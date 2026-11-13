# 01 — Introduction

Deploying a Laravel application to production involves more than uploading source code. You need a server that can run PHP, a database, a web server, a domain name, and a secure HTTPS connection.

This guide shows how to deploy an **existing Laravel application to an Ubuntu VPS using CloudPanel**, without manually configuring every server component from scratch.

CloudPanel provides a web interface for managing PHP sites, Nginx, databases, SSL certificates, and other server settings. You'll still use SSH and a few terminal commands for Laravel-specific tasks.

## What you'll set up

```text
Developer's computer
        |
        | Push application code
        v
   Git repository
        |
        | Clone / deploy
        v
  Ubuntu VPS + CloudPanel <------ DNS A record <------ Domain name
        |
        +-- Nginx -> Laravel (PHP-FPM)
        +-- MySQL / MariaDB
        +-- Built frontend assets (if applicable)
        |
        v
   HTTPS website
```

The domain registrar and VPS provider do **not** need to be the same company.

Applications built with Laravel, Vue, and Inertia.js can also be deployed using this approach. Vue assets are typically compiled with Vite before being served by Laravel.

Applications using Inertia server-side rendering (SSR) require an additional running Node.js process, which is outside this guide's basic setup.

## Prerequisites

Before starting, have the following ready:

- **Laravel application:** An existing project that works locally, with its environment requirements understood.
- **Git repository:** Your application source code hosted on GitHub or another Git provider.
- **VPS:** A fresh server running an operating system supported by CloudPanel, with administrator/SSH access. For small applications, **4 GB RAM** is a comfortable starting point.
- **Domain name:** A registered domain with access to its DNS records.
- **SSH client:** A terminal with SSH installed; key-based authentication is recommended.
- **Backups:** A plan to keep database and uploaded-file backups outside the VPS.

For the latest supported operating systems and server requirements, consult the [CloudPanel Requirements](https://www.cloudpanel.io/docs/v2/requirements/).

## Terminal Requirements

This guide uses **Bash-compatible commands** for consistency across Windows, macOS, and Linux.

| Operating System | Recommended Terminal                        |
| ---------------- | ------------------------------------------- |
| Windows          | [Git Bash](https://git-scm.com/install/win) |
| macOS            | Terminal                                    |
| Linux            | Terminal                                    |

**Windows users:** Install Git for Windows, which includes Git Bash. You don't need WSL or a separate Linux installation.

Windows Command Prompt and PowerShell can also run many SSH commands, but some commands and file paths may differ. For consistency, this guide uses **Git Bash on Windows**.

Throughout this guide, commands will be executed in two different environments:

- **Local Computer:** Your Windows, macOS, or Linux computer, used for SSH connections and SSH key management.
- **Ubuntu VPS:** Your remote server, where CloudPanel, PHP, Laravel, and other services are installed.

Each chapter will indicate where commands should be executed.

## Conventions Used in This Guide

Commands use example values. Replace them with your actual configuration before running anything.

| Placeholder          | Description                                         |
| -------------------- | --------------------------------------------------- |
| `SERVER_IP`          | Public IPv4 address of your VPS                     |
| `example.com`        | Your domain name                                    |
| `SITE_USER`          | Dedicated site user created in CloudPanel           |
| `~/.ssh/laravel_vps` | Example SSH private key path on your local computer |
| `REPOSITORY_URL`     | Your application's Git clone URL                    |
| `PHP_BIN`            | PHP CLI executable appropriate for your project     |

**Important:**

- Never commit credentials, `.env` files, SSH private keys, or production secrets to Git.
- Run application commands as the dedicated site user unless a step explicitly requires elevated privileges.
- Review commands and filesystem paths before executing them.

## What This Guide Covers

You'll learn how to:

1. Choose a domain and VPS provider.
2. Configure your VPS and SSH access.
3. Install CloudPanel securely.
4. Connect your domain using DNS.
5. Deploy your Laravel application from GitHub.
6. Configure the database and frontend assets.
7. Enable HTTPS and production settings.
8. Update and back up your application.

This guide assumes you already know Laravel development. It is **not** a Laravel programming tutorial or a complete Linux administration handbook.

Queue workers, scheduling, and WebSockets may need additional configuration depending on your application.

---

**Next:** [02 — Choosing a Domain and VPS](02-domain-and-vps.md)
