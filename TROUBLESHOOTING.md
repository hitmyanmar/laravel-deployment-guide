
# Laravel Deployment — Troubleshooting Guide

This guide covers common problems when deploying and managing Laravel applications on an Ubuntu VPS using CloudPanel.

If your application isn't working correctly, use the sections below to identify the problem and find a solution.

For deployment instructions, see the [Laravel Deployment Guide](README.md).

## Before Troubleshooting

First, connect to your VPS from your local computer:

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

Replace `SERVER_IP`, `SITE_USER`, and `example.com` with your actual values.

### Basic Diagnostic Commands

Run these commands as your site user:

```bash
# Check Laravel application
php artisan about

# Check PHP version
php -v

# Check Git status
git status

# Check Laravel logs
tail -n 50 storage/logs/laravel.log

# Check file permissions
ls -ld storage bootstrap/cache
```

For server-level checks, use a root or sudo session:

```bash
# Check disk space
df -h

# Check memory
free -h

# Check Nginx
sudo systemctl status nginx

# Check MySQL
sudo systemctl status mysql
```

> **Important:** Always read error messages and logs before changing configurations. Avoid running unfamiliar commands directly on production.

---

## 1. 500 Internal Server Error

**Problem:** Your website displays:

```text
500 Internal Server Error
```

This usually means Laravel or PHP encountered an error while processing the request.

### Possible Causes

- Incorrect `.env` configuration.
- Missing Composer dependencies.
- Incorrect PHP version.
- File permission problems.
- Database connection failures.
- Application code errors.

### Solutions

**Step 1 — Check Laravel Logs**

```bash
tail -n 50 storage/logs/laravel.log
```

**Step 2 — Check Dependencies**

```bash
composer install --no-dev --optimize-autoloader
```

**Step 3 — Check PHP Version**

```bash
php -v
```

Make sure the PHP version is compatible with your application. Also check the site's PHP version in CloudPanel.

**Step 4 — Check Production Configuration**

Open `.env`:

```bash
nano .env
```

