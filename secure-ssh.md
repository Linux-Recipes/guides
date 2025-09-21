# Secure SSH on Linux (Keys, UFW, Fail2ban)

Harden your SSH service with **public key authentication**, **firewall rate limiting**, and **Fail2ban** bans for brute-force IPs. This guide is safe to copy/paste and links to linux.recipes tools and command docs where helpful.

## Prerequisites
- A sudo-capable user
- SSH already installed (`openssh-server`)
- Basic terminal access

---

## 1) Create an SSH key pair (on your laptop)
Use a modern algorithm (ed25519). Protect the private key with a passphrase.

```bash
ssh-keygen -t ed25519 -a 100 -f ~/.ssh/id_ed25519
# -a 100 = key derivation rounds (slows offline attacks)
# Accept the default path and set a strong passphrase when prompted.