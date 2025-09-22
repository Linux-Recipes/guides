# WireGuard VPN Server (wg-quick) on Ubuntu/Debian
<!-- difficulty: Intermediate -->
<!-- tags: wireguard, vpn, networking, security -->
<!-- summary: Set up a fast, modern VPN with WireGuard, including NAT, firewall, and a ready-to-use client config. -->

Deploy a **fast, modern VPN** with WireGuard. This guide sets up a `wg0` server, enables **IP forwarding & NAT**, opens the **UDP port**, and creates a **client config** you can use on laptops and phones. Copy/paste friendly.

## Prerequisites
- Ubuntu/Debian server with sudo
- A public IP (or DNS) reachable on UDP `51820`
- Your default network interface name (e.g., `eth0`, `ens3`, `enp0s1`)

---

## 1) Install WireGuard (and QR tool for phones)
~~~bash
sudo apt update
sudo apt install -y wireguard qrencode
~~~

> [!TIP]
> `qrencode -t ansiutf8` lets you show a QR in the terminal for quick phone setup (iOS/Android).

---

## 2) Create keys & base directory
~~~bash
# Lock down the directory first
sudo install -d -m 0700 /etc/wireguard

# Generate server keypair
sudo bash -c 'umask 077; wg genkey | tee /etc/wireguard/server.key | wg pubkey > /etc/wireguard/server.pub'

# (Optional) Pre-create one client keypair
sudo bash -c 'umask 077; wg genkey | tee /etc/wireguard/client1.key | wg pubkey > /etc/wireguard/client1.pub'

# Show the generated public keys
echo "Server public key:"
sudo cat /etc/wireguard/server.pub
echo "Client1 public key:"
sudo cat /etc/wireguard/client1.pub
~~~

---

## 3) Enable IP forwarding (routing)
~~~bash
# Enable immediately
echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward >/dev/null

# Make persistent
echo 'net.ipv4.ip_forward=1' | sudo tee /etc/sysctl.d/99-wireguard-forwarding.conf >/dev/null
sudo sysctl --system
~~~

---

## 4) Create `/etc/wireguard/wg0.conf`
Replace **`eth0`** with your real network interface (run `ip -o -4 route show to default` to see it).  
You can change the VPN subnet (`10.8.0.0/24`) and port (`51820`) if you like.

~~~bash
WG_IF="wg0"
WG_PORT="51820"
WG_NET_CIDR="10.8.0.0/24"
WG_SERVER_ADDR="10.8.0.1/24"
WAN_IF="eth0"                          # <--- CHANGE to your real NIC (e.g., ens3)

SERVER_PRIV=$(sudo cat /etc/wireguard/server.key)

sudo bash -c "cat > /etc/wireguard/${WG_IF}.conf" <<EOF
[Interface]
Address = ${WG_SERVER_ADDR}
ListenPort = ${WG_PORT}
PrivateKey = ${SERVER_PRIV}
SaveConfig = false

# NAT and forwarding (iptables)
PostUp   = iptables -t nat -A POSTROUTING -s ${WG_NET_CIDR} -o ${WAN_IF} -j MASQUERADE
PostUp   = iptables -A FORWARD -i ${WAN_IF} -o ${WG_IF} -m state --state RELATED,ESTABLISHED -j ACCEPT
PostUp   = iptables -A FORWARD -i ${WG_IF} -o ${WAN_IF} -j ACCEPT
PostDown = iptables -t nat -D POSTROUTING -s ${WG_NET_CIDR} -o ${WAN_IF} -j MASQUERADE
PostDown = iptables -D FORWARD -i ${WAN_IF} -o ${WG_IF} -m state --state RELATED,ESTABLISHED -j ACCEPT
PostDown = iptables -D FORWARD -i ${WG_IF} -o ${WAN_IF} -j ACCEPT
EOF

# Lock it down
sudo chmod 600 /etc/wireguard/${WG_IF}.conf
~~~

> [!NOTE]
> On Debian/Ubuntu with nftables, these `iptables` commands are translated via the compatibility layer. If you run a pure-nftables setup, you can swap the PostUp/Down for nft rules instead.

