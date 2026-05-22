# Auth

**Strategy: stateless JWT with short-lived access tokens + long-lived refresh tokens, both stored server-side as revocable records.**

Flow:
1. Android obtains a Google ID token via Google Sign-In SDK.
2. Client sends `POST /auth/google { idToken }`.
3. Server verifies the token using the Firebase Admin SDK (or `google-auth-library`) — this validates signature against Google's public keys and confirms `aud` matches the app's client ID.
4. Server upserts a `User` record keyed on `googleSub` from the token claims.
5. Server issues a JWT access token (15-minute expiry) and a refresh token (UUID stored in a `refresh_tokens` table, 30-day expiry).
6. Client uses access token for all API calls. On 401, client exchanges refresh token for a new access token via `POST /auth/refresh`.

**Why not fully stateless (refresh token in JWT):** For a tiny private app the ops cost of a `refresh_tokens` table is near zero, and it buys immediate revocation (useful if a test device is lost). Fully stateless refresh tokens cannot be revoked without a denylist — same complexity, less control.

**Why not sessions/cookies:** The client is Android, not a browser. Bearer tokens are the natural fit and avoid cookie/CSRF complexity.

**Why the refresh token is returned in the response body (not an HttpOnly cookie):** The HttpOnly-cookie rule is a browser/XSS mitigation that has no force on a native client. See [`ADR-0001`](../../decisions/0001-bearer-tokens-not-httponly-cookies.md) for the full rationale and the revisit trigger (a future browser client).
