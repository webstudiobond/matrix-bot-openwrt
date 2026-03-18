# Matrix Bot for OpenWrt
[![ShellCheck Lint](https://github.com/webstudiobond/matrix-bot-openwrt/actions/workflows/shellcheck.yml/badge.svg)](https://github.com/webstudiobond/matrix-bot-openwrt/actions/workflows/shellcheck.yml)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

A lightweight, POSIX shell–based Matrix bot for remote router management over the [Matrix protocol](https://matrix.org/). Runs natively on OpenWrt without any runtime dependencies beyond what the standard image provides.

**Features:**
- Remote control via Matrix chat: services, interfaces, Wi-Fi, WOL, clients, system info
- **E2EE support** via [`matrix-commander-rs`](https://github.com/8go/matrix-commander-rs) over SSH (primary transport)
- **HTTP fallback** using the Matrix Client-Server API directly (no E2EE)
- **Auto mode**: E2EE listener primary, HTTP listener as fallback/parallel
- Security alerts forwarded to a dedicated admin room on unauthorized access attempts
- Managed by `procd` — automatic restart on crash, logs via `logread`

**Compatibility:** Successfully tested on OpenWrt 25.12.0 (Xiaomi Mi Router 3G).

---

## Architecture

```
OpenWrt Router
│
├── /etc/config/bot.conf          ← single config file (credentials, room IDs, etc.)
├── /etc/init.d/matrixbot         ← procd init script (start/stop/enable)
├── /etc/matrix_bot_known_hosts   ← SSH host key store (created during setup)
│
└── /usr/lib/matrix/
    ├── matrix_bot                ← main bot process (listener + command handler)
    └── matrix_send               ← standalone message sender (called by bot + usable directly)
```

**Transport priority (sending messages):**

```
matrix_send → [1] SSH → matrix-commander-rs (E2EE)
                      ↓ on failure
             [2] HTTP → Matrix Client API (plaintext)
```

**Listening priority (receiving commands):**

```
auto mode → listen_e2ee (SSH/matrix-commander-rs) — primary
          → listen_http (Matrix /sync polling)    — fallback / parallel
```

---

## Requirements

### OpenWrt packages

Install on the router:

```sh
apk update
apk add curl jq openssh-client
```

| Package | Purpose |
|---|---|
| `curl` | HTTP transport for sending/receiving Matrix API calls |
| `jq` | JSON parsing (fallback: built-in `jsonfilter`) |
| `openssh-client` | SSH transport to the E2EE host running `matrix-commander-rs` |

> `jsonfilter` is included in OpenWrt by default and is used automatically if `jq` is not installed. `jq` is strongly recommended for correctness with complex sync payloads.

For Wake-on-LAN support:

```sh
apk add etherwake
```

For Nginx reload support — Nginx must already be installed:

```sh
apk add nginx
```

### External host (E2EE only)

You need a separate machine (Linux server, VPS, Raspberry Pi, etc.) with:
- [`matrix-commander-rs`](https://github.com/8go/matrix-commander-rs) installed and **already logged in** to your Matrix account
- SSH server running and accessible from the router
- A dedicated SSH key pair for the bot (see [SSH Key Setup](#ssh-key-setup))

If you do not need E2EE, you can run the bot in `--no-e2ee` (HTTP-only) mode and skip the external host entirely.

---

## Matrix Setup

### 1. Register a bot account

Register a dedicated Matrix account for the bot on your homeserver (e.g. `@mybot:matrix.org`). Do **not** use your personal account.

### 2. Get an access token

Log in to the bot account and retrieve its access token:

```sh
curl -s -X POST "https://matrix.example.com/_matrix/client/v3/login" \
  -H "Content-Type: application/json" \
  -d '{"type":"m.login.password","user":"mybot","password":"yourpassword"}'
```

Copy the `access_token` field from the response.

### 3. Create rooms

Create (or reuse) the following rooms in your Matrix client:

| Room | Purpose | Variable |
|---|---|---|
| **Command room** | Where you send commands to the bot. Can be E2EE-encrypted. | `MATRIX_ROOM_E2EE_ID` |
| **Plaintext room** | Optional secondary room without encryption. | `MATRIX_ROOM_ID` |
| **Admin/alert room** | Where the bot sends security alerts (unauthorized access). Does **not** accept commands. | `MATRIX_ROOM_ADMIN` |

Invite the bot account to all rooms. Get room IDs from your client (usually under Room Settings → Advanced) — they look like `!opaque:server.tld`.

---

## SSH Key Setup

Generate a dedicated ED25519 key pair **on the router**:

```sh
mkdir -p /etc/matrix_bot
ssh-keygen -t ed25519 -f /etc/matrix_bot/id_ed25519 -N "" -C "matrix-bot@openwrt"
chmod 600 /etc/matrix_bot/id_ed25519
```

Copy the public key to the external E2EE host:

```sh
cat /etc/matrix_bot/id_ed25519.pub
# Append the output to ~/.ssh/authorized_keys on the remote host
```

Register the remote host key (run once — uses your config values):

```sh
. /etc/config/bot.conf && \
ssh -i "$SSH_KEY" -p "$SSH_PORT" \
    -o StrictHostKeyChecking=accept-new \
    -o UserKnownHostsFile=/etc/matrix_bot_known_hosts \
    -o BatchMode=yes \
    -o ConnectTimeout=10 \
    "$SSH_USER@$SSH_HOST" exit 2>&1 && \
echo "Host key saved." && cat /etc/matrix_bot_known_hosts
```

This stores the host key in `/etc/matrix_bot_known_hosts`. The bot uses `StrictHostKeyChecking=yes` against this file — MITM attacks will be rejected.

---

## Installation

### 1. Create the directory

```sh
mkdir -p /usr/lib/matrix
```

### 2. Download scripts

```sh
curl -sSL "https://raw.githubusercontent.com/webstudiobond/matrix-bot-openwrt/refs/heads/main/usr/lib/matrix/matrix_bot" -o /usr/lib/matrix/matrix_bot
curl -sSL "https://raw.githubusercontent.com/webstudiobond/matrix-bot-openwrt/refs/heads/main/usr/lib/matrix/matrix_send" -o /usr/lib/matrix/matrix_send
curl -sSL "https://raw.githubusercontent.com/webstudiobond/matrix-bot-openwrt/refs/heads/main/etc/init.d/matrixbot" -o /etc/init.d/matrixbot
```

### 3. Set permissions

```sh
chmod 700 /usr/lib/matrix/matrix_bot
chmod 700 /usr/lib/matrix/matrix_send
chmod 755 /etc/init.d/matrixbot
```

The scripts must be owned by `root` and not writable by anyone else — they execute as root and handle credentials.

### 4. Create the config file

```sh
touch /etc/config/bot.conf
chmod 600 /etc/config/bot.conf
```

Edit `/etc/config/bot.conf` — see [Configuration](#configuration) below.

### 5. Enable and start the service

```sh
/etc/init.d/matrixbot enable
/etc/init.d/matrixbot start
```

Check logs:

```sh
logread -e matrix
```

---

## Configuration

All settings live in `/etc/config/bot.conf`. This file is sourced as a shell script.

```sh
# /etc/config/bot.conf
# Permissions must be 600 (chmod 600 /etc/config/bot.conf)

# =====================
# Matrix
# =====================

# Base URL of your Matrix homeserver (no trailing slash)
MATRIX_URL='https://matrix.example.com'

# Room IDs — get from your Matrix client (Room Settings → Advanced)
# Primary room: E2EE-encrypted, used for sending commands
MATRIX_ROOM_E2EE_ID='!AbCdEfGhIj:matrix.example.com'
# Secondary room: plaintext (optional, can be same as above if no E2EE needed)
MATRIX_ROOM_ID='!KlMnOpQrSt:matrix.example.com'
# Admin/alert room: bot sends security warnings here. NOT a command room.
MATRIX_ROOM_ADMIN='!UvWxYzAbCd:matrix.example.com'

# Full Matrix user ID of the bot account (must be invited to all rooms above)
MATRIX_BOT_USER='@mybot:matrix.example.com'
# Full Matrix user ID of the admin (the only user whose commands are accepted)
MATRIX_ADMIN_USER='@me:matrix.example.com'

# Access token of the bot account (from login response)
MATRIX_ACCESS_TOKEN='syt_XXXXXXXXXXXXXXXXXXXXXXXX'

# =====================
# SSH (E2EE transport)
# =====================
# Leave blank to disable E2EE and use HTTP-only mode.

# Hostname or IP of the machine running matrix-commander-rs
SSH_HOST='192.168.1.100'
SSH_PORT='22'
SSH_USER='myuser'
# Path to the private key generated during setup
SSH_KEY='/etc/matrix_bot/id_ed25519'

# =====================
# Wake-on-LAN
# =====================

# MAC address of a specific machine to wake with the 'wol_pc' command (optional)
MAC_PC='AA:BB:CC:DD:EE:FF'

# =====================
# Wi-Fi display
# =====================

# Set to 1 for detailed Wi-Fi info (hardware, BSSID, signal, TX power, etc.)
# Set to 0 or leave empty for simple mode (SSID, channel, rate, key)
WIFI_DETAILED='0'

# =====================
# Service control
# =====================

# Space-separated whitelist of services allowed to be restarted via 'restart' command.
# Only services in this list AND present in /etc/init.d/ can be restarted.
# Default (used if this variable is absent): dnsmasq firewall network odhcpd cron uhttpd
SVC_WANTED='dnsmasq firewall network odhcpd cron uhttpd nginx'
```

> **Security note:** `/etc/config/bot.conf` contains your Matrix access token and SSH key path. It must be readable only by root (`chmod 600`). Never commit this file to version control.

---

## Usage

Once the bot is running and invited to your rooms, send commands as `MATRIX_ADMIN_USER` in the command room.

### Quick start

Send `help` or `start` to get the full command list from the bot itself.

### Command reference

#### System info

| Command | Description |
|---|---|
| `uptime` | Router uptime and load average |
| `memory` | RAM usage in MB (total / used / free) |
| `meminfo` | Detailed `/proc/meminfo` (first 5 entries) |
| `wan_ip` | Public WAN IP address (queries external services with local fallback) |

#### Network clients

| Command | Description |
|---|---|
| `clients` | Full network report: all Wi-Fi + wired clients |
| `wifi_clients` | Wi-Fi associated clients (IP, IPv6, MAC, signal, hostname) |
| `wired_clients` | Wired LAN clients from ARP table (excluding Wi-Fi MACs) |

#### Service management

| Command | Description |
|---|---|
| `restart <service>` | Restart a service from the whitelist (e.g. `restart dnsmasq`) |
| `reload nginx` | Test Nginx config, then reload if valid |

Only services listed in `SVC_WANTED` (or the default list) **and** present in `/etc/init.d/` can be restarted.

#### Network interfaces

| Command | Description |
|---|---|
| `ifup <iface>` | Bring up a UCI-defined network interface (e.g. `ifup wan`) |
| `ifdown <iface>` | Bring down a UCI-defined network interface |

#### Wi-Fi control

| Command | Description |
|---|---|
| `wifi` / `wifi_info` | Wi-Fi status (SSID, encryption, channel, rate, key) |
| `wifi_up_2_4` | Enable 2.4 GHz radio (radio0) |
| `wifi_down_2_4` | Disable 2.4 GHz radio |
| `wifi_reload_2_4` | Reload 2.4 GHz radio config |
| `wifi_up_5` | Enable 5 GHz radio (radio1) |
| `wifi_down_5` | Disable 5 GHz radio |
| `wifi_reload_5` | Reload 5 GHz radio config |

Set `WIFI_DETAILED='1'` in config for extended output (hardware chip, BSSID, country, TX power, signal/noise, OCV, client count).

#### Wake-on-LAN

| Command | Description |
|---|---|
| `wol <MAC>` | Send WOL magic packet to any MAC (format: `AA:BB:CC:DD:EE:FF`) |
| `wol_pc` | Send WOL magic packet to `MAC_PC` from config |

Requires `etherwake` package. The LAN interface is detected automatically from UCI (`network.lan.device`), falling back to `br-lan`.

---

## Operating Modes

The bot can be started in three modes. The procd init script uses `--e2ee` by default.

| Flag | Mode | Description |
|---|---|---|
| *(none / auto)* | `auto` | E2EE listener (SSH) as primary, HTTP `/sync` polling as parallel fallback |
| `--e2ee` | E2EE only | SSH + `matrix-commander-rs` listener only. No HTTP polling. |
| `--no-e2ee` | HTTP only | Matrix `/sync` long-polling only. No SSH. No E2EE. |

To change the mode, edit `/etc/init.d/matrixbot` and modify the `procd_set_param command` line:

```sh
# E2EE only (default):
procd_set_param command /bin/sh "$SCRIPT" --e2ee

# HTTP only (no external host needed):
procd_set_param command /bin/sh "$SCRIPT" --no-e2ee
```

Then restart the service:

```sh
/etc/init.d/matrixbot restart
```

---

## Running Manually / Debugging

Stop the procd service first to avoid conflicts:

```sh
/etc/init.d/matrixbot stop
```

Run in foreground with debug output:

```sh
/usr/lib/matrix/matrix_bot -d --no-e2ee
```

Flags can be combined:

```sh
/usr/lib/matrix/matrix_bot -d --e2ee
/usr/lib/matrix/matrix_bot -d          # auto mode with debug
```

Send a test message directly:

```sh
/usr/lib/matrix/matrix_send --room-id '!RoomID:server.tld' 'Hello from router'
```

With debug output:

```sh
/usr/lib/matrix/matrix_send -d --room-id '!RoomID:server.tld' 'Test message'
```

Force HTTP transport (skip SSH attempt):

```sh
/usr/lib/matrix/matrix_send --no-e2ee --room-id '!RoomID:server.tld' 'HTTP only test'
```

View live logs:

```sh
logread -f -e matrix
```

---

## Security Model

- **Single admin:** only `MATRIX_ADMIN_USER` can issue commands. Any other sender triggers an alert to `MATRIX_ROOM_ADMIN`.
- **Room whitelist:** events from any room not listed in `MATRIX_ROOM_E2EE_ID` / `MATRIX_ROOM_ID` are silently dropped.
- **Service whitelist:** only services in `SVC_WANTED` can be restarted. Service names are validated against `[a-zA-Z0-9_-]` before use.
- **Input sanitization:** all command arguments are stripped to a safe character set before use. Shell metacharacters are blocked.
- **SSH host verification:** `StrictHostKeyChecking=yes` with a dedicated `known_hosts` file. Run the host registration step before first use.
- **Credentials not exposed in process list:** the Matrix access token is written to a `chmod 600` temporary file and passed to `curl -K` rather than on the command line.
- **E2EE context awareness:** the bot tracks which rooms have encryption enabled. In HTTP mode, encrypted messages arrive as opaque `m.room.encrypted` events — the bot detects this and warns rather than silently failing.

---

## File Permissions Summary

| Path | Owner | Mode | Notes |
|---|---|---|---|
| `/usr/lib/matrix/matrix_bot` | root | `700` | Main bot process |
| `/usr/lib/matrix/matrix_send` | root | `700` | Message sender |
| `/etc/init.d/matrixbot` | root | `755` | procd init script |
| `/etc/config/bot.conf` | root | `600` | Credentials — never world-readable |
| `/etc/matrix_bot/id_ed25519` | root | `600` | SSH private key |
| `/etc/matrix_bot_known_hosts` | root | `600` | SSH host key store |

---

## Troubleshooting

**Bot does not respond to commands**
- Confirm the bot account is invited to and has joined the room.
- Confirm `MATRIX_ADMIN_USER` exactly matches the full Matrix user ID (including `@` and `:server`).
- Check `logread -e matrix` for authentication or network errors.

**`Host key verification failed`**
- The SSH host key has not been registered. Run the [SSH Key Setup](#ssh-key-setup) registration command.

**`Could not determine WAN IP`**
- All external IP services timed out and `ifstatus wan` returned no address. Check WAN connectivity on the router.

**Service restart returns `Access Denied`**
- The service is not in `SVC_WANTED`. Add it to the config and restart the bot.

**E2EE messages not received**
- The SSH connection to the `matrix-commander-rs` host failed. Run the bot with `-d` flag and check the `RUST LOG:` lines for the reason.
- Verify `matrix-commander-rs` is running and logged in on the remote host.

**`No wireless interfaces found`**
- `iwinfo` returned no results. Ensure `kmod-mac80211` and the appropriate driver are installed, and that Wi-Fi is enabled.