---

## 5) Add your first peer (client)
This creates a client with VPN IP `10.8.0.2/32`. We’ll output a ready-to-use client config file.

~~~bash
WG_IF="wg0"
CLIENT_NAME="client1"
CLIENT_PRIV=$(sudo cat /etc/wireguard/${CLIENT_NAME}.key)
CLIENT_PUB=$(sudo cat /etc/wireguard/${CLIENT_NAME}.pub)
SERVER_PUB=$(sudo cat /etc/wireguard/server.pub)

# Determine your server's public endpoint (set manually if needed)
SERVER_ENDPOINT="$(curl -fsS ifconfig.me || echo YOUR.SERVER.IP):51820"

# Append peer to the server config
sudo tee -a /etc/wireguard/${WG_IF}.conf >/dev/null <<EOF

[Peer]
# ${CLIENT_NAME}
PublicKey = ${CLIENT_PUB}
AllowedIPs = 10.8.0.2/32
PersistentKeepalive = 25
EOF

# Create client config (save locally)
cat > ${CLIENT_NAME}.conf <<EOF
[Interface]
PrivateKey = ${CLIENT_PRIV}
Address = 10.8.0.2/32
DNS = 1.1.1.1

[Peer]
PublicKey = ${SERVER_PUB}
Endpoint = ${SERVER_ENDPOINT}
AllowedIPs = 0.0.0.0/0, ::/0
PersistentKeepalive = 25
EOF

echo "Wrote client config: $(pwd)/${CLIENT_NAME}.conf"
~~~

> [!TIP]
> If your server listens on a non-default port or has DNS, set `SERVER_ENDPOINT="vpn.example.com:51820"` before generating the client file.

---

## 6) Firewall: allow UDP port
If you use UFW:
~~~bash
sudo ufw allow 51820/udp
sudo ufw status
~~~

If you manage the firewall elsewhere (e.g., cloud provider), open **UDP 51820** there.

---

## 7) Start and enable the VPN
~~~bash
# Start now and enable on boot
sudo systemctl enable --now wg-quick@wg0

# Show active state and peers
sudo systemctl status wg-quick@wg0 --no-pager
sudo wg show
~~~

---

## 8) Use the client config (desktop or phone)
- **Desktop (Linux/macOS/Windows):** Import `client1.conf` into your WireGuard app.
- **Mobile (iOS/Android):** Show a QR and scan it in the WireGuard app:
~~~bash
qrencode -t ansiutf8 < client1.conf
~~~

---

## Verify connectivity
From the client, connect and then:
~~~bash
# Should see your server's wg0 peer
ping -c3 10.8.0.1

# On the server, watch handshakes/logs
sudo wg show
journalctl -u wg-quick@wg0 -f
~~~

---

## Add more peers later
Repeat the client key/config steps with new IPs (`10.8.0.3/32`, `10.8.0.4/32`, …) and append each peer to `/etc/wireguard/wg0.conf`. Then:
~~~bash
sudo systemctl restart wg-quick@wg0
sudo wg show
~~~

---

## Troubleshooting
- **No handshake:** Check the correct **UDP port** is open, and that the client’s **PublicKey** matches the server’s peer entry.
- **No internet through VPN:** Ensure `net.ipv4.ip_forward=1` and NAT (PostUp MASQUERADE) is correct for your **WAN interface**.
- **Wrong interface name:** Replace `eth0` with your real NIC (e.g., `ens3`, `enp0s1`).
- **Double NAT / CGNAT:** If your server is behind NAT, forward UDP `51820` on the upstream router to your server.

---

## Remove / stop WireGuard
~~~bash
sudo systemctl disable --now wg-quick@wg0
sudo rm -f /etc/wireguard/wg0.conf
sudo sysctl -w net.ipv4.ip_forward=0
~~~

> [!WARNING]
> Removing `wg0.conf` deletes your peers and server settings. Keep backups of client `.conf` files if you want to re-create the setup quickly.
