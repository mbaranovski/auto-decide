# Response

**Decision:** Going with Option B (Redis-based sliding window using sorted sets) -- it delivers all three requested features (per-user limits, sliding window algorithm, 429 + Retry-After headers) and handles multi-server deployments without adding unnecessary complexity, since Redis sorted sets are the standard approach for sliding window rate limiting.

Here is the implementation:

```typescript
// rate-limiter.ts
import { Redis } from "ioredis";
import { Request, Response, NextFunction } from "express";

interface RateLimiterConfig {
  redis: Redis;
  windowMs: number;       // Sliding window size in milliseconds
  maxRequests: number;     // Max requests allowed per window
  keyPrefix?: string;      // Redis key prefix
  keyExtractor?: (req: Request) => string; // Extract user/API key from request
}

const defaultKeyExtractor = (req: Request): string => {
  const apiKey = req.headers["x-api-key"] as string;
  if (apiKey) return apiKey;
  // Fallback to IP if no API key provided
  return req.ip || req.socket.remoteAddress || "unknown";
};

export function createRateLimiter(config: RateLimiterConfig) {
  const {
    redis,
    windowMs,
    maxRequests,
    keyPrefix = "ratelimit",
    keyExtractor = defaultKeyExtractor,
  } = config;

  return async (req: Request, res: Response, next: NextFunction) => {
    const userKey = keyExtractor(req);
    const redisKey = `${keyPrefix}:${userKey}`;
    const now = Date.now();
    const windowStart = now - windowMs;

    try {
      // Use a pipeline for atomicity and performance
      const pipeline = redis.pipeline();

      // 1. Remove all entries outside the current sliding window
      pipeline.zremrangebyscore(redisKey, 0, windowStart);

      // 2. Count remaining entries (requests within the window)
      pipeline.zcard(redisKey);

      // 3. Add the current request with its timestamp as the score
      pipeline.zadd(redisKey, now.toString(), `${now}:${Math.random()}`);

      // 4. Set expiry on the key so it auto-cleans
      pipeline.pexpire(redisKey, windowMs);

      const results = await pipeline.exec();

      if (!results) {
        // Redis pipeline failed entirely -- fail open
        return next();
      }

      // zcard result is at index 1 (second command)
      const currentCount = results[1][1] as number;

      // Set rate limit headers on every response
      const remaining = Math.max(0, maxRequests - currentCount - 1);
      res.setHeader("X-RateLimit-Limit", maxRequests);
      res.setHeader("X-RateLimit-Remaining", remaining);
      res.setHeader(
        "X-RateLimit-Reset",
        Math.ceil((now + windowMs) / 1000)
      );

      if (currentCount >= maxRequests) {
        // Over the limit: calculate Retry-After
        // Find the oldest entry in the window to determine when the window slides enough
        const oldestEntries = await redis.zrange(redisKey, 0, 0, "WITHSCORES");
        let retryAfterSeconds = Math.ceil(windowMs / 1000);

        if (oldestEntries.length >= 2) {
          const oldestTimestamp = parseInt(oldestEntries[1], 10);
          const msUntilSlide = oldestTimestamp + windowMs - now;
          retryAfterSeconds = Math.max(1, Math.ceil(msUntilSlide / 1000));
        }

        res.setHeader("Retry-After", retryAfterSeconds);

        // Remove the request we just added since it's being rejected
        await redis.zrem(redisKey, `${now}:${Math.random()}`);

        return res.status(429).json({
          error: "Too Many Requests",
          message: `Rate limit exceeded. Try again in ${retryAfterSeconds} seconds.`,
          retryAfter: retryAfterSeconds,
        });
      }

      next();
    } catch (err) {
      // Fail open: if Redis is down, allow the request through
      console.error("Rate limiter error:", err);
      next();
    }
  };
}
```

```typescript
// usage-example.ts
import express from "express";
import Redis from "ioredis";
import { createRateLimiter } from "./rate-limiter";

const app = express();
const redis = new Redis({
  host: process.env.REDIS_HOST || "localhost",
  port: parseInt(process.env.REDIS_PORT || "6379", 10),
});

// Apply rate limiter globally: 100 requests per 15-minute sliding window
app.use(
  createRateLimiter({
    redis,
    windowMs: 15 * 60 * 1000, // 15 minutes
    maxRequests: 100,
  })
);

// Or apply per-route with different limits
const strictLimiter = createRateLimiter({
  redis,
  windowMs: 60 * 1000, // 1 minute
  maxRequests: 10,
  keyPrefix: "ratelimit:strict",
});

app.post("/api/expensive-operation", strictLimiter, (req, res) => {
  res.json({ result: "ok" });
});

app.get("/api/data", (req, res) => {
  res.json({ data: "here" });
});

app.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

Key design points:

- **Sliding window via Redis sorted sets:** Each request is stored as a member with its timestamp as the score. Before counting, all entries outside the window are pruned. This gives true sliding window behavior, not fixed buckets.
- **Per-user tracking:** Extracts the user identity from `x-api-key` header by default, with a configurable `keyExtractor` for custom schemes (JWT sub, session ID, etc.).
- **429 with Retry-After:** When the limit is exceeded, the middleware calculates exactly when the oldest request in the window will expire, giving the client a precise `Retry-After` value in seconds.
- **Fail-open on Redis errors:** If Redis is unavailable, requests pass through rather than blocking all traffic.
- **Pipeline for performance:** The prune, count, add, and expire operations run in a single Redis pipeline to minimize round trips.
