# Cloudflare Tunnel Setup Guide for Chronolox Relayer OS

This guide provides step-by-step instructions for setting up Cloudflare Tunnel with Chronolox Relayer OS, including screenshots of the console wizard.

## Overview

Chronolox Relayer OS supports **Cloudflare Tunnel (cloudflared)** as the recommended ingress mode because it requires **zero open inbound ports** — all traffic enters the appliance via an outbound Cloudflare Tunnel connection, providing maximum security against direct attacks.

**Prerequisites:**
- A Cloudflare account with the Zero Trust dashboard
- A domain name (e.g., `relayer.example.com`) added to your Cloudflare account
- Chronolox Relayer OS ISO booted on your VPS or VM

## Step 1: Boot Chronolox Relayer OS

After attaching the ISO to your VPS and booting, you'll see the Chronolox Relayer OS welcome screen:

![Screenshot: Chronolox Relayer OS Boot Screen](screenshots/01-boot-screen.png)

The first-boot wizard will guide you through the initial setup process.

## Step 2: Complete Initial Setup Steps

Before configuring Cloudflare Tunnel, complete these initial wizard steps:

### 2.1 LUKS2 Vault Initialization

The wizard will detect eligible disks and prompt you to initialize the LUKS2 encrypted vault:

![Screenshot: Disk Selection](screenshots/02-disk-selection.png)

**Important:** This will permanently erase all data on the selected disk. Type `YES /dev/vdX` to confirm.

### 2.2 Set Vault Passphrase

Set a strong passphrase (minimum 10 characters) that will:
- Encrypt your LUKS2 vault (AES-XTS-512 + Argon2id)
- Protect the local appliance console menu

![Screenshot: Passphrase Setup](screenshots/03-passphrase-setup.png)

### 2.3 Network Configuration

The wizard will detect your network settings. Keep the default DHCP configuration unless you need a static IP:

![Screenshot: Network Configuration](screenshots/04-network-config.png)

### 2.4 Operator Wallet Setup

Choose one of two options:

**Option 1:** Import an existing Relayer Private Key from Rabby/MetaMask
**Option 2:** Generate a new Relayer Hot Wallet

![Screenshot: Wallet Setup](screenshots/05-wallet-setup.png)

If you generate a new wallet, **import the displayed Private Key into Rabby/MetaMask immediately** before typing `SAVED` to confirm.

## Step 3: Select Cloudflare Tunnel Ingress Mode

After wallet setup, the wizard presents the HTTPS Ingress options:

```
Step 4/5 — HTTPS Ingress & Reverse Proxy Mode
  [1] Cloudflare Tunnel (cloudflared) — Recommended (Zero open inbound ports)
  [2] Nginx + Let's Encrypt (certbot) — Automatic TLS on ports 80 & 443
  [3] Nginx + Custom TLS Certificate — Paste your own PEM fullchain & private key
  [4] Local / VM Test Mode — Self-signed TLS on 443 + direct HTTP on 8081
```

![Screenshot: Ingress Mode Selection](screenshots/06-ingress-mode-selection.png)

**Select option [1] for Cloudflare Tunnel.**

## Step 4: Enter Public HTTPS URL

The wizard will prompt for your public HTTPS URL:

```
Enter your Public HTTPS URL (e.g. https://relayer.example.com):
```

![Screenshot: Public URL Input](screenshots/07-public-url-input.png)

**Example:** `https://relayer.yourdomain.com`

This URL must be a domain you control in Cloudflare.

## Step 5: Choose Cloudflare Authentication Method

The wizard offers two authentication methods:

```
Cloudflare Tunnel Authentication:
  [1] Zero Trust Dashboard Token (Recommended — shown in the dashboard's
      'cloudflared service install <TOKEN>' command)
  [2] CLI-created Tunnel Credentials JSON (locally-managed — minted by the
      cloudflared CLI; the dashboard no longer offers a credentials download)
```

![Screenshot: Cloudflare Auth Method](screenshots/08-cloudflare-auth-method.png)

### Method 1: Zero Trust Dashboard Token (Recommended)

This is the simplest method. Follow these steps:

#### 5.1 Create a Tunnel in Cloudflare Zero Trust

1. Log in to your Cloudflare Zero Trust dashboard: https://one.dash.cloudflare.com
2. Navigate to **Networking** → **Tunnels**
3. Click **Create a tunnel**

![Screenshot: Cloudflare Tunnels Dashboard](screenshots/09-cloudflare-tunnels-dashboard.png)

#### 5.2 Configure Tunnel Settings

1. **Tunnel name:** Enter a descriptive name (e.g., `chronolox-relayer-01`)
2. **Tunnel type:** Select **Cloudflared** (self-hosted)
3. Click **Next**

![Screenshot: Create Tunnel](screenshots/10-create-tunnel.png)

#### 5.3 Choose Installation Method

Select **Generic / Linux** as the installation method:

![Screenshot: Installation Method](screenshots/11-installation-method.png)

#### 5.4 Copy the Tunnel Token

Cloudflare will display a command like:

```bash
cloudflared service install <TOKEN>
```

