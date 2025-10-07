# Certbot + Nginx front proxy (Let's Encrypt) and Redis

Overview
- Purpose: obtain and renew Let's Encrypt certificates via webroot, run Nginx as a front proxy (HTTP->HTTPS), and run core infra (Redis).
- HTTP (port 80): serves ACME challenges and redirects to HTTPS (after certs exist).
- HTTPS (port 443): serves using the renewed certificates from ./certs.
- Certbot: issues the initial cert and renews it periodically.
- Redis: provided as a core infra service on its own bridge network.

Layout
- dc-initial-cert.yml: bootstrap stack to obtain the first certificate using conf/nginx-cert.conf (no redirect).
- dc-front-proxy.yml: normal runtime stack using conf/nginx.conf (HTTP redirect + HTTPS with live certs). Includes a manual certbot service for scheduled renew.
- conf/nginx-cert.conf: Nginx config for initial issuance; serves ACME and returns 200 OK for all other paths (no redirect).
- conf/nginx.conf: Nginx config for production; HTTP serves ACME and redirects, HTTPS uses live certs.
- acme-webroot/: shared webroot for ACME challenges.
- certs/: Let’s Encrypt config and live certificates.

Assumptions
- Linux user: website-host
- Project path: /home/website-host/certbot (i.e., ~/certbot for that user)
- Domain: madsraad.com (and www.madsraad.com) points to this server (A/AAAA DNS records).
- Ports 80 and 443 are open to the internet.
- Docker and Docker Compose plugin are installed; website-host is in the docker group.

Quick prep
```bash
# as website-host
mkdir -p ~/certbot/{conf,acme-webroot,certs}
# Files in this repo should already be under ~/certbot
# Verify domain settings in:
# - conf/nginx-cert.conf
# - conf/nginx.conf
# Both are pre-set to: madsraad.com www.madsraad.com
```
## initial certification
Initial certificate issuance (one-time)
1) Start only Nginx for initial issuance:
```bash
cd ~/certbot
docker compose -f dc-initial-cert.yml up -d nginx
```

2) Obtain the certificate via webroot:
```bash
# Replace email if desired
docker compose -f dc-initial-cert.yml run --rm certbot \
  certonly --webroot -w /var/www/certbot \
  -d madsraad.com -d www.madsraad.com \
  --email admin@madsraad.com --agree-tos --no-eff-email
```

3) Verify certificates exist:
```bash
ls -l ~/certbot/certs/live/madsraad.com
# Expect: fullchain.pem, privkey.pem, etc.
```

4) Switch to the normal front-proxy stack:
```bash
docker compose -f dc-initial-cert.yml down
docker compose -f dc-front-proxy.yml up -d
```
- Nginx will now serve HTTPS with the issued certificate and redirect HTTP->HTTPS.
- Redis can be started as needed (see below).

If you want to Running services manually:
```bash
cd ~/certbot
docker compose -f dc-front-proxy.yml up -d           # start nginx and redis (restart: always keeps them up across reboots)
```
### Renewal of certifications and start of infrastructure apps
Set up automatic renewals (systemd user timer)
- The dc-front-proxy.yml contains a certbot service configured for manual runs (profiles: ["manual"]). We trigger it via a systemd user timer so it runs twice daily, then reload Nginx.

1) Ensure the front-proxy stack is running:
```bash
cd ~/certbot
docker compose -f dc-front-proxy.yml up -d
```

2) Create systemd user units (as website-host):
```bash
mkdir -p ~/.config/systemd/user
```

~/.config/systemd/user/certbot-renew.service
```ini
[Unit]
Description=Renew Let's Encrypt certificates (webroot) for nginx-proxy
Wants=network-online.target
After=network-online.target

[Service]
Type=oneshot
WorkingDirectory=/home/website-host/certbot
# Runs the certbot service from dc-front-proxy.yml (profiles: manual)
ExecStart=/usr/bin/docker compose -f dc-front-proxy.yml run --rm certbot
# Reload nginx to pick up renewed certs (container_name: nginx-proxy)
ExecStartPost=/usr/bin/docker kill -s HUP nginx-proxy
```

~/.config/systemd/user/certbot-renew.timer
```ini
[Unit]
Description=Run certbot renew twice daily

[Timer]
OnCalendar=*-*-* 00,12:00:00
RandomizedDelaySec=300
Persistent=true

[Install]
WantedBy=timers.target
```

3) Enable lingering so user timers run without an active login:
```bash
# run as root or with sudo:
sudo loginctl enable-linger website-host
```

4) Enable and start the timer (as website-host):
```bash
systemctl --user daemon-reload
systemctl --user enable --now certbot-renew.timer
systemctl --user status certbot-renew.timer
```

Manual renewal test
```bash
# ad-hoc renew attempt
cd ~/certbot
docker compose -f dc-front-proxy.yml run --rm certbot
# then reload nginx if needed
docker kill -s HUP nginx-proxy
```