Verify:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com
```

If you've changed your environment settings, refresh the configuration cache:

```bash
php artisan optimize
```

> **Security:** Never enable `APP_DEBUG=true` on a public production website to troubleshoot errors. Use application logs instead.

If Laravel logs are unavailable, open **CloudPanel → Sites → Manage → Logs** to inspect PHP-FPM and Nginx errors.

---

## 2. 502 Bad Gateway

**Problem:** Your website displays:

```text
502 Bad Gateway
```

This usually means Nginx cannot communicate correctly with the PHP-FPM service or another configured backend.

### Solutions

Run these commands as root or with sudo.

**Check Nginx:**

```bash
sudo systemctl status nginx
```

**Check PHP-FPM:**

For example, if your website uses PHP 8.4:

```bash
sudo systemctl status php8.4-fpm
```

Replace `php8.4-fpm` with the service corresponding to your website's PHP version.

**Check Nginx Configuration:**

```bash
sudo nginx -t
```

**Check Server Resources:**

```bash
free -h
df -h
```

Also inspect your site's Nginx and PHP-FPM logs through CloudPanel.

If a service has stopped, investigate why it stopped before restarting it.

You can restart supported services through **CloudPanel → Admin Area → Instance → Services**.

> **Note:** A 502 error can also occur when a reverse proxy points to an incorrect port or a backend application isn't running.

---

## 3. 403 Forbidden or 404 Not Found

**Problem:** Your website displays a 403 or 404 error even though the Laravel application files exist.

### Possible Causes

- Incorrect website document root.
- Missing `public/index.php`.
- Incorrect Nginx configuration.
- Missing Laravel route.

### Solutions

**Step 1 — Verify Document Root**

In CloudPanel:

1. Navigate to **Sites → Your Website → Manage**.
2. Open **Settings → Domain Settings**.
3. Verify the Root Directory.

It should point to:

```text
/home/SITE_USER/htdocs/example.com/public
```

**Step 2 — Check Laravel Public Files**

```bash
ls -la public/
```

Make sure `index.php` exists.

**Step 3 — Check Laravel Routes**

```bash
php artisan route:list
```

Verify that the requested route is registered.

**Step 4 — Check Nginx Configuration**

```bash
sudo nginx -t
```

If you changed the Vhost configuration, review it in CloudPanel.

> **Important:** Never point your website root directly to the Laravel project directory. It must point to `public`.

---

## 4. CSS or JavaScript Not Loading

**Problem:** Your Laravel application loads, but CSS, JavaScript, or Vue/Inertia pages aren't working correctly.

You may also see:

```text
Vite manifest not found
```

### Possible Causes

- Frontend assets haven't been built.
- Node.js version is incompatible.
- Missing `public/build` directory.
- Incorrect Vite configuration.
- Browser caching.

### Solutions

**Step 1 — Check Node.js**

```bash
node -v
npm -v
```

**Step 2 — Install Dependencies**

```bash
npm ci
```

**Step 3 — Build Frontend Assets**

```bash
npm run build
```

**Step 4 — Verify Build Output**

```bash
ls -la public/build
```

For a typical Laravel Vite application, verify that `manifest.json` exists inside `public/build`.

Also check your browser's Developer Tools → Network tab for failed CSS or JavaScript requests.

> **Note:** Use `npm run build` for production. You normally don't need to run `npm run dev` on your production VPS.

---

## 5. Database Connection Error

**Problem:** Laravel displays an error such as:

```text
SQLSTATE[HY000] [1045] Access denied for user
```

or:

```text
SQLSTATE[HY000] [2002] Connection refused
```

### Possible Causes

- Incorrect database credentials.
- MySQL service isn't running.
- Incorrect database host or port.
- Database or database user doesn't exist.
- Database permissions are insufficient.

### Solutions

**Step 1 — Check MySQL**

As root or sudo:

```bash
sudo systemctl status mysql
```

**Step 2 — Check Laravel Database Configuration**

As your site user:

```bash
nano .env
```

Verify:

```dotenv
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=example_db
DB_USERNAME=example_user
DB_PASSWORD="YOUR_DATABASE_PASSWORD"
```

Compare these values with your database credentials in CloudPanel.

**Step 3 — Refresh Configuration**

```bash
php artisan config:clear
php artisan optimize
```

**Step 4 — Check Migrations**

```bash
php artisan migrate:status
```

If the database connection still fails, inspect the exact error and verify database access through CloudPanel.

> **Important:** Do not recreate the production database or reset its password without understanding how existing applications depend on it.

---

## 6. Permission Denied

**Problem:** Laravel cannot write cache, logs, sessions, or uploaded files.

Common errors include:

```text
Permission denied
```

```text
The stream or file could not be opened for writing
```

### Solutions

**Step 1 — Check Current User**

```bash
whoami
```

You should normally be working as the dedicated CloudPanel site user.

**Step 2 — Check Permissions**

```bash
ls -ld storage bootstrap/cache
```

**Step 3 — Ensure Appropriate Write Permissions**

If ownership and group access are already configured correctly:

```bash
chmod -R ug+rwX storage bootstrap/cache
```

If errors continue, inspect the directory ownership and PHP-FPM process user.

> **Security:** Never use `chmod -R 777` as a general solution. Fix the actual ownership or permission problem instead.

---

## 7. APP_KEY Missing or Application Key Error

**Problem:** Laravel reports:

```text
No application encryption key has been specified.
```

### Solutions

For a **brand-new application**, generate the key:

```bash
php artisan key:generate
```

Then refresh the configuration:

```bash
php artisan optimize
```

For an **existing production application**, restore the original `APP_KEY` from a secure backup if it is missing.

> **Warning:** Do not regenerate the application key on an existing production application. Changing it can make previously encrypted data and sessions unreadable.

---

## 8. 419 Page Expired or Login Session Problems

**Problem:** Users see:

```text
419 Page Expired
```

Or they cannot stay logged in after authentication.

### Possible Causes

- Session configuration problems.
- Incorrect application domain.
- Missing or expired CSRF token.
- Session storage permissions.
- Incorrect cookie settings.
- Session backend unavailable.

### Solutions

**Step 1 — Check Environment Configuration**

Open `.env`:

```bash
nano .env
```

Verify your application URL:

```dotenv
APP_URL=https://example.com
```

Also review your application's `SESSION_DRIVER` and any custom `SESSION_DOMAIN` or secure-cookie settings.

**Step 2 — Check Session Storage**

If using database sessions, verify that the required session table exists.

If using Redis sessions, confirm Redis is running.

If using file sessions, verify the storage directory permissions.

**Step 3 — Check Browser Cookies**

Use your browser's Developer Tools to check whether session cookies are being stored and sent correctly.

**Step 4 — Refresh Configuration**

After making necessary configuration changes:

```bash
php artisan optimize
```

> **Important:** Don't disable CSRF protection to fix a 419 error. Identify the session or token problem instead.

---

## 9. SSL Certificate or HTTPS Problems

**Problem:** Your browser displays:

```text
Your connection is not private
```

Or the website doesn't load over HTTPS.

### Solutions

**Step 1 — Check DNS**

On your local computer:

```bash
nslookup example.com
```

Make sure your DNS configuration is correct.

If you're using Cloudflare proxying, the returned IP may belong to Cloudflare rather than your VPS.

**Step 2 — Check SSL in CloudPanel**

1. Open **Sites → Your Website → Manage**.
2. Navigate to **SSL/TLS**.
3. Check the installed certificate.
4. Install a Let's Encrypt certificate if required.

**Step 3 — Check HTTPS**

```bash
curl -I https://example.com
```

**Step 4 — Check Firewall**

Make sure ports `80` and `443` are reachable.

If Cloudflare proxying is enabled, use **Full (strict)** with a valid origin certificate.

> **Note:** SSL issuance can fail if DNS records aren't correct or domain validation cannot reach your server.

See [08 — SSL & Production Security](docs/08-ssl-security.md).

---

## 10. Git Pull Failed

**Problem:** You cannot pull the latest application code from GitHub.

You may see:

```text
Permission denied (publickey)
```

or:

```text
fatal: Not possible to fast-forward
```

### Solutions

**Step 1 — Check Git Status**

```bash
git status
git branch --show-current
git remote -v
```

**Step 2 — Verify GitHub Access**

For a private repository using the Deploy Key from Chapter 06:

```bash
ssh -i ~/.ssh/github_deploy -o IdentitiesOnly=yes -T git@github.com
```

GitHub should confirm successful authentication, although it may return a nonzero exit code because shell access isn't provided.

**Step 3 — Review Git History**

```bash
git fetch origin
git log --oneline -5
```

If your branch has diverged, review the local and remote commits before deciding how to reconcile them.

> **Important:** Avoid `git reset --hard`, `git push --force`, or deleting your project directory without understanding the consequences.

See [09 — Updating Production](docs/09-updating-production.md).

---

## 11. Composer Install Failed

**Problem:** Composer reports missing PHP extensions, incompatible versions, or dependency errors.

### Solutions

**Check PHP and Composer:**

```bash
php -v
composer --version
```

**Check Platform Requirements:**

```bash
composer check-platform-reqs
```

**Check Required Extensions:**

```bash
php -m
```

Compare the output with your project's requirements.

Also verify the PHP version selected in CloudPanel.

> **Note:** CloudPanel can use a different PHP version for your website and your SSH terminal. Both environments must support the dependencies they execute.

Do not use `--ignore-platform-reqs` to hide production compatibility problems.

---

## 12. Queue Jobs Not Processing

**Problem:** Background tasks such as emails or notifications aren't being processed.

### Possible Causes

- Queue worker isn't running.
- Incorrect queue connection.
- Supervisor process stopped.
- Job failed.
- Queue configuration cache is outdated.

### Solutions

**Step 1 — Check Queue Configuration**

Review `QUEUE_CONNECTION` in `.env`.

For example:

```dotenv
QUEUE_CONNECTION=database
```

**Step 2 — Check Supervisor**

As root or sudo:

```bash
sudo supervisorctl status
```

**Step 3 — Check Failed Jobs**

As the site user:

```bash
php artisan queue:failed
```

**Step 4 — Restart Workers**

```bash
php artisan queue:restart
```

**Step 5 — Check Logs**

```bash
tail -n 50 ~/queue-worker.log
```

If using database queues, make sure the required database tables exist.

See [Advanced — Queue Workers & Scheduler](docs/advanced/01-queue-and-scheduler.md).

---

## 13. Laravel Scheduler Not Running

**Problem:** Scheduled commands aren't executing automatically.

### Solutions

**Step 1 — List Scheduled Tasks**

```bash
php artisan schedule:list
```

**Step 2 — Run the Scheduler Manually**

```bash
php artisan schedule:run
```

**Step 3 — Verify CloudPanel Cron**

Open:

**Sites → Your Website → Manage → Cron Jobs**

Make sure a Cron Job runs every minute.

Example:

```bash
cd /home/SITE_USER/htdocs/example.com && /usr/bin/php artisan schedule:run >> /dev/null 2>&1
```

Verify that the PHP path matches the version required by your application.

Also check the VPS timezone and the timezone configured for your scheduled tasks.

---

## 14. Laravel Reverb WebSocket Not Connecting

**Problem:** Real-time notifications, chat, or other WebSocket features aren't working.

### Solutions

**Step 1 — Check Reverb Process**

As root or sudo:

```bash
sudo supervisorctl status laravel-reverb
```

**Step 2 — Check Internal Port**

```bash
ss -ltn | grep ':9000'
```

In our guide, Reverb listens on:

```text
127.0.0.1:9000
```

**Step 3 — Check Environment Configuration**

Verify that your Reverb host, port, scheme, and credentials match the deployed configuration.

**Step 4 — Check Nginx Vhost**

Make sure the `/app/` and `/apps/` paths proxy to the correct internal Reverb port.

**Step 5 — Check Browser**

Open Developer Tools → Network → WS.

Look for a secure WebSocket connection:

```text
wss://example.com/app/YOUR_APP_KEY
```

A successful handshake returns HTTP status `101`.

If connections succeed but events don't arrive, also check broadcasting authorization and queue workers.

See [Advanced — Laravel Reverb](docs/advanced/02-laravel-reverb.md).

---

## 15. Redis Connection Refused

**Problem:** Laravel cannot connect to Redis.

### Solutions

**Check Redis Service:**

```bash
sudo systemctl status redis-server
```

**Test Redis:**

```bash
redis-cli ping
```

For a local instance without authentication, the expected result is:

```text
PONG
```

**Check Listening Port:**

```bash
sudo ss -lntp | grep ':6379'
```

Verify your Laravel `.env`:

```dotenv
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

