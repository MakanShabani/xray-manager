# xray-manager

A single-file Bash script for managing [Xray](https://github.com/XTLS/Xray-core) proxy users on a Linux server. Handles first-time server setup, user creation, config generation, and export — with support for three proxy configurations per user out of the box.

---

## Proxy Configurations

Each user gets four ready-to-import connection links:

| # | Protocol | Transport | Security | Port |
|---|---|---|---|---|
| 1 | VLESS | WebSocket | TLS + CDN | 443 |
| 2 | VLESS | gRPC | TLS + CDN | 443 |
| 3 | VMess | HTTPUpgrade | Plain (host spoofing) | 8080 |
| 4 | VLESS | TCP | Reality (TLS 1.3) | 8443 |

- **VLESS + WS/gRPC + CDN** — traffic routes through Cloudflare CDN, hiding your server IP behind a trusted domain
- **VMess + HTTPUpgrade** — lightweight direct connection disguised as HTTP traffic to a known host
- **VLESS + Reality** — direct connection with TLS 1.3 that borrows the fingerprint of a real website; no CDN needed, no third-party decryption, strongest censorship resistance

---

## Requirements

- Ubuntu / Debian server
- Root or sudo access
- [Xray-core](https://github.com/XTLS/Xray-core) installed
- A domain pointed at your server (for TLS + CDN configs)

> `nginx`, `certbot`, `jq`, and `openssl` are installed automatically by the script if missing.

---

## Installation

```bash
curl -O https://raw.githubusercontent.com/MakanShabani/xray-manager/main/xray-manager
chmod +x xray-manager
sudo mv xray-manager /usr/local/bin/xray-manager
```

---

## Commands

```
xray-manager setup              First-time server setup
xray-manager create             Add a new user
xray-manager print              Print configs for all users
xray-manager print <username>   Print configs for a specific user
xray-manager export             Export all configs to ~/xray-configs.txt
```

---

## Usage

### First-time setup

Run once on a fresh server. Installs dependencies, creates the Xray config skeleton, writes two nginx sites, opens firewall ports, and optionally runs certbot.

```bash
sudo xray-manager setup
```

You will be prompted for:

- Server public IP
- CDN domain (for VLESS WS + gRPC, e.g. `proxy.example.com`)
- VMess host / nginx server name (e.g. `news.example.com`)
- Reality SNI domain (site to impersonate, e.g. `www.google.com`)
- Public ports: CDN (default `443`), VMess (default `8080`), Reality (default `8443`)
- Internal Xray ports: WS (default `10000`), gRPC (default `10001`), VMess (default `10080`)
- Paths: WS path, gRPC service name, VMess HTTPUpgrade path
- Email address for certbot TLS certificate

All answers are saved to `xray-manager.conf` in the same directory as the script and reused for every subsequent command.

After setup, your server will have:

```
/usr/local/etc/xray/config.json               Xray config (skeleton, no users yet)
/etc/nginx/sites-available/<cdn-domain>.conf  VLESS WS + gRPC over TLS
/etc/nginx/sites-available/<vmess-host>.conf  VMess HTTPUpgrade
/usr/local/bin/xray-manager.conf              Saved defaults
```

---

### Adding a user

```bash
sudo xray-manager create
```

Prompts for a username, then:

- Generates a UUID
- Generates a Reality keypair (`xray x25519`) and a short ID
- Adds the user to all inbounds in `config.json`
- Creates the Reality inbound on first use
- Restarts Xray
- Prints all four connection links
- Saves links and Reality keys to `/root/xray_user_<username>.txt`

---

### Printing configs

```bash
# Single user
sudo xray-manager print john

# All users
sudo xray-manager print
```

Reads UUIDs, Reality public keys, and short IDs directly from `config.json`. You only need to enter the server IP if it was not saved during setup.

If the username is not found, the script tells you to run `create`.

---

### Exporting all configs

```bash
sudo xray-manager export
```

Writes all user connection links to `~/xray-configs.txt` in plain text, ready to copy or share.

---

## Configuration File

After `setup`, a file called `xray-manager.conf` is created next to the script. It stores all your defaults so you never have to re-enter them:

```bash
DEFAULT_SERVER_IP="1.2.3.4"
DEFAULT_CDN_DOMAIN="proxy.example.com"
DEFAULT_VMESS_HOST="news.example.com"
DEFAULT_REALITY_SNI="www.google.com"
DEFAULT_CDN_PORT="443"
DEFAULT_VMESS_PORT="8080"
DEFAULT_REALITY_PORT="8443"
VLESS_WS_PATH="/slark"
VLESS_GRPC_SERVICE="slarkg"
VMESS_PATH="/your-secret-path"
XRAY_VLESS_WS_PORT="10000"
XRAY_VLESS_GRPC_PORT="10001"
XRAY_VMESS_PORT="10080"
```

You can edit this file directly at any time to change defaults without re-running setup.

---

## Nginx Sites

The script creates two nginx site configs:

**TLS site** (`<cdn-domain>.conf`) — handles VLESS WebSocket and gRPC, proxied from nginx (with TLS termination) to Xray on localhost:

```
Client → Cloudflare CDN → nginx:443 (TLS) → Xray:10000 (WS)
                                           → Xray:10001 (gRPC)
```

**VMess site** (`<vmess-host>.conf`) — listens on a plain HTTP port, drops all traffic except the exact secret path:

```
Client → nginx:8080 → Xray:10080 (HTTPUpgrade)
```

**Reality** bypasses nginx entirely — Xray listens directly on `0.0.0.0:8443` and handles TLS itself:

```
Client → Xray:8443 (Reality/TLS 1.3)
```

---

## Xray Installation

The script does not install Xray automatically. Install it with the official installer before running `setup`:

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
```

---

## File Layout

```
/usr/local/bin/xray-manager           The script
/usr/local/bin/xray-manager.conf      Saved defaults (created by setup)
/usr/local/etc/xray/config.json       Xray config
/etc/nginx/sites-available/           Nginx site configs
/root/xray_user_<name>.txt            Per-user config file (created by create)
~/xray-configs.txt                    Full export (created by export)
```

---

## License

MIT
