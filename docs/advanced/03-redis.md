
# 03 — Redis for Laravel

Redis is a fast, in-memory data store commonly used by Laravel applications for caching, sessions, and background queues.

Using Redis can reduce database queries and improve application performance.

In this guide, we'll configure Redis on our Ubuntu VPS and connect it to an existing Laravel application.

## Prerequisites

- Laravel application deployed and working.
- CloudPanel installed and configured.
- SSH access to the VPS.
- Access to your application's production `.env` file.

> **Note:** Redis is optional. For small Laravel applications, database or file-based cache, session, and queue drivers may be sufficient.

Official documentation:
- https://redis.io/docs/latest/
- https://laravel.com/docs/redis

---

## 1. Check Whether Redis Is Installed

CloudPanel supports Redis, so it may already be installed on your VPS.

Connect from your local computer:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

Check the Redis service:

```bash
systemctl status redis-server
```

Test the connection:

```bash
redis-cli ping
```

If Redis is running correctly, you should see:

```text
PONG
```

If Redis is installed and working, skip to **Step 3**.

## 2. Install Redis (If Needed)

If Redis is not installed, you can install it from Ubuntu's package repositories.

Run these commands as root or a user with sudo privileges:

```bash
sudo apt update
sudo apt install -y redis-server
```

Enable and start Redis:

```bash
sudo systemctl enable --now redis-server
```

Verify the installation:

```bash
redis-cli ping
```

Expected output:

```text
PONG
```

> **Important:** If your VPS already has a Redis instance managed by CloudPanel or another application, do not replace its installation or configuration without reviewing the existing setup.

## 3. Secure Redis

Redis should not be publicly accessible from the internet.

For our single-VPS setup, Laravel and Redis will communicate locally.

### Check Redis Configuration

The standard Redis configuration file on Ubuntu is:

```text
/etc/redis/redis.conf
```

Open it as root:

```bash
sudo nano /etc/redis/redis.conf
```

Verify these settings:

```text
bind 127.0.0.1
protected-mode yes
port 6379
```

If the settings are already configured correctly, no changes are needed.

If you changed the configuration, restart Redis:

```bash
sudo systemctl restart redis-server
```

Verify that Redis listens locally:

```bash
sudo ss -lntp | grep ':6379'
```

The listening address should be `127.0.0.1:6379`, not a public IP or `0.0.0.0:6379`.

> **Security:**
>
> - Never publicly expose port `6379`.
> - Do not allow port `6379` through your VPS provider's public firewall.
> - Keep Redis protected mode enabled.
> - Do not disable Redis security protections just to resolve connection errors.

This guide assumes one trusted VPS running the application and Redis together.

If multiple untrusted applications or site users share the same VPS, configure Redis authentication and access controls. Different Redis database numbers are not a security boundary.

Official reference: https://redis.io/docs/latest/management/security/

---

## 4. Check Laravel Redis Support

Laravel needs a PHP Redis client to communicate with Redis.

Two common options are:

- **PhpRedis:** A PHP extension, recommended for performance.
- **Predis:** A PHP package that doesn't require a PHP extension.

### Option A — Use PhpRedis

Switch to your CloudPanel site user:

```bash
sudo -iu SITE_USER
cd ~/htdocs/example.com
```

Check whether the PHP Redis extension is available:

```bash
php -m | grep -i redis
```

If the output contains `redis`, the extension is available for that PHP CLI version.

Configure your Laravel `.env`:

```dotenv
REDIS_CLIENT=phpredis
```

> **Important:** CloudPanel may use different PHP versions for CLI and PHP-FPM. Make sure the Redis extension is also enabled for your website's PHP version.

### Option B — Use Predis

If PhpRedis isn't available, Predis is an alternative.

In your **local development environment**, install Predis:

```bash
composer require predis/predis
```

Commit and push the updated `composer.json` and `composer.lock` files to GitHub.

Then deploy the changes to your VPS using Chapter 09.

Configure `.env`:

```dotenv
REDIS_CLIENT=predis
```

> **Recommendation:** Use PhpRedis if it's already available. Predis is a convenient alternative when you want to avoid installing additional PHP extensions.

## 5. Configure Redis Connection

Open your Laravel `.env` file:

```bash
nano .env
```

Configure the Redis connection:

```dotenv
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

REDIS_DB=0
REDIS_CACHE_DB=1
REDIS_PREFIX=example_production_
```

These settings connect Laravel to Redis running on the same VPS.

| Variable | Purpose |
|---|---|
| `REDIS_HOST` | Redis server address |
| `REDIS_PORT` | Redis server port |
| `REDIS_DB` | Default Redis database |
| `REDIS_CACHE_DB` | Redis database for Laravel cache |
| `REDIS_PREFIX` | Prefix to distinguish application keys |

The `REDIS_PASSWORD=null` example assumes your local Redis instance has no authentication configured.

If Redis requires authentication, configure the appropriate credentials instead.