If Redis uses authentication, configure the appropriate credentials.

Make sure the correct PhpRedis extension or Predis package is available.

> **Security:** Don't expose Redis publicly or disable authentication just to fix connection problems.

See [Advanced — Redis](docs/advanced/03-redis.md).

---

## 16. Cannot Access CloudPanel

**Problem:** You cannot open CloudPanel on port `8443`.

### Solutions

If you're using the SSH tunnel method from Chapter 04, run this on your local computer:

```bash
ssh -i ~/.ssh/laravel_vps -N -L 8443:127.0.0.1:8443 root@SERVER_IP
```

Keep the terminal open.

Then visit:

```text
https://localhost:8443
```

If it fails:

1. Verify SSH access to the VPS.
2. Make sure another local application isn't already using port `8443`.
3. Check whether the CloudPanel service is running on the VPS.
4. Review server and provider firewall settings.

> **Security:** Don't expose port `8443` publicly just to troubleshoot CloudPanel. Use the SSH tunnel or another secured administration method.

---

## 17. Website Suddenly Becomes Slow

**Problem:** Your application is much slower than usual.

### Possible Causes

- High CPU usage.
- Insufficient RAM.
- Disk space running low.
- Slow database queries.
- Too many background workers.
- Redis or other services consuming memory.

### Solutions

