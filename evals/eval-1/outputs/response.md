**Decision:** Going with Option A -- WebSockets for all real-time features over a single persistent connection. The user explicitly asked for real-time typing indicators and read receipts alongside messaging; deferring them (Option C) drops two of three requested features, and polling every 2 seconds (Option B) is not genuinely real-time for something like a typing indicator that needs sub-second feedback.

Here's the implementation plan:

**WebSocket message protocol:**

All communication flows over a single WebSocket connection per client. Messages are JSON with a `type` discriminator:

```json
// Chat message
{ "type": "message", "chatId": "abc", "senderId": "user1", "content": "Hello", "id": "msg_001", "timestamp": 1708600000 }

// Typing indicator
{ "type": "typing", "chatId": "abc", "userId": "user1", "isTyping": true }

// Read receipt
{ "type": "read_receipt", "chatId": "abc", "userId": "user2", "lastReadMessageId": "msg_001" }
```

**Server architecture:**

- A WebSocket server (e.g., `ws` in Node.js, or native support in frameworks like Fastify/Hono) that authenticates on connection upgrade and maps each socket to a user ID.
- A room/channel registry that tracks which users are in which chats, so broadcasts go only to participants.
- Message persistence to a database on `type: "message"`. Typing indicators are ephemeral -- broadcast only, never stored. Read receipts are persisted (update a `last_read_message_id` per user per chat).

**Client implementation:**

- A `WebSocketManager` class that handles:
  - Connection and reconnection with exponential backoff (start at 1s, cap at 30s)
  - A message router that dispatches incoming frames to registered handlers by `type`
  - A send queue that buffers messages during reconnection
- Typing indicator logic: emit `isTyping: true` on the first keystroke, then throttle to one event per 3 seconds while the user continues typing. Emit `isTyping: false` after 2 seconds of inactivity. On the receiving side, auto-clear the indicator after 4 seconds with no refresh (guards against missed stop events).
- Read receipt logic: when the chat view is focused and a new message scrolls into the viewport, send a read receipt with the latest visible message ID. Debounce to avoid flooding during fast scrolling.

**Message flow:**

```
User A types in chat "abc":
  Client A -> Server: { type: "typing", chatId: "abc", userId: "A", isTyping: true }
  Server -> Client B: (broadcast same typing event)
  Client B renders "A is typing..."

User A sends a message:
  Client A -> Server: { type: "message", chatId: "abc", senderId: "A", content: "Hey!", id: "msg_042" }
  Server persists message, assigns timestamp
  Server -> Client B: (broadcast the message)
  Client A -> Server: { type: "typing", chatId: "abc", userId: "A", isTyping: false }

User B reads the message:
  Client B -> Server: { type: "read_receipt", chatId: "abc", userId: "B", lastReadMessageId: "msg_042" }
  Server persists receipt, notifies Client A
  Client A renders "Read" under msg_042
```

I'll start with the WebSocket server and connection management module, then build out handlers for each message type, then the client-side manager.
