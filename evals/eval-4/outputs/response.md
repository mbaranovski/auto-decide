**Decision:** Going with Option B (S3 + pre-signed URLs + third-party/native preview) -- it delivers all four features while respecting the S3 constraint, and it leverages S3's built-in pre-signed URL expiration rather than reinventing it with cron jobs and custom link shorteners.

Here's the implementation plan:

## Architecture

```
[Client Browser]
    |
    v
[API Server (Node/Express or similar)]
    |
    +-- POST /upload         --> streams file to S3, stores metadata in DB
    +-- POST /share          --> generates pre-signed URL with TTL, returns short link
    +-- GET  /share/:id      --> looks up link, checks expiration, redirects or serves preview
    +-- GET  /preview/:id    --> renders preview page with embedded file viewer
    |
    v
[S3 Bucket]                  [Database (metadata, links, expiration)]
```

## Implementation

### 1. File Upload

```typescript
// POST /api/upload
import { S3Client, PutObjectCommand } from "@aws-sdk/client-s3";

const s3 = new S3Client({ region: process.env.AWS_REGION });

app.post("/api/upload", upload.single("file"), async (req, res) => {
  const fileKey = `uploads/${uuidv4()}-${req.file.originalname}`;

  await s3.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET,
    Key: fileKey,
    Body: req.file.buffer,
    ContentType: req.file.mimetype,
  }));

  // Store metadata in DB
  const fileRecord = await db.files.create({
    key: fileKey,
    originalName: req.file.originalname,
    mimeType: req.file.mimetype,
    size: req.file.size,
    uploadedBy: req.user.id,
    uploadedAt: new Date(),
  });

  res.json({ fileId: fileRecord.id, name: fileRecord.originalName });
});
```

### 2. Share via Link with Expiration

```typescript
// POST /api/share
import { GetObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";

app.post("/api/share", async (req, res) => {
  const { fileId, expiresInHours } = req.body;

  const file = await db.files.findById(fileId);
  if (!file) return res.status(404).json({ error: "File not found" });

  const expiresInSeconds = expiresInHours * 3600;
  const expiresAt = new Date(Date.now() + expiresInSeconds * 1000);

  // Generate pre-signed URL with built-in expiration
  const presignedUrl = await getSignedUrl(
    s3,
    new GetObjectCommand({
      Bucket: process.env.S3_BUCKET,
      Key: file.key,
    }),
    { expiresIn: expiresInSeconds }
  );

  // Store share link metadata in DB for tracking and preview routing
  const shareLink = await db.shareLinks.create({
    fileId: file.id,
    presignedUrl,
    expiresAt,
    shortCode: generateShortCode(), // e.g., nanoid(10)
  });

  res.json({
    shareUrl: `${process.env.BASE_URL}/s/${shareLink.shortCode}`,
    expiresAt: shareLink.expiresAt,
  });
});
```

### 3. Link Resolution and Expiration Check

```typescript
// GET /s/:shortCode
app.get("/s/:shortCode", async (req, res) => {
  const link = await db.shareLinks.findByShortCode(req.params.shortCode);

  if (!link) return res.status(404).render("not-found");

  if (new Date() > link.expiresAt) {
    return res.status(410).render("link-expired");
  }

  // Redirect to preview page (not raw download)
  res.redirect(`/preview/${link.shortCode}`);
});
```

### 4. File Preview in Browser

