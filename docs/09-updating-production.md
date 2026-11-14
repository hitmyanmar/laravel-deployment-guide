# 09 — Updating a Production Laravel Application

After deploying your Laravel application, you'll eventually need to release new features, fix bugs, or update existing functionality.

You don't need to reinstall CloudPanel or redeploy everything from scratch.

You can update your existing application by pulling the latest code from GitHub and running the necessary Laravel commands.

## Prerequisites

Before updating your application, make sure:

- Your Laravel application is already deployed and working.
- Your latest code changes have been pushed to GitHub.
- You have SSH access to your VPS.
- Your GitHub repository access is still working.
- You have a recent backup of important production data.

> **Important:** This guide uses a simple Git-based deployment process. It is suitable for smaller applications but may cause brief downtime. For larger production applications, consider automated or release-based deployment.

## 1. Connect to Your VPS

From your **local computer**, connect using SSH:

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

Replace `SITE_USER` and `example.com` with your actual values.

## 2. Check Your Git Repository

Before pulling new changes, check the current Git branch:

```bash
git branch --show-current
```

Check for uncommitted changes:

```bash
git status
```

Make sure you're on the correct production branch and have no unexpected local changes.

> **Important:** Do not overwrite or discard production changes without reviewing them. Production application code should normally be updated through Git, not edited directly on the VPS.

## 3. Enable Maintenance Mode (Optional)

If your update involves database changes or may temporarily break the application, you can enable Laravel's maintenance mode:

```bash
php artisan down
```

Visitors will see a maintenance response while you update the application.

You can skip this step for small updates when downtime is not required, but in-place deployments are not guaranteed to be interruption-free.

## 4. Pull the Latest Code from GitHub

Update your application with:

```bash
git pull --ff-only
```

This downloads the latest changes from your configured upstream branch.

The `--ff-only` option prevents Git from automatically creating a merge commit if your local and remote branches have diverged.

If Git reports an error, review the branch history or uncommitted changes before proceeding. Do not use `git reset --hard` blindly on a production server.

> **Note:** If you're using a private GitHub repository, the Deploy Key configured in Chapter 06 should allow the VPS to pull the latest code.

## 5. Update Composer Dependencies

Install the PHP dependencies required by the latest code:

```bash
composer install --no-dev --optimize-autoloader
```

Composer uses your project's `composer.lock` file to install the locked dependency versions.

The `--no-dev` option skips development-only packages.

> **Important:** Use `composer install` rather than `composer update` during production deployment. Dependency versions should be updated and tested during development before deployment.

## 6. Update Frontend Assets

If your Laravel application uses Vue, React, Inertia.js, or Vite, rebuild the frontend assets:

```bash
npm ci
npm run build
```

For a typical Laravel Vite application, the new assets will be generated inside:

```text
public/build/
```

You can skip this step if your update doesn't require rebuilding frontend assets.

> **Note:** Use `npm ci` with a committed `package-lock.json`. Run these commands as the CloudPanel site user, not root.

## 7. Run Database Migrations

If your latest changes include database migrations, run:

```bash
php artisan migrate
```

Laravel will execute pending migrations.

When running in production, Laravel may ask for confirmation before continuing.

Check migration status:

```bash
php artisan migrate:status
```

> **Important:**
>
> - Back up your production database before running migrations.
> - Review and test migrations before deployment.
> - Avoid destructive database changes unless necessary.
> - Do not run `migrate:fresh` on an existing production database.

If the update introduces new environment variables, configure them in `.env` before running commands that depend on them. Clear the configuration cache first if necessary:

```bash
php artisan config:clear
```

Never regenerate your existing production `APP_KEY` during an ordinary application update.

## 8. Optimize Laravel

After updating your application, rebuild Laravel's production caches:

```bash
php artisan optimize
```

This caches configuration, routes, events, and views to improve performance.

If you encounter problems related to outdated cached configuration, review your environment settings and rebuild the caches.

## 9. Restart Background Workers (Optional)

If your application uses Laravel queue workers, restart them so they use the latest application code:

```bash
php artisan queue:restart
```

This tells workers to finish their current jobs and exit gracefully.

Your process manager, such as Supervisor, must be configured to start the workers again automatically.

If your application uses other long-running services such as Laravel Reverb or Octane, restart or reload those services according to their configuration.

> **Note:** Skip this step if your application doesn't use queue workers or other persistent processes.

## 10. Disable Maintenance Mode

If you enabled maintenance mode earlier, bring the application back online:

```bash
php artisan up
```

Make sure the deployment completed successfully before running this command.

## 11. Verify Your Updated Application

Open your live website and test the features affected by the update.

For example:

1. Open the homepage.
2. Test login and authentication.
3. Check newly added features.
4. Verify database operations.
5. Confirm Vue/Inertia frontend assets load correctly.

If your application encounters an error, check the Laravel logs:

```bash
tail -n 50 storage/logs/laravel.log
```

The log filename may differ depending on your logging configuration.

You can also verify the currently deployed Git commit:

```bash
git log -1 --oneline
```

### What if the deployment fails?

If the update fails:

1. Review the error message and Laravel logs.
2. Keep maintenance mode enabled if the application is not safe to serve.
3. Fix the issue or redeploy a previously tested application version.
4. Restore database or uploaded-file backups if necessary.

> **Important:** Reverting Git code does not automatically reverse database migrations. Some migrations cannot be safely rolled back. Always have a recovery plan before making production changes.

## 12. Deployment Checklist

Before considering the update complete:

- [ ] Latest code successfully pulled from GitHub.
- [ ] Composer dependencies updated.
- [ ] Frontend assets rebuilt if required.
- [ ] Database migrations completed if required.
- [ ] Laravel production caches rebuilt.
- [ ] Background workers restarted if applicable.
- [ ] Maintenance mode disabled.
- [ ] Website and important features tested.
- [ ] No unexpected production errors.

Your Laravel application is now running the latest deployed version.

---

## Quick Command Reference

For a typical Laravel application, the update process is:

```bash
# Navigate to your project
cd ~/htdocs/example.com

# Check repository
git status

# Get latest code
git pull --ff-only

# Install PHP dependencies
composer install --no-dev --optimize-autoloader

# Build frontend assets (if applicable)
npm ci
npm run build

# Apply database migrations (if applicable)
php artisan migrate

# Optimize Laravel
php artisan optimize

# Restart queue workers (if applicable)
php artisan queue:restart
```

Run these commands as your **CloudPanel site user**.

This is a command reference, not an unattended deployment script. Review changes, create necessary backups, and use maintenance mode where appropriate before executing production updates.

---

**Next:** [10 — Backups & Recovery](10-backups.md)