> **Important:** Use a unique Redis prefix for each Laravel application or environment. This helps prevent applications from accidentally sharing cache keys or processing each other's queues.

---

## 6. Use Redis for Laravel Cache

Redis is commonly used to store temporary application data, such as frequently accessed records or expensive query results.

Open `.env` and set:

```dotenv
CACHE_STORE=redis
```

Keep your existing session and queue drivers unchanged unless you want to move them to Redis.

Refresh Laravel's configuration:

```bash
php artisan optimize
```

### Test Laravel Cache

Open Laravel Tinker:

```bash
php artisan tinker
```

Store a temporary value:

```php
cache()->put('redis-guide-test', 'Hello Redis', 60);
```

Retrieve it:

```php
cache()->get('redis-guide-test');
```

Expected result:

```text
"Hello Redis"
```

Remove the test value:

```php
cache()->forget('redis-guide-test');
```

Exit Tinker:

```php
exit
```

Laravel is now using Redis for application caching.

> **Note:** Do not use Redis commands such as `FLUSHALL` or `FLUSHDB` to clear Laravel's cache. These commands may delete session, queue, or other application data stored in Redis.

For Laravel cache management, use the appropriate Artisan commands and review their effect on shared cache stores.

---

## 7. Use Redis for Laravel Sessions (Optional)

Laravel can also store user sessions in Redis instead of the database or filesystem.

To enable Redis sessions, update `.env`:

```dotenv
SESSION_DRIVER=redis
SESSION_CONNECTION=default
```

Refresh Laravel's configuration:

```bash
php artisan optimize
```

New sessions will now be stored using the configured Redis connection.

> **Important:** Changing the session driver on a running production application can log out existing users because their previous sessions are not automatically migrated. Plan this change carefully.

You can continue using database sessions if your application does not need Redis sessions.

---

## 8. Use Redis for Laravel Queues (Optional)

Redis can also store Laravel's background jobs.

If your application already uses the database queue driver, you can optionally switch to Redis.

Update `.env`:

```dotenv
QUEUE_CONNECTION=redis
```

Refresh Laravel's configuration:

```bash
php artisan optimize
```

### Restart Queue Workers

If you configured Supervisor in the previous advanced chapter, restart the workers:

```bash
php artisan queue:restart
```

If you changed the queue connection, also restart the Supervisor-managed workers from a root or sudo session:

```bash
sudo supervisorctl restart laravel-worker:*
```

This ensures workers start with the new Redis queue configuration.

> **Important:** Changing from database queues to Redis does not automatically transfer pending jobs. Before switching, allow existing jobs to finish or plan a safe migration.

### Verify Queue Workers

Check Supervisor status:

```bash
sudo supervisorctl status
```

Your queue workers should show `RUNNING`.

If you haven't configured Supervisor yet, follow:

[01 — Laravel Queue Workers & Scheduler](01-queue-and-scheduler.md)

> **Note:** Redis queues require a working background worker. Setting `QUEUE_CONNECTION=redis` alone does not process jobs.

---

## 9. Monitor Redis

Redis stores data in memory, so monitoring memory usage is important.

Check Redis connectivity:

```bash
redis-cli ping
```

Check memory usage:

```bash
redis-cli INFO memory | grep used_memory_human
```

Check Redis service status:

```bash
sudo systemctl status redis-server
```

You can also inspect Redis logs:

```bash
sudo journalctl -u redis-server -n 50
```

> **Important:** Redis cache data is generally disposable, but sessions and queued jobs may not be. If Redis is used for important queue workloads, configure appropriate persistence, memory limits, monitoring, and recovery procedures. Avoid eviction policies that could silently discard queued jobs.

Redis should not replace your application's primary MySQL database or its backup strategy.

---

## 10. Final Checklist

Before finishing, verify:

- [ ] Redis is installed and running.
- [ ] `redis-cli ping` returns `PONG`.
- [ ] Redis listens only on a trusted local interface.
- [ ] Port `6379` is not publicly accessible.
- [ ] Laravel has a working PhpRedis or Predis client.
- [ ] Redis environment variables are configured correctly.
- [ ] Unique Redis prefixes are configured where necessary.
- [ ] Laravel caching works with Redis.
- [ ] Sessions work if Redis sessions are enabled.
- [ ] Queue workers run correctly if Redis queues are enabled.
- [ ] Redis memory usage is monitored.

Your Laravel application can now use Redis for caching, sessions, and background jobs.

---

## Quick Command Reference

```bash
# Test Redis
redis-cli ping

# Check Redis service
sudo systemctl status redis-server

# Check Redis memory
redis-cli INFO memory | grep used_memory_human

# Check PHP Redis extension
php -m | grep -i redis

# Refresh Laravel configuration
php artisan optimize

# Restart Laravel queue workers
php artisan queue:restart
```

---

**Back to:** [Laravel Deployment Guide](../../README.md)

