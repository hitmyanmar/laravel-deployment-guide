
# 05 — Domain and DNS Configuration

Before deploying your Laravel application, you need to point your domain name to your VPS using DNS records.

This allows users to access your website using a domain such as `example.com` instead of your server's IP address.

## 1. Access Your DNS Management

Log in to your domain registrar or DNS provider and find the **DNS Management**, **DNS Records**, or **DNS Zone** section.

For example:

- **Spaceship:** Domain Management → DNS
- **Namecheap:** Domain List → Manage → Advanced DNS
- **Cloudflare:** Select your domain → DNS → Records

> **Note:** Your domain registrar and DNS provider may be different. Make sure you're editing the DNS records at the provider managing your domain's authoritative nameservers.

## 2. Point Your Domain to the VPS

To connect your main domain (`example.com`) to your VPS, create an **A record**.

| Type | Name | Value | TTL |
| --- | --- | --- | --- |
| A | @ | SERVER_IP | Auto |

Replace `SERVER_IP` with your VPS public IPv4 address.

- **Type:** `A` maps a domain to an IPv4 address.
- **Name:** `@` represents your root domain (`example.com`).
- **Value:** Your VPS public IPv4 address.
- **TTL:** Leave it as Auto or the provider's default.

Some DNS providers use a blank Name field instead of `@`.

### Configure the www Subdomain (Optional)

If you also want visitors to access your website using `www.example.com`, create a **CNAME record**.

| Type | Name | Value | TTL |
| --- | --- | --- | --- |
| CNAME | www | example.com | Auto |

Your DNS records should look similar to this:

| Type | Name | Value |
| --- | --- | --- |
| A | @ | SERVER_IP |
| CNAME | www | example.com |

> **Note:** DNS only directs visitors to your server. It does not automatically redirect `www.example.com` to `example.com` or vice versa. You can configure the preferred domain later.

## 3. Using a Subdomain (Optional)

If you want to host your Laravel application on a subdomain such as `app.example.com`, create an A record:

| Type | Name | Value | TTL |
| --- | --- | --- | --- |
| A | app | SERVER_IP | Auto |

This points `app.example.com` to your VPS without affecting the main domain.

You can use the same VPS for multiple domains or subdomains by creating separate websites in CloudPanel.

## 4. Using Cloudflare DNS (Optional)

If you're using Cloudflare to manage DNS, you'll see a **Proxy Status** option when creating A or CNAME records.

Cloudflare provides two options:

- **DNS Only (Gray Cloud):** Traffic connects directly to your VPS.
- **Proxied (Orange Cloud):** Website traffic passes through Cloudflare before reaching your VPS.

For initial deployment, you can use **DNS Only** to simplify DNS verification and SSL troubleshooting.

After configuring a valid SSL certificate on your VPS, you can enable **Proxied** if you want to use Cloudflare's security and performance features.

If enabling the Cloudflare proxy, configure SSL/TLS mode to **Full (strict)** when your origin server has a valid certificate.

> **Important:** Cloudflare Registrar requires Cloudflare nameservers. If you bought your domain elsewhere and want to use Cloudflare DNS, you must configure the nameservers provided by Cloudflare.

Official documentation: https://developers.cloudflare.com/dns/

## 5. Verify DNS Configuration

After saving your DNS records, verify that your domain resolves correctly.

Open a terminal on your **local computer**.

Check your main domain:

```bash
nslookup example.com
```

Check the `www` subdomain if configured:

```bash
nslookup www.example.com
```

Replace `example.com` with your actual domain name.

**Expected result:** If you are using DNS Only, the domain should resolve to your VPS public IPv4 address.

If you're using Cloudflare Proxied mode, the lookup will normally return Cloudflare IP addresses instead.

You can also check your domain's nameservers:

```bash
nslookup -type=NS example.com
```

These commands can be used on Windows (Git Bash), macOS, and Linux when `nslookup` is available.

### DNS Propagation

DNS changes may take some time to appear everywhere due to caching and nameserver updates.

If your domain doesn't resolve correctly:

1. Verify the DNS records contain the correct IP address.
2. Make sure you're editing the correct DNS provider.
3. Check for conflicting or outdated A, AAAA, or CNAME records.
4. Allow time for DNS caches to refresh.
5. Run `nslookup` again.

> **Important:** Do not remove unrelated DNS records such as MX or TXT records used for email and domain verification. Only modify records relevant to your website.

## 6. Verify Configuration

Before continuing, make sure:

- [ ] Your domain has an A record pointing to your VPS.
- [ ] The `www` or application subdomain is configured if needed.
- [ ] DNS records are managed by the correct provider.
- [ ] Your domain resolves correctly using `nslookup`.
- [ ] There are no conflicting DNS records.

Your domain is now configured to point to your VPS.

The website itself may not be accessible yet because we haven't deployed the Laravel application.

---

**Next:** [06 — Deploy Laravel from GitHub](06-deploy-laravel.md)
