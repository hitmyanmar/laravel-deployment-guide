
# 01 — Laravel Queue Workers & Scheduler

Some Laravel applications need to execute scheduled tasks or process background jobs, such as sending emails, generating reports, or processing notifications.

CloudPanel provides Cron Job management, but long-running Laravel queue workers need an additional process manager such as **Supervisor**.

This guide explains how to configure both services on an Ubuntu VPS.

## Prerequisites

- Laravel application deployed and working.
- CloudPanel installed and configured.
- SSH access to the VPS.
- Laravel production environment configured.

> **Note:** The Laravel Scheduler and Queue Workers are independent. You can configure either one or both depending on your application's requirements.

---

## 1. Laravel Scheduler (CloudPanel Cron Jobs)

Laravel's Scheduler allows applications to execute tasks automatically at specified intervals.

For example:

- Sending daily reports.
- Cleaning expired records.
- Running scheduled notifications.
- Executing periodic maintenance tasks.

Laravel only requires **one Cron Job** to manage all scheduled tasks.

### Step 1 — Check Your Laravel Schedule

Connect to your VPS and switch to the CloudPanel site user:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

```bash
sudo -iu SITE_USER
cd ~/htdocs/example.com
```

List your application's scheduled tasks:

```bash
php artisan schedule:list
```

This displays the scheduled commands and their next execution times.

You can also manually execute due tasks:

```bash
php artisan schedule:run
```

### Step 2 — Configure Cron in CloudPanel

1. Log in to CloudPanel.
2. Navigate to **Sites → Your Website → Manage**.
3. Open **Cron Jobs**.
4. Add a new Cron Job.
5. Select the **Every Minute** template.
6. Enter the following command:

```bash
cd /home/SITE_USER/htdocs/example.com && /usr/bin/php artisan schedule:run >> /dev/null 2>&1
```

Replace `SITE_USER` and `example.com` with your actual values.

The command uses an absolute PHP path because Cron may have a different environment from your interactive terminal.

You can check your PHP path on the VPS:

```bash
command -v php
```

If it is different from `/usr/bin/php`, update the Cron command accordingly. Make sure the PHP version is compatible with your application.

Save the Cron Job.

CloudPanel will now execute Laravel's scheduler every minute using the site's Cron configuration.

> **Important:** Create only one scheduler Cron Job per Laravel application. Your scheduled tasks themselves are managed inside Laravel.

### Step 3 — Verify the Scheduler

Run:

```bash
php artisan schedule:list
```

Verify that your scheduled tasks are listed correctly.

For troubleshooting, you can temporarily replace `/dev/null` with an appropriate log file to capture the Cron output.

Official documentation:

- https://www.cloudpanel.io/docs/v2/frontend-area/cron-jobs/
- https://laravel.com/docs/scheduling

---

## 2. Laravel Queue Workers

Laravel queues allow you to process tasks in the background instead of making users wait for them to finish.

For example:

- Sending emails.
- Processing uploaded images.
- Generating PDF reports.
- Processing notifications.
- Importing large datasets.

Laravel provides the `queue:work` command, but it stops when the terminal closes.

For production applications, we'll use **Supervisor** to keep queue workers running automatically.

### Step 1 — Configure Laravel Queue Connection

Connect to your VPS as the CloudPanel site user and navigate to your Laravel application.

Open your `.env` file:

```bash
nano .env
```

For a simple application, you can use Laravel's database queue driver:

```dotenv
QUEUE_CONNECTION=database
```

Make sure your database configuration is working.

Laravel needs a `jobs` table for database queues. Most newer Laravel applications already include the required migration.

If your application does not have it, create the migration during development and deploy it:

```bash
php artisan make:queue-table
php artisan migrate
```

Do not create a duplicate migration if the table already exists.

After changing your production environment, refresh Laravel's configuration cache:

```bash
php artisan optimize
```

> **Note:** If you use `QUEUE_CONNECTION=sync`, jobs run immediately and a background worker is not required. The following steps are intended for asynchronous queue connections such as `database` or `redis`.

### Step 2 — Test Queue Worker

Before installing Supervisor, test the worker manually:

```bash
php artisan queue:work --tries=3
```

This starts a worker that processes queued jobs.

If the application has pending jobs, they should begin processing.

Press `Ctrl + C` to stop the worker.

> **Important:** Do not keep manually started queue workers running alongside Supervisor unless you intentionally need additional workers.

---

## 3. Install Supervisor

Supervisor is a process manager that keeps background applications running.

It can automatically restart Laravel queue workers if they stop or the server reboots.

### Step 1 — Install Supervisor

Connect to your VPS as root or a user with sudo privileges.

