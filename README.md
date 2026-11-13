
# Laravel Deployment Guide

A practical, step-by-step guide to deploying an existing **Laravel application on an Ubuntu VPS using CloudPanel**.

Learn how to configure your server, connect a domain, deploy Laravel from GitHub, enable HTTPS, update your application, and manage backups.

This guide also includes optional advanced topics such as queue workers, Laravel Reverb, Redis, and automated deployments.

## Who Is This Guide For?

This guide is for developers who:

- Already know how to build Laravel applications.
- Want to deploy their applications to a real VPS.
- Are new to server administration or production deployment.
- Prefer practical instructions over complicated infrastructure setups.

**You don't need prior VPS or CloudPanel experience.**

> This guide focuses on deploying an **existing Laravel application**, not teaching Laravel development from scratch.

## Technologies Covered

| Technology | Purpose |
| --- | --- |
| Ubuntu Server | VPS operating system |
| CloudPanel | Server and website management |
| Nginx | Web server |
| PHP & Composer | Laravel backend |
| MySQL | Application database |
| Node.js & npm | Frontend asset builds |
| Git & GitHub | Source code management |
| Let's Encrypt | Free SSL certificates |
| Supervisor | Background process management |
| Laravel Reverb | Real-time WebSockets |
| Redis | Cache, sessions, and queues |
| GitHub Actions | Automated deployment |

Not every application needs all these technologies. Advanced services are optional.

## Prerequisites

Before getting started, you should have:

- Basic knowledge of Laravel.
- An existing Laravel application hosted on GitHub.
- A domain name.
- An Ubuntu VPS or plans to purchase one.
- Basic familiarity with Git and terminal commands.

### Supported Development Computers

You can follow this guide using:

- **Windows:** Git Bash (Git for Windows)
- **macOS:** Terminal
- **Linux:** Terminal

We'll use Bash-compatible commands throughout the guide.

Commands are labeled according to where they should run: your **local computer** or your **Ubuntu VPS**.

## Core Deployment Guide

Follow these chapters in order if you're deploying a Laravel application for the first time.

| # | Chapter | What You'll Learn |
| --- | --- | --- |
| 01 | [Introduction](docs/01-introduction.md) | Overview, requirements, and deployment workflow |
| 02 | [Domain & VPS](docs/02-domain-and-vps.md) | Choose and purchase a domain and VPS |
| 03 | [VPS Initial Setup](docs/03-vps-initial-setup.md) | SSH keys, server access, and firewall |
| 04 | [CloudPanel Installation](docs/04-cloudpanel-installation.md) | Install CloudPanel and secure admin access |
| 05 | [Domain & DNS](docs/05-domain-dns.md) | Connect your domain to the VPS |
| 06 | [Deploy Laravel](docs/06-deploy-laravel.md) | Create a PHP site and clone Laravel from GitHub |
| 07 | [Production Setup](docs/07-production-setup.md) | Composer, `.env`, MySQL, migrations, and assets |
| 08 | [SSL & Security](docs/08-ssl-security.md) | Enable HTTPS and verify production security |
| 09 | [Updating Production](docs/09-updating-production.md) | Deploy new changes using Git |
| 10 | [Backups & Recovery](docs/10-backups.md) | Back up and restore your application data |

**Start here:** [01 — Introduction](docs/01-introduction.md)

## Advanced Topics (Optional)

Once your Laravel application is running, you can explore additional production features.

| # | Chapter | What You'll Learn |
| --- | --- | --- |
| 01 | [Queue Workers & Scheduler](docs/advanced/01-queue-and-scheduler.md) | CloudPanel Cron Jobs and Supervisor |
| 02 | [Laravel Reverb](docs/advanced/02-laravel-reverb.md) | Production WebSockets and real-time features |
| 03 | [Redis](docs/advanced/03-redis.md) | Redis caching, sessions, and queues |
| 04 | [CI/CD with GitHub Actions](docs/advanced/04-ci-cd.md) | Automated Laravel deployments |

These chapters are independent of the core guide. Follow only the topics your application needs.

## Deployment Workflow

The main deployment process looks like this:

```text
Domain + Ubuntu VPS
        |
        v
Initial Server Setup
        |
        v
Install CloudPanel
        |
        v
Configure Domain DNS
        |
        v
Create Laravel Website
        |
        v
Clone Project from GitHub
        |
        v
Configure Production Environment
        |
        v
Install Dependencies & Build Assets
        |
        v
Enable HTTPS
        |
        v
Application Live!
        |
        v
Updates, Monitoring & Backups
```

## Example Application

This guide works with typical Laravel applications, including:

- Laravel + Blade
- Laravel + Vue + Inertia.js
- Laravel + React + Inertia.js
- Laravel REST APIs

Some applications may require additional configuration depending on their dependencies and architecture.

The guide uses `example.com`, `SITE_USER`, and `SERVER_IP` as placeholders. Replace them with your actual values when following the instructions.

## Security Notes

Deploying an application to production involves handling server credentials and sensitive application data.

Always follow these basic practices:

- Use SSH keys for server access.
- Restrict CloudPanel administrator access.
- Never commit production `.env` files or private keys to GitHub.
- Keep `APP_DEBUG=false` in production.
- Use HTTPS for public websites.
- Avoid exposing unnecessary server ports.
- Keep your server and application dependencies updated.
- Maintain backups outside your VPS.

The guide demonstrates a practical starting point for deployment, not a complete security audit or a high-availability infrastructure design.

## Official Documentation

For further information, refer to:

- [Laravel Documentation](https://laravel.com/docs)
- [CloudPanel Documentation](https://www.cloudpanel.io/docs/v2/)
- [Ubuntu Server Documentation](https://ubuntu.com/server/docs)
- [GitHub Documentation](https://docs.github.com/)
- [Let's Encrypt](https://letsencrypt.org/)

## Contributing

Contributions and improvements are welcome.

If you find an error, have a suggestion, or want to improve an explanation, feel free to open a GitHub Issue or submit a Pull Request.

Please keep contributions practical, clear, and beginner-friendly for developers learning server deployment.

## Disclaimer

This guide is provided for educational purposes.

Server environments, software versions, and hosting providers may differ. Always verify commands and configurations before executing them on a production server.

Back up important data before making significant changes.

---

**Happy Deploying!**

Made with ❤️ by [HiT Myanmar](https://hitmyanmar.com)
