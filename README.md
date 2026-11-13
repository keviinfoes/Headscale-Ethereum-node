# Guide | Home Ethereum Full Node with Headscale Remote Access (Ubuntu / Geth / Lighthouse)

*Last updated: 13-11-26*

This is a guide to running your own Ethereum full node on a small, headless home server (a NUC or similar) and reaching it from your laptop and phone, wherever you are, through a self-hosted VPN. 

**This guide assumes a working ethereum node based on [Somer Esat's Guide to Staking on Ethereum (Ubuntu/Lighthouse)](https://someresat.medium.com/guide-to-staking-on-ethereum-ubuntu-lighthouse-773f5d982e03).** 

This guide includes instructions on how to:
- Prepare your router and the NUC's BIOS for an always-on home server
- Install a non-staking full node by following the relevant parts of Somer's guide
- (Optional, recommended) Add Google Authenticator two-factor login to SSH
- Install and configure Headscale on the same machine and connect your devices to it
- Expose the node's JSON-RPC endpoint **only** to devices on the VPN
- Point wallet and dApps at your own node
- (Optional) Monitor the node with Prometheus and Grafana

## Table of Contents

- [Change Log](#change-log)
- [Acknowledgements](#acknowledgements)
- [Disclaimer](#disclaimer)
- [Prerequisites](#prerequisites)
- [Requirements](#requirements)
- [Overview](#overview)
- [Step 1 - Prepare the Router and BIOS](#step-1---prepare-the-router-and-bios)
- [Step 2 - Install the Node with Somer's Guide](#step-2---install-the-node-with-somers-guide)
- [Step 3 - Verify the Checkpoint and Sync Status](#step-3---verify-the-checkpoint-and-sync-status)
- [Step 4 - Install Headscale](#step-4---install-headscale)
- [Step 5 - Connect Your Devices](#step-5---connect-your-devices)
- [Step 6 - Expose the Node RPC over the VPN](#step-6---expose-the-node-rpc-over-the-vpn)
- [Step 7 - Use Your Node from Wallets and dApps](#step-7---use-your-node-from-wallets-and-dapps)
- [Step 8 - Monitoring with Prometheus and Grafana (Optional)](#step-8---monitoring-with-prometheus-and-grafana-optional)
- [Step 9 - Maintenance and Updates](#step-9---maintenance-and-updates)
- [Appendix A - Headscale with a Domain and TLS](#appendix-a---headscale-with-a-domain-and-tls)
- [Appendix B - Other Execution Clients and Archive Nodes](#appendix-b---other-execution-clients-and-archive-nodes)
- [Appendix C - Useful Commands](#appendix-c---useful-commands)
- [Troubleshooting](#troubleshooting)
- [Final Remarks](#final-remarks)
- [References](#references)

## Change Log

| Date *(dd-mm-yy)* | Change |
| :--- | :--- |
| *13-11-26* | Published |

## Acknowledgements

The Ethereum node part of this setup is entirely [Somer Esat's work](https://github.com/SomerEsat/ethereum-staking-guides), and the structure of this guide is modelled on his. Thanks also to the Headscale maintainers and the ethereum teams.

## Disclaimer

I am not a professional sysadmin or ginesecurity ener. This setup works for me and I wanted to write it down properly. It is a home setup, not a production deployment. Please read every command before you run it, and read the security notes. Somer's guide has its own disclaimer, which applies to the parts taken from it. Feedback and corrections are welcome.

## Prerequisites

This guide assumes basic knowledge of Ethereum, Linux and the command line. You need:

- Everything listed under **Prerequisites** in [Somer's guide](https://someresat.medium.com/guide-to-staking-on-ethereum-ubuntu-lighthouse-773f5d982e03). The parts about staking ETH do not apply.
- Ubuntu Server 24.04 LTS installed, as Somer's guide requires. Tick **Install OpenSSH server** in the installer, so you can manage the machine over SSH without a monitor.
- Admin access to your home router to set a DHCP reservation and forward **one** TCP port (for the Headscale control plane).
- A real public IPv4 address at home. 

## Requirements

Hardware: see **Requirements** in Somer's guide. Staking is not needed for a full node, so the CPU and RAM recommendations there are more than enough. Storage is the real constraint: use an NVMe SSD of at least 2 TB, preferably 4 TB.

Software installed by this guide, on top of Somer's:

| Software | Version | Releases |
| :--- | :--- | :--- |
| Headscale | 0.29.7 | https://github.com/juanfont/headscale/releases |
| Tailscale | latest stable | https://tailscale.com/download |

## Overview

Everything runs on the one machine. The first two services come from Somer's guide; this guide only changes their flags.

| Service | Installed by | Listens on | Reachable from |
| :--- | :--- | :--- | :--- |
| `geth` | Somer, Step 8 | 30303 tcp/udp (p2p), 8551 tcp (engine API), 8545 tcp (RPC, added in Step 6) | p2p: internet. Engine API: localhost. RPC: VPN only. |
| `lighthousebeacon` | Somer, Step 11 | 9000 tcp/udp + 9001 udp (p2p), 5052 tcp (beacon API) | p2p: internet. Beacon API: localhost (optionally VPN). |
| `headscale` | This guide, Step 4 | 8080 tcp | internet (port-forwarded) |
| `tailscaled` | This guide, Step 5 | 41641 udp | peers |
| `grafana-server` (optional) | This guide, Step 8 | 3000 tcp | VPN only |

How the pieces fit together:

1. **Geth and Lighthouse** form the Ethereum node, exactly as in Somer's guide but without a validator.
2. **Headscale** is a tiny coordination server. It hands out VPN IPs in `100.64.0.0/10` and public keys. The actual VPN traffic flows peer-to-peer (WireGuard) between your devices and the NUC; 
3. **Tailscale** is the client. It runs on the NUC too, so the NUC gets a VPN IP.
4. The **firewall** (UFW, set up in Somer's guide) gets one extra rule: the Geth RPC port is allowed from the VPN range only.

## Step 1 - Prepare the Router and BIOS

Somer's guide starts from a running Ubuntu Server. This step covers the two things a home server needs on top of that.

### Give the NUC a fixed LAN address

The port forward in Step 4 needs the NUC to keep the same address on your home network. In your router, create a **DHCP reservation** for the NUC. Find its current address and MAC address on the NUC with:

```
ip -br addr
ip -br link
```

The reserved address is `<NUC_LAN_IP>` in the rest of this guide.

### Check it powers back on after an outage

In the NUC BIOS (`F2` at boot), check that **Power > After Power Failure** is not set to **Stay Off**. Otherwise the node stays down after a power cut until someone presses the power button.

## Step 2 - Install the Node with Somer's Guide

Open [Somer's Guide to Staking on Ethereum (Ubuntu/Lighthouse)](https://someresat.medium.com/guide-to-staking-on-ethereum-ubuntu-lighthouse-773f5d982e03) and work through it with the table below. Everything marked **Skip** is only needed for staking.

| Somer's step | Do it? | Notes |
| :--- | :--- | :--- |
| Step 1 — Generate Staking Data | **Skip** | No validator. |
| Step 2 — Create the Server User | Do | |
| Step 3 — Update the Server | Do | |
| Step 4 — Secure the Server | Do | Write down your SSH port; it is `<YourSSHPortNumber>` in this guide. Forward the p2p ports 30303 and 9000 (and UDP 9001) in your router as the notes there describe. **Never** forward your SSH port or 8545. |
| Step 5 — Create a Swap Space | Do | Optional, as Somer notes. |
| Step 6 — Configure Timekeeping | Do | |
| Step 7 — Generate Client Authentication Secret | Do | Creates `/var/lib/jwtsecret/jwt.hex`. |
| Step 8 — Configure the Execution Client | Do | This guide's examples use **Geth**. Any of Somer's five clients works; see [Appendix B](#appendix-b---other-execution-clients-and-archive-nodes) for the RPC flags of the others. |
| Step 9 — Install the Lighthouse Consensus Client | Do | |
| Step 10 — Import the Validator Keys | **Skip** | No validator. |
| Step 11 — Configure the Beacon Node Service | Do | For the checkpoint sync URL, pick one from https://eth-clients.github.io/checkpoint-sync-endpoints/, for example `https://mainnet.checkpoint.sigp.io`. |
| Step 12 — Configure the Validator Service | **Skip** | No validator. |
| Step 13 — Fund the Validator Keys | **Skip** | Do **not** deposit anything. |

When you are done, `geth` and `lighthousebeacon` are running as services and syncing. The rest of this guide assumes the paths and service names from Somer's guide.

### Recommended: two-factor SSH login with Google Authenticator

This is optional, but recommended. Somer's guide secures SSH with a custom port and the firewall. This adds a one-time code from an authenticator app on your phone (Google Authenticator or any other TOTP app) to every SSH login, so a leaked password or key alone is not enough to get in.

> **Warning:** A mistake here can lock you out of SSH. Keep your current SSH session open until you have logged in successfully from a second terminal.

Install the PAM module:

```
sudo apt install -y libpam-google-authenticator
```

Generate your secret. Run this as your own user, **not** with `sudo`:

```
google-authenticator -t -d -f -r 3 -R 30 -w 3
```

| Flag | What it does |
| :--- | :--- |
| `-t` | Time-based codes (TOTP). |
| `-d` | Each code can be used only once. |
| `-f` | Writes the config to `~/.google_authenticator` without asking. |
| `-r 3 -R 30` | At most 3 login attempts per 30 seconds. |
| `-w 3` | Accepts the previous and next code too, to allow for small clock differences. |

It prints a QR code. Scan it with your authenticator app, and enter the code the app shows to confirm. It also prints **emergency scratch codes**. Store them somewhere safe, outside the NUC: each one gets you in once if you lose your phone.

Tell PAM to ask for the code on SSH logins:

```
sudo nano /etc/pam.d/sshd
```

Add this line at the end of the file. Save and exit.

```
auth required pam_google_authenticator.so
```

Tell SSH to use it. Create a drop-in config file:

```
sudo nano /etc/ssh/sshd_config.d/10-google-authenticator.conf
```

If you log in with a **password** (the default after Somer's guide), paste:

```
UsePAM yes
KbdInteractiveAuthentication yes
PasswordAuthentication no
AuthenticationMethods keyboard-interactive
```

SSH then asks for your password and then the verification code. `PasswordAuthentication no` is correct here: the password is still asked, but through the same prompt as the code, so it cannot be used without it.

If you log in with an **SSH key**, paste this instead:

```
UsePAM yes
KbdInteractiveAuthentication yes
PasswordAuthentication no
AuthenticationMethods publickey,keyboard-interactive
```

With a key you probably don't want to be asked for your password as well. To ask only for the code, put a `#` in front of the `@include common-auth` line in `/etc/pam.d/sshd`.

Check the config and restart SSH:

```
sudo sshd -t
sudo systemctl restart ssh
```

`sshd -t` prints nothing if the config is valid. Now open a **second** terminal and log in:

```
ssh -p <YourSSHPortNumber> <yourusername>@<NUC_LAN_IP>
```

Here `<yourusername>` is the user you created in Somer's Step 2. You should be asked for a `Verification code`. Only close your first session once this works.

> **Note:** The codes are time-based, so the NUC's clock must be right. Somer's Step 6 (timekeeping) takes care of that. If codes are suddenly rejected, check `timedatectl` first.

## Step 3 - Verify the Checkpoint and Sync Status

Somer's guide does not cover verifying the checkpoint. Checkpoint sync means trusting the provider for the starting point, so check it against an independent source.

Install `jq`, which the commands below use:

```
sudo apt install -y jq
```

Ask your node for its finalized checkpoint:

```
curl -s http://127.0.0.1:5052/eth/v1/beacon/states/finalized/finality_checkpoints | jq .data.finalized
```

Compare with a second provider from the [community list](https://eth-clients.github.io/checkpoint-sync-endpoints/), for example:

```
curl -s https://beaconstate.ethstaker.cc/eth/v1/beacon/states/finalized/finality_checkpoints | jq .data.finalized
```

The `epoch` and `root` must match. If finalization moved on between the two calls, run the first one again. If they still do not match, stop `lighthousebeacon`, remove `/var/lib/lighthouse/beacon` and sync again from a different URL.

Check sync progress at any time:

```
curl -s http://127.0.0.1:5052/eth/v1/node/syncing | jq .data.is_syncing
sudo journalctl -fu geth
```

Lighthouse is synced when `is_syncing` is `false`. Geth is synced when it logs `Imported new potential chain segment` for every new block. Geth takes several hours to a day. You can carry on with the next steps while it syncs.

## Step 4 - Install Headscale

Headscale is the self-hosted control server. Your devices' Tailscale clients will log in to it instead of to Tailscale's cloud.

### Create the service user and directories

```
sudo useradd \
  --create-home \
  --home-dir /var/lib/headscale/ \
  --system \
  --user-group \
  --shell /usr/sbin/nologin \
  headscale
sudo mkdir -p /etc/headscale
```

### Download and verify Headscale

Go to https://github.com/juanfont/headscale/releases and note the latest version. Download the Linux amd64 binary and the checksums file:

```
cd ~
wget https://github.com/juanfont/headscale/releases/download/v0.29.7/headscale_0.29.7_linux_amd64
wget https://github.com/juanfont/headscale/releases/download/v0.29.7/checksums.txt
sha256sum --check --ignore-missing checksums.txt
```

Expect `headscale_0.29.7_linux_amd64: OK`. Install:

```
chmod +x headscale_0.29.7_linux_amd64
sudo mv headscale_0.29.7_linux_amd64 /usr/local/bin/headscale
rm checksums.txt
headscale version
```

> **Note:** Headscale also ships a `.deb` package that installs the user, directories and service for you. This guide uses the plain binary in `/usr/local/bin`, the same way Somer's guide installs Geth and Lighthouse. Do not mix the two.

### Create the configuration

Start from the example config for the release branch you installed:

```
sudo wget https://raw.githubusercontent.com/juanfont/headscale/release-branch/0.29/config-example.yaml \
  -O /etc/headscale/config.yaml
sudo nano /etc/headscale/config.yaml
```

Find and set these keys (search with `Ctrl+W`). Everything else can stay at its default.

| Key | Value | Why |
| :--- | :--- | :--- |
| `server_url` | `http://<PUBLIC_HOME_IP>:8080` | What clients connect to. Must be reachable from the internet. |
| `listen_addr` | `0.0.0.0:8080` | Listen on all interfaces so the port forward reaches it. |
| `metrics_listen_addr` | `127.0.0.1:9090` | Leave as is (localhost only). |
| `database.type` | `sqlite` | Default. |
| `database.sqlite.path` | `/var/lib/headscale/db.sqlite` | Default. |
| `unix_socket` | `/var/run/headscale/headscale.sock` | Default. The CLI talks to the server through this socket. |
| `dns.base_domain` | `vpn.internal` | MagicDNS suffix. Must **not** be the same as the `server_url` domain. Any made-up internal name is fine. |

Replace `<PUBLIC_HOME_IP>` with your home's public IP address. Find it on the NUC with:

```
curl -4 ifconfig.me
```

> **Note:** Most home internet connections have a public IP that changes occasionally. See [Appendix A](#appendix-a---headscale-with-a-domain-and-tls) for using a domain name with dynamic DNS instead. This guide first gets everything working with the raw IP so there is less to debug at once.

Press `Ctrl+X` then `Y` then `Enter` to save and exit. Set permissions so only the headscale user can read the config and data:

```
sudo chown -R headscale:headscale /etc/headscale /var/lib/headscale
sudo chmod 750 /etc/headscale /var/lib/headscale
sudo chmod 640 /etc/headscale/config.yaml
```

> **Security note:** With a raw IP and plain HTTP, the control plane traffic (node registration, key exchange, pre-auth keys) crosses the internet unencrypted. The WireGuard tunnels themselves are still encrypted end to end. For a HTTPS setup see [Appendix A](#appendix-a---headscale-with-a-domain-and-tls).

### Create the systemd service

This is the hardened unit file that Headscale ships with, adjusted for the binary path.

```
sudo nano /etc/systemd/system/headscale.service
```

Paste the following. Save and exit.

```
[Unit]
After=network.target
Description=headscale coordination server for Tailscale
X-Restart-Triggers=/etc/headscale/config.yaml

[Service]
Type=simple
User=headscale
Group=headscale
ExecStart=/usr/local/bin/headscale serve
ExecReload=/usr/bin/kill -HUP $MAINPID
Restart=always
RestartSec=5

WorkingDirectory=/var/lib/headscale
ReadWritePaths=/var/lib/headscale

AmbientCapabilities=CAP_NET_BIND_SERVICE CAP_CHOWN
CapabilityBoundingSet=CAP_NET_BIND_SERVICE CAP_CHOWN
LockPersonality=true
NoNewPrivileges=true
PrivateDevices=true
PrivateMounts=true
PrivateTmp=true
ProcSubset=pid
ProtectClock=true
ProtectControlGroups=true
ProtectHome=true
ProtectHostname=true
ProtectKernelLogs=true
ProtectKernelModules=true
ProtectKernelTunables=true
ProtectProc=invisible
ProtectSystem=strict
RemoveIPC=true
RestrictAddressFamilies=AF_INET AF_INET6 AF_UNIX
RestrictNamespaces=true
RestrictRealtime=true
RestrictSUIDSGID=true
RuntimeDirectory=headscale
RuntimeDirectoryMode=0750
StateDirectory=headscale
StateDirectoryMode=0750
SystemCallArchitectures=native
SystemCallFilter=@chown
SystemCallFilter=@system-service
SystemCallFilter=~@privileged
UMask=0077

[Install]
WantedBy=multi-user.target
```

The important lines: `RuntimeDirectory=headscale` creates `/var/run/headscale` for the CLI socket on every start, `ProtectSystem=strict` makes the whole filesystem read-only to the service except `/var/lib/headscale`, and `CAP_NET_BIND_SERVICE` lets it bind port 443 later if you do Appendix A.

### Start Headscale

```
sudo systemctl daemon-reload
sudo systemctl start headscale
sudo systemctl status headscale
```

It should say `active (running)`. If it exits immediately, the config has a problem: `sudo journalctl -u headscale -n 50` tells you which line. Enable it to start on reboot:

```
sudo systemctl enable headscale
```

Check that the CLI can talk to the server over the socket, and create a Headscale **user** (a user groups devices; one is enough for a home setup):

```
sudo headscale users create home
sudo headscale users list
```

Note the numeric `ID` of the user; it is `<USER_ID>` from here on (usually `1`).

### Firewall and router

Add the control-plane port to the UFW rules from Somer's Step 4:

```
sudo ufw allow 8080/tcp comment 'Headscale control plane'
```

In your router, forward the port to the NUC:

| Setting | Value |
| :--- | :--- |
| Protocol | TCP |
| External port | 8080 |
| Internal port | 8080 |
| Internal IP | `<NUC_LAN_IP>` |

Test from **outside** your network, for example from your phone on its mobile network, by opening `http://<PUBLIC_HOME_IP>:8080/health`. You should see `{"status":"pass"}`. From the NUC itself:

```
curl http://127.0.0.1:8080/health
```

> **Note:** Many routers cannot reach their own public IP from inside the LAN ("NAT hairpinning"). If the test fails from your laptop on your home network but works from your phone's mobile network, that is why, and it is fine. Tailscale clients handle it.

## Step 5 - Connect Your Devices

### 5.1 Connect the NUC itself

The NUC needs the Tailscale client so it gets a VPN IP. Install it with the official script, which adds Tailscale's apt repository and package:

```
curl -fsSL https://tailscale.com/install.sh | sh
```

Point it at your Headscale server:

```
sudo tailscale up --login-server http://<PUBLIC_HOME_IP>:8080
```

It prints a URL ending in a registration ID, like `http://<PUBLIC_HOME_IP>:8080/register/<AUTH_ID>`. You do not need to open it. Instead, in another SSH session on the NUC, register the node to your user:

```
sudo headscale auth register --user home --auth-id <AUTH_ID>
```

Back in the first session, `tailscale up` finishes with `Success.` Get the NUC's VPN IP:

```
tailscale ip -4
```

Write this address down. Wherever the rest of this guide says `<NUC_VPN_IP>`, use this address. It stays the same for the life of the node. Check the node list:

```
sudo headscale nodes list
```

### 5.2 Create a pre-auth key for other devices

Rather than approving every device by hand, create a short-lived key:

```
sudo headscale preauthkeys create --user <USER_ID> --expiration 1h
```

It prints a key like `hskey-auth-...`. Use it within the hour. One key per device is good hygiene; create another one when you need it.

### 5.3 macOS

With Homebrew (CLI and daemon, no GUI needed):

```
brew install tailscale
sudo brew services start tailscale
tailscale up --login-server http://<PUBLIC_HOME_IP>:8080 --authkey hskey-auth-<KEY>
tailscale ip -4
tailscale ping <NUC_VPN_IP>
```

If you prefer the App Store or standalone Tailscale app, open `http://<PUBLIC_HOME_IP>:8080/apple` on the Mac. Headscale serves instructions there for pointing the GUI app at a custom control server.

### 5.4 iPhone / iPad and Android

- **iOS:** install Tailscale from the App Store. Open it, go to **Settings** (gear) and tap **Alternate coordination server URL** (if you do not see it, open `http://<PUBLIC_HOME_IP>:8080/apple` on the phone for the current steps). Enter `http://<PUBLIC_HOME_IP>:8080`, then log in. Safari opens the registration page with an auth ID. Register it from the NUC exactly like in 5.1: `sudo headscale auth register --user home --auth-id <AUTH_ID>`.
- **Android:** install Tailscale from the Play Store. Open it, tap the three-dot menu **three times** quickly until **Change server** appears. Enter `http://<PUBLIC_HOME_IP>:8080`, log in, and register the auth ID from the NUC.

### 5.5 Linux or Windows laptop

Linux is the same as the NUC: install, then `sudo tailscale up --login-server http://<PUBLIC_HOME_IP>:8080 --authkey hskey-auth-<KEY>`. Windows: install Tailscale, then open `http://<PUBLIC_HOME_IP>:8080/windows` for the registry setting that points it at your server, and log in.

### 5.6 Verify

On the NUC:

```
sudo headscale nodes list
```

Every device should be listed with an IP in `100.64.0.0/10` and `online`. From any device, `tailscale ping <NUC_VPN_IP>` should answer within a few tries, ideally with `direct` (peer-to-peer) rather than `via DERP` (relayed through Tailscale's public relay servers, which is slower but still encrypted).

You can now also SSH into the NUC from anywhere, using the user you created in Somer's Step 2 as `<yourusername>`: `ssh -p <YourSSHPortNumber> <yourusername>@<NUC_VPN_IP>`.

## Step 6 - Expose the Node RPC over the VPN

The Geth service from Somer's guide only has the engine API for Lighthouse; it does not serve JSON-RPC to wallets. We add an HTTP RPC endpoint and open it to the VPN range and to nothing else.

### Firewall: only the VPN range may reach the RPC

```
sudo ufw allow from 100.64.0.0/10 to any port 8545 proto tcp comment 'Geth RPC via VPN'
```

Optional, for tools that use the beacon API:

```
sudo ufw allow from 100.64.0.0/10 to any port 5052 proto tcp comment 'Lighthouse API via VPN'
```

### Add the RPC flags to Geth

Edit the Geth service from Somer's Step 8:

```
sudo nano /etc/systemd/system/geth.service
```

Add the `--http` lines to the end of `ExecStart`. The result looks like this, with the five new lines at the bottom:

```
ExecStart=/usr/local/bin/geth \
  --mainnet \
  --datadir /var/lib/geth \
  --authrpc.jwtsecret /var/lib/jwtsecret/jwt.hex \
  --db.engine pebble \
  --http \
  --http.addr 0.0.0.0 \
  --http.port 8545 \
  --http.api eth,net,web3,txpool \
  --http.vhosts "*" \
  --http.corsdomain "*"
```

Mind the trailing backslash: every line except the last needs one.

| Flag | What it does |
| :--- | :--- |
| `--http` | Enables the JSON-RPC HTTP server. |
| `--http.addr 0.0.0.0` | Listen on all interfaces, so traffic arriving on the Tailscale interface is accepted. UFW makes sure only VPN sources get through. |
| `--http.port 8545` | Fixed RPC port, the one opened in UFW. |
| `--http.api eth,net,web3,txpool` | Which API namespaces to expose. **Never** add `admin`, `debug` or `personal`. |
| `--http.vhosts "*"` | Accept any `Host` header, such as `<NUC_VPN_IP>:8545`. Without it Geth answers with `invalid host specified` and MetaMask shows a "Vhost security error". |
| `--http.corsdomain "*"` | Lets browser wallets (MetaMask, Rabby) on any origin query the node. Required for browser extensions. |

Reload and restart:

```
sudo systemctl daemon-reload
sudo systemctl restart geth
sudo ss -tlnp | grep 8545
```

The last command should show `0.0.0.0:8545`.

If you also opened 5052, add `--http-address 0.0.0.0` to the Lighthouse service from Somer's Step 11 in the same way, then reload and restart it:

```
sudo nano /etc/systemd/system/lighthousebeacon.service
sudo systemctl daemon-reload
sudo systemctl restart lighthousebeacon
sudo ss -tlnp | grep 5052
```

The last command should show `0.0.0.0:5052`.

> **Security note:** These flags are only safe because UFW restricts port 8545 to `100.64.0.0/10`. If that rule is ever removed, the RPC is open to your LAN and, if you ever forwarded the port, the internet. A stricter alternative is `--http.addr <NUC_VPN_IP>`, which binds only the VPN interface. The downside is that Geth then fails to start if it comes up before `tailscaled` has brought the interface up, so you would need `After=tailscaled.service` and `Requires=tailscaled.service` in the unit.

### Test from a device on the VPN

From your laptop (connected to the VPN):

```
curl -s -X POST -H "Content-Type: application/json" \
  --data '{"jsonrpc":"2.0","method":"eth_blockNumber","params":[],"id":1}' \
  http://<NUC_VPN_IP>:8545
```

You should get `{"jsonrpc":"2.0","id":1,"result":"0x..."}` with the current block number in hex. Convert it (`printf "%d\n" 0x...`) and compare with https://etherscan.io. While Geth is still syncing, the number will lag behind.

Now test that the RPC is **not** reachable from the LAN: disconnect the laptop from the VPN (`tailscale down`) and run the same `curl` against `http://<NUC_LAN_IP>:8545`. It must hang and time out. If it answers, your UFW rule is wrong. Reconnect with `tailscale up`.

## Step 7 - Use Your Node from Wallets and dApps

### MetaMask / Rabby

Add a network manually, or edit the existing Ethereum Mainnet entry's RPC URL:

| Field | Value |
| :--- | :--- |
| Network name | `Ethereum (home node)` |
| RPC URL | `http://<NUC_VPN_IP>:8545` |
| Chain ID | `1` |
| Currency symbol | `ETH` |
| Block explorer | `https://etherscan.io` |

MetaMask Mobile works the same way as long as the Tailscale app on the phone is connected.

> **Note:** MetaMask may warn that the RPC URL is `http` rather than `https`. Inside a WireGuard tunnel the traffic is already encrypted, so that is acceptable here.

### Redirect dApp traffic to your node (Requestly)

Most dApps talk to their own RPC provider (Infura, Alchemy, etc.) directly from the browser, bypassing your wallet's RPC setting. To route those calls through your node:

1. Install the Requestly browser extension: [Chrome](https://requestly.com/downloads/chrome/) (also for Brave and Edge) or [Firefox](https://requestly.com/downloads/firefox/). The free plan is enough for this one rule.
2. Create a [**Redirect** rule](https://docs.requestly.com/general/http-rules/rule-types/redirect-rule). Source: URL **matches regex** `.*(rpc|eth|mainnet|infura|alchemy|ankr|node|chain).*`. Destination: `http://<NUC_VPN_IP>:8545`.
3. Open a dApp, open the browser's Network tab, and confirm the JSON-RPC `POST` requests go to `<NUC_VPN_IP>:8545`.

The regex is deliberately broad. Expect to add exceptions, since it also catches unrelated URLs that happen to contain those words.

> **Note:** Catching and redirecting is best effort, not a guarantee. Many dApps use their own custom RPC endpoints, with URLs that do not match the regex, so those calls still go to the dApp's provider. Check the Network tab for each dApp you use. For full control you need to block every RPC connection a dApp makes on its own and forward only to your node.

> **Note:** A full node serves recent state and all blocks and logs, which covers wallets and the large majority of dApps. Calls that need historical **state** (e.g. `eth_call` at an old block, `debug_traceTransaction` far in the past) need an archive node. See [Appendix B](#appendix-b---other-execution-clients-and-archive-nodes).

## Step 8 - Monitoring with Prometheus and Grafana (Optional)

Somer's guide does not include monitoring. This step adds Prometheus to collect metrics, Node Exporter for CPU/disk/memory, and Grafana for dashboards, reachable only over the VPN.

### Enable client metrics

Add these flags to the end of `ExecStart` in the Geth service from Somer's Step 8 (remember the backslash on the line before):

```
  --metrics \
  --metrics.addr 127.0.0.1 \
  --metrics.port 6060
```

And these to the Lighthouse service from Somer's Step 11:

```
  --metrics \
  --metrics-address 127.0.0.1 \
  --metrics-port 5054
```

```
sudo systemctl daemon-reload
sudo systemctl restart geth lighthousebeacon
```

### Prometheus

```
sudo useradd --no-create-home --shell /bin/false prometheus
sudo mkdir -p /etc/prometheus /var/lib/prometheus
```

Get the latest `linux-amd64` release from https://github.com/prometheus/prometheus/releases (3.16.0 at time of writing):

```
cd ~
wget https://github.com/prometheus/prometheus/releases/download/v3.16.0/prometheus-3.16.0.linux-amd64.tar.gz
wget https://github.com/prometheus/prometheus/releases/download/v3.16.0/sha256sums.txt
sha256sum --check --ignore-missing sha256sums.txt
tar xvf prometheus-3.16.0.linux-amd64.tar.gz
sudo cp prometheus-3.16.0.linux-amd64/{prometheus,promtool} /usr/local/bin/
sudo chown prometheus:prometheus /usr/local/bin/{prometheus,promtool}
rm -rf prometheus-3.16.0.linux-amd64* sha256sums.txt
```

Configuration:

```
sudo nano /etc/prometheus/prometheus.yml
```

```
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['127.0.0.1:9095']
  - job_name: node_exporter
    static_configs:
      - targets: ['127.0.0.1:9100']
  - job_name: geth
    metrics_path: /debug/metrics/prometheus
    static_configs:
      - targets: ['127.0.0.1:6060']
  - job_name: lighthouse_beacon
    static_configs:
      - targets: ['127.0.0.1:5054']
```

```
sudo chown -R prometheus:prometheus /etc/prometheus /var/lib/prometheus
sudo nano /etc/systemd/system/prometheus.service
```

```
[Unit]
Description=Prometheus
Wants=network-online.target
After=network-online.target

[Service]
User=prometheus
Group=prometheus
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/prometheus \
    --config.file=/etc/prometheus/prometheus.yml \
    --storage.tsdb.path=/var/lib/prometheus \
    --storage.tsdb.retention.time=30d \
    --web.listen-address=127.0.0.1:9095

[Install]
WantedBy=multi-user.target
```

> **Note:** Prometheus normally listens on 9090, but Headscale's metrics endpoint already uses 9090 on localhost, so we use 9095.

```
sudo systemctl daemon-reload
sudo systemctl enable --now prometheus
sudo systemctl status prometheus
```

### Node Exporter

```
sudo useradd --no-create-home --shell /bin/false node_exporter
cd ~
wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/node_exporter-1.12.1.linux-amd64.tar.gz
wget https://github.com/prometheus/node_exporter/releases/download/v1.12.1/sha256sums.txt
sha256sum --check --ignore-missing sha256sums.txt
tar xvf node_exporter-1.12.1.linux-amd64.tar.gz
sudo cp node_exporter-1.12.1.linux-amd64/node_exporter /usr/local/bin/
sudo chown node_exporter:node_exporter /usr/local/bin/node_exporter
rm -rf node_exporter-1.12.1.linux-amd64* sha256sums.txt
sudo nano /etc/systemd/system/node_exporter.service
```

```
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
Restart=always
RestartSec=5
ExecStart=/usr/local/bin/node_exporter --web.listen-address=127.0.0.1:9100

[Install]
WantedBy=multi-user.target
```

```
sudo systemctl daemon-reload
sudo systemctl enable --now node_exporter
curl -s http://127.0.0.1:9095/api/v1/targets | jq '.data.activeTargets[] | {job: .labels.job, health: .health}'
```

All four targets should be `up`.

### Grafana

Install from Grafana's apt repository:

```
sudo apt install -y apt-transport-https software-properties-common
sudo mkdir -p /etc/apt/keyrings
wget -q -O - https://apt.grafana.com/gpg.key | gpg --dearmor | sudo tee /etc/apt/keyrings/grafana.gpg > /dev/null
echo "deb [signed-by=/etc/apt/keyrings/grafana.gpg] https://apt.grafana.com stable main" | sudo tee /etc/apt/sources.list.d/grafana.list
sudo apt update
sudo apt install -y grafana
sudo systemctl enable --now grafana-server
```

Allow it from the VPN only:

```
sudo ufw allow from 100.64.0.0/10 to any port 3000 proto tcp comment 'Grafana via VPN'
```

From a device on the VPN open `http://<NUC_VPN_IP>:3000`. Log in with `admin` / `admin` and set a new password. Then:

1. **Connections > Data sources > Add data source > Prometheus**, URL `http://127.0.0.1:9095`, **Save & test**.
2. **Dashboards > New > Import** and import:
   - Node Exporter Full, dashboard ID `1860`.
   - Geth: the dashboard JSON from https://github.com/ethereum/go-ethereum/tree/master/grafana (or search Grafana.com for "Geth").
   - Lighthouse: the beacon node dashboard from https://github.com/sigp/lighthouse-metrics.

## Step 9 - Maintenance and Updates

### Geth and Lighthouse

Update them exactly as described in Somer's guide: **Appendix C — Updating Geth** and **Appendix F — Updating Lighthouse**. Hard forks **require** updating before the fork block. Your extra flags in the service files are not affected by an update.

For log size, see Somer's **Appendix I — Manage Systemd Journal Logs**.

### Headscale

Read the [changelog](https://github.com/juanfont/headscale/blob/main/CHANGELOG.md) first. Minor versions sometimes change config keys and need a database migration, which happens automatically on start but is one-way. **Back up first:**

```
sudo systemctl stop headscale
sudo cp -a /var/lib/headscale /var/lib/headscale.bak-$(date +%F)
sudo cp /etc/headscale/config.yaml /etc/headscale/config.yaml.bak-$(date +%F)
```

Then download, checksum and move the new binary over `/usr/local/bin/headscale` as in Step 4, diff your config against the new `config-example.yaml` for that release branch, and start:

```
sudo systemctl start headscale
sudo journalctl -u headscale -n 50
sudo headscale nodes list
```

### Tailscale

Tailscale was installed from its apt repository, so the regular update from Somer's Step 3 updates it too:

```
sudo apt update && sudo apt upgrade -y
```

### Node key expiry

Headscale expires device keys after 180 days by default. A device that drops off the VPN with `tailscale status` saying it needs to log in again just needs `tailscale up --login-server ...` again, which reuses its registration. You can also see and extend expiry on the server: `sudo headscale nodes list` and `sudo headscale nodes expire --help`.

### Backups

The things you cannot regenerate:

- `/etc/headscale/config.yaml`
- `/var/lib/headscale/` (database, noise private key). Losing it means re-registering every device.

Chain data is not worth backing up; it resyncs.

## Appendix A - Headscale with a Domain and TLS

Once everything works on `http://<PUBLIC_HOME_IP>:8080`, upgrade the control plane to HTTPS. You need a domain (any cheap one) and a DNS record pointing to your home IP.

1. **Dynamic DNS.** Create an `A` record `vpn.example.com` and keep it updated when your home IP changes. Most routers support DynDNS / No-IP / DuckDNS, or run `ddclient` on the NUC. Some registrars (Cloudflare, Porkbun, deSEC) have APIs with ready-made update scripts.
2. **Router.** Forward TCP **80** and TCP **443** to `<NUC_LAN_IP>`. Port 80 is needed for the Let's Encrypt HTTP-01 challenge. You can remove the 8080 forward afterwards.
3. **Firewall.** `sudo ufw allow 80/tcp comment 'ACME'` and `sudo ufw allow 443/tcp comment 'Headscale HTTPS'`. Later `sudo ufw delete allow 8080/tcp`.
4. **Config.** In `/etc/headscale/config.yaml` set:

   | Key | Value |
   | :--- | :--- |
   | `server_url` | `https://vpn.example.com` |
   | `listen_addr` | `0.0.0.0:443` |
   | `tls_letsencrypt_hostname` | `vpn.example.com` |
   | `acme_email` | your email |
   | `tls_letsencrypt_challenge_type` | `HTTP-01` |
   | `tls_letsencrypt_listen` | `":http"` |
   | `dns.base_domain` | something other than `example.com`, e.g. `vpn.internal` |

   The service unit from Step 4 already has `CAP_NET_BIND_SERVICE`, so the headscale user may bind ports 80 and 443.
5. **Restart** with `sudo systemctl restart headscale`, watch `sudo journalctl -fu headscale` for the certificate being issued, then open `https://vpn.example.com/health`.
6. **Re-point clients.** On each device run `tailscale up --login-server https://vpn.example.com` (phones: change the server URL in the app). Existing registrations carry over.

If you would rather not forward port 80, use a reverse proxy such as Caddy in front of Headscale, or the `TLS-ALPN-01` challenge type. See https://headscale.net/stable/ref/tls/.

## Appendix B - Other Execution Clients and Archive Nodes

Somer's Step 8 covers Besu, Erigon, Geth, Nethermind and Reth. Whichever you install, Step 6 of this guide works the same way: add the HTTP RPC flags to that client's service file. The flags differ per client:

| Client | Flags to add |
| :--- | :--- |
| Geth | `--http --http.addr 0.0.0.0 --http.port 8545 --http.api eth,net,web3,txpool --http.vhosts "*" --http.corsdomain "*"` |
| Erigon | `--http --http.addr 0.0.0.0 --http.port 8545 --http.api eth,net,web3,txpool,erigon,trace --http.vhosts "*" --http.corsdomain "*"` |
| Reth | `--http --http.addr 0.0.0.0 --http.port 8545 --http.api eth,net,web3,txpool,trace --http.corsdomain "*"` (no vhosts flag) |
| Nethermind | `--JsonRpc.Enabled true --JsonRpc.Host 0.0.0.0 --JsonRpc.Port 8545` |
| Besu | `--rpc-http-enabled=true --rpc-http-host=0.0.0.0 --rpc-http-port=8545 --rpc-http-cors-origins="*" --host-allowlist="*"` |

Verify exact flags against `--help` for the version you install, and check whether Somer's service file already sets any of them.

If you need historical state (tracing old transactions, `eth_call` at arbitrary old blocks), you need an archive node. Geth's archive mode is not practical for home use. Erigon and Reth can run as archive nodes in roughly 3 TB; see their documentation for the archive settings.

## Appendix C - Useful Commands

| Task | Command |
| :--- | :--- |
| Service status | `sudo systemctl status <service>` |
| Follow logs | `sudo journalctl -fu <service>` |
| Last 100 log lines | `sudo journalctl -u <service> -n 100 --no-pager` |
| Firewall rules | `sudo ufw status numbered` |
| Delete a firewall rule | `sudo ufw delete <number>` |
| Headscale users / nodes | `sudo headscale users list`, `sudo headscale nodes list` |
| New pre-auth key | `sudo headscale preauthkeys create --user <USER_ID> --expiration 1h` |
| Remove a device | `sudo headscale nodes delete --identifier <ID>` |
| Tailscale state on a device | `tailscale status`, `tailscale ip -4`, `tailscale netcheck` |
| Geth peers | `curl -s -X POST -H "Content-Type: application/json" --data '{"jsonrpc":"2.0","method":"net_peerCount","params":[],"id":1}' http://127.0.0.1:8545` |
| Lighthouse peers | `curl -s http://127.0.0.1:5052/eth/v1/node/peer_count \| jq` |
| Open ports | `sudo ss -tulnp` |

## Troubleshooting

For problems with installing or syncing Geth and Lighthouse themselves, use the support channels listed in Somer's guide. This section covers the parts this guide adds.

**Headscale will not start.** `sudo journalctl -u headscale -n 50`. Almost always a YAML indentation error or a `dns.base_domain` that overlaps `server_url`. Permissions: `ls -la /etc/headscale /var/lib/headscale` must show `headscale:headscale`.

**`headscale` CLI says it cannot connect / permission denied.** The CLI uses the unix socket; run it with `sudo`. Check `/var/run/headscale/headscale.sock` exists (created by `RuntimeDirectory=` when the service starts).

**Phone cannot reach the control server.** Open `http://<PUBLIC_HOME_IP>:8080/health` on your phone using its mobile network. If it fails, check the router port forward, `sudo ufw status` for 8080, and that `listen_addr` is `0.0.0.0:8080` not `127.0.0.1:8080`. Also check for CGNAT (see Prerequisites).

**Devices connect but `tailscale ping` only works `via DERP`.** Direct connections need UDP 41641 outbound (almost always open). It still works, just relayed. `tailscale netcheck` shows what is blocked.

**`curl` to `<NUC_VPN_IP>:8545` hangs.** On the NUC: `sudo ss -tlnp | grep 8545` must show `0.0.0.0:8545`; `sudo ufw status` must show the `100.64.0.0/10` rule; `tailscale status` on both ends must show the peer. Try `curl http://127.0.0.1:8545` on the NUC itself to separate Geth problems from network problems.

**Geth will not start after editing the service.** Usually a missing trailing backslash in `ExecStart`. `sudo journalctl -u geth -n 30` shows the flag it choked on.

**MetaMask: "Vhost security error" or `invalid host specified`.** `--http.vhosts "*"` is missing from the Geth service, or you did not `daemon-reload` + `restart` after editing.

**MetaMask: CORS error in the console.** `--http.corsdomain "*"` is missing.

## Final Remarks

That's it! You now run your own piece of Ethereum infrastructure and can start using it with your wallet and dapps.

A few parting thoughts:

- Keep an eye on hard fork announcements at https://blog.ethereum.org and on the change log of Somer's guide. Updating the clients in time is the only recurring chore.
- If you find a mistake in this guide, please open an issue or a pull request.

## References

[Somer Esat — Guide to Staking on Ethereum (Ubuntu/Lighthouse)](https://someresat.medium.com/guide-to-staking-on-ethereum-ubuntu-lighthouse-773f5d982e03)

[Somer Esat's Ethereum Staking Guides](https://github.com/SomerEsat/ethereum-staking-guides)

[Community checkpoint sync endpoints](https://eth-clients.github.io/checkpoint-sync-endpoints/)

[Headscale documentation](https://headscale.net/stable/)

[Headscale install (official releases)](https://headscale.net/stable/setup/install/official/)

[Headscale getting started](https://headscale.net/stable/usage/getting-started/)

[Headscale TLS](https://headscale.net/stable/ref/tls/)

[Headscale changelog](https://github.com/juanfont/headscale/blob/main/CHANGELOG.md)

[Tailscale download](https://tailscale.com/download)

[Geth JSON-RPC server documentation](https://geth.ethereum.org/docs/interacting-with-geth/rpc)

[Prometheus](https://prometheus.io/download/) and [Node Exporter](https://github.com/prometheus/node_exporter)

[Grafana install on Debian/Ubuntu](https://grafana.com/docs/grafana/latest/setup-grafana/installation/debian/)

[Requestly for Chrome](https://requestly.com/downloads/chrome/), [Requestly for Firefox](https://requestly.com/downloads/firefox/) and [Redirect rule docs](https://docs.requestly.com/general/http-rules/rule-types/redirect-rule)