**Copy the `<TOKEN>` part only** (it looks like a long Base64 string, typically starting with `eyJ`).

![Screenshot: Copy Tunnel Token](screenshots/12-copy-tunnel.png)

#### 5.5 Paste Token in Relayer OS Console

Return to the Relayer OS console and paste the token:

```
Zero Trust dashboard -> Networking -> Tunnels -> your tunnel:
Routes tab -> Add route -> Published application ->
relayer.yourdomain.com -> http://127.0.0.1:8081
Paste Cloudflare Tunnel Token (input hidden): [paste token here]
```

![Screenshot: Paste Token](screenshots/13-paste-token.png)

The token input is hidden for security.

#### 5.6 Configure Public Hostname Routing

**Before pressing Enter in the Relayer OS console**, complete this step in Cloudflare:

1. Open your tunnel and go to the **Routes** tab
2. Click **Add route** and select **Published application**
3. **Hostname:** Enter your domain (e.g., `relayer.yourdomain.com`)
4. **Service URL:** Enter `http://127.0.0.1:8081`
5. Click **Add route**

![Screenshot: Configure Public Hostname](screenshots/14-configure-hostname.png)

This tells Cloudflare to route traffic for `relayer.yourdomain.com` to the Relayer's HTTP port 8081 on the appliance.

#### 5.7 Complete Setup in Relayer OS

Press Enter after pasting the token. The wizard will confirm:

```
✓ Cloudflare Tunnel persisted in LUKS2 vault (UFW will block all inbound ports).
```

![Screenshot: Cloudflare Setup Complete](screenshots/15-cloudflare-complete.png)

### Method 2: CLI-created Tunnel Credentials JSON

For advanced users who prefer locally-managed tunnels (ingress rules in your own `config.yml` instead of dashboard-managed routes). **The Zero Trust dashboard no longer offers a `credentials.json` download** — tunnels created in the dashboard are remotely-managed and hand out a token only. The credentials file is minted by the cloudflared CLI:

#### 5.1 Mint the Credentials File (on any machine with a browser)

```bash
cloudflared tunnel login                     # opens a browser; writes cert.pem
cloudflared tunnel create chronolox-relayer  # -> ~/.cloudflared/<TUNNEL-UUID>.json
```

Lost the credentials file for an existing tunnel? Recover it with the tunnel token command (works for remotely-managed tunnels too):

```bash
cloudflared tunnel token <tunnel-name-or-uuid> --cred-file out.json
```

![Screenshot: Create Tunnel via CLI](screenshots/16-create-tunnel-cli.png)

#### 5.2 Paste Credentials in Relayer OS Console

The wizard prints the exact commands above, then asks for the JSON. Paste the file content and finish with an empty line — the Tunnel UUID is extracted from the JSON automatically (you are only prompted for it if extraction fails):

```
Paste your Tunnel credentials JSON (finish with an empty line):
{"AccountTag":"...","TunnelSecret":"...","TunnelID":"..."}
[paste empty line to finish]
```

![Screenshot: Enter Credentials JSON](screenshots/17-enter-credentials.png)

## Step 6: Complete Remaining Wizard Steps

After Cloudflare Tunnel configuration, continue with the remaining wizard steps:

### 6.1 Optional: Telegram Alerts & Custom RPC Keys

Configure Telegram bot token and chat ID for critical alerts, plus optional Alchemy API keys for enhanced RPC fallback:

```
Step 5/5 — Optional Telegram Alerts & Custom RPC Keys
Enter Telegram Bot Token (or press Enter to skip):
Enter Telegram Chat ID (or press Enter to skip):
Enter Alchemy API Key (comma-separated, or press Enter to skip):
```

![Screenshot: Telegram Alerts](screenshots/18-telegram-alerts.png)

### 6.2 No SSH on the Appliance

The wizard finishes with a notice instead of an SSH policy step: `openssh-server` and `sudo` are **not in the image** — the appliance has no network login path at all. Management happens exclusively through the provider's VGA/serial console and the dashboard's Emergency Root Shell (gated by a fresh passphrase + typed confirmation).

![Screenshot: Remote Access Notice](screenshots/20-remote-access-notice.png)

## Step 7: Configuration Complete

The wizard will display a success message:

```
✓ Chronolox Relayer OS configuration complete!
✓ Relayer service started
✓ Cloudflare Tunnel active
✓ UFW firewall enabled (all inbound ports blocked)
```

![Screenshot: Setup Complete](screenshots/21-setup-complete.png)

## Step 8: Verify Cloudflare Tunnel Status

### 8.1 Check Tunnel Health in Cloudflare Dashboard

Navigate to Cloudflare Zero Trust → **Networking** → **Tunnels** and verify your tunnel shows as **Healthy**:

![Screenshot: Tunnel Healthy](screenshots/22-tunnel-healthy.png)

### 8.2 Access Relayer Dashboard

Open your browser and navigate to your public URL:

```
https://relayer.yourdomain.com
```

You should see the Chronolox Relayer dashboard:

![Screenshot: Relayer Dashboard](screenshots/23-relayer-dashboard.png)

