
# Laravel Deployment Cheatsheet

A quick reference for commonly used commands when deploying and managing Laravel applications on an Ubuntu VPS with CloudPanel.

This cheatsheet covers SSH, Git, Laravel, Composer, Node.js, Nginx, Supervisor, Redis, and backups.

For step-by-step instructions, see the [Laravel Deployment Guide](README.md).

## Before You Start

Replace these placeholders with your actual values:

| Placeholder | Description |
| --- | --- |
| `SERVER_IP` | Your VPS public IP address |
| `SITE_USER` | Your CloudPanel website's Linux user |
| `example.com` | Your application domain |
| `example_db` | Your database name |
| `main` | Your production Git branch |

**Where to run commands:**

- **Local:** Windows Git Bash, macOS Terminal, or Linux Terminal.
- **VPS (root):** Server administration commands.
- **VPS (site user):** Laravel, Composer, npm, and application Git commands.

> **Note:** These examples assume your default `php` command uses a version compatible with your application. Check with `php -v`. CloudPanel may use a different PHP version for your website.

---

## 1. SSH — Server Access

### Connect to VPS

Run on your local computer:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

### Switch to CloudPanel Site User

Run on VPS as root:

```bash
sudo -iu SITE_USER
```

### Connect Directly as Site User

If your SSH public key has been added to that site user:

```bash
ssh -i ~/.ssh/laravel_vps SITE_USER@SERVER_IP
```

### Generate an SSH Key

Run on your local computer:

```bash
mkdir -p ~/.ssh

ssh-keygen -t ed25519 -f ~/.ssh/laravel_vps -C "laravel-vps"
```

Do not overwrite an existing SSH key without checking it first.

### Display Public Key

```bash
cat ~/.ssh/laravel_vps.pub
```

Never share your private key.

### Access CloudPanel Through SSH Tunnel

Run on your local computer:

```bash
ssh -i ~/.ssh/laravel_vps -N -L 8443:127.0.0.1:8443 root@SERVER_IP
```

Keep the terminal open and visit:

```text
https://localhost:8443
```

The SSH tunnel does not automatically block public access to port `8443`. Restrict that port through your firewall as explained in Chapter 03.

---

## 2. Server Information

Run on your VPS:

```bash
# Ubuntu version
lsb_release -a

# Current user
whoami

# Current directory
pwd

# Disk usage
df -h

# Memory usage
free -h

# Running processes
top

# Listening TCP ports
sudo ss -lntp

# PHP version
php -v

# Node.js version
node -v

# npm version
npm -v

# Composer version
composer --version
```

### Check Services

```bash
sudo systemctl status nginx
sudo systemctl status mysql
sudo systemctl status supervisor
```

Use the corresponding service name if your VPS uses a different database service.

---

## 3. Navigate to Laravel Project

Run as the CloudPanel site user:

```bash
cd ~/htdocs/example.com
```

Useful commands:

```bash
# List files
ls -la

# Show current directory
pwd

# Check directory ownership
ls -ld storage bootstrap/cache
```

Your Laravel application should normally be located at:

```text
/home/SITE_USER/htdocs/example.com
```

Your public website root should point to:

```text
/home/SITE_USER/htdocs/example.com/public
```

---

## 4. Git — Source Code Management

Run inside your Laravel project as the site user:

```bash
# Check working tree
git status

# Check current branch
git branch --show-current

# Show repository remote
git remote -v

# Get latest changes
git fetch origin

# Pull latest changes
git pull --ff-only

# Show latest commit
git log -1 --oneline

# Show recent commits
git log --oneline -5
```

### Switch Branch

```bash
git switch main
```

Only switch branches intentionally. Ensure the working tree is clean and the branch is suitable for production.

### Clone a Public Repository

```bash
git clone https://github.com/OWNER/REPO.git example.com
```

### Clone a Private Repository

If the server's GitHub Deploy Key is already configured:

```bash
GIT_SSH_COMMAND="ssh -i $HOME/.ssh/github_deploy -o IdentitiesOnly=yes" \
git clone git@github.com:OWNER/REPO.git example.com
```