Run on your VPS:

```bash
# Check CPU and running processes
top

# Check memory
free -h

# Check disk usage
df -h

# Check active services
sudo systemctl status nginx
sudo systemctl status mysql
```

For applications using Redis:

```bash
redis-cli INFO memory | grep used_memory_human
```

Also review Laravel logs, database performance, and queue worker activity.

> **Note:** Don't immediately upgrade your VPS without identifying the actual bottleneck. A slow query or poorly configured worker can cause problems even on a larger server.

---

## Quick Troubleshooting Reference

| Problem | First Thing to Check |
| --- | --- |
| 500 Internal Server Error | Laravel and PHP logs |
| 502 Bad Gateway | PHP-FPM and Nginx |
| 403 Forbidden | Document root and permissions |
| 404 Not Found | Laravel routes and Vhost |
| Missing CSS / JS | `npm run build` |
| Database Error | `.env` and MySQL |
| Permission Denied | File ownership and permissions |
| Missing APP_KEY | Existing `.env` and key backup |
| 419 Page Expired | Sessions, cookies, CSRF |
| SSL Error | DNS and CloudPanel SSL |
| Git Pull Failed | SSH key, branch, Git status |
| Composer Error | PHP version and extensions |
| Queue Not Working | Supervisor and failed jobs |
| Scheduler Not Running | CloudPanel Cron Job |
| Reverb Not Connecting | Supervisor, Nginx, WebSocket port |
| Redis Error | Redis service and connection |
| CloudPanel Not Accessible | SSH tunnel and port 8443 |
| Website Slow | CPU, RAM, disk and database |

---

## Before Asking for Help

If you still can't resolve an issue, collect the following information:

1. The exact error message.
2. The command or action that caused it.
3. Your Laravel and PHP versions.
4. The relevant application or service log lines.
5. What you changed before the problem started.

Avoid sharing passwords, API tokens, database credentials, private SSH keys, or your production `.env` contents.

When asking for help on GitHub, provide a minimal example or sanitized log output.

---

## Final Security Reminders

- Keep `APP_DEBUG=false` in production.
- Never expose your `.env` file.
- Never use `chmod -R 777` as a general permission fix.
- Avoid destructive database commands on production.
- Don't regenerate an existing production `APP_KEY`.
- Don't expose Redis or internal WebSocket ports publicly.
- Back up important data before major changes.
- Review logs before restarting or changing services.

---

## Official References

- [Laravel Documentation](https://laravel.com/docs)
- [CloudPanel Documentation](https://www.cloudpanel.io/docs/v2/)
- [CloudPanel Logs](https://www.cloudpanel.io/docs/v2/frontend-area/logs/)
- [Laravel Deployment](https://laravel.com/docs/12.x/deployment)
- [Laravel Queues](https://laravel.com/docs/12.x/queues)
- [Laravel Reverb](https://laravel.com/docs/12.x/reverb)

---

**Back to:** [Laravel Deployment Guide](README.md)

Made with ❤️ by [HiT Myanmar](https://hitmyanmar.com)
