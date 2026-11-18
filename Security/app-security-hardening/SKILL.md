---
name: app-security-hardening
description: Audit and harden a mobile app with its backend, admin panel and server, layer by layer - device-bound sessions, short tokens with refresh rotation, two-factor, certificate pinning, anti-tamper / RASP, Play Integrity and App Attest, screenshot blocking, server lockdown (SSH keys only, firewall, non-root), CI secret and dependency scanning, encrypted key backups. Use when someone asks to secure, harden, pentest-proof or security-audit an app, add a specific protection from this list, or check which protections an app is missing. Built for Flutter + Node (Fastify/Express) + Next.js + a Linux VPS, with notes for other stacks.
---

# App security hardening

A tested playbook from hardening a production fitness app (Flutter app, Fastify + Postgres API,
Next.js admin panel, Ubuntu VPS with nginx + PM2). Each layer below was built, tested and shipped
without locking out a single real user. Follow the same order and the same safety rules.

## How to work

1. **Audit first** (`references/audit.md`). Read the code and, if you have access, the server.
   Write a short table: layer → present / missing / weak → risk. Show it to the user before
   changing anything.
2. **Agree the plan.** Pick layers in priority order (below). Explain each in one plain sentence;
   many users are not security people.
3. **One layer at a time:** implement → test (automated tests where the codebase has them) →
   verify on a real device or the real server → ship → next layer.
4. **Report** in plain words: what is now protected, what is not yet, what the user must do
   themselves (Play Console, Apple, password manager).

## Safety rules (never skip)

- **Never lock out real users.** Anything that can refuse a legitimate person (integrity checks,
  RASP blocking, request signing, pinning) ships in **monitor / opt-in mode first**: old app
  versions keep working, failures are logged, and you switch to enforce only after real data shows
  honest phones pass.
- **Never lock yourself out of a server.** Before turning off SSH passwords: add a second backup
  key, test a NEW connection with each key while the current session stays open, keep a backup of
  the old config. Same for firewall changes: confirm SSH stays allowed first.
- **Ship the app before the server depends on it.** Pinning a new certificate authority, a new
  required header, a new token format: the app version that understands it must be out first.
- **Secrets never go into git, chat, URLs or logs.** Generate them on the machine that uses them.
  If one leaks, rotate it.
- **Ask before outward or irreversible actions:** deploys, merges, deleting data, store consoles.
- **Monitor ≠ working.** In monitor mode, make sure the check actually *runs* and its results are
  visible somewhere (an admin "Security" page). A check that never runs proves nothing.

## The layers, in priority order

| # | Layer | Guide |
|---|-------|-------|
| 1 | Business-logic races (double spend, double redeem) and auth basics: lockout, constant-time login, no user enumeration | `references/server-api.md` |
| 2 | Vulnerable dependencies and framework RCEs (`npm audit`, Next.js advisories) | `references/ci-cd.md` |
| 3 | Short access tokens (15 min) + rotating refresh tokens with reuse detection | `references/sessions.md` |
| 4 | Device-bound sessions: every request signed by a key in the Secure Enclave / Android Keystore | `references/sessions.md` |
| 5 | Two-factor (TOTP + recovery codes) for app and admin; new sign-in alerts | `references/sessions.md` |
| 6 | Certificate pinning to the CA roots + Android network security config | `references/mobile.md` |
| 7 | Runtime protection: freeRASP (all events), native anti-debug / anti-hook, periodic re-check | `references/mobile.md` |
| 8 | Screenshot / recording block, app-switcher blur | `references/mobile.md` |
| 9 | Device integrity: Google Play Integrity + Apple App Attest, monitor → enforce | `references/integrity.md` |
| 10 | Security events stored + admin Security page | `references/server-api.md` |
| 11 | Admin panel behind mTLS (client certificate) + 2FA | `references/server-hardening.md` |
| 12 | Server: SSH keys only, firewall minimal, fail2ban, app as non-root user, secrets 600, HTTPS auto-renew | `references/server-hardening.md` |
| 13 | CI: tests gate deploys, gitleaks, dependency review, Dependabot security-only | `references/ci-cd.md` |
| 14 | Backups: database before each deploy, off-server; signing keys encrypted with age | `references/backups.md` |

Read only the guides for the layers you are working on.

## Explaining to the user

Keep it short and concrete: what an attacker could do before, what stops them now, what (if
anything) the user has to do. Example: "Before: someone who copied your login token could use your
account from their own phone. Now: every request must be signed by a key locked inside your phone's
chip, so a copied token is useless."