See [06 — Deploy Laravel](docs/06-deploy-laravel.md) for the complete setup.

> **Important:** Avoid `git reset --hard` on production unless you have reviewed exactly which changes will be lost.

---

## 5. Composer — PHP Dependencies

Run inside your Laravel project:

```bash
# Check Composer
composer --version

# Install dependencies
composer install

# Install production dependencies
composer install --no-dev --optimize-autoloader

# Check dependency platform requirements
composer check-platform-reqs
```

For production deployments, prefer:

```bash
composer install --no-dev --optimize-autoloader
```

> Use `composer install`, not `composer update`, when deploying an application's existing, tested lockfile.

---

## 6. Node.js — Frontend Assets

For Laravel applications using Vue, React, Inertia.js, or Vite:

```bash
# Check versions
node -v
npm -v

# Install locked dependencies
npm ci

# Build production assets
npm run build
```

Typical Vite build output:

```text
public/build/
```

Check generated assets:

```bash
ls -la public/build
```

> `npm ci` requires a committed `package-lock.json`. A typical Laravel + Inertia application without SSR does not need a continuously running Node.js server.

---

## 7. Laravel Artisan Commands

Run inside your Laravel project as the site user.

### Application Information

```bash
# Laravel application details
php artisan about

# List registered routes
php artisan route:list

# Show available commands
php artisan list
```

### Application Setup

```bash
# Generate APP_KEY (new deployment only)
php artisan key:generate

# Create public storage link (if needed)
php artisan storage:link
```

Do not regenerate `APP_KEY` on an existing production application.

### Database Migrations

```bash
# Run pending migrations
php artisan migrate

# Check migration status
php artisan migrate:status

# Preview migration SQL
php artisan migrate --pretend
```

For non-interactive deployment automation:

```bash
php artisan migrate --force
```

> Review migrations and back up existing production data before running potentially destructive changes. Never run `migrate:fresh` on a live production database.

### Production Optimization

```bash
# Cache production configuration, routes, views, and events
php artisan optimize

# Clear configuration cache
php artisan config:clear

# Cache configuration again
php artisan config:cache
```

Use `php artisan optimize` after deploying code and finalizing `.env` settings.

### Maintenance Mode

```bash
# Enable maintenance mode
php artisan down

# Disable maintenance mode
php artisan up
```

Use maintenance mode for updates that may temporarily make the application unavailable.

---

## 8. Laravel Production Environment

Important `.env` values:

```dotenv
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
```

Additional variables depend on your application.

**Never:**

- Commit production `.env` files to GitHub.
- Expose `APP_KEY` or database passwords.
- Enable `APP_DEBUG=true` on a public production website.

---

## 9. Laravel Logs & Troubleshooting

### View Laravel Logs

```bash
tail -n 50 storage/logs/laravel.log
```

### Follow Laravel Logs in Real Time

```bash
tail -f storage/logs/laravel.log
```

The log filename may differ depending on your Laravel logging configuration.

### Check File Permissions

```bash
ls -ld storage bootstrap/cache
```

If appropriate ownership is already configured, ensure the required write permissions:

```bash
chmod -R ug+rwX storage bootstrap/cache
```

Never use `chmod -R 777` as a general fix.

---

## 10. Nginx & CloudPanel

### Test Nginx Configuration

Run as root or with sudo:

```bash
sudo nginx -t
```

### Check Nginx Status

```bash
sudo systemctl status nginx
```

### Reload Nginx

```bash
sudo systemctl reload nginx
```

### View Nginx Service Logs

```bash
sudo journalctl -u nginx -n 50
```

### Useful CloudPanel Locations

| Task | Location |
| --- | --- |
| Create PHP Site | Sites → Add Site |
| Change PHP Version | Sites → Manage → Settings |
| Change Document Root | Sites → Manage → Settings |
| Install SSL | Sites → Manage → SSL/TLS |
| Configure Cron Jobs | Sites → Manage → Cron Jobs |
| Edit Nginx Vhost | Sites → Manage → Vhost |
| Manage Databases | Sites → Manage → Databases |
| Configure Remote Backups | Admin Area → Backups |

