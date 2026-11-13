
# 04 — Automated Laravel Deployment with GitHub Actions

In the previous chapters, we deployed and updated Laravel applications manually using SSH and Git commands.

With **GitHub Actions**, we can automate the deployment process whenever new code is pushed to GitHub.

In this guide, we'll configure a simple CI/CD workflow for an existing Laravel application hosted on a CloudPanel VPS.

## Prerequisites

- Laravel application already deployed and working.
- Application source code hosted on GitHub.
- CloudPanel site user with SSH access.
- GitHub repository access configured on the VPS (Chapter 06).
- Laravel production environment configured.
- A working manual deployment process (Chapter 09).

> **Important:** This guide uses a simple, in-place deployment strategy. It is suitable for small applications but does not guarantee zero downtime.

Official documentation: https://docs.github.com/en/actions

---

## 1. How Automated Deployment Works

Our deployment process will look like this:

```text
Developer pushes code to GitHub
               |
               v
      GitHub Actions starts
               |
               v
     Connect to VPS via SSH
               |
               v
       Pull latest Git code
               |
               v
     Install PHP dependencies
               |
               v
       Build frontend assets
               |
               v
       Run Laravel migrations
               |
               v
         Optimize Laravel
               |
               v
        Deployment complete
```

We'll use a dedicated SSH key to let GitHub Actions access the VPS securely.

> **Note:** The SSH key used by GitHub Actions to access the VPS is different from the GitHub Deploy Key configured in Chapter 06. The VPS still uses its existing GitHub Deploy Key to pull repository changes.

## 2. Create an SSH Key for GitHub Actions

We need a dedicated SSH key pair for automated deployments.

On your **local computer**, open Git Bash (Windows) or Terminal (macOS/Linux).

Create the SSH directory if needed:

```bash
mkdir -p ~/.ssh
```

Generate a new key pair:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/laravel_ci_cd -C "github-actions-deploy"
```

For this automated deployment key, leave the passphrase empty when prompted. GitHub Actions cannot respond to an interactive passphrase prompt in this workflow.

Do not overwrite an existing key with the same filename.

This creates:

```text
~/.ssh/laravel_ci_cd       # Private key
~/.ssh/laravel_ci_cd.pub   # Public key
```

Keep the private key secure.

### Add the Public Key to CloudPanel

Display your public key:

```bash
cat ~/.ssh/laravel_ci_cd.pub
```

Copy the entire key.

In CloudPanel:

1. Navigate to **Sites → Your Website → Manage**.
2. Open **Settings**.
3. Find **Site User Settings**.
4. Add the public SSH key to your primary site user.
5. Save the changes.

CloudPanel also supports managing SSH users and public keys through the **SSH/FTP** section.

> **Security:** Use your dedicated site user instead of root. This limits the deployment workflow's server permissions.

### Test SSH Connection

From your local computer, run:

```bash
ssh -i ~/.ssh/laravel_ci_cd SITE_USER@SERVER_IP
```

Replace:

- `SITE_USER` with your CloudPanel site username.
- `SERVER_IP` with your VPS IP address.

The connection should work without requesting the site's account password.

If you're asked to verify the server's host key, compare its fingerprint with the trusted fingerprint of your VPS before accepting it.

## 3. Verify Your Server's SSH Host Key

GitHub Actions should verify that it connects to your actual VPS, rather than blindly trusting any server presenting itself at the configured address.

First, on the VPS as root, display the server's SSH host-key fingerprint:

```bash
ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub
```

On your local computer, retrieve the public host key:

```bash
ssh-keyscan -t ed25519 SERVER_IP
```

Check the fingerprint of the retrieved key:

```bash
ssh-keyscan -t ed25519 SERVER_IP 2>/dev/null | ssh-keygen -lf -
```

Confirm that the fingerprint matches the one obtained directly from your VPS.

Once verified, copy the complete output of:

```bash
ssh-keyscan -t ed25519 SERVER_IP
```

We'll save this as `DEPLOY_KNOWN_HOSTS` in GitHub.

> **Important:** Do not blindly trust the output of `ssh-keyscan`. Always verify the host-key fingerprint against a trusted source before using it.

## 4. Configure GitHub Actions Secrets

We need to store the VPS connection information securely in GitHub.

Open your GitHub repository:

1. Go to **Settings**.
2. Select **Secrets and variables → Actions**.
3. Open **Secrets**.
4. Click **New repository secret**.

Create these secrets:

| Secret Name | Value |
| --- | --- |
| `DEPLOY_HOST` | Your VPS IP address |
| `DEPLOY_USER` | Your CloudPanel site username |
| `DEPLOY_SSH_KEY` | Contents of the private key `laravel_ci_cd` |
| `DEPLOY_KNOWN_HOSTS` | Verified SSH host-key entry from Step 3 |

### Copy the Private Key

On your local computer, run:

```bash
cat ~/.ssh/laravel_ci_cd
```

Copy the complete private key, including its BEGIN and END lines, into the `DEPLOY_SSH_KEY` secret.

> **Security:**
>
> - Never commit SSH private keys to your repository.
> - Never share the deployment private key publicly.
> - Do not use your personal or root SSH key for automated deployments.
> - Restrict who can modify production deployment workflows.

Official documentation: https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions

## 5. Create the GitHub Actions Workflow

In your Laravel project, create the following directory:

```text
.github/workflows/
```

Inside it, create a file named:

```text
deploy.yml
```

Your project structure should look like:

```text
laravel-project/
├── app/
├── bootstrap/
├── config/
├── public/
├── resources/
├── routes/
├── .github/
│   └── workflows/
│       └── deploy.yml
├── artisan
└── composer.json
```

### Add the Deployment Workflow

Open `.github/workflows/deploy.yml` and add:

```yaml
name: Deploy Laravel

