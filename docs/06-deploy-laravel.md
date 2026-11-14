# 06 — Deploy Laravel from GitHub

In this chapter, we'll create a Laravel site in CloudPanel and deploy an existing Laravel application from GitHub to our VPS.

Official CloudPanel Laravel guide: https://www.cloudpanel.io/docs/v2/php/applications/laravel/

## Prerequisites

- CloudPanel installed and configured.
- Domain pointing to your VPS.
- An existing Laravel application hosted on GitHub.
- Access to CloudPanel and your VPS via SSH.

## 1. Create a Laravel Site in CloudPanel

1. Log in to CloudPanel.
2. Navigate to **Sites → Add Site**.
3. Select **Create a PHP Site**.
4. Configure the following:
   - **Application:** Laravel (choose a compatible template).
   - **Domain Name:** `example.com`
   - **PHP Version:** Compatible with your Laravel application.
   - **Site User:** Create a dedicated user, e.g. `example-user`.
   - **Site User Password:** Use a strong password.
5. Click **Create**.

CloudPanel will create the website configuration, Nginx virtual host, and site user.

> **Note:** Available Laravel templates may change with CloudPanel updates. Choose a compatible template and verify the PHP version required by your project.

## 2. Connect as the Site User

CloudPanel creates a dedicated Linux user for each website.

We should use this **site user** to manage application files instead of working as root.

From your local computer, connect to the VPS using the SSH key configured in Chapter 03:

```bash
ssh -i ~/.ssh/laravel_vps root@SERVER_IP
```

Then switch to the CloudPanel site user:

```bash
sudo -iu SITE_USER
```

Replace `SITE_USER` with the username created in CloudPanel.

Verify your current user:

```bash
whoami
pwd
```

You should now be working inside the site's user environment, typically `/home/SITE_USER`.

> **Important:** The VPS SSH key and GitHub Deploy Key are separate credentials. The first provides access to your server; the second allows the server to access your GitHub repository.

## 3. Configure GitHub Access

If your GitHub repository is **public**, you can clone it using HTTPS without configuring a GitHub SSH key.

If your repository is **private**, configure a GitHub Deploy Key.

### Generate a GitHub Deploy Key

Run the following commands on the VPS as the **site user**:

```bash
mkdir -p ~/.ssh
chmod 700 ~/.ssh

ssh-keygen -t ed25519 -f ~/.ssh/github_deploy -C "laravel-deploy"
```

Do not overwrite an existing key with the same filename.

Display the public key:

```bash
cat ~/.ssh/github_deploy.pub
```

Copy the entire public key.

### Add the Deploy Key to GitHub

1. Open your GitHub repository.
2. Go to **Settings → Deploy keys**.
3. Click **Add deploy key**.
4. Enter a title such as `Production VPS`.
5. Paste the public key.
6. Leave **Allow write access** unchecked.
7. Click **Add key**.

This allows the VPS to read the specified GitHub repository without using your personal GitHub credentials.

> **Security:** Never share the private key (`github_deploy`) or add it to GitHub. Only the public key (`github_deploy.pub`) should be uploaded.

### Test GitHub Connection

Run the following command on your VPS as the site user:

```bash
ssh -i ~/.ssh/github_deploy -o IdentitiesOnly=yes -T git@github.com
```

During your first connection, SSH may ask you to verify GitHub's host key. Compare the fingerprint with [GitHub's official SSH fingerprints](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/githubs-ssh-key-fingerprints) before accepting it.

A successful connection should display an authentication confirmation. GitHub does not provide interactive SSH shell access, so the command may return exit status `1` even after successful authentication.

Official reference: https://docs.github.com/en/authentication/connecting-to-github-with-ssh/managing-deploy-keys

## 4. Clone Your Laravel Application

First, navigate to the `htdocs` directory:

```bash
cd ~/htdocs
```

Check the existing files:

```bash
pwd
ls -la
```

CloudPanel normally creates a directory matching your website's domain.

For example:

```text
/home/SITE_USER/htdocs/example.com
```

### Prepare the Website Directory

If this is a **new website** containing only CloudPanel's default placeholder files, rename the placeholder directory:

```bash
mv example.com example.com.cloudpanel-backup
```

> **Important:** Only rename the directory after confirming that it contains no important application data. Never run this step on an existing production website.

If the target directory doesn't exist, you can skip this step.

### Clone a Public Repository

For a public GitHub repository:

```bash
git clone https://github.com/OWNER/REPO.git example.com
```

Replace `OWNER/REPO` with your actual GitHub repository.

### Clone a Private Repository

If you configured a GitHub Deploy Key in Step 3, run:

```bash
GIT_SSH_COMMAND="ssh -i $HOME/.ssh/github_deploy -o IdentitiesOnly=yes" \
git clone git@github.com:OWNER/REPO.git example.com
```

Then navigate into the project:

```bash
cd ~/htdocs/example.com
```

Configure Git to use the same SSH key for future updates:

```bash
git config core.sshCommand "ssh -i $HOME/.ssh/github_deploy -o IdentitiesOnly=yes"
```

This configuration applies only to the cloned repository.

> **Note:** Replace `example.com` with the actual directory name used by CloudPanel. For example, a site created as `www.example.com` may have a directory with that name.

## 5. Verify Your Laravel Project

Navigate to your application directory if you aren't already there:

```bash
cd ~/htdocs/example.com
```

Check the application files:

```bash
ls -la
```

You should see the standard Laravel project structure, including:

```text
app/
bootstrap/
config/
database/
public/
resources/
routes/
storage/
artisan
composer.json
```

Verify the current Git branch:

```bash
git branch --show-current
```

Check the Git repository:

```bash
git remote -v
```

Make sure you've cloned the correct application and branch.

## 6. Verify the Website Root Directory

Laravel applications must serve HTTP requests through the **public** directory.

For example:

```text
/home/SITE_USER/htdocs/example.com/public
```

In CloudPanel:

1. Navigate to **Sites**.
2. Select your website and open **Manage**.
3. Find **Domain Settings → Root Directory**.
4. Verify that the website's document root points to the Laravel `public` directory.
5. Save the configuration if changes are required.

If you selected a compatible Laravel template, CloudPanel may already have configured the correct root directory.

> **Security:** Never point the public web root to the Laravel project root. Doing so could expose sensitive application files such as `.env`.

## 7. Verify Deployment Files

Before proceeding, confirm:

- [ ] Laravel PHP site is created in CloudPanel.
- [ ] Application files are owned by the dedicated site user.
- [ ] GitHub repository has been cloned successfully.
- [ ] GitHub Deploy Key is configured if using a private repository.
- [ ] Correct Git branch is checked out.
- [ ] Website root directory points to Laravel's `public` directory.

Your Laravel application source code is now available on the VPS.

The application may not work in the browser yet. We still need to install dependencies, configure the environment and database, and build frontend assets.

---

**Next:** [07 — Laravel Production Setup](07-production-setup.md)
