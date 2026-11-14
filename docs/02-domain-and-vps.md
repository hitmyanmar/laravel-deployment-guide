# 02 — Choosing a Domain and VPS

Before deploying Laravel with CloudPanel, you need a **domain name** and a **virtual private server (VPS)**. These can be purchased from different companies. You do not need to buy hosting from your domain registrar.

This guide uses an Ubuntu VPS with a public IPv4 address and a domain such as `example.com`.

## 1. Choose a domain registrar

A **domain registrar** registers and renews your domain. A **DNS provider** manages records that direct the domain to your server. They may be the same company, but they do not have to be.

| Registrar            | Website                                        |
| -------------------- | ---------------------------------------------- |
| Spaceship            | https://www.spaceship.com/                     |
| Namecheap            | https://www.namecheap.com/                     |
| Cloudflare Registrar | https://www.cloudflare.com/products/registrar/ |

Before buying, check:

- **Renewal cost**, not just the first-year promotional price.
- **Domain extension** (`.com`, `.dev`, etc.) and its annual price.
- **DNS management**, including A, AAAA and CNAME records.
- **Account security**, including two-factor authentication (2FA) and domain lock.
- **Transfer restrictions** and any applicable fees.

Cloudflare Registrar requires the use of Cloudflare's authoritative nameservers.

### Register a domain

1. Search for an available domain at your selected registrar.
2. Check the checkout total **and renewal price**.
3. Register the domain in an account you control.
4. Enable 2FA and automatic renewal or renewal reminders.
5. Find your DNS management page; you will use it in [05 — Domain and DNS](05-domain-dns.md).

> You can prepare your VPS before buying a domain, but a domain is needed for the usual DNS and HTTPS setup.

## 2. Choose a VPS provider

A **VPS** is a virtual Linux server where you will run CloudPanel, Nginx, PHP, Laravel and a database.

| Provider     | Product                                                    |
| ------------ | ---------------------------------------------------------- |
| Hostinger    | [VPS Hosting](https://www.hostinger.com/vps-hosting)       |
| DigitalOcean | [Droplets](https://www.digitalocean.com/products/droplets) |

These are examples, not a ranking. Any provider offering a clean, supported Linux VPS with root or sudo access can work.

### Suggested server resources

For a relatively small Laravel application with a local database:

| Resource | Starting point       | More comfortable option            |
| -------- | -------------------- | ---------------------------------- |
| RAM      | 2 GB                 | 4 GB                               |
| CPU      | 1 shared vCPU        | 2 shared vCPUs                     |
| Storage  | 25 GB SSD            | 40–60 GB SSD                       |
| OS       | Supported Ubuntu LTS | Ubuntu 26.04 LTS                   |
| Network  | Public IPv4          | Public IPv4 with adequate transfer |

These are planning suggestions, not performance guarantees. CloudPanel's minimum is **1 CPU core, 2 GB RAM and 10 GB disk**. See the [official requirements](https://www.cloudpanel.io/docs/v2/requirements/).

If your app uses Redis, queues or Laravel Reverb, allow additional resources.

### Compare the full cost

Before purchasing, compare:

- Monthly vs annual billing, discounts and **renewal pricing**.
- IPv4 charges, SSD storage, bandwidth and backups.
- Datacenter location and latency to your users.
- Available operating-system images and rescue-console access.
- Support policy and whether the server is **unmanaged**.

After purchasing, your VPS provider will provide a public IP address and initial access instructions. Depending on the provider, you may receive a root password or be asked to configure an SSH public key during provisioning. Keep your credentials private.

## Next step

Continue to [03 — VPS Initial Setup & Security](03-vps-initial-setup.md).
