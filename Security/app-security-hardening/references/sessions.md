# Sessions, device binding, two-factor

## Short access + rotating refresh tokens

- Access token: JWT, 15 minutes, carries `sid` (session id) and the user's `tokenVersion`.
- Refresh token: 32 random bytes (base64url), 30 days, stored **hashed** (sha256) in a
  `auth_sessions` table: `id, user_id, family_id, refresh_hash, token_version, device_key,
  user_agent, expires_at, used_at, revoked_at`.
- `POST /auth/refresh`: atomically claim the row
  (`UPDATE … SET revoked_at=now(), used_at=now() WHERE id=$1 AND revoked_at IS NULL RETURNING`),
  then create the next session in the **same family**.
- **Reuse detection:** a refresh token presented again after it was used = stolen → revoke the
  whole family. Allow a ~20 s grace window that returns `409 refresh_race` instead, because an app
  firing two requests that both got 401 will refresh twice.
- `authenticate` checks the session row is live (not revoked, not expired, tokenVersion matches).
  Bumping `users.token_version` signs the person out everywhere (password change, "sign out all").
- **Roll out safely:** the app opts in with a header (e.g. `x-app-session: refresh`) on sign-in.
  Apps without the header keep the old long token until everyone has updated.
- App side: one interceptor; on 401 call refresh once (shared future so parallel requests wait on
  the same refresh), retry the request once, sign out only if refresh fails.

## Device-bound sessions (request signing)

Goal: a copied token is useless on any other device.

- The app creates a **P-256 key in the Secure Enclave (iOS) / Android Keystore (StrongBox if
  available)**, non-exportable. Native code via a method channel: `publicKey`, `sign(bytes)`.
- On sign-in it sends the public key (uncompressed point, 65 bytes, base64) in a header; the
  server stores it on the session.
- Every request of that session carries `x-ts` (ms) and `x-sig` (base64 DER ECDSA-SHA256) over:
  ```
  `${ts}\n${METHOD}\n${path?query}\n${sha256 hex of the exact body bytes}`
  ```
- Server: needs the **raw body** (add a content-type parser that keeps `req.rawBody`), rejects if
  `|now - ts| > 2 min`, verifies with `crypto.verify('sha256', msg, {key, dsaEncoding:'der'}, sig)`.
- App: serialize the JSON body to a string **before** signing and send that exact string, or the
  bytes won't match.
- Phones that can't make a key simply get unbound sessions (never block them).
- Test: sign with a test key in the test suite; assert a request with a wrong/missing signature on a
  bound session is 401, and that an unbound session still works.

## Two-factor (TOTP, RFC 6238)

- `POST /me/2fa/setup` → random 20-byte secret (base32) + `otpauth://totp/App:email?secret=…&issuer=App`
  (show as QR + copyable text). Store the secret **encrypted** (AES-256-GCM, key derived from a
  server secret with HKDF/scrypt), not yet enabled.
- `POST /me/2fa/enable {code}` → verify, enable, return **8 recovery codes** once (store hashes).
- Login with 2FA on: password OK → return `{twoFactorRequired:true, challenge}` where challenge is a
  short-lived JWT with `purpose:'2fa'`. **`authenticate` must reject purpose tokens.**
  `POST /auth/2fa {challenge, code}` → real tokens.
- Accept ±1 time step; **store the last used step** so a code can't be replayed.
- Recovery code: single use, case-insensitive.
- Disable needs password **and** a code. Return 400 (not 401) for a wrong password here, or admin
  panels that sign out on 401 will log the user out.
- Test against the RFC vector: secret `12345678901234567890`, t=59 s → `94287082` (last 6: `287082`).

## New sign-in alerts

After a *complete* sign-in (after 2FA, not at the password step) send a push + in-app notification
"New sign-in on <device>". Do it in the background so sign-in doesn't wait; tests must poll for it.

## Login hardening

- Rate limit `/auth/*` per IP and per account; lock for 15 min after ~10 failures.
- Unknown email: still run the password hash against a dummy hash (equal timing), same error text.
- Passwords: scrypt/argon2/bcrypt with per-user salt.
