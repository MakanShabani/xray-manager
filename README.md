# xray-manager

A single-file Bash script for managing [Xray](https://github.com/XTLS/Xray-core) proxy users on a Linux server. Handles first-time server setup, user creation, config generation, export, and cleanup — with four proxy configurations per user out of the box.

---

## Proxy Configurations

Each user gets four ready-to-import connection links:

| # | Protocol | Transport | Security | Port |
|---|---|---|---|---|
| 1 | VLESS | WebSocket | TLS + CDN + ECH | 443 |
| 2 | VLESS | gRPC | TLS + CDN + ECH | 443 |
| 3 | VMess | HTTPUpgrade | Plain (host spoofing) | 8080 |
| 4 | VLESS | TCP | Reality (TLS 1.3) | 8443 |

- **VLESS + WS/gRPC + CDN + ECH** — traffic routes through Cloudflare CDN; client configs enable Encrypted Client Hello (ECH is client-side only — not in the server `config.json`)
- **VMess + HTTPUpgrade** — lightweight direct connection disguised as HTTP traffic to a known host
- **VLESS + Reality** — direct connection with TLS 1.3 that borrows the fingerprint of a real website; no CDN needed, no third-party decryption, strongest censorship resistance

---

## Requirements

- Ubuntu / Debian server
- Root or sudo access
- [Xray-core](https://github.com/XTLS/Xray-core) installed
- A domain pointed at your server (for TLS + CDN configs)
- Port **80** reachable on the server during initial Let's Encrypt certificate issuance

> `nginx`, `certbot`, `jq`, `openssl`, and `qrencode` are installed automatically by the script if missing.

---

## Installation

```bash
curl -O https://raw.githubusercontent.com/MakanShabani/xray-manager/master/xray-manager.sh
chmod +x xray-manager.sh
sudo mv xray-manager.sh /usr/local/bin/xray-manager
```

Verify the download before moving it — the first line should read `#!/bin/bash`, not `404: Not Found`.

---

## Commands

```
xray-manager setup              First-time server setup
xray-manager create             Add a new user
xray-manager print              Print configs for all users
xray-manager print <username>   Print configs for a specific user
xray-manager delete <username>  Remove a single user
xray-manager export             Export all configs to /root/xray-configs/xray-configs.txt
xray-manager reset              Remove all users and restore config skeleton
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

**SSL certificate flow:** if no certificate exists yet, setup writes a temporary HTTP-only nginx config on port 80 for the ACME challenge, starts nginx, then runs certbot with `--webroot`. After the certificate is issued, the full TLS config on port 443 is written automatically. If Cloudflare proxies your domain (orange cloud), set the DNS record to **DNS only** (grey cloud) until the certificate is issued.

**Existing Xray install:** if the Xray installer already created `/usr/local/etc/xray/config.json` (even empty), setup detects a missing or invalid skeleton and writes the correct one.

---

### Adding a user

```bash
sudo xray-manager create
```

Prompts for a username, then:

- Generates a UUID
- Generates a Reality keypair (`xray x25519`) and a short ID
- Adds the user to all inbounds in `config.json`
- Creates the Reality inbound on first use (or adds to the existing one)
- Validates the config with `xray run -test` before restarting
- Restarts Xray
- Prints all four connection links and terminal QR codes
- Saves share links, **client JSON snippets** (for CDN+TLS with ECH), Reality keys, and QR code PNGs to `/root/xray-configs/`

**CDN+TLS client settings (WS and gRPC):** each user file includes ready-to-paste Xray client outbound JSON with:

```json
"tlsSettings": {
  "serverName": "<cdn-domain>",
  "fingerprint": "chrome",
  "echConfigList": "<cdn-domain>+udp://1.1.1.1",
  "echForceQuery": "full"
}
```

Share links also include `fp=chrome` and a URL-encoded `ech` parameter. ECH does not affect the server-side Xray config.

**Per-user files created:**

```
/root/xray-configs/xray_user_<username>.txt
/root/xray-configs/xray_user_<username>_ws.png
/root/xray-configs/xray_user_<username>_grpc.png
/root/xray-configs/xray_user_<username>_vmess.png
/root/xray-configs/xray_user_<username>_reality.png
```

If the Reality inbound was removed (e.g. after deleting the last user), `create` recreates it with a **new** keypair. If other users still exist, the new user is added to the existing Reality inbound and shares its keys.

---

### Printing configs

```bash
# Single user
sudo xray-manager print john

# All users
sudo xray-manager print
```

Reads UUIDs, Reality public keys, and short IDs directly from `config.json`. You only need to enter the server IP if it was not saved during setup.

Shows each link in the terminal with a scannable QR code and refreshes the PNG files in `/root/xray-configs/`.

If the username is not found, the script tells you to run `create`.

---

### Deleting a user

```bash
sudo xray-manager delete john
```

Removes the user from all inbounds in `config.json` (including Reality and their `shortId`), deletes `/root/xray-configs/xray_user_<username>.txt` and QR PNG files if they exist, validates the config, and restarts Xray.

If the deleted user was the **last** Reality user, the entire Reality inbound is removed. The next `create` will recreate it automatically. The export file is not updated automatically — run `export` again if you need a fresh copy.

---

### Exporting all configs

```bash
sudo xray-manager export
```

Writes all user connection links to `/root/xray-configs/xray-configs.txt` in plain text, ready to copy or share.

---

### Resetting all users

```bash
sudo xray-manager reset
```

Removes every user from `config.json`, restores the empty setup skeleton (including removing the Reality inbound), deletes everything in `/root/xray-configs/`, and restarts Xray. You must type `yes` to confirm. Nginx sites and `xray-manager.conf` are not changed.

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
DEFAULT_ECH_DNS="udp://1.1.1.1"
DEFAULT_ECH_FORCE_QUERY="full"
```

`DEFAULT_ECH_DNS` and `DEFAULT_ECH_FORCE_QUERY` apply to **client configs only** (share links and JSON snippets). The CDN domain is combined automatically: `echConfigList` becomes `<cdn-domain>+udp://1.1.1.1`.

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
/usr/local/bin/xray-manager                    The script
/usr/local/bin/xray-manager.conf               Saved defaults (created by setup)
/usr/local/etc/xray/config.json                Xray config
/usr/local/etc/xray/config.json.bak.*          Automatic backups before changes
/etc/nginx/sites-available/                    Nginx site configs
/root/xray-configs/                            All saved user configs and exports
/root/xray-configs/xray_user_<name>.txt        Per-user config file (created by create)
/root/xray-configs/xray_user_<name>_ws.png     QR code — VLESS WebSocket
/root/xray-configs/xray_user_<name>_grpc.png   QR code — VLESS gRPC
/root/xray-configs/xray_user_<name>_vmess.png   QR code — VMess
/root/xray-configs/xray_user_<name>_reality.png QR code — VLESS Reality
/root/xray-configs/xray-configs.txt            Full export (created by export)
```

---

## Troubleshooting

**Xray fails to start (exit code 23)**

Run the config test to see the exact error:

```bash
sudo xray run -test -config /usr/local/etc/xray/config.json
sudo journalctl -u xray -n 30 --no-pager
```

Common causes: empty or invalid `config.json`, empty Reality `privateKey` after a failed `create`, or port already in use.

**Empty `config.json` after setup**

Re-run setup with the latest script — it detects empty or invalid configs and writes the skeleton even if the file already exists.

**Certbot / nginx SSL errors**

Setup uses a temporary HTTP config on port 80 before certificates exist. Ensure port 80 is open and the domain points directly to the server (disable Cloudflare proxy during issuance). After a failed run, re-run `sudo xray-manager setup` once DNS and firewall are correct.

**Manual certificate issuance**

```bash
sudo mkdir -p /var/www/certbot
sudo certbot certonly --webroot -w /var/www/certbot \
  -d your.cdn.domain --email you@example.com --agree-tos
sudo xray-manager setup
```

---

## License

MIT
