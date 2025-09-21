# Let's Encrypt for Nginx (Zero-Downtime HTTPS + Auto-Renew)

Turn any Nginx site into **HTTPS** with free TLS certificates from **Let’s Encrypt**, using `certbot` and the Nginx plugin. Includes **firewall rules**, **HTTP→HTTPS redirect**, and **auto-renew** checks. Safe to copy/paste.

## Prerequisites
- A public domain (e.g., `example.com`) pointing to your server’s IP (A/AAAA record)
- Nginx installed and serving something on **port 80**
- A sudo-capable user
- (Optional) Firewall enabled (UFW or firewalld)

---

## 1) Install Certbot + Nginx plugin
Choose your family:

### Debian/Ubuntu
```bash
sudo apt update
sudo apt install -y nginx
sudo apt install -y certbot python3-certbot-nginx
```

### RHEL/CentOS/Alma/Rocky (EPEL required)
```bash
sudo dnf install -y epel-release
sudo dnf install -y nginx certbot python3-certbot-nginx
sudo systemctl enable --now nginx
```

> [!NOTE]
> You only need `nginx` here if it wasn’t installed yet. Make sure it’s running and listening on :80 before requesting a cert.

---

## 2) Open the firewall (if applicable)

### UFW (Ubuntu/Debian)
```bash
sudo ufw allow 'Nginx Full'   # opens 80/tcp and 443/tcp
sudo ufw status
```

### firewalld (RHEL family)
```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --reload
```

---

## 3) (Optional) Create a minimal HTTP server block
If you don’t already have one, create a basic site on port 80. Replace `example.com`.

```bash
sudo mkdir -p /var/www/example.com/html
echo '<h1>It works (HTTP)!</h1>' | sudo tee /var/www/example.com/html/index.html >/dev/null

sudo tee /etc/nginx/conf.d/example.com.conf >/dev/null <<'NGINX'
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;
    root /var/www/example.com/html;
    index index.html;
    access_log /var/log/nginx/example.com.access.log;
    error_log  /var/log/nginx/example.com.error.log;
    location / {
        try_files $uri $uri/ =404;
    }
}
NGINX

sudo nginx -t && sudo systemctl reload nginx
```

> [!TIP]
> Use the **Command Builder** to craft safe variations: [/tools/builder](/tools/builder)

---

## 4) Request and install the certificate (Nginx plugin)
Replace the domains with yours:

```bash
sudo certbot --nginx -d example.com -d www.example.com
```

Follow the prompt to:
- choose redirect (recommended),
- provide email (for renewal notices),
- accept the ToS.

Certbot will edit your Nginx config, add SSL blocks, and reload Nginx.

---

## 5) Verify HTTPS and auto-renew
Check the site over HTTPS in your browser first. Then verify certbot’s **systemd timer**:

```bash
systemctl list-timers | grep certbot || true
sudo certbot renew --dry-run
```

> [!NOTE]
> Certbot installs a systemd **timer** by default, so you **do not** need a cron entry. If you prefer scheduling by hand, see our systemd guide: [/guides/systemd-service/](/guides/systemd-service/)

---

## 6) Harden TLS (optional, safe defaults)
Certbot’s config is already good. To add a couple of explicit options, you can extend your HTTPS server block:

```nginx
# inside the `server { listen 443 ssl; ... }` block
ssl_protocols TLSv1.2 TLSv1.3;
ssl_prefer_server_ciphers off;
# Enable OCSP stapling if chain provides it:
ssl_stapling on;
ssl_stapling_verify on;
resolver 1.1.1.1 1.0.0.1 valid=300s;
resolver_timeout 5s;
```

Then test and reload:

```bash
sudo nginx -t && sudo systemctl reload nginx
```

> [!WARNING]
> Avoid pasting outdated cipher suites from random blogs. The snippet above relies on OpenSSL’s modern defaults and is usually enough.

---

## 7) Alternate method: Webroot (no direct Nginx edits)
If the Nginx plugin isn’t available, use **webroot**. Ensure `/.well-known/acme-challenge/` is served by your site:

```bash
sudo mkdir -p /var/www/example.com/html/.well-known/acme-challenge
sudo chown -R www-data:www-data /var/www/example.com/html
```

Add this location to your port-80 server block (adjust root if different):

```nginx
location ^~ /.well-known/acme-challenge/ {
    root /var/www/example.com/html;
    default_type "text/plain";
}
```

Reload and request certs:

```bash
sudo nginx -t && sudo systemctl reload nginx
sudo certbot certonly --webroot -w /var/www/example.com/html \
  -d example.com -d www.example.com
```

Now manually add a 443 server block with `ssl_certificate`/`ssl_certificate_key` that point to `/etc/letsencrypt/live/example.com/fullchain.pem` and `privkey.pem`, then reload.

---

## 8) Troubleshooting
- **DNS not pointing** → `curl -I http://example.com` from elsewhere should hit your server.
- **Port 80 blocked** → Let’s Encrypt must reach `http://example.com/.well-known/...` over **80/tcp**.
- **SELinux** (RHEL) → If you changed paths, you may need: `sudo setsebool -P httpd_can_network_connect 1`.
- **Multiple server blocks** → Ensure the correct `server_name` and no conflicting default servers.
- **Rate limits** → Use `--dry-run` or `--staging` for repeated tests.

---

## Renewals and certificate paths
- Live certs: `/etc/letsencrypt/live/<your-domain>/`
- Renewals are automatic via systemd timer; you can force one with:
```bash
sudo certbot renew --force-renewal
```

> [!DANGER]
> Never edit files in `/etc/letsencrypt/live/` directly; they are symlinks managed by Certbot. Always change Nginx config, then `nginx -t && systemctl reload nginx`.

---

## Remove certificates (if needed)
```bash
sudo certbot delete
sudo rm -f /etc/nginx/conf.d/example.com.conf
sudo nginx -t && sudo systemctl reload nginx
```

> [!TIP]
> Migrating a job (renewal hooks, log rotation, etc.)? Consider a **systemd service + timer** instead of cron for reliability: [/guides/systemd-service/](/guides/systemd-service/)