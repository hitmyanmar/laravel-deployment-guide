
# 02 — Laravel Reverb (WebSockets)

Laravel Reverb provides real-time WebSocket communication for Laravel applications.

It can be used for features such as:

- Real-time notifications.
- Live chat.
- Live order updates.
- Real-time dashboards.

In this guide, we'll configure Laravel Reverb on an Ubuntu VPS using **CloudPanel, Nginx, and Supervisor**.

## Prerequisites

- Laravel application deployed and working.
- HTTPS configured for your domain.
- SSH access to the VPS.
- Supervisor installed (see the previous advanced chapter).
- Laravel broadcasting configured in your application.

Official documentation: https://laravel.com/docs/12.x/reverb

---

## 1. Install Laravel Reverb

If your application already uses Laravel Reverb, skip this step.

Otherwise, install Reverb **in your development environment**:

```bash
php artisan install:broadcasting --reverb
```

This installs the required broadcasting packages and configuration.

Commit and push the changes to GitHub, then deploy them to your VPS using Chapter 09.

> **Note:** Avoid installing new packages directly on the production server. Install and test them during development, then deploy the updated `composer.lock`, `package-lock.json`, and application files.

## 2. Configure Reverb for Production

Connect to your VPS from your **local computer**:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

Switch to your CloudPanel site user:

```bash
sudo -iu SITE_USER
cd ~/htdocs/example.com
```

Open your environment file:

```bash
nano .env
```

Configure Reverb:

```dotenv
BROADCAST_CONNECTION=reverb

REVERB_APP_ID=YOUR_APP_ID
REVERB_APP_KEY=YOUR_APP_KEY
REVERB_APP_SECRET=YOUR_APP_SECRET

REVERB_SERVER_HOST=127.0.0.1
REVERB_SERVER_PORT=9000

REVERB_HOST=example.com
REVERB_PORT=443
REVERB_SCHEME=https

VITE_REVERB_APP_KEY="${REVERB_APP_KEY}"
VITE_REVERB_HOST="${REVERB_HOST}"
VITE_REVERB_PORT="${REVERB_PORT}"
VITE_REVERB_SCHEME="${REVERB_SCHEME}"
```

Replace the example domain and credentials with your actual values.

Use securely generated Reverb application credentials. If you already have credentials from a working Reverb installation, preserve them rather than generating different ones unnecessarily.

### Understanding the Configuration

| Variable | Purpose |
| --- | --- |
| `REVERB_SERVER_HOST` | IP address where Reverb listens |
| `REVERB_SERVER_PORT` | Internal Reverb port |
| `REVERB_HOST` | Public hostname used to reach Reverb |
| `REVERB_PORT` | Public HTTPS/WebSocket port |
| `REVERB_SCHEME` | Public connection scheme |

For our configuration:

```text
Browser (wss://example.com)
          |
          v
    Nginx (HTTPS:443)
          |
          v
    Reverb (127.0.0.1:9000)
```

Nginx handles the SSL certificate, so Reverb itself does not need a separate SSL certificate.

> **Security:** We use `127.0.0.1` so Reverb only accepts local connections. Do not publicly expose port `9000` through your VPS firewall or provider firewall.

After saving `.env`, refresh Laravel's configuration cache:

```bash
php artisan optimize
```

If your frontend uses Laravel Echo with Vite, rebuild the assets to include the production WebSocket configuration:

```bash
npm run build
```

## 3. Test Laravel Reverb

Before configuring Supervisor, verify that Reverb starts successfully.

Run the following command as your **site user**:

```bash
php artisan reverb:start --host=127.0.0.1 --port=9000
```

If Reverb starts successfully, it should remain running and wait for WebSocket connections.

Press `Ctrl + C` to stop it after testing.

> **Note:** Port `9000` is an example. If another service already uses this port, choose an available local port and update the Reverb and Nginx configurations consistently.

## 4. Configure Nginx Reverse Proxy

We need Nginx to forward WebSocket requests from our HTTPS domain to Laravel Reverb.

Laravel Reverb handles two important paths:

- `/app/` — WebSocket connections.
- `/apps/` — Broadcasting API requests.

### Step 1 — Open CloudPanel Vhost Editor

1. Log in to CloudPanel.
2. Navigate to **Sites → Your Website → Manage**.
3. Open **Vhost**.
4. Back up the existing Vhost configuration before editing.

### Step 2 — Add WebSocket Proxy Configuration

Inside the existing HTTPS `server` block for your Laravel website, add these two Nginx locations:

```nginx
location ^~ /app/ {
    proxy_pass http://127.0.0.1:9000;
    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}

location ^~ /apps/ {
    proxy_pass http://127.0.0.1:9000;
    proxy_http_version 1.1;

    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "upgrade";
    proxy_set_header Host $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

    proxy_read_timeout 3600s;
    proxy_send_timeout 3600s;
}
```

**Important:**

- Add these locations inside the HTTPS-enabled server block, not outside it.
- Do not replace the entire existing Vhost configuration.
- Keep the existing Laravel PHP configuration and `location /` block.
- Do not create duplicate location blocks if your Vhost already contains them.
- Use the same internal port as `REVERB_SERVER_PORT`.

Save the configuration.

CloudPanel validates Nginx Vhost syntax when saving.

If the configuration fails validation, review the error and restore the original configuration if necessary.

Official documentation: https://www.cloudpanel.io/docs/v2/frontend-area/vhost/