on:
  push:
    branches:
      - main

  workflow_dispatch:

permissions:
  contents: read

concurrency:
  group: production-deploy
  cancel-in-progress: false

jobs:
  deploy:
    name: Deploy to VPS
    runs-on: ubuntu-latest

    steps:
      - name: Configure SSH
        env:
          DEPLOY_SSH_KEY: ${{ secrets.DEPLOY_SSH_KEY }}
          DEPLOY_KNOWN_HOSTS: ${{ secrets.DEPLOY_KNOWN_HOSTS }}
        run: |
          mkdir -p ~/.ssh
          chmod 700 ~/.ssh

          printf '%s\n' "$DEPLOY_SSH_KEY" > ~/.ssh/deploy_key
          chmod 600 ~/.ssh/deploy_key

          printf '%s\n' "$DEPLOY_KNOWN_HOSTS" > ~/.ssh/known_hosts
          chmod 600 ~/.ssh/known_hosts

      - name: Deploy Application
        env:
          DEPLOY_HOST: ${{ secrets.DEPLOY_HOST }}
          DEPLOY_USER: ${{ secrets.DEPLOY_USER }}
        run: |
          ssh -i ~/.ssh/deploy_key \
            -o IdentitiesOnly=yes \
            -o BatchMode=yes \
            -o StrictHostKeyChecking=yes \
            "$DEPLOY_USER@$DEPLOY_HOST" 'bash -se' <<'REMOTE'
          set -e

          # Navigate to Laravel project
          cd ~/htdocs/example.com

          # Verify production branch
          if [ "$(git branch --show-current)" != "main" ]; then
            echo "Incorrect production branch"
            exit 1
          fi

          # Stop if there are unexpected local changes
          if [ -n "$(git status --porcelain)" ]; then
            echo "Uncommitted changes found on server"
            exit 1
          fi

          # Pull latest code
          git pull --ff-only origin main

          # Install PHP dependencies
          composer install --no-dev --optimize-autoloader --no-interaction

          # Build frontend assets
          npm ci
          npm run build

          # Run database migrations
          php artisan migrate --force

          # Optimize Laravel
          php artisan optimize

          echo "Deployment completed successfully"
          REMOTE
```

Replace `example.com` with your actual Laravel project directory.

If your production branch is not `main`, update both the workflow trigger and the Git branch checks.

### Understanding the Workflow

| Configuration | Purpose |
| --- | --- |
| `on: push` | Runs after code is pushed |
| `branches: main` | Deploys changes from the main branch |
| `workflow_dispatch` | Allows manual execution |
| `permissions: contents: read` | Limits the workflow's GitHub token permissions |
| `concurrency` | Prevents overlapping deployments |
| `Configure SSH` | Prepares SSH credentials and verifies server identity |
| `Deploy Application` | Executes deployment commands on the VPS |

The remote script uses `set -e`, which stops execution when a command fails.

The `--ff-only` flag prevents unexpected Git merge commits on the production server.

> **Important:** `php artisan migrate --force` is required here because GitHub Actions cannot respond to Laravel's interactive production confirmation prompt. Only deploy migrations that have been reviewed and tested, and ensure database backups are available.

## 6. Run Your First Deployment

Commit your workflow file:

```bash
git add .github/workflows/deploy.yml
git commit -m "Add Laravel deployment workflow"
git push origin main
```

After pushing, open your GitHub repository.

1. Navigate to the **Actions** tab.
2. Select **Deploy Laravel**.
3. Open the latest workflow run.
4. Review the deployment logs.

If all commands complete successfully, the workflow will show a successful status.

### Run Deployment Manually

You can also start the workflow manually:

1. Open **GitHub → Actions**.
2. Select **Deploy Laravel**.
3. Click **Run workflow**.
4. Select the `main` branch.
5. Click **Run workflow**.

The workflow will connect to your VPS and deploy the latest code from `main`.

> **Note:** If a push triggers the workflow automatically, you do not need to trigger it manually as well.

## 7. Restart Background Services (Optional)

If your application uses Laravel Queue Workers or Laravel Reverb, restart them after deployment so they load the latest code.

Add the following commands to the remote deployment script, after `php artisan optimize`:

```bash
# Queue Workers
php artisan queue:restart

# Laravel Reverb
php artisan reverb:restart
```

Only include the commands for services you actually use.

Make sure Supervisor is configured to restart these services automatically.

See:

- [01 — Queue Workers & Scheduler](01-queue-and-scheduler.md)
- [02 — Laravel Reverb](02-laravel-reverb.md)

## 8. Protect Production Deployments (Recommended)

Automatically deploying every push to `main` is convenient, but production deployments should still be controlled.

GitHub provides deployment environments that can support branch restrictions, environment secrets, and approval rules, depending on your repository and GitHub plan.

### Create a Production Environment

1. Open your repository's **Settings**.
2. Navigate to **Environments**.
3. Create an environment named `production`.
4. Configure available branch restrictions or deployment approvals.
5. Move deployment secrets into the environment if desired.

Then add this line to your deployment job:

```yaml
jobs:
  deploy:
    name: Deploy to VPS
    runs-on: ubuntu-latest
    environment: production
```

GitHub will apply the configured environment protection rules before the deployment job starts.

> **Note:** Environment features and approval requirements depend on your GitHub plan and whether the repository is public or private. If your plan doesn't support the desired protection rules, you can continue using repository secrets with appropriate branch protection.

Official documentation: https://docs.github.com/en/actions/how-tos/deploy/configure-and-manage-deployments/manage-environments

## 9. Verify the Deployment

After a successful workflow run, open your live application.

Check:

1. Homepage and authentication.
2. Recently updated features.
3. Database operations.
4. Vue/Inertia frontend assets.
5. Queue workers and real-time features, if configured.

You can also connect to your VPS and verify the latest deployed commit:

```bash
cd ~/htdocs/example.com
git log -1 --oneline
```

### Troubleshooting

**SSH Permission Denied**

- Verify the private key stored in GitHub Secrets.
- Confirm the matching public key is added to the CloudPanel site user.
- Check the site username and SSH permissions.

**Host Key Verification Failed**

- Verify `DEPLOY_KNOWN_HOSTS`.
- Check that the VPS host key has not changed unexpectedly.
- Reverify the fingerprint before updating the secret.

**Git Pull Failed**

- Verify the production branch.
- Check for uncommitted changes on the VPS.
- Confirm the existing GitHub Deploy Key still works.

**Composer or npm Failed**

- Check the PHP and Node.js versions installed on the VPS.
- Verify the application's dependency lockfiles.
- Review the workflow logs.

**Migration Failed**

- Review the migration error.
- Keep the application in a safe state while investigating.
- Restore from a verified backup if recovery requires it.

> **Important:** This workflow stops when a command fails, but it does not automatically roll back the deployed code, database migrations, or generated assets.

## 10. Production Considerations

Before enabling automatic deployments for an important application:

- Review and test code before merging into `main`.
- Protect your production branch against unauthorized changes.
- Maintain recent database and uploaded-file backups.
- Keep production `.env` and SSH private keys out of Git.
- Use a dedicated site user for deployment.
- Avoid manually modifying tracked application files on the VPS.
- Review database migrations before deployment.
- Use maintenance mode when updates require downtime.
- Monitor deployment failures and application errors.

GitHub-hosted runners must be able to reach your VPS's SSH port. If your provider firewall allows SSH only from trusted IP addresses, configure an appropriate secure runner or deployment access method rather than disabling important firewall protections.

Also, this workflow updates files directly inside the live application directory. During deployment, some requests may encounter temporary errors.

For applications requiring **zero-downtime deployments**, consider a release-based approach with separate release directories and an atomic symlink switch.

---

## 11. Final Checklist

- [ ] Dedicated deployment SSH key created.
- [ ] Public key added to the CloudPanel site user.
- [ ] SSH host fingerprint verified.
- [ ] GitHub Actions secrets configured.
- [ ] `.github/workflows/deploy.yml` created.
- [ ] Correct production branch selected.
- [ ] Manual SSH access tested.
- [ ] First GitHub Actions deployment completed.
- [ ] Database migrations and frontend assets verified.
- [ ] Background services restart if required.
- [ ] Production branch and credentials are protected.

Your Laravel application can now be updated automatically using GitHub Actions.

---

## Quick Comparison

| Manual Deployment | Automated Deployment |
| --- | --- |
| Connect to VPS manually | GitHub Actions connects automatically |
| Run `git pull` | Workflow executes Git commands |
| Run Composer and npm | Workflow installs and builds |
| Run migrations manually | Workflow runs reviewed migrations |
| Run optimization commands | Workflow optimizes Laravel |
| Check terminal output | Review GitHub Actions logs |

Both approaches use the same underlying Laravel deployment commands.

The main difference is that GitHub Actions performs these steps automatically whenever the configured workflow runs.

---

**Back to:** [Laravel Deployment Guide](../../README.md)
