# Create a systemd Service (Auto-Restart, Logs, Security)

Turn any script or binary into a reliable **background service** with `systemd`. You’ll get **auto-restart**, **start on boot**, and **journal logs** out of the box. Safe to copy/paste.

---

## Quick start — copy/paste this script

```bash
#!/usr/bin/env bash
# Create a systemd service + optional drop-in + optional timer in ONE go.

set -euo pipefail

# ========= CONFIG =========
APP="myapp"                               # service name (no spaces)
BIN_SRC="/path/to/your/app"               # <- change this to your binary/script
PORT="8080"                               # example env var
ENVIRONMENT="production"                  # example env var

# Timer (optional)
ENABLE_TIMER=true                         # set to false to skip timer
TIMER_ONCALENDAR="*-*-* 03:30:00"         # daily at 03:30
# ==========================

APP_USER="$APP"
APP_GROUP="$APP"
APP_DIR="/opt/$APP"
BIN_DST="$APP_DIR/$APP"
ENV_DIR="/etc/$APP"
ENV_FILE="$ENV_DIR/$APP.env"
LOG_DIR="/var/log/$APP"

UNIT_FILE="/etc/systemd/system/$APP.service"
DROPIN_DIR="/etc/systemd/system/$APP.service.d"
OVERRIDE_FILE="$DROPIN_DIR/override.conf"
EXECSTART_DROPIN="$DROPIN_DIR/execstart.conf"

TIMER_FILE="/etc/systemd/system/$APP.timer"

echo "==> Preparing user and directories for '$APP'"

# Create system user/group if missing
if ! id -u "$APP_USER" &>/dev/null; then
  sudo useradd --system --no-create-home --shell /usr/sbin/nologin "$APP_USER"
fi

# App dir
sudo install -d -o "$APP_USER" -g "$APP_GROUP" -m 0755 "$APP_DIR"

# Put your binary/script in place (or symlink an existing one)
if [[ -n "${BIN_SRC:-}" && -e "$BIN_SRC" ]]; then
  sudo install -m 0755 "$BIN_SRC" "$BIN_DST"
else
  echo "!! BIN_SRC '$BIN_SRC' not found. Creating a placeholder script."
  echo '#!/usr/bin/env bash' | sudo tee "$BIN_DST" >/dev/null
  echo 'echo "Hello from myapp"; sleep 2' | sudo tee -a "$BIN_DST" >/dev/null
  sudo chmod 0755 "$BIN_DST"
fi
sudo chown "$APP_USER:$APP_GROUP" "$BIN_DST"

# Env file (optional)
sudo install -d -m 0755 "$ENV_DIR"
sudo bash -c "cat > '$ENV_FILE' <<EOF
PORT=$PORT
ENV=$ENVIRONMENT
EOF"

# Log dir (only if your app writes files; journal works without it)
sudo install -d -o "$APP_USER" -g "$APP_GROUP" -m 0750 "$LOG_DIR"

echo "==> Writing unit file: $UNIT_FILE"
sudo bash -c "cat > '$UNIT_FILE' <<EOF
[Unit]
Description=$APP service
After=network-online.target
Wants=network-online.target

[Service]
User=$APP_USER
Group=$APP_GROUP
WorkingDirectory=$APP_DIR

# Load env vars if present
EnvironmentFile=-$ENV_FILE

# Your executable (script or binary)
ExecStart=$BIN_DST

# Restart policy
Restart=on-failure
RestartSec=3

# Reasonable limits
LimitNOFILE=65536

# If you need privileged ports (<1024), uncomment:
# AmbientCapabilities=CAP_NET_BIND_SERVICE

# Security hardening
NoNewPrivileges=yes
PrivateTmp=yes
PrivateDevices=yes
ProtectSystem=strict
ProtectHome=read-only
ProtectKernelTunables=yes
ProtectKernelModules=yes
ProtectControlGroups=yes
RestrictSUIDSGID=yes
LockPersonality=yes
MemoryDenyWriteExecute=yes
CapabilityBoundingSet=
AmbientCapabilities=
SystemCallFilter=@system-service @basic-io

# Allow writes only where needed
ReadWritePaths=$APP_DIR $LOG_DIR

# Logging to journal
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF"

echo "==> Reloading systemd, enabling and starting service"
sudo systemctl daemon-reload
sudo systemctl enable "$APP.service"
sudo systemctl restart "$APP.service"
sudo systemctl --no-pager --full status "$APP.service" || true

# -------- Optional: drop-in override (safe edits without touching main unit) --------
echo "==> Creating drop-in override: $OVERRIDE_FILE"
sudo install -d "$DROPIN_DIR"
sudo bash -c "cat > '$OVERRIDE_FILE' <<EOF
# $OVERRIDE_FILE
[Service]
EnvironmentFile=-$ENV_FILE
Environment=\"LOG_LEVEL=info\"
WorkingDirectory=$APP_DIR

Restart=on-failure
RestartSec=3s
LimitNOFILE=65536

NoNewPrivileges=yes
PrivateTmp=yes
ProtectSystem=strict
ProtectHome=read-only
ReadWritePaths=$APP_DIR $LOG_DIR
CapabilityBoundingSet=
AmbientCapabilities=
StandardOutput=journal
StandardError=journal
EOF"

# Example: change ExecStart via drop-in (disabled by default)
# sudo bash -c "cat > '$EXECSTART_DROPIN' <<'EOF'
# [Service]
# ExecStart=
# ExecStart=/opt/myapp/myapp --config /etc/myapp/config.yml
# EOF"

echo "==> Applying drop-in"
sudo systemctl daemon-reload
sudo systemctl restart "$APP.service"
sudo systemctl cat "$APP.service" | sed -n '1,120p' || true

# -------- Optional: systemd timer --------
if [[ "${ENABLE_TIMER}" == "true" ]]; then
  echo "==> Creating timer: $TIMER_FILE"
  sudo bash -c "cat > '$TIMER_FILE' <<EOF
[Unit]
Description=Run $APP on a schedule

[Timer]
OnCalendar=$TIMER_ONCALENDAR
Persistent=true
Unit=$APP.service

[Install]
WantedBy=timers.target
EOF"
  sudo systemctl daemon-reload
  sudo systemctl enable --now "$APP.timer"
  systemctl list-timers | (grep -E \"$APP\.timer\" || true)
fi

cat <<'EOS'

# ===== Usage & Logs =====
# Start/stop/restart:
#   sudo systemctl start  '"$APP"'.service
#   sudo systemctl stop   '"$APP"'.service
#   sudo systemctl restart '"$APP"'.service
#
# Enable/disable at boot:
#   sudo systemctl enable '"$APP"'.service
#   sudo systemctl disable '"$APP"'.service
#
# Show effective unit (with drop-ins):
#   systemctl cat '"$APP"'.service
#
# Logs (journal):
#   journalctl -u '"$APP"'.service -f
#   journalctl -u '"$APP"'.service -n 200
#   journalctl -u '"$APP"'.service -p err -n 100
#
# ===== Uninstall =====
#   sudo systemctl disable --now '"$APP"'.service '"$APP"'.timer 2>/dev/null || true
#   sudo rm -f /etc/systemd/system/'"$APP"'.service /etc/systemd/system/'"$APP"'.timer
#   sudo systemctl daemon-reload
#   sudo userdel '"$APP"' 2>/dev/null || true
#   sudo rm -rf '"$APP_DIR"' '"$LOG_DIR"' '"$ENV_DIR"'
EOS

echo "==> Done. Tip: tail logs with:  journalctl -u $APP.service -f"