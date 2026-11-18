# Server hardening (Ubuntu VPS, nginx, PM2)

Always keep your current SSH session open while changing SSH or the firewall, and test a **new**
connection before closing it.

## SSH: keys only

```bash
# 1. a backup key (on the admin's machine), added to the server, tested
ssh-keygen -t ed25519 -N "" -C "server-backup-key" -f ~/.ssh/server_backup
ssh root@HOST "cat >> ~/.ssh/authorized_keys" < ~/.ssh/server_backup.pub
ssh -o IdentitiesOnly=yes -i ~/.ssh/server_backup root@HOST echo ok
# 2. a drop-in that is read FIRST (sshd keeps the first value it reads; cloud-init files
#    like 50-cloud-init.conf often say PasswordAuthentication yes)
cat > /etc/ssh/sshd_config.d/00-hardening.conf <<'EOF'
PasswordAuthentication no
KbdInteractiveAuthentication no
PermitRootLogin prohibit-password
MaxAuthTries 3
EOF
sshd -t && systemctl reload ssh
sshd -T | grep -E "passwordauthentication|permitrootlogin"
# 3. from the admin's machine: each key works, a password is refused
ssh -o PubkeyAuthentication=no -o PreferredAuthentications=password root@HOST true   # → Permission denied (publickey)
```
Have the user save the backup private key in their password manager, then delete the local copy.
Mention the provider's web console as the last-resort way in.

## Firewall

`ufw status numbered`; keep only `22, 80, 443`. Remove leftovers from panels (aaPanel/FTP: 20, 21,
888, passive ranges like 39000:40000, random panel ports) after checking with `ss -tlnp` that
nothing needed listens there. Keep app ports (3000, 4000 …) bound to 127.0.0.1 behind nginx;
verify from outside that they time out.

fail2ban `sshd` jail on.

## Run the app as a normal user

```bash
useradd -m -s /bin/bash app
install -d -m 700 -o app -g app /home/app/.ssh
grep "deploy-key-comment" /root/.ssh/authorized_keys > /home/app/.ssh/authorized_keys
chown app:app /home/app/.ssh/authorized_keys && chmod 600 /home/app/.ssh/authorized_keys
chown -R app:app /var/www/app
chmod 600 /var/www/app/**/.env* /var/www/app/backend/*-key.json
# move processes: stop under root → start under app → delete from root's PM2
pm2 stop api admin
sudo -iu app bash -c "cd /var/www/app/backend && pm2 start dist/src/index.js --name api && \
  cd /var/www/app/admin && pm2 start npx --name admin -- next start -H 127.0.0.1 -p 4001 && pm2 save"
pm2 delete api admin && pm2 save
pm2 startup systemd -u app --hp /home/app   # boot start for the app user
```
Check: health endpoint 200, `ps -o user=` shows `app`, logs clean (service account readable).
Then point CI deploys at the user (`DEPLOY_USER=app` secret) and let the next deploy prove it
(backup → build → migrate → reload → health check all as `app`). Ports above 1024 need no root.

## Admin panel behind mTLS

nginx for the admin host:
```nginx
ssl_client_certificate /etc/nginx/mtls/admin-ca.crt;
ssl_verify_client on;
```
Make a small CA, issue a client cert per admin device, give them a `.p12` (password-protected) to
install. Keep the CA key off the server. Plus 2FA inside the panel.

## HTTPS

`certbot renew --dry-run` must succeed for every certificate; `systemctl list-timers | grep certbot`.
If a certificate expires: apps with pinning or strict TLS stop connecting, browsers show
"not secure"; push still works (goes through Google/Apple).

## Secrets on disk

`.env` and service-account files: owner = app user, mode 600. Never print them in logs or chat.