## 5. Configure Supervisor

Reverb is a long-running service.

We need Supervisor to keep it running automatically, including after server restarts.

### Step 1 — Create Supervisor Configuration

Connect to the VPS as root or a user with sudo privileges.

Create the configuration file:

```bash
sudo nano /etc/supervisor/conf.d/laravel-reverb.conf
```

Add the following:

```ini
[program:laravel-reverb]
process_name=%(program_name)s
command=/usr/bin/php /home/SITE_USER/htdocs/example.com/artisan reverb:start --host=127.0.0.1 --port=9000
directory=/home/SITE_USER/htdocs/example.com
user=SITE_USER

autostart=true
autorestart=true
startsecs=5

redirect_stderr=true
stdout_logfile=/home/SITE_USER/reverb.log

stopasgroup=true
killasgroup=true
stopwaitsecs=30
```

Replace:

- `SITE_USER` with your CloudPanel site username.
- `example.com` with your application's directory.
- `/usr/bin/php` with the correct PHP executable path if different.

To check your PHP executable:

```bash
command -v php
```

Make sure this executable uses a PHP version compatible with your application.

> **Important:** Run Reverb as the dedicated CloudPanel site user, not as root.

### Step 2 — Start Reverb with Supervisor

After saving the file, run:

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

Check the status:

```bash
sudo supervisorctl status
```

Expected output:

```text
laravel-reverb    RUNNING
```

Supervisor will automatically restart Reverb if the process stops unexpectedly.

## 6. Configure Laravel Queue Workers

Laravel broadcasting events commonly use the queue system.

If your application uses events implementing `ShouldBroadcast`, you need a working queue connection and queue worker.

For example, your `.env` might contain:

```dotenv
QUEUE_CONNECTION=database
```

Ensure that the required queue database tables exist and that your queue worker is running.

If you haven't configured queue workers, follow:

[01 — Laravel Queue Workers & Scheduler](01-queue-and-scheduler.md)

> **Note:** Events implementing `ShouldBroadcastNow` are broadcast immediately and do not require a queue worker for that event.

## 7. Verify WebSocket Connection

Open your application in a browser.

1. Open **Developer Tools**.
2. Select the **Network** tab.
3. Filter requests by **WS** (WebSocket).
4. Refresh your application.
5. Look for a connection to your domain.

The WebSocket URL should look similar to:

```text
wss://example.com/app/YOUR_APP_KEY
```

A successful WebSocket handshake should return:

```text
101 Switching Protocols
```

If the connection succeeds but real-time updates are not received, check your Laravel broadcasting events, channel authorization, and queue workers.

### Check Reverb Logs

On the VPS:

```bash
tail -n 50 /home/SITE_USER/reverb.log
```

### Check Supervisor Status

```bash
sudo supervisorctl status laravel-reverb
```

### Check Internal Listening Port

```bash
ss -ltn | grep ':9000'
```

You should see Reverb listening on `127.0.0.1:9000`.

If the browser reports a `502 Bad Gateway`, verify that Reverb is running and that the Nginx proxy port matches the Reverb port.

## 8. Restart Reverb After Deployment

Reverb is a long-running PHP process, so it needs to be restarted after relevant application or configuration changes.

As the CloudPanel site user, run:

```bash
php artisan reverb:restart
```

This gracefully terminates the existing Reverb process.

Supervisor will automatically start it again.

For a manual restart using Supervisor:

```bash
sudo supervisorctl restart laravel-reverb
```

Run Supervisor commands as root or a user with the required permissions.

> **Note:** Restarting Reverb temporarily disconnects connected clients. Laravel Echo clients normally attempt to reconnect automatically.

## 9. Production Security

Before using Reverb in production, verify the following:

- Reverb listens only on `127.0.0.1`.
- Port `9000` is not publicly exposed.
- The website has a valid SSL certificate.
- Browser connections use `wss://`.
- Reverb credentials are stored securely in `.env`.
- The application secret is never exposed through `VITE_` variables.
- Private and presence channels use proper Laravel authorization.

You should also restrict Reverb's allowed origins in `config/reverb.php` to your trusted domains rather than allowing all origins.

For example:

```php
'allowed_origins' => ['example.com'],
```

Make this change in your application source code, test it, and deploy it through GitHub.

## 10. Final Checklist

Before finishing, confirm:

- [ ] Laravel Reverb is installed.
- [ ] Production environment variables are configured.
- [ ] Reverb starts successfully on `127.0.0.1:9000`.
- [ ] Nginx proxies `/app/` and `/apps/` to Reverb.
- [ ] Supervisor keeps Reverb running.
- [ ] Queue workers are running if required.
- [ ] Frontend assets include the correct Reverb configuration.
- [ ] Browser WebSocket connections use `wss://`.
- [ ] Real-time events are received successfully.
- [ ] Reverb restarts after deployment.

Your Laravel application can now support real-time communication using Laravel Reverb.

---

## Quick Command Reference

```bash
# Check Reverb status
sudo supervisorctl status laravel-reverb

# Restart Reverb gracefully (site user)
php artisan reverb:restart

# Restart using Supervisor (root/sudo)
sudo supervisorctl restart laravel-reverb

# View Reverb logs
tail -n 50 /home/SITE_USER/reverb.log

# Check the listening port
ss -ltn | grep ':9000'

# Refresh Laravel configuration (site user)
php artisan optimize
```

---

**Back to:** [Laravel Deployment Guide](../../README.md)
