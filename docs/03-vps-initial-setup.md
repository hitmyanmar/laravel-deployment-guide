# 03 — VPS Initial Setup & Security

Before installing CloudPanel, we need to prepare our Ubuntu VPS, configure SSH access, and secure the server using a firewall.

This guide uses **Ubuntu 26.04 LTS**. Check [CloudPanel's supported operating systems](https://www.cloudpanel.io/docs/v2/requirements/) before provisioning.

## 1. Prepare Your VPS

Depending on your VPS provider, you can select the operating system during purchase or install it afterward.

1. Install **Ubuntu 26.04 LTS** on your VPS.
2. Obtain your server's **public IPv4 address** and initial SSH credentials.
3. Make sure your provider offers a **web console or rescue console** for recovery.
4. Enable **Two-Factor Authentication (2FA)** on your hosting provider account.

> **Note:** CloudPanel requires a clean server. Reinstalling an existing VPS may erase its data. Back up important files before reinstalling.

## 2. Connect to Your VPS

If your VPS provider supplied a root password, open your **local terminal** and run:

```bash
ssh root@SERVER_IP
```

Enter the root password when prompted.

If your VPS already has your SSH public key installed, you can connect using the corresponding private key:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

Replace `SERVER_IP` with your VPS public IP address. Some providers use a username other than `root`.

Once connected, verify your operating system and server resources:

```bash
cat /etc/os-release
free -h
df -h
```

You can disconnect from the VPS using:

```bash
exit
```

## 3. Set Up SSH Key Authentication

SSH keys provide a more secure way to access your VPS without relying on a server password.

Each computer can have its own SSH key pair, and a VPS can authorize multiple public keys.

### Generate an SSH Key Pair

Run the following commands on your **local computer**, not on the VPS.

First, make sure the `.ssh` directory exists:

```bash
mkdir -p ~/.ssh
```

Generate a new SSH key pair:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/laravel_vps -C "laravel-vps"
```

Set a passphrase when prompted for additional security.

> **Important:** If `laravel_vps` already exists, do not overwrite it. Choose another filename instead.

This generates two files:

- `~/.ssh/laravel_vps` — **Private key** (keep it on your computer).
- `~/.ssh/laravel_vps.pub` — **Public key** (add it to your VPS).

Display your public key:

```bash
cat ~/.ssh/laravel_vps.pub
```

### Add Your Public Key to the VPS

There are two scenarios depending on your VPS provider.

**Option A — During VPS Creation**

Some providers allow you to configure an SSH public key while creating a VPS.

1. Generate your SSH key pair before creating the VPS.
2. Copy the public key using the command above.
3. Find the **SSH Keys** section in your VPS provider's dashboard.
4. Paste your public key.
5. Complete VPS creation.

The provider will configure the public key for the appropriate SSH user.

**Option B — Existing VPS with Password Authentication**

If your VPS provider supplied a root password and the server is already running, you can add your public key manually.

Run this command on your **local computer**:

```bash
cat ~/.ssh/laravel_vps.pub | ssh root@SERVER_IP \
'umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys; chmod 700 ~/.ssh; chmod 600 ~/.ssh/authorized_keys'
```

Enter your VPS root password when prompted.

This command:

1. Reads your local SSH public key.
2. Connects to your VPS using SSH.
3. Creates the server's `.ssh` directory if necessary.
4. Adds the public key to `~/.ssh/authorized_keys`.
5. Sets secure file permissions.

This works in **Windows Git Bash, macOS Terminal, and Linux Terminal**.

If your provider uses a different SSH username, replace `root` with that username.

### Verify SSH Key Authentication

Open a **new terminal** on your local computer and run:

```bash
ssh -o PreferredAuthentications=publickey -o PasswordAuthentication=no -i ~/.ssh/laravel_vps root@SERVER_IP
```

If the connection succeeds, your SSH key authentication is working.

You may be prompted for your **private key passphrase**, but not your VPS root password.

> **Security:** After verifying SSH key authentication and confirming recovery-console access, consider disabling SSH password authentication for better security. Do not disable it before testing your SSH key.

### Access Your VPS from Multiple Computers

If you want to access the same VPS from another computer:

1. Generate a separate SSH key pair on that computer.
2. Add its public key to the VPS user's `~/.ssh/authorized_keys` file.
3. Connect using that computer's private key.

Each computer can have its own SSH key. You do not need to share the same private key between computers.

---

## 4. Configure the Firewall

CloudPanel uses **TCP port 8443** for its administration interface.

Before installing CloudPanel, restrict public access to this port. Otherwise, someone could access the initial setup page and create the first administrator account before you do.

Choose **one** of the following firewall methods.

### Option A — VPS Provider Firewall (Recommended)

If your VPS provider offers a firewall or security group:

1. Open the provider's **Firewall** or **Security Groups** page.
2. Create inbound rules allowing:
   - **TCP 22** (SSH) — from your trusted IP if practical.
   - **TCP 80** (HTTP) — from anywhere.
   - **TCP 443** (HTTPS) — from anywhere.
3. Do **not** allow public access to:
   - **TCP 8443** — CloudPanel administration.
   - **TCP 3306** — MySQL.
4. Ensure the firewall blocks other unsolicited inbound connections.
5. Apply the firewall rules to your VPS.
6. Verify that SSH access still works.

> These steps assume an allowlist-based firewall. Verify your provider's default firewall behavior before applying the rules.

### Option B — Ubuntu UFW Firewall

If your provider does not offer a network firewall, you can use Ubuntu's built-in UFW firewall.

Connect to your VPS and run:

```bash
sudo apt update
sudo apt install -y ufw
```

Configure the firewall rules:

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing

sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

sudo ufw deny 8443/tcp
```

Enable the firewall:

```bash
sudo ufw enable
```

Check the active rules:

```bash
sudo ufw status verbose
```

**Important:**

- If SSH uses a different port, allow that port before enabling UFW.
- Keep your existing SSH session open while configuring the firewall.
- Test SSH access from a second terminal.
- Make sure your VPS provider offers console or rescue access in case you get locked out.
- CloudPanel may modify UFW rules during installation. Recheck the rules afterward and ensure port **8443 remains blocked from public access**.

A provider-level firewall is preferable because it can prevent public access to port 8443 even if CloudPanel changes the server's UFW rules.

---

## 5. Verify VPS Readiness

Before installing CloudPanel, make sure:

- [ ] Ubuntu 26.04 LTS is installed on a clean VPS.
- [ ] SSH access works from your local computer.
- [ ] SSH key authentication is configured and tested.
- [ ] VPS console or rescue access is available.
- [ ] Port **8443** is not publicly accessible.
- [ ] Port **3306** is not publicly accessible.
- [ ] Firewall rules are configured without blocking SSH.

Your VPS is now ready for CloudPanel installation.

---

**Next:** [04 — CloudPanel Installation](04-cloudpanel-installation.md)
