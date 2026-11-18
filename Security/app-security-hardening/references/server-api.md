# API-level fixes

## Races on money and counters

Bad: `SELECT balance` → check in JS → `UPDATE balance = newValue`. Two parallel requests both pass.
Good, one atomic statement:
```sql
UPDATE wallets SET balance = balance - $1
WHERE user_id = $2 AND balance >= $1
RETURNING balance;   -- no row returned → insufficient funds
```
Or `SELECT … FOR UPDATE` inside a transaction. Same for one-time coupons/referrals: a unique
constraint (`UNIQUE(user_id, code)`) + `INSERT … ON CONFLICT DO NOTHING RETURNING`.
Test: fire 10 parallel requests with `Promise.all` and assert exactly one succeeds.

## Errors that don't sign people out

Clients usually sign out on any 401. Use 401 only for "your session is invalid". A wrong password
inside a signed-in action (disable 2FA, change email) → 400 `wrong_password`.

## Security events + admin page

Table `security_events(id, user_id null, event text, platform, ip, created_at)`.
- Sources: RASP alerts from the app, failed integrity checks, (optionally) lockouts.
- Validate `event` against a known list (enum) — never store free text from clients.
- Admin endpoint: newest 100, filter all/serious, 7-day counts per event. Admin page shows a
  label per event in the user's language, a "serious" badge, the user and IP.
- `trustProxy` must be set correctly behind nginx (`TRUST_PROXY=1` hop) or every IP is 127.0.0.1;
  set it to the number of hops, never `true` (spoofable X-Forwarded-For).

## Rate limits

Per route: auth 5–10/min, security events 10/min, integrity 20/min, everything else a sane global.

## Secrets

- Sign/encrypt with keys derived from one server secret (HKDF with a label per use), so rotating
  is one place.
- Push credentials: Firebase service account via a file path env var
  (`FIREBASE_SERVICE_ACCOUNT_FILE`), file mode 600. iOS push via FCM needs an APNs **.p8 key valid
  for Sandbox & Production**, uploaded in Firebase → Cloud Messaging (`BadEnvironmentKeyInToken`
  / `Invalid APNs credential` = wrong key environment).
