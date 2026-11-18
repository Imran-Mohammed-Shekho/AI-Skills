# Backups and key safety

## Database

- Before every deploy: `pg_dump "$DATABASE_URL" | gzip > ~/backups/app-$(date +%Y%m%d-%H%M%S).sql.gz`,
  keep the last 14.
- **Off the server** too (a hacked or dead server takes local backups with it): daily cron →
  encrypt with `age -r <public key>` → upload to object storage (R2/S3/B2) with a write-only key.
  Test a restore once.

## Signing keys and secrets (the "lost keystore" problem)

Losing the Android upload keystore or `.env` hurts. Gather every secret file into one folder
(per app: `upload-keystore.jks` + `key.properties`, server `.env`, service-account JSON, admin
`.p12`, SSH keys), then encrypt with **age** and an X25519 key (not a human password, which can be
guessed):

```bash
brew install age
umask 077
age-keygen -o backup-secret-key.txt
tar -czf - keys-folder | age -r "$(age-keygen -y backup-secret-key.txt)" -o keys-backup.tar.age
# prove it restores before deleting anything
t=$(mktemp -d); age -d -i backup-secret-key.txt keys-backup.tar.age | tar -xzf - -C "$t" && diff -r keys-folder "$t/keys-folder"
```
- `backup-secret-key.txt` (one line, `AGE-SECRET-KEY-1…`) → the user's **password manager** as a
  secure note (Bitwarden free works; text fits). Verify the saved text matches the file.
- `keys-backup.tar.age` → anywhere: a private repo, iCloud, Drive. Without the key it's useless.
- Then securely delete the plain folder and key file (`rm -P`, not Trash).
- Never put raw keys in a git repo, even private: account compromise, accidental public, history
  is forever.

Restore: `age -d -i backup-secret-key.txt keys-backup.tar.age | tar -xzf -`.

## Lost Android upload key

With Play App Signing users are unaffected. Make a new key, export
`keytool -export -rfc -keystore upload-keystore.jks -alias upload -file upload_certificate.pem`,
then Play Console → Protected with Play → Play Store protection → Manage Play app signing →
Request upload key reset. Update CI secrets with the new keystore.
