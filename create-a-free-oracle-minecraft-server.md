<!-- level: Beginner -->
<!-- tags: minecraft, server, oracle, bedrock -->

# Run a Minecraft Bedrock Server on Linux (Oracle Cloud Free Tier)

Host a free **Minecraft Bedrock Edition** server on Oracle Cloud (Ampere A1 ARM) or any Linux VM.  
Safe to copy/paste — works with ARM-based Oracle Free Tier.

## Prerequisites
- Oracle Cloud Free Tier account (or any Linux server)
- A sudo-capable user
- Minecraft Bedrock clients (Xbox, iOS, Android, Windows 10/11, Switch, PS4)

---

## 1) Update system

sudo apt update && sudo apt upgrade -y   # Ubuntu/Debian
# OR
sudo dnf update -y                       # Oracle Linux 8/9

## 2) Install dependencies

sudo apt install unzip curl -y           # Ubuntu/Debian
# OR
sudo dnf install unzip curl -y           # Oracle Linux

## 3) Create a server directory

mkdir -p ~/bedrock
cd ~/bedrock

## 4) Download Minecraft Bedrock server

Find the latest download at:
https://www.minecraft.net/en-us/download/server/bedrock

Then copy the link and run:

curl -O https://minecraft.azureedge.net/bin-linux/bedrock-server-<VERSION>.zip
unzip bedrock-server-<VERSION>.zip
chmod +x bedrock_server

## 5) Open the firewall & Oracle ports

On the VM itself:

sudo firewall-cmd --permanent --zone=public --add-port=19132/udp
sudo firewall-cmd --reload

On Oracle Cloud Console → Networking → VCN → Security Lists → Add Ingress Rule:

Protocol: UDP

Port: 19132

Source: 0.0.0.0/0

## 6) Start the server

./bedrock_server

It will generate server.properties on first run.
Edit to adjust server name, max players, gamemode, etc.

## 7) Keep it running (tmux)

sudo dnf install tmux -y   # or sudo apt install tmux -y
tmux new -s bedrock
./bedrock_server
# detach with Ctrl+B then D
tmux attach -t bedrock

## 8) Connect from Minecraft

Use your Oracle VM public IP

Port: 19132 (default)

Done! You now have a cross-platform Bedrock server running free on Oracle Cloud.
Friends on Xbox, PlayStation, Switch, iOS, Android, Windows 10/11 can all join.
