# Nginx Reverse Proxy + Let’s Encrypt (HTTPS) for Any App

Put your app (Node, Python, Go, PHP, etc.) behind **Nginx** with automatic **HTTPS** via **Let’s Encrypt**. Works great for apps on `127.0.0.1:3000` (or a Unix socket). Safe to copy/paste.

## Prerequisites
- A domain pointing to your server’s public IP (A/AAAA record)
- Ports **80** and **443** open in your firewall (see [/guides/ufw-hardening/](/guides/ufw-hardening/))
- App listening on `127.0.0.1:3000` (adjust as needed)
- Debian/Ubuntu-like server (paths similar elsewhere)

---

## 1) Install Nginx and Certbot
```bash
sudo apt update
sudo apt install -y nginx certbot python3-certbot-nginx

# Allow HTTP + HTTPS through UFW (if you use UFW)
sudo ufw allow 'Nginx Full'
sudo ufw delete allow 'Nginx HTTP' 2>/dev/null || true

# Check Nginx health
sudo systemctl enable --now nginx
sudo nginx -t && sudo systemctl reload nginx
```

> [!TIP]
> If you proxy to a **Unix socket** (e.g., `/run/myapp.sock`), skip opening port 3000 to the world; only Nginx needs to read the socket.

---

## 2) Create a minimal server block (HTTP only for now)
Replace `example.com` with your domain. If you also want `www`, add it to `server_name`.

```bash
sudo tee /etc/nginx/sites-available/example.com >/dev/null <<'EOF'
server {
    listen 80;
    listen [::]:80;
    server_name example.com;

    # ACME HTTP-01 challenge
    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type "text/plain";
    }

    # Proxy to your app
    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_http_version 1.1;

        # WebSocket support
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;

        # Pass client details to the app
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts sane defaults
        proxy_connect_timeout   5s;
        proxy_send_timeout     60s;
        proxy_read_timeout     60s;
    }
}

# Helper map for WebSockets (above)
map $http_upgrade $connection_upgrade {
    default upgrade;
    ''      close;
}
EOF

sudo ln -sf /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/example.com
sudo nginx -t && sudo systemctl reload nginx
```

> [!NOTE]
> If your app listens on a **Unix socket**, use:
> ```nginx
> proxy_pass http://unix:/run/myapp.sock:;
> ```

---

## 3) Obtain and install a TLS certificate (auto HTTPS)
```bash
# This edits your Nginx block and adds TLS + 301 redirects automatically
sudo certbot --nginx -d example.com

# If you also want www:
# sudo certbot --nginx -d example.com -d www.example.com

# Dry-run renewal to verify
sudo certbot renew --dry-run
```

> [!TIP]
> Certbot installs a **systemd timer** to renew certs automatically. Check with:
> ```bash
> systemctl list-timers | grep certbot
> ```

---

## 4) Add security & performance headers (optional but recommended)
```bash
sudo tee /etc/nginx/snippets/secure_headers.conf >/dev/null <<'EOF'
# Basic sane defaults; tweak to your app’s needs
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header X-XSS-Protection "0" always;

# If your app is purely static or carefully audited, consider CSP:
# add_header Content-Security-Policy "default-src 'self'; object-src 'none'; base-uri 'self'; frame-ancestors 'self';" always;
EOF
```

Include the snippet in your TLS server block. Certbot usually created a `server` listening on `443`. Edit it:

```bash
sudo nano /etc/nginx/sites-available/example.com
```

Inside the **TLS** server block (`listen 443 ssl;`), add:

```nginx
include snippets/secure_headers.conf;

# Optional gzip (safe defaults)
gzip on;
gzip_types text/plain text/css application/json application/javascript application/xml+rss;
gzip_vary on;
```

Then:
```bash
sudo nginx -t && sudo systemctl reload nginx
```

---

## 5) Proxy to a socket (alternative to port 3000)
If your app exposes `/run/myapp.sock`, change the `proxy_pass` and tighten firewall:

```nginx
location / {
    proxy_pass http://unix:/run/myapp.sock:;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection $connection_upgrade;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```

And (if using UFW) block external access to your app’s TCP port entirely:
```bash
sudo ufw deny 3000/tcp
```

---

## 6) Zero-downtime reloads and log checks
```bash
# Validate and reload
sudo nginx -t && sudo systemctl reload nginx

# Tail access/error logs
sudo tail -n 200 /var/log/nginx/access.log
sudo tail -n 200 /var/log/nginx/error.log

# Show the active server blocks
sudo nginx -T | sed -n '1,200p'
```

---

## 7) HTTP/2 & (optional) HTTP/3
- **HTTP/2** is enabled by default in most certbot-generated configs (`listen 443 ssl http2;`).
- **HTTP/3/QUIC** needs a newer Nginx build and UDP/443 open. If you enable it:
  ```nginx
  listen 443 quic reuseport;   # plus your existing listen 443 ssl http2;
  add_header Alt-Svc 'h3=":443"; ma=86400';
  ```
  And open:
  ```bash
  sudo ufw allow 443/udp
  ```

---

## 8) Common gotchas
- **DNS not propagated** yet → Let’s Encrypt fails. Check with `dig +short A example.com`.
- **Port 80 blocked** → HTTP-01 challenge fails. Ensure firewall/NAT lets **80/443** through.
- **Upstream 502/504** → App not running, wrong port/socket path, or permissions on the socket.
- **Mixed content** → Update app/frontend to use `https://` URLs.

---

## Quick copy/paste checklist
```bash
# 1) Install
sudo apt update && sudo apt install -y nginx certbot python3-certbot-nginx

# 2) Open firewall
sudo ufw allow 'Nginx Full' && sudo ufw delete allow 'Nginx HTTP' 2>/dev/null || true

# 3) Create site (edit domain + upstream)
DOM=example.com
sudo tee /etc/nginx/sites-available/$DOM >/dev/null <<EOF
server {
    listen 80;
    listen [::]:80;
    server_name $DOM;

    location ^~ /.well-known/acme-challenge/ {
        root /var/www/html;
        default_type "text/plain";
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade \$http_upgrade;
        proxy_set_header Connection \$connection_upgrade;
        proxy_set_header Host \$host;
        proxy_set_header X-Real-IP \$remote_addr;
        proxy_set_header X-Forwarded-For \$proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto \$scheme;
        proxy_connect_timeout 5s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
    }
}
map \$http_upgrade \$connection_upgrade { default upgrade; '' close; }
EOF
sudo ln -sf /etc/nginx/sites-available/$DOM /etc/nginx/sites-enabled/$DOM
sudo nginx -t && sudo systemctl reload nginx

# 4) Get cert + auto-redirect
sudo certbot --nginx -d $DOM
sudo certbot renew --dry-run

# 5) (Optional) add headers
sudo tee /etc/nginx/snippets/secure_headers.conf >/dev/null <<'EOF'
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
EOF
sudo sed -i '/listen 443 ssl/,$ s|}|    include snippets/secure_headers.conf;\n}|' /etc/nginx/sites-available/$DOM
sudo nginx -t && sudo systemctl reload nginx
```

> [!TIP]
> Pair this with a **systemd service** for your app (see [/guides/systemd-service/](/guides/systemd-service/)) and **Fail2ban** for brute-force protection (see [/guides/fail2ban-hardening/](/guides/fail2ban-hardening/)).