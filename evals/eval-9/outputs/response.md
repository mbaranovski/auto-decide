**Decision:** Going with Option B — it is the only option that delivers every feature the user explicitly requested without substitution or deferral. The user specifically asked for refresh tokens, not long-lived JWTs (ruling out A), and asked for all features now, not in phases (ruling out C).

Here is the implementation plan for the complete authentication system:

## Architecture Overview

The system will have the following components:

### 1. Email/Password Authentication
- Registration endpoint with email validation and password hashing (bcrypt)
- Login endpoint that validates credentials and issues token pairs
- Passwords stored as salted bcrypt hashes, never in plaintext

### 2. OAuth Integration (Google and GitHub)
- OAuth 2.0 authorization code flow for both providers
- Google OAuth: redirect to Google's consent screen, exchange authorization code for user profile, link or create account
- GitHub OAuth: redirect to GitHub's authorization endpoint, exchange code for access token, fetch user profile from GitHub API, link or create account
- Account linking: if an OAuth login matches an existing email/password account, link them rather than creating a duplicate

### 3. Password Reset via Email
- "Forgot password" endpoint generates a time-limited reset token (e.g., 1-hour expiry)
- Token is stored hashed in the database (not sent in plaintext to the DB)
- Reset email contains a link with the token
- Reset endpoint validates the token, accepts a new password, invalidates the token after use
- Rate limiting on reset requests to prevent abuse

### 4. Session Management with Refresh Token Rotation
- On successful authentication (any method), issue a pair:
  - **Access token**: short-lived JWT (15 minutes), used for API authorization
  - **Refresh token**: longer-lived opaque token (7 days), stored hashed in the database
- Refresh endpoint accepts a valid refresh token, issues a new access/refresh pair, and invalidates the old refresh token (rotation)
- Refresh token reuse detection: if a previously rotated token is presented, invalidate the entire token family (signals potential theft)
- Logout endpoint invalidates all refresh tokens for the session

### Data Model

```
users
  id (UUID, PK)
  email (unique, indexed)
  password_hash (nullable — null for OAuth-only users)
  email_verified (boolean)
  created_at, updated_at

oauth_accounts
  id (UUID, PK)
  user_id (FK -> users)
  provider (enum: google, github)
  provider_user_id (string)
  unique constraint on (provider, provider_user_id)

password_reset_tokens
  id (UUID, PK)
  user_id (FK -> users)
  token_hash (string)
  expires_at (timestamp)
  used (boolean)

refresh_tokens
  id (UUID, PK)
  user_id (FK -> users)
  token_hash (string)
  family_id (UUID — for reuse detection)
  expires_at (timestamp)
  revoked (boolean)
  created_at
```

### API Endpoints

```
POST   /auth/register          — email/password registration
POST   /auth/login             — email/password login, returns access + refresh tokens
POST   /auth/refresh           — exchange refresh token for new token pair
POST   /auth/logout            — revoke refresh token(s)

GET    /auth/google            — redirect to Google OAuth consent
GET    /auth/google/callback   — handle Google OAuth callback
GET    /auth/github            — redirect to GitHub OAuth authorization
GET    /auth/github/callback   — handle GitHub OAuth callback

POST   /auth/forgot-password   — request password reset email
POST   /auth/reset-password    — submit new password with reset token
```

This delivers every feature in the original request: email/password login, Google OAuth, GitHub OAuth, email-based password reset, and session management with short-lived access tokens and refresh token rotation.