### 8.3 Verify No Inbound Ports Open

On the Relayer OS console, check UFW status:

```
ufw status
```

Expected output for Cloudflare mode:
```
Status: active

To                         Action      From
--                         ------      ----
```

No inbound ports should be listed — all traffic enters via the outbound Cloudflare Tunnel.

![Screenshot: UFW Status](screenshots/24-ufw-status.png)

## Security Benefits of Cloudflare Tunnel Mode

1. **Zero Open Inbound Ports:** UFW blocks all inbound traffic. Only the outbound Cloudflare Tunnel connection is allowed.
2. **DDoS Protection:** Cloudflare's global network absorbs DDoS attacks before they reach your appliance.
3. **TLS Termination:** Cloudflare handles TLS termination, reducing CPU load on the appliance.
4. **Hidden Origin:** Your appliance's IP address is never exposed to end users.
5. **Access Controls:** Cloudflare Zero Trust policies can restrict access by geography, IP, or authentication.

## Troubleshooting

### Tunnel Shows "Unhealthy" in Cloudflare Dashboard

1. Check Cloudflare Tunnel service status on the appliance:
   ```bash
   systemctl status cloudflared
   ```

2. View Cloudflare Tunnel logs:
   ```bash
   journalctl -u cloudflared -f
   ```

3. Verify network connectivity:
   ```bash
   ping 1.1.1.1
   ```

### Cannot Access Relayer Dashboard

1. Verify the public hostname routing in Cloudflare matches your setup:
   - Hostname: `relayer.yourdomain.com`
   - Service: `http://127.0.0.1:8081`

2. Check relayer service status:
   ```bash
   systemctl status chronolox-relayer
   ```

3. Check UFW status (should have no inbound rules):
   ```bash
   ufw status
   ```

### Token Format Error

If you see "Invalid Cloudflare Tunnel token format":

1. Ensure you copied the entire token (not the full command)
2. The token should be a long Base64 string starting with `eyJ` or similar
3. Do not include quotes or extra whitespace

## Reconfiguring Cloudflare Tunnel

If you need to change Cloudflare Tunnel settings after initial setup:

1. Unlock the LUKS2 vault at the console
2. Access the Locked Appliance Console menu
3. Select option **4: Reconfigure Ingress**
4. Follow the wizard prompts to update Cloudflare Tunnel settings

![Screenshot: Reconfigure Ingress](screenshots/25-reconfigure-ingress.png)

## Switching Ingress Modes

To switch from Cloudflare Tunnel to another ingress mode (Let's Encrypt, Custom TLS, or Local Test):

1. Access the Locked Appliance Console menu
2. Select option **4: Reconfigure Ingress**
3. Choose the new ingress mode
4. The wizard will automatically:
   - Stop/disable the previous service (cloudflared or nginx)
   - Configure the new service
   - Update UFW firewall rules accordingly

## Additional Resources

- [Cloudflare Zero Trust Documentation](https://developers.cloudflare.com/cloudflare-one/)
- [Cloudflare Tunnel Documentation](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/)
- [Create a tunnel (dashboard)](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/get-started/create-remote-tunnel/)
- [Create a locally-managed tunnel (CLI)](https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/do-more-with-tunnels/local-management/create-local-tunnel/)
- [Chronolox Relayer OS Plan](../CHRONOLOX-RELAYER-OS-PLAN.md)
- [Relayer Onboarding Guide](https://chronolox.com/relayer-os)

## Security Notes

- **Never share your Cloudflare Tunnel token** — it grants full control over your tunnel
- **Rotate tokens regularly** — regenerate tokens periodically for enhanced security
- **Monitor tunnel logs** — review Cloudflare Tunnel logs for suspicious activity
- **Use Access Policies** — implement Cloudflare Zero Trust access policies for additional protection
- **Keep LUKS2 passphrase secure** — this is the only credential protecting your encrypted vault

## Appendix: Console Wizard Cloudflare Mode Implementation

The Cloudflare Tunnel mode in the console wizard (`chronolox-console.sh`) performs the following:

1. **Token Storage:** Stores the tunnel token in `/etc/cloudflared/tunnel.token` with `0600` permissions
2. **Systemd Service:** Creates `/etc/systemd/system/cloudflared.service` with:
   - `LoadCredential=tunnel_token:/etc/cloudflared/tunnel.token` — prevents token from appearing in `/proc/<pid>/environ`
   - `DynamicUser=yes` — runs as unprivileged user
   - `--token-file %d/tunnel_token` — passes token via file descriptor
   - Full systemd hardening: `ProtectSystem=strict`, `ProtectProc=invisible`, `RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6`
3. **UFW Firewall:** Configures UFW to block all inbound ports (since traffic enters via outbound tunnel)
4. **Nginx Disabled:** Stops and disables nginx (not needed with Cloudflare Tunnel)
5. **Persistence:** All configuration persisted in LUKS2 vault under `/mnt/chrono-vault/etc/systemd-units/`

This ensures maximum security with the tunnel token never exposed in process listings or environment variables.
