**Decision:** Going with Option A -- the unified notification service with channel plugins and a preferences table. It is the only option that delivers every feature in the original request (all three channels plus per-channel user preferences) while also providing a clean extension path for future channels without rework.

---

## Implementation Plan

### 1. Database Schema

**notifications table**

| Column | Type | Description |
|---|---|---|
| id | UUID / PK | Unique notification ID |
| user_id | FK -> users | Recipient |
| channel | ENUM('email','push','in_app') | Delivery channel |
| title | VARCHAR(255) | Short headline |
| body | TEXT | Full message content |
| payload | JSONB | Channel-specific data (e.g. email subject, deep-link URL) |
| status | ENUM('pending','sent','delivered','failed') | Delivery state |
| created_at | TIMESTAMP | When the notification was created |
| sent_at | TIMESTAMP | When delivery was attempted |

**notification_preferences table**

| Column | Type | Description |
|---|---|---|
| id | UUID / PK | Row ID |
| user_id | FK -> users | Owner of the preference |
| channel | ENUM('email','push','in_app') | Which channel this row controls |
| enabled | BOOLEAN DEFAULT true | Whether the user wants this channel |
| updated_at | TIMESTAMP | Last change |

Unique constraint on `(user_id, channel)` so each user has exactly one row per channel.

### 2. Channel Plugin Interface

```python
from abc import ABC, abstractmethod
from dataclasses import dataclass

@dataclass
class NotificationPayload:
    user_id: str
    title: str
    body: str
    metadata: dict  # channel-specific extras

class NotificationChannel(ABC):
    """Base class every channel plugin must implement."""

    @abstractmethod
    def send(self, payload: NotificationPayload) -> bool:
        """Attempt delivery. Return True on success."""
        ...

    @abstractmethod
    def channel_name(self) -> str:
        """Return the canonical channel key, e.g. 'email'."""
        ...
```

Three concrete implementations:

- `EmailChannel` -- wraps an SMTP/SES client.
- `PushChannel` -- wraps FCM / APNs.
- `InAppChannel` -- writes a row to an `in_app_notifications` table (or the same `notifications` table filtered by channel) and emits a WebSocket event.

### 3. Notification Service (Router)

```python
class NotificationService:
    def __init__(self, channels: list[NotificationChannel], preference_repo):
        self._channels = {ch.channel_name(): ch for ch in channels}
        self._preferences = preference_repo

    def notify(self, user_id: str, title: str, body: str, metadata: dict | None = None):
        """Send a notification to every channel the user has enabled."""
        prefs = self._preferences.get_for_user(user_id)
        payload = NotificationPayload(
            user_id=user_id,
            title=title,
            body=body,
            metadata=metadata or {},
        )
        for channel_name, channel in self._channels.items():
            if prefs.get(channel_name, True):  # default to enabled
                channel.send(payload)
```

The service iterates over registered channels, checks the user's preference for each, and dispatches accordingly. Adding a fourth channel later means writing one new class and registering it -- no routing logic changes.

### 4. Preferences API

| Method | Endpoint | Purpose |
|---|---|---|
| GET | `/api/notifications/preferences` | Return current user's preferences for all channels |
| PATCH | `/api/notifications/preferences` | Update one or more channel preferences (`{ "email": false, "push": true }`) |

Preferences default to all-enabled on first access (no row = enabled).

### 5. Delivery Sequence

1. An event in the system triggers `notification_service.notify(user_id, ...)`.
2. The service loads the user's channel preferences.
3. For each channel where `enabled = true`, the corresponding plugin's `send()` method is invoked.
4. A row is written to the `notifications` table with the delivery status.
5. Failures are logged; a retry mechanism (e.g. a background job) can re-attempt failed sends.

This gives us all three channels, per-channel user preferences, and a plugin architecture that stays out of the way until a new channel actually needs to be added.