```typescript
// GET /preview/:shortCode
app.get("/preview/:shortCode", async (req, res) => {
  const link = await db.shareLinks.findByShortCode(req.params.shortCode);
  if (!link || new Date() > link.expiresAt) {
    return res.status(410).render("link-expired");
  }

  const file = await db.files.findById(link.fileId);

  // Determine preview strategy based on MIME type
  const previewStrategy = getPreviewStrategy(file.mimeType);

  res.render("preview", {
    fileName: file.originalName,
    fileSize: file.size,
    mimeType: file.mimeType,
    previewStrategy,
    presignedUrl: link.presignedUrl,
    downloadUrl: link.presignedUrl,
    shortCode: link.shortCode,
  });
});

function getPreviewStrategy(mimeType: string): PreviewStrategy {
  // Images: native browser <img> tag
  if (mimeType.startsWith("image/")) {
    return { type: "image" };
  }
  // PDFs: native browser <iframe> or <embed>
  if (mimeType === "application/pdf") {
    return { type: "pdf" };
  }
  // Video: native browser <video> tag
  if (mimeType.startsWith("video/")) {
    return { type: "video" };
  }
  // Audio: native browser <audio> tag
  if (mimeType.startsWith("audio/")) {
    return { type: "audio" };
  }
  // Text/code: fetch content and render in <pre> or with syntax highlighting
  if (mimeType.startsWith("text/") || mimeType === "application/json") {
    return { type: "text" };
  }
  // Office documents: Google Docs Viewer
  if ([
    "application/msword",
    "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
    "application/vnd.ms-excel",
    "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
    "application/vnd.ms-powerpoint",
    "application/vnd.openxmlformats-officedocument.presentationml.presentation",
  ].includes(mimeType)) {
    return { type: "google-docs-viewer" };
  }
  // Fallback: download only
  return { type: "download-only" };
}
```

### 5. Preview Template (HTML)

```html
<!-- views/preview.ejs -->
<div class="preview-container">
  <header>
    <h1>{{ fileName }}</h1>
    <span class="file-meta">{{ fileSize | formatBytes }} &middot; {{ mimeType }}</span>
    <a href="{{ downloadUrl }}" class="btn-download">Download</a>
  </header>

  <div class="preview-content">
    {% if previewStrategy.type == "image" %}
      <img src="{{ presignedUrl }}" alt="{{ fileName }}" />

    {% elif previewStrategy.type == "pdf" %}
      <iframe src="{{ presignedUrl }}" width="100%" height="800px"></iframe>

    {% elif previewStrategy.type == "video" %}
      <video controls src="{{ presignedUrl }}"></video>

    {% elif previewStrategy.type == "audio" %}
      <audio controls src="{{ presignedUrl }}"></audio>

    {% elif previewStrategy.type == "text" %}
      <pre id="text-preview"></pre>
      <script>
        fetch("{{ presignedUrl }}")
          .then(r => r.text())
          .then(t => document.getElementById("text-preview").textContent = t);
      </script>

    {% elif previewStrategy.type == "google-docs-viewer" %}
      <iframe src="https://docs.google.com/gview?url={{ presignedUrl | urlencode }}&embedded=true"
              width="100%" height="800px"></iframe>

    {% else %}
      <p>Preview not available for this file type. <a href="{{ downloadUrl }}">Download instead.</a></p>
    {% endif %}
  </div>
</div>
```

### 6. Database Schema

```sql
CREATE TABLE files (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  key         TEXT NOT NULL,         -- S3 object key
  original_name TEXT NOT NULL,
  mime_type   TEXT NOT NULL,
  size        BIGINT NOT NULL,
  uploaded_by UUID REFERENCES users(id),
  uploaded_at TIMESTAMPTZ DEFAULT now()
);

CREATE TABLE share_links (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  file_id       UUID REFERENCES files(id) ON DELETE CASCADE,
  short_code    TEXT UNIQUE NOT NULL,
  presigned_url TEXT NOT NULL,
  expires_at    TIMESTAMPTZ NOT NULL,
  created_at    TIMESTAMPTZ DEFAULT now()
);

CREATE INDEX idx_share_links_short_code ON share_links(short_code);
CREATE INDEX idx_share_links_expires_at ON share_links(expires_at);
```

## Key Design Notes

- **Expiration is double-enforced**: the pre-signed URL itself expires (S3-level), and the application checks `expires_at` before serving the preview page. This means even if someone has the raw pre-signed URL cached, S3 will reject it after TTL.
- **Preview is pragmatic**: images, PDFs, video, and audio use native browser capabilities (zero dependencies). Office documents use Google Docs Viewer. Text files are fetched and rendered. No custom renderer code to maintain.
- **Pre-signed URL TTL note**: S3 pre-signed URLs have a max TTL of 7 days when signed with temporary credentials (STS). For longer expiration windows, the share link record should regenerate the pre-signed URL on access (check if the stored one is expired, generate a fresh one if the share link itself is still valid).
