# 10 — Backups & Recovery

Backups are essential for protecting your Laravel application from data loss, server failures, accidental deletion, and unsuccessful deployments.

CloudPanel provides built-in database backups and supports remote backups, making it easier to protect your application without manually configuring every backup process.

Official documentation:

- [CloudPanel Database Backups](https://www.cloudpanel.io/docs/v2/frontend-area/databases/)
- [CloudPanel Remote Backups](https://www.cloudpanel.io/docs/v2/admin-area/backups/)

## 1. What Should You Back Up?

A Laravel application typically contains several types of important data.

| Data             | What to back up                               |
| ---------------- | --------------------------------------------- |
| Database         | Users, orders, application records, etc.      |
| Uploaded Files   | Images, documents, and user uploads           |
| Environment      | Production `.env` and important configuration |
| Application Code | Git repository and deployed version           |

Application source code is normally stored in GitHub, but the database, uploaded files, and production environment configuration require separate protection.

> **Important:** Keep backups outside your VPS. If the VPS fails or is compromised, backups stored only on that server may also be lost.

## 2. Automatic Database Backups in CloudPanel

CloudPanel automatically creates a backup of each database.

By default:

- **Frequency:** Every night at 3:15 AM (server timezone).
- **Retention:** 7 days.
- **Location:** The website user's backup directory.

```text
/home/SITE_USER/backups/
```

### Check Existing Backups

Connect to your VPS and switch to the CloudPanel site user, as explained in previous chapters.

Run:

```bash
ls -lh ~/backups
```

You should see backup files or directories created by CloudPanel.

> **Note:** Check that automatic backups are running successfully. Seven days of backups may not be enough for every production application.

For more information, see the [CloudPanel Database Documentation](https://www.cloudpanel.io/docs/v2/frontend-area/databases/).

## 3. Create a Manual Database Backup

Before deploying major updates or changing your database structure, it's a good idea to create a manual backup.

CloudPanel provides a CLI command for exporting databases.

### Create a Backup Directory

Run these commands on the VPS as your **site user**:

```bash
mkdir -p ~/backups/manual
chmod 700 ~/backups/manual
cd ~/backups/manual
```

### Export Your Database

```bash
clpctl db:export --databaseName=example_db --file=example_db-backup.sql.gz
```

Replace `example_db` with your actual database name.

The `.sql.gz` extension tells CloudPanel to create a compressed database backup.

Verify the backup file:

```bash
ls -lh example_db-backup.sql.gz
```

You should see the generated backup file.

> **Security:** Database backups may contain user information and other sensitive data. Never store them inside Laravel's `public` directory or commit them to GitHub.

## 4. Back Up Uploaded Files

If your Laravel application stores uploaded files locally, those files must also be backed up.

For example, Laravel's public storage directory is commonly:

```text
storage/app/public/
```

### Create an Uploaded Files Backup

Run the following command as your site user:

```bash
tar -czf ~/backups/manual/uploads-backup.tar.gz \
-C ~/htdocs/example.com storage/app/public
```

Replace `example.com` with your actual website directory.

This creates a compressed archive of the uploaded files.

Verify the archive:

```bash
ls -lh ~/backups/manual/uploads-backup.tar.gz
```

> **Note:** This example assumes your application stores uploads in `storage/app/public`. If you use other local directories, include them as well.

If your application uses Amazon S3 or another external object-storage provider, configure backups for that storage separately.

## 5. Configure Remote Backups in CloudPanel

Local backups are useful, but they won't protect your application if the entire VPS becomes unavailable.

CloudPanel supports remote backups to services such as:

- Amazon S3
- DigitalOcean Spaces
- Wasabi
- Dropbox
- Google Drive (supported Google Workspace accounts)
- SFTP storage
- Other Rclone-compatible storage providers

### Set Up Remote Backups

1. Log in to CloudPanel.
2. Open **Admin Area → Backups**.
3. Select your preferred storage provider.
4. Enter the required storage credentials.
5. Save the configuration.
6. Click **Create Backup** to test it.
7. Verify that the backup appears in your remote storage.

CloudPanel can back up site directories and web-server configurations. Its remote backup process includes site home directories by default, with certain exclusions.

Because database dumps are stored separately from the live MySQL database files, verify that your remote backup includes a recent database dump.

> **Security:**
>
> - Use restricted storage credentials.
> - Protect access to remote backups.
> - Encrypt sensitive backups when appropriate.
> - Configure a suitable backup schedule and retention policy.
> - Verify that backups are completing successfully.

Official guide: https://www.cloudpanel.io/docs/v2/admin-area/backups/

## 6. Restore a Database Backup

If your database becomes damaged or data is accidentally deleted, you may need to restore a previous backup.

CloudPanel provides a database import command.

### Test Database Restoration

Before restoring production data, it's safer to test the backup using a separate database.

1. Open CloudPanel.
2. Navigate to your website's **Databases** section.
3. Create a new empty test database, for example `example_restore_db`.
4. Connect to your VPS as the site user.
5. Navigate to your backup directory.

```bash
cd ~/backups/manual
```

Import the backup into the test database:

```bash
clpctl db:import --databaseName=example_restore_db --file=example_db-backup.sql.gz
```

Replace the example database names and filename with your own values.

Check the restored database through CloudPanel or your database management tool.

> **Important:** Database imports can overwrite existing tables or records. Never import a backup into your live production database without understanding the consequences and preparing a recovery plan.

For a real production recovery, stop application writes, identify the correct backup, and restore it using an appropriate database recovery procedure.

## 7. Restore Uploaded Files

You can also extract an uploaded-files backup.

First, inspect the archive:

```bash
tar -tzf ~/backups/manual/uploads-backup.tar.gz
```

To test that the backup can be extracted safely, use a separate directory:

```bash
mkdir -p ~/restore-test

tar -xzf ~/backups/manual/uploads-backup.tar.gz \
-C ~/restore-test
```

Verify the extracted files:

```bash
ls -la ~/restore-test/storage/app/public
```

For an actual recovery, restore the required files to the correct application storage directory and verify their ownership and permissions.

> **Important:** Avoid extracting untrusted archives directly into a live application directory. Test the backup contents first.

## 8. Back Up Your Production Environment

Your Laravel `.env` file contains configuration that may be difficult to recreate after losing a server.

It can include:

- Application key (`APP_KEY`)
- Database connection details
- Mail configuration
- External service credentials
- Cache, session, and queue configuration

Keep a secure backup of this configuration outside the VPS.

**Never commit your production `.env` file to a public or private application repository.**

Store sensitive configurations in encrypted backup storage or a trusted secrets-management system.

## 9. Verify Your Backup Strategy

A successful backup is useful only if you can restore it.

Regularly verify:

- [ ] Automatic database backups are running.
- [ ] Backups include the required database records.
- [ ] Uploaded files are included in the backup plan.
- [ ] Production environment configuration is protected.
- [ ] Backup copies exist outside the VPS.
- [ ] Backup schedules and retention periods are appropriate.
- [ ] A test database restore has succeeded.
- [ ] A test uploaded-files restore has succeeded.
- [ ] Backup storage is protected from unauthorized access.

For important production systems, consider maintaining multiple backup copies and testing recovery periodically.

---

## Final Notes

You have now learned how to:

1. Choose a domain name and VPS.
2. Prepare and secure an Ubuntu server.
3. Install CloudPanel.
4. Configure domain DNS records.
5. Deploy an existing Laravel application.
6. Configure Laravel for production.
7. Enable HTTPS and verify security.
8. Update a production application.
9. Create and restore application backups.

Your Laravel application is now deployed and ready for ongoing maintenance.

Remember to keep Laravel, CloudPanel, PHP, and your server packages updated.

---

**Back to:** [Laravel Deployment Guide](../README.md)
