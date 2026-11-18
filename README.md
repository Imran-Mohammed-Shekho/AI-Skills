# AI Skills

Skills for AI coding agents (Claude Code and others that read `SKILL.md`).

## Security

| Skill | What it does |
|---|---|
| [app-security-hardening](Security/app-security-hardening/SKILL.md) | Audits and hardens a mobile app, its API, admin panel and server, layer by layer: short tokens with refresh rotation, device-bound request signing, two-factor, certificate pinning, RASP / anti-tamper, Play Integrity and App Attest, screenshot blocking, server lockdown, CI scanning, encrypted key backups. |

## Install

Copy a skill folder into your skills directory:

```bash
git clone https://github.com/Imran-Mohammed-Shekho/AI-Skills.git
cp -R AI-Skills/Security/app-security-hardening ~/.claude/skills/
```

Then ask your agent to "secure my app", or run `/app-security-hardening`.
