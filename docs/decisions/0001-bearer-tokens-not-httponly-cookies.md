# ADR-0001: Refresh token in the response body, not an HttpOnly cookie

- **Status:** Accepted
- **Date:** 2026-05-22

## Context and Problem Statement

`POST /auth/google` returns `{ accessToken, refreshToken, user }` and the client later exchanges the refresh token via `POST /auth/refresh { refreshToken }` (see [`backend/architecture/api.md`](../backend/architecture/api.md), [`backend/architecture/auth.md`](../backend/architecture/auth.md)).

A common security guideline says a refresh token must never be returned in the response body — it should be set as an `HttpOnly; Secure` cookie so client-side JavaScript can't read it. The question raised: does returning the refresh token in the JSON body constitute a vulnerability here, and should we move it into an HttpOnly cookie?

The constraint that drives the answer: **Liftly's only client is a native Android app** (no web/browser client — see the stack docs). The "HttpOnly cookie for refresh tokens" rule is a *browser* mitigation. Its sole purpose is to keep the token out of reach of JavaScript executing in a DOM, i.e., to limit the blast radius of an XSS bug. A native Android client has no DOM and no untrusted JavaScript execution against its token store, so the threat the mitigation addresses does not exist in this context.

## Considered Options

1. **Refresh token in the response body; client stores it in OS-backed secure storage.** Server returns `{ accessToken, refreshToken, user }`; Android keeps it in a Tink-encrypted Proto DataStore wrapped by an Android Keystore key (see [`android/architecture/auth.md`](../android/architecture/auth.md)).
2. **Refresh token as an `HttpOnly; Secure` cookie via `Set-Cookie`; body returns only `accessToken`.** Android adopts an OkHttp `CookieJar` with persistence, and the API gains CSRF defenses.

## Decision

**Option 1: keep the refresh token in the response body.**

For a native Android client this is not a vulnerability. The token at rest is protected by Keystore-backed encryption, which is the native-platform equivalent of what HttpOnly provides on the web. An attacker who could read the token from the JSON body in memory could equally read it from a cookie store on the same device — the cookie changes nothing about the native threat model.

This confirms and extends the existing "Why not sessions/cookies" note in [`backend/architecture/auth.md`](../backend/architecture/auth.md): bearer tokens are the natural fit for a non-browser client and avoid cookie/CSRF complexity.

## Consequences

**Positive**
- No change to the documented design; the OkHttp `Authenticator` refresh flow (`POST /auth/refresh { refreshToken }`) and Keystore-backed DataStore storage stay as-is.
- No cookie jar, no cookie persistence across app restarts, and no CSRF surface or its mitigations to maintain.
- The token's protection is anchored in OS secure hardware (TEE) rather than a transport-layer flag aimed at a threat the client doesn't have.

**Negative**
- The choice is correct *only while the client stays native*. The moment a browser client exists, refresh-token-in-body becomes a real XSS exposure and this ADR must be revisited (see Revisit Trigger).
- Anyone applying generic web-security checklists to this API will flag the body-returned refresh token; this ADR is the standing answer to that flag.

## Revisit Trigger

Revisit when **any** of these hold:

1. A **browser/web client** (or any client executing untrusted JavaScript against the token store) is added or seriously planned. At that point the refresh token should move to an `HttpOnly; Secure` cookie for that client, with CSRF protection, and the API likely needs per-client token handling.
2. The refresh-token transport or storage model changes for another reason (e.g., switching to BFF/session-based auth).

## Pros and Cons of the Options

### Option 1: Refresh token in body + OS-backed secure storage
- **Good:** matches the native threat model; no CSRF/cookie machinery; protection rooted in Keystore/TEE; zero change to the current design.
- **Bad:** not portable to a future browser client; trips generic web-security checklists.

### Option 2: HttpOnly cookie, body returns only the access token
- **Good:** the correct pattern *if* a browser client existed; keeps the refresh token out of JS reach in a DOM.
- **Bad:** solves a threat (XSS/DOM JS access) that a native Android client doesn't have; adds an OkHttp `CookieJar`, cookie persistence, and a CSRF surface plus its defenses — the exact complexity the bearer-token decision avoided.