CloudPanel should normally manage your site's Nginx configuration. Avoid modifying generated configuration files manually unless necessary.

---

## 11. Laravel Scheduler

### Show Scheduled Tasks

```bash
php artisan schedule:list
```

### Run Due Tasks Manually

```bash
php artisan schedule:run
```

### CloudPanel Cron Job

Configure a Cron Job to execute **every minute**:

```bash
cd /home/SITE_USER/htdocs/example.com && /usr/bin/php artisan schedule:run >> /dev/null 2>&1
```

Check the PHP path using:

```bash
command -v php
```

Use the correct PHP executable in the Cron command.

Only one Scheduler Cron Job is needed per Laravel application.

See [Advanced — Queue Workers & Scheduler](docs/advanced/01-queue-and-scheduler.md).

---

## 12. Laravel Queue Workers & Supervisor

### Test Queue Worker Manually

Run as the site user:

```bash
php artisan queue:work --tries=3
```

Press `Ctrl + C` to stop the manual worker.

### Restart Queue Workers After Deployment

```bash
php artisan queue:restart
```

### Failed Jobs

```bash
# List failed jobs
php artisan queue:failed

# Retry a specific job
php artisan queue:retry JOB_ID
```

Review the cause before retrying a failed job.

### Supervisor Commands

Run as root or with sudo:

```bash
# Check all workers
sudo supervisorctl status

# Reload changed Supervisor configurations
sudo supervisorctl reread
sudo supervisorctl update

# Restart Laravel workers
sudo supervisorctl restart laravel-worker:*

# Stop Laravel workers
sudo supervisorctl stop laravel-worker:*

# Start Laravel workers
sudo supervisorctl start laravel-worker:*
```

### View Queue Worker Logs

```bash
sudo tail -n 50 /home/SITE_USER/queue-worker.log
```

These commands assume the Supervisor program name used in the advanced queue chapter.

---

## 13. Laravel Reverb

If your application uses real-time WebSocket features:

```bash
# Restart Reverb gracefully (site user)
php artisan reverb:restart

# Check Supervisor status (root/sudo)
sudo supervisorctl status laravel-reverb

# Restart via Supervisor (root/sudo)
sudo supervisorctl restart laravel-reverb

# View Reverb logs
tail -n 50 /home/SITE_USER/reverb.log

# Check internal WebSocket port
ss -ltn | grep ':9000'
```

Our advanced guide uses port `9000` on `127.0.0.1`.

Do not expose the internal Reverb port publicly.

See [Advanced — Laravel Reverb](docs/advanced/02-laravel-reverb.md).

---

## 14. Redis

If your application uses Redis:

```bash
# Test Redis connection
redis-cli ping

# Check Redis service
sudo systemctl status redis-server

# Check memory usage
redis-cli INFO memory | grep used_memory_human

# Check listening port
sudo ss -lntp | grep ':6379'

# Check PHP Redis extension
php -m | grep -i redis
```

Expected result for `redis-cli ping`:

```text
PONG
```

Redis should only be accessible to trusted local services in our single-VPS setup.

Do not run `FLUSHALL` or `FLUSHDB` casually, especially when Redis is used for sessions or queues.

See [Advanced — Redis](docs/advanced/03-redis.md).

---

## 15. Database Backups

### Check Existing CloudPanel Backups

Run as the site user:

```bash
ls -lh ~/backups
```

### Export Database

```bash
mkdir -p ~/backups/manual
chmod 700 ~/backups/manual

cd ~/backups/manual

clpctl db:export --databaseName=example_db --file=example_db-backup.sql.gz
```

### Verify Backup File

```bash
ls -lh example_db-backup.sql.gz
```

### Import Database (Recovery Only)

```bash
clpctl db:import --databaseName=example_restore_db --file=example_db-backup.sql.gz
```

