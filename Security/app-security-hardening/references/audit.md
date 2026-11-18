# Audit checklist

Read-only. Produce a table: layer → present / missing / weak → what an attacker could do.

## Code
- **Money and counters:** wallet debits, coupon/referral redemption, stock. Is "check balance then
  subtract" done in one atomic statement (`UPDATE … SET balance = balance - $1 WHERE balance >= $1
  RETURNING`) or a row lock? Two parallel requests must not both succeed.
- **Login:** rate limit per account and per IP; lockout after N failures; same response and same
  time for "no such email" and "wrong password" (always run the password hash).
- **Tokens:** lifetime? Refresh flow? Can a stolen token be used from another device? Is there a
  way to sign out everywhere (token version / session table)?
- **Authorization:** every route checks the *owner* of the object, not only "signed in". Look for
  IDs taken from the request body or URL without an ownership check.
- **Input:** schema validation (zod/joi) on every body, size limits, upload type checks.
- **Admin:** separate role, 2FA, not reachable from the open internet if possible.
- **Secrets in the repo:** `git log -p | grep -iE "secret|api_key|password|BEGIN .*PRIVATE"`,
  committed `.env`, keystores, `.p8`, `.p12`, service-account JSON. Check `.gitignore`.
- **Dependencies:** `npm audit --omit=dev`, `flutter pub outdated`, framework advisories.

## App
- TLS pinning? Android `network_security_config` (no cleartext, no user CAs)?
- RASP / anti-tamper? Root/jailbreak, Frida, debugger, emulator, repackaging detection.
- Screenshots of sensitive screens? App-switcher preview?
- Tokens stored in Keychain / Keystore (flutter_secure_storage), not SharedPreferences?
- Release build obfuscated (`--obfuscate --split-debug-info`)?

## Server (if you have SSH)
- `sshd -T | grep -E "passwordauthentication|permitrootlogin"`; check `/etc/ssh/sshd_config.d/*`
  too (cloud-init files often re-enable passwords; the first value read wins).
- `journalctl -u ssh --since "24 hours ago" | grep -c "Failed password"` (brute force volume).
- `ufw status` / `iptables -S`: only 22, 80, 443 should be open. `ss -tlnp`: services bound to
  0.0.0.0 that should be 127.0.0.1.
- Which user runs the app (`ps -o user= -p <pid>`)? Root = full server on any RCE.
- `.env` permissions (should be 600, owned by the app user).
- `certbot renew --dry-run`; `systemctl list-timers | grep certbot`.
- Backups: where, how often, are any off the server?
- Error logs: repeated 401/500s, push credential errors, crash loops (PM2 restart count).
