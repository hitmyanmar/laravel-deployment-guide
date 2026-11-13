# 08 — SSL & Production Security

After deploying and configuring our Laravel application, we need to enable HTTPS and verify that our website is secure for production.

CloudPanel makes SSL certificate installation simple using **Let's Encrypt**, which provides free SSL certificates.

Official documentation: https://www.cloudpanel.io/docs/v2/frontend-area/tls/

## Prerequisites

Before continuing, make sure:

- Laravel application is deployed and configured.
- Domain DNS records point to your VPS.
- Ports **80 (HTTP)** and **443 (HTTPS)** are publicly accessible.
- You can access the CloudPanel dashboard.

## 1. Install an SSL Certificate

CloudPanel creates a self-signed SSL certificate by default. However, we should replace it with a trusted SSL certificate for our website.

### Install Let's Encrypt SSL

1. Log in to CloudPanel.
2. Navigate to **Sites → Your Website → Manage**.
3. Open **SSL/TLS**.
4. Click **Actions → New Let's Encrypt Certificate**.
5. Enter the domain names you want to secure:
   - `example.com`
   - `www.example.com` (if configured)
6. Click **Create and Install**.

CloudPanel will request and install the SSL certificate.

> **Important:** Every domain included in the certificate must have valid DNS records pointing to your server. Make sure port 80 is reachable for HTTP-based certificate validation.

If you're using Cloudflare, switching the DNS records to **DNS Only** temporarily can simplify the initial certificate installation.

## 2. Enable HTTPS

HTTPS encrypts communication between your website and its visitors.

CloudPanel normally configures **HTTP to HTTPS redirection automatically** when creating a PHP site.

After installing your SSL certificate, open:

```text
https://example.com
```

Verify that your browser displays a valid, trusted HTTPS connection without a certificate warning.

### Test HTTP Redirection

Try opening:

```text
http://example.com
```

It should automatically redirect to:

```text
https://example.com
```

You can also verify this using a terminal on your **local computer**:

```bash
curl -I http://example.com
```

Expected result: An HTTP redirect such as `301` or `308` pointing to the HTTPS version.

Check HTTPS:

```bash
curl -I https://example.com
```

Expected result: A successful HTTPS connection with a valid certificate.

> **Note:** If HTTP redirection is not working, check your CloudPanel site's Vhost configuration. CloudPanel's default PHP site template normally handles this automatically.

## 3. Configure Cloudflare SSL (Optional)

If you're using Cloudflare with **Proxied (Orange Cloud)** DNS records, make sure Cloudflare uses the correct SSL/TLS encryption mode.

1. Log in to Cloudflare.
2. Select your domain.
3. Navigate to **SSL/TLS → Overview**.
4. Select **Full (strict)** encryption mode.

Full (strict) encrypts traffic between visitors and Cloudflare, and between Cloudflare and your VPS. It also validates your origin server's SSL certificate.

Your VPS must have a valid Let's Encrypt certificate or another certificate trusted by Cloudflare.

> **Important:** Avoid Flexible SSL mode. It does not provide end-to-end HTTPS encryption and can cause redirect loops with your origin server.

Official documentation: https://developers.cloudflare.com/ssl/origin-configuration/ssl-modes/full-strict/

If you're not using Cloudflare, you can skip this section.

## 4. Verify Laravel Production Configuration

Connect to your VPS from your **local computer**:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

Switch to your CloudPanel site user:

```bash
sudo -iu SITE_USER
```

Navigate to your Laravel application:

```bash
cd ~/htdocs/example.com
```

Open your environment file:

```bash
nano .env
```

Make sure these values are configured:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com
```

- `APP_ENV=production` identifies the production environment.
- `APP_DEBUG=false` prevents Laravel from displaying detailed debugging information to visitors.
- `APP_URL` should use your actual HTTPS domain.

Also verify that your database credentials, mail settings, and other production environment variables are correct.

If you changed your `.env` file, rebuild Laravel's configuration cache:

```bash
php artisan optimize
```

> **Security:** Never enable `APP_DEBUG=true` on a public production website. Never commit your production `.env` file or `APP_KEY` to GitHub.

## 5. Verify Laravel's Public Directory

Laravel applications must use the `public` directory as the website's document root.

For example:

```text
/home/SITE_USER/htdocs/example.com/public
```

In CloudPanel:

1. Navigate to **Sites → Your Website → Manage**.
2. Open **Domain Settings**.
3. Check the **Root Directory**.
4. Make sure it points to the Laravel `public` directory.

This prevents visitors from directly accessing sensitive application files stored outside the public directory.

> **Important:** Never configure the website root to point directly to the Laravel project directory.

## 6. Verify File Permissions

Laravel needs write access to these directories:

```text
storage/
bootstrap/cache/
```

Check their permissions:

```bash
ls -ld storage bootstrap/cache
```

Make sure the appropriate site user and PHP process can write to these directories.

If permissions need adjustment, refer to the setup instructions in Chapter 07.

> **Security:** Never use `chmod -R 777` to fix Laravel permission problems.

## 7. Test Your Live Laravel Application

Open your website in a browser and test the important features.

For example:

1. Open the homepage.
2. Test login and registration.
3. Navigate between pages.
4. Verify database operations.
5. Test file uploads if your application supports them.
6. Check that Vue/Inertia frontend assets load correctly.

### Check Laravel Logs

If your application shows an error, inspect the Laravel log:

```bash
tail -n 50 storage/logs/laravel.log
```

The log file may not exist yet if Laravel hasn't written any entries.

Also check your browser's Developer Tools for failed network requests or missing frontend assets.

## 8. Final Security Checklist

Before making your website available to users, confirm:

- [ ] A valid SSL certificate is installed.
- [ ] HTTP redirects to HTTPS.
- [ ] Cloudflare uses Full (strict) if proxying is enabled.
- [ ] `APP_ENV=production` is configured.
- [ ] `APP_DEBUG=false`.
- [ ] Laravel's document root points to `public`.
- [ ] `.env` and private application files are not publicly accessible.
- [ ] Storage permissions are configured correctly.
- [ ] CloudPanel port **8443** remains restricted.
- [ ] The homepage, authentication, and main application features work.

Your Laravel application is now accessible through a secure HTTPS connection.

---

**Next:** [09 — Updating a Production Laravel Application](09-updating-production.md)