Import into a separate, empty test database first.

> **Warning:** Importing data into a database containing existing records can overwrite or conflict with those records. Verify the database name before executing an import.

### Back Up Uploaded Files

```bash
tar -czf ~/backups/manual/uploads-backup.tar.gz \
-C ~/htdocs/example.com storage/app/public
```

This example backs up Laravel's local public upload directory. Add other application-specific upload locations if needed.

Remember to keep a verified backup copy outside the VPS.

See [10 — Backups & Recovery](docs/10-backups.md).

---

## 16. DNS & HTTPS Testing

Run on your local computer.

### Check Domain DNS

```bash
nslookup example.com
```

### Check WWW DNS

```bash
nslookup www.example.com
```

### Check Nameservers

```bash
nslookup -type=NS example.com
```

### Test HTTP Redirect

```bash
curl -I http://example.com
```

### Test HTTPS

```bash
curl -I https://example.com
```

Verify that HTTP redirects to HTTPS and that your domain uses a valid SSL certificate.

If Cloudflare proxying is enabled, use **Full (strict)** SSL mode with a valid origin certificate.

---

## 17. Typical Production Update

Run these commands as your CloudPanel site user.

Before deploying, review the changes, check your backups, and decide whether maintenance mode is necessary.

```bash
# Navigate to application
cd ~/htdocs/example.com

# Check working tree
git status

# Pull latest code
git pull --ff-only

# Update PHP dependencies
composer install --no-dev --optimize-autoloader

# Build frontend assets (if needed)
npm ci
npm run build

# Run reviewed database migrations (if needed)
php artisan migrate

# Optimize Laravel
php artisan optimize

# Restart queue workers (if used)
php artisan queue:restart

# Restart Reverb (if used)
php artisan reverb:restart
```

Do not blindly run every optional command. Use only the services required by your application.

For major changes, enable maintenance mode before updating and disable it after verifying the deployment.

See [09 — Updating Production](docs/09-updating-production.md).

---

## 18. Common Problems

| Problem | What to Check |
| --- | --- |
| 500 Internal Server Error | Laravel logs, `.env`, PHP version, permissions |
| 502 Bad Gateway | PHP-FPM, Nginx, or reverse-proxy service |
| Database Connection Error | `DB_*` values, MySQL status, database credentials |
| Missing CSS / JavaScript | `npm run build`, `public/build` |
| Permission Denied | Ownership and permissions of `storage` and `bootstrap/cache` |
| Git Pull Failed | Branch, uncommitted changes, deploy key |
| SSL Certificate Error | DNS, port 80, domain name, certificate |
| Queue Jobs Not Processing | Queue connection, Supervisor, failed jobs |
| Reverb Not Connecting | Supervisor, Nginx proxy, Reverb port, WSS |
| Redis Connection Refused | Redis service, host, port, firewall |

For additional guidance, see [Troubleshooting](TROUBLESHOOTING.md).

---

## Security Reminders

- Use dedicated SSH keys and site users.
- Keep CloudPanel port `8443` restricted.
- Never expose MySQL port `3306`, Redis port `6379`, or internal Reverb port `9000` unnecessarily.
- Keep `APP_DEBUG=false` in production.
- Never commit `.env`, credentials, or SSH private keys.
- Test and back up production databases before destructive changes.
- Keep external backups of important application data.
- Review commands before running them on production.

---

## Official References

- [Laravel Documentation](https://laravel.com/docs)
- [CloudPanel Documentation](https://www.cloudpanel.io/docs/v2/)
- [CloudPanel CLI Commands](https://www.cloudpanel.io/docs/v2/cloudpanel-cli/site-user-commands/)
- [Git Documentation](https://git-scm.com/docs)
- [Supervisor Documentation](https://supervisord.org/)
- [Redis Documentation](https://redis.io/docs/latest/)

---

**Back to:** [Laravel Deployment Guide](README.md)

Made with ❤️ by [HiT Myanmar](https://hitmyanmar.com)
