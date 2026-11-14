# 07 — Laravel Production Setup

After cloning our Laravel application from GitHub, we need to configure the database, install dependencies, set up environment variables, and build frontend assets.

In this chapter, we'll prepare our Laravel application for production.

## Prerequisites

- Laravel application cloned successfully in Chapter 06.
- CloudPanel PHP site already created.
- Access to CloudPanel and the VPS.
- A PHP version compatible with your Laravel application.

> **Note:** Run Laravel application commands as your CloudPanel **site user**, not root, unless otherwise specified.

## 1. Create a Database in CloudPanel

First, create a MySQL database for your Laravel application.

1. Log in to CloudPanel.
2. Navigate to **Sites → Your Website → Manage**.
3. Open **Databases**.
4. Click **Add Database**.
5. Enter the following information:
   - **Database Name:** `example_db`
   - **Database User:** `example_user`
   - **Database Password:** Generate a strong password.
6. Click **Add Database**.

Save your database credentials securely. We'll use them in the Laravel `.env` file.

> **Security:** Use a dedicated database user for your application. Never use the MySQL root account as your application's database user.

## 2. Connect to Your Laravel Application

On your **local computer**, connect to your VPS:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

Switch to the CloudPanel site user:

```bash
sudo -iu SITE_USER
```

Navigate to your Laravel project:

```bash
cd ~/htdocs/example.com
```

Replace `SITE_USER` and `example.com` with your actual values.

Verify the current directory:

```bash
pwd
ls -la
```

## 3. Install Composer Dependencies

CloudPanel provides PHP and Composer, but you should verify that the CLI PHP version matches your Laravel application's requirements.

For example:

```bash
php -v
```

Check Composer:

```bash
composer --version
```

Install Laravel's production dependencies:

```bash
composer install --no-dev --prefer-dist --optimize-autoloader --no-interaction
```

This installs the PHP packages defined in `composer.lock` without development-only dependencies.

If Composer reports missing PHP extensions, verify your application's requirements and enable the necessary extensions through CloudPanel.

## 4. Configure Laravel Environment

Create the `.env` file from `.env.example`:

```bash
cp .env.example .env
```

Open the file:

```bash
nano .env
```

Configure the production environment and database credentials:

```dotenv
APP_NAME="Example App"
APP_ENV=production
APP_DEBUG=false
APP_URL=https://example.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=example_db
DB_USERNAME=example_user
DB_PASSWORD="YOUR_DATABASE_PASSWORD"
```

Replace the example values with your actual domain and database credentials.

Also configure any additional environment variables required by your application, such as mail, cache, sessions, queues, and third-party services.

Save the file using `Ctrl + O`, press `Enter`, then exit Nano using `Ctrl + X`.

### Generate Application Key

For a **new deployment**, generate the Laravel application key:

```bash
php artisan key:generate
```

This creates the `APP_KEY` value in your `.env` file.

> **Security:**
>
> - Always use `APP_DEBUG=false` in production.
> - Never commit `.env` or `APP_KEY` to GitHub.
> - Never regenerate the application key on an existing production application without understanding the consequences for encrypted data.
> - Keep a secure backup of your production environment configuration.

## 5. Run Database Migrations

After configuring the database connection, run:

```bash
php artisan migrate --force
```

The `--force` option allows migrations to run in the production environment without an interactive confirmation.

Check migration status:

```bash
php artisan migrate:status
```

> **Important:** Only run migrations that you've reviewed and tested. For an existing database, create a backup before performing schema changes.

Do not run `php artisan db:seed` unless your application specifically requires production seed data.

## 6. Install Node.js (For Frontend Assets)

If your Laravel application uses **Vue, React, Inertia.js, or Vite**, you need Node.js to install JavaScript dependencies and build frontend assets.

First, check whether Node.js and npm are already installed:

```bash
node -v
npm -v
```

If Node.js is installed and its version meets your project's requirements, skip to Step 7.

### Install Node.js on Ubuntu

Node.js installation requires root or sudo privileges.

Open a **separate terminal on your local computer** and connect to the VPS as root:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

The following example installs **Node.js 24 LTS** using NodeSource.

Download the setup script:

```bash
curl -fsSL https://deb.nodesource.com/setup_24.x -o nodesource_setup.sh
```

Review the downloaded script before executing it, then run:

```bash
bash nodesource_setup.sh
```

Install Node.js:

```bash
apt install -y nodejs
```

Verify installation:

```bash
node -v
npm -v
```

Official reference: https://github.com/nodesource/distributions

> **Note:** Node.js 24 is used as an example. Check your project's `package.json` and use a compatible, supported Node.js version.

Once Node.js is installed, return to your **site user** terminal for the remaining steps.

## 7. Install and Build Frontend Assets

Navigate to your Laravel project as the CloudPanel site user:

```bash
cd ~/htdocs/example.com
```

Install JavaScript dependencies:

```bash
npm ci
```

This uses `package-lock.json` to install the locked dependencies.

If your project doesn't have a `package-lock.json`, generate and commit one during development before using `npm ci` on the VPS.

Build the production frontend assets:

```bash
npm run build
```

For a standard Laravel application using Vite, this normally generates production assets inside:

```text
public/build/
```

Verify the build output:

```bash
ls -la public/build
```

> **Important:**
>
> - Run `npm ci` and `npm run build` as the site user, not root.
> - Node.js is needed during the build, but a typical Vue + Inertia.js application without SSR does not need a continuously running Node.js process.
> - Applications using Inertia SSR require additional server-side runtime configuration.

## 8. Configure Storage and Permissions

Laravel needs write access to these directories:

```text
storage/
bootstrap/cache/
```

Make sure the application directories are writable by the appropriate site user and PHP process.

If the site user already owns the application files, you can ensure appropriate permissions with:

```bash
chmod -R ug+rwX storage bootstrap/cache
```

If your application uses Laravel's local public storage disk for uploaded files, create the storage link:

```bash
php artisan storage:link
```

This links:

```text
public/storage → storage/app/public
```

> **Security:** Never use `chmod -R 777` to resolve permission errors. Also, the storage link is only needed when your application uses publicly accessible local storage.

## 9. Optimize Laravel for Production

Laravel provides a command to cache configuration, routes, events, and views for production:

```bash
php artisan optimize
```

If you update environment variables or need to clear cached configuration, use the appropriate Laravel cache-clearing commands before rebuilding caches.

> **Note:** Run optimization after completing the environment configuration. Remember that cached configuration must be refreshed when relevant `.env` values change.

## 10. Verify Production Setup

Check that Laravel can run successfully:

```bash
php artisan about
```

Review the application log if necessary:

```bash
tail -n 50 storage/logs/laravel.log
```

The log file might not exist yet if Laravel hasn't written any log entries.

Before proceeding, confirm:

- [ ] MySQL database and dedicated database user created.
- [ ] Composer dependencies installed successfully.
- [ ] `.env` configured with production values.
- [ ] Application key generated.
- [ ] Database migrations completed.
- [ ] Node.js installed if required.
- [ ] Frontend assets built successfully.
- [ ] Storage permissions are correct.
- [ ] Laravel production optimization completed.
- [ ] `APP_DEBUG=false`.

Your Laravel application is now configured for production.

Next, we'll enable HTTPS and verify the security of the live application.

---

**Next:** [08 — SSL & Production Security](08-ssl-security.md)
