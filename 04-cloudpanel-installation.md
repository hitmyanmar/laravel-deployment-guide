# 04 — CloudPanel Installation

CloudPanel is a free server control panel for managing PHP applications, Nginx, databases and SSL certificates on a VPS.

Official documentation: https://www.cloudpanel.io/docs/v2/getting-started/other/

## Prerequisites

- A **fresh VPS** with Ubuntu **26.04 LTS** and root or sudo access.
- At least **2 GB RAM** (4 GB recommended for a small Laravel app).
- SSH access and firewall protection completed in [03 — VPS Initial Setup & Security](03-vps-initial-setup.md).

## 1. Connect to Your VPS

From your local computer, connect using the SSH key configured in Chapter 03:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

If your provider still uses password authentication, use `ssh root@SERVER_IP` instead.

## 2. Update Your Server

On the VPS, update packages and install prerequisites:

```bash
apt update && apt -y upgrade && apt -y install curl wget sudo
```

Reboot and reconnect if system updates require it.

## 3. Install CloudPanel

This example uses **MySQL 8.4**:

```bash
curl -fsS https://installer.cloudpanel.io/ce/v2/install.sh -o install.sh && \
echo "8146dbe0a488e7088b04071b0c34d59aa0ab1fe9dcec382d395fd155c9e6c476 install.sh" | \
sha256sum -c && sudo DB_ENGINE=MYSQL_8.4 bash install.sh
```

**Important:** Before running this command, check the [official installation guide](https://www.cloudpanel.io/docs/v2/getting-started/other/) for the latest installer command and **SHA-256 checksum**. Stop if verification fails.

After installation, check your firewall. In particular, **port 8443 must not be reachable from untrusted networks**:

```bash
sudo ufw status verbose
```

If your VPS uses a provider-level firewall, verify its inbound rules as well. CloudPanel may modify local UFW rules during installation. Make sure port 8443 remains inaccessible from untrusted public networks.

## 4. Access CloudPanel Through an SSH Tunnel

On your **local computer**, open a separate terminal:

```bash
ssh -i ~/.ssh/laravel_vps -N -L 8443:127.0.0.1:8443 root@SERVER_IP
```

If you use password authentication, omit `-i ~/.ssh/laravel_vps`.

SSH authenticates using your private key (and may ask for its passphrase) or your server password. A person who knows only the server IP **cannot create the tunnel** without valid SSH credentials.

Keep this terminal open. In your browser, visit:

```text
https://localhost:8443
```

A self-signed certificate warning may appear during the initial setup. Proceed only after verifying the SSH connection is to your intended VPS.

> The SSH tunnel is **not a firewall**. Keep port 8443 blocked from public internet access. The tunnel also does not bypass CloudPanel's own login.

## 5. Create Your Administrator Account

1. Open the first-run setup page through the SSH tunnel.
2. Create the **first administrator account** with a strong, unique password.
3. Log in and enable **two-factor authentication (2FA)**.
4. Keep the management port private even after the account has been created.

The first-admin registration is a one-time setup step. Other users cannot simply register a new administrator through that initial page.

When you're finished using CloudPanel, press `Ctrl + C` in the SSH tunnel terminal.

## Verify Installation

- [ ] CloudPanel opens at `https://localhost:8443` through the tunnel.
- [ ] The administrator account exists and 2FA is enabled.
- [ ] Port **8443** is blocked from untrusted public networks.
- [ ] SSH access still works.

**Next:** [05 — Domain and DNS Configuration](05-domain-dns.md).