```bash
sudo apt update
sudo apt install -y supervisor
```

Enable and start Supervisor:

```bash
sudo systemctl enable --now supervisor
```

Verify that it is running:

```bash
sudo systemctl status supervisor
```

### Step 2 — Create Queue Worker Configuration

Create a Supervisor configuration file:

```bash
sudo nano /etc/supervisor/conf.d/laravel-worker.conf
```

Add the following configuration:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=/usr/bin/php /home/SITE_USER/htdocs/example.com/artisan queue:work --sleep=3 --tries=3 --timeout=60 --max-time=3600
directory=/home/SITE_USER/htdocs/example.com
autostart=true
autorestart=true
numprocs=1
user=SITE_USER
redirect_stderr=true
stdout_logfile=/home/SITE_USER/queue-worker.log
stopasgroup=true
killasgroup=true
stopwaitsecs=3600
```

Replace:

- `SITE_USER` — Your CloudPanel site username.
- `example.com` — Your Laravel application's directory.
- `/usr/bin/php` — Your PHP executable path, if different.

Start with `numprocs=1` for a small application. You can increase the number of workers later if needed.

**Important:**

- The worker must run as the dedicated CloudPanel site user, not root.
- Ensure the configured PHP version matches your application.
- The worker's `--timeout` must be shorter than the queue connection's `retry_after` setting to reduce the risk of duplicate processing.
- `stopwaitsecs` should exceed the duration of your longest-running job.

### Step 3 — Start Queue Workers

After saving the Supervisor configuration, run:

```bash
sudo supervisorctl reread
sudo supervisorctl update
```

Check the worker status:

```bash
sudo supervisorctl status
```

Expected output will look similar to:

```text
laravel-worker:laravel-worker_00    RUNNING
```

Supervisor will now keep the Laravel queue worker running and restart it when necessary.

---

## 4. Managing Queue Workers

You can manage workers using Supervisor.

**Check worker status:**

```bash
sudo supervisorctl status
```

**Stop the worker:**

```bash
sudo supervisorctl stop laravel-worker:*
```

**Start the worker:**

```bash
sudo supervisorctl start laravel-worker:*
```

**Restart the worker:**

```bash
sudo supervisorctl restart laravel-worker:*
```

**Check worker logs:**

```bash
sudo tail -n 50 /home/SITE_USER/queue-worker.log
```

> **Note:** Run Supervisor management commands as root or a user with the required sudo permissions.

---

## 5. Restart Workers After Deployment

Laravel queue workers are long-running processes.

After deploying new application code, you should restart them so they use the latest version.

As the CloudPanel site user, run:

```bash
php artisan queue:restart
```

This tells Laravel queue workers to finish their current jobs and exit gracefully.

Supervisor will automatically start new worker processes.

> **Important:** Laravel uses its configured cache to send the restart signal. Make sure your cache is working and Supervisor is configured to restart exited workers.

For more information, see [09 — Updating Production](../09-updating-production.md).

---

## 6. Check Failed Queue Jobs

Sometimes background jobs fail because of application errors, external API issues, or database problems.

Laravel provides commands to inspect and retry failed jobs.

**List failed jobs:**

```bash
php artisan queue:failed
```

**Retry a specific failed job:**

```bash
php artisan queue:retry JOB_ID
```

Replace `JOB_ID` with the failed job identifier.

**Check Laravel logs:**

```bash
tail -n 50 storage/logs/laravel.log
```

> **Important:** Review why a job failed before retrying it. Retrying jobs that send payments, emails, or other external requests could repeat an operation if the job is not designed to handle retries safely.

---

## 7. Verify Configuration

Before finishing, make sure:

- [ ] Laravel Scheduler is configured if the application uses scheduled tasks.
- [ ] The Scheduler Cron Job runs every minute.
- [ ] Cron runs with the correct site user and PHP executable.
- [ ] The queue connection is configured correctly.
- [ ] The required queue database tables exist, if using database queues.
- [ ] Supervisor is installed and running.
- [ ] Queue workers run as the CloudPanel site user.
- [ ] Queue workers restart automatically.
- [ ] Failed jobs and worker logs can be inspected.

Your Laravel application can now run scheduled tasks and process background jobs automatically.

---

## Quick Summary

| Feature | Tool |
| --- | --- |
| Scheduled Tasks | Laravel Scheduler |
| Scheduler Execution | CloudPanel Cron Jobs |
| Background Jobs | Laravel Queue Workers |
| Worker Process Management | Supervisor |
| Failed Jobs | Laravel Artisan Commands |

You do not need to install Supervisor if your application only uses Laravel Scheduler and does not need persistent queue workers.

---

**Back to:** [Laravel Deployment Guide](../../README.md)
