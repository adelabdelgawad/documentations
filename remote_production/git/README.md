
# GitHub Deploy Key Setup with Write Access

Setup a deploy key to allow your Ubuntu VM to push and pull from your GitHub repository.

## On Ubuntu VM

### 1. Generate SSH Deploy Key

Create a dedicated SSH key for this repository:

```
ssh-keygen -t ed25519 -C "deploy-key-repo-name" -f ~/.ssh/deploy_key_repo
```

Press `Enter` when asked for passphrase (leave empty for automation, or set one for security).

This creates two files:
- **Private key**: `~/.ssh/deploy_key_repo`
- **Public key**: `~/.ssh/deploy_key_repo.pub`

### 2. Display the Public Key

```
cat ~/.ssh/deploy_key_repo.pub
```

Copy the entire output (starts with `ssh-ed25519`).

### 3. Set Correct Permissions

```
chmod 600 ~/.ssh/deploy_key_repo
chmod 644 ~/.ssh/deploy_key_repo.pub
```

## On GitHub

### 1. Navigate to Repository Settings

1. Go to your GitHub repository
2. Click **Settings** tab
3. Click **Deploy keys** in the left sidebar

### 2. Add Deploy Key

1. Click **Add deploy key** button
2. **Title**: Enter a descriptive name (e.g., "Ubuntu VM Deploy Key" or "Production Server")
3. **Key**: Paste the public key content you copied earlier
4. ✅ **Important: Check "Allow write access"** checkbox
5. Click **Add key**

The deploy key is now added with write permissions.

## Configure SSH on Ubuntu VM

### 1. Create/Edit SSH Config

```
nano ~/.ssh/config
```

### 2. Add Configuration

Add this configuration (replace `username` and `repository` with your actual values):

```
Host github.com-myrepo
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_repo
    IdentitiesOnly yes
```

**Example:**
```
Host github.com-fastapi
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_repo
    IdentitiesOnly yes
```

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`.

### 3. Set Config Permissions

```
chmod 600 ~/.ssh/config
```

### 4. Test SSH Connection

```
ssh -T git@github.com-myrepo
```

Replace `github.com-myrepo` with your Host name from the config.

You should see:
```
Hi username/repository! You've successfully authenticated, but GitHub does not provide shell access.
```

## Clone or Update Repository

### Option A: Clone New Repository

Use the custom host from your SSH config:

```
git clone git@github.com-myrepo:username/repository.git
```

**Example:**
```
git clone git@github.com-fastapi:myusername/fastapi-app.git
```

### Option B: Update Existing Repository

If you already have the repository cloned:

```
cd /path/to/your/repository
git remote set-url origin git@github.com-myrepo:username/repository.git
```

**Example:**
```
cd ~/projects/fastapi-app
git remote set-url origin git@github.com-fastapi:myusername/fastapi-app.git
```

### Verify Remote URL

```
git remote -v
```

Should show your custom host:
```
origin  git@github.com-myrepo:username/repository.git (fetch)
origin  git@github.com-myrepo:username/repository.git (push)
```

## Test Write Access

### 1. Make a Test Change

```
cd /path/to/your/repository
echo "# Deploy key test" >> test.md
git add test.md
git commit -m "Test deploy key write access"
```

### 2. Push to GitHub

```
git push origin main
```

Replace `main` with your default branch name (`master`, `develop`, etc.) if different.

If successful, your changes are now on GitHub!

## Common Git Commands

### Pull Latest Changes

```
git pull origin main
```

### Push Changes

```
git add .
git commit -m "Your commit message"
git push origin main
```

### Check Status

```
git status
```

### View Remote Configuration

```
git remote -v
```

## Troubleshooting

### Error: "Permission denied (publickey)"

**Check SSH connection:**
```
ssh -T git@github.com-myrepo
```

**Verify key is loaded:**
```
ssh-add -l
```

**Add key to SSH agent:**
```
eval "$(ssh-agent -s)"
ssh-add ~/.ssh/deploy_key_repo
```

### Error: "Repository not found"

**Check remote URL format:**
```
git remote -v
```

Should be: `git@github.com-myrepo:username/repository.git`

**Update if incorrect:**
```
git remote set-url origin git@github.com-myrepo:username/repository.git
```

### Error: "Failed to push - insufficient permission"

Verify on GitHub that:
1. Deploy key is added under repository Settings → Deploy keys
2. ✅ **"Allow write access" is checked**
3. The public key matches your private key

To verify key match:
```
ssh-keygen -y -f ~/.ssh/deploy_key_repo
```

Compare output with the key on GitHub.

### Wrong Key Being Used

Ensure `IdentitiesOnly yes` is in your SSH config to prevent other keys from being tried first.

### Debug SSH Connection

Use verbose mode to see what's happening:
```
ssh -vT git@github.com-myrepo
```

## Security Best Practices

1. **Use separate deploy keys** for each repository or server
2. **Only enable write access when necessary**
3. **Use descriptive key titles** on GitHub to identify servers
4. **Rotate keys periodically** (every 6-12 months)
5. **Never commit private keys** to the repository
6. **Restrict deploy key usage** to specific repositories only

## Multiple Repositories

If you need deploy keys for multiple repositories:

### SSH Config Example

```
Host github.com-repo1
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_repo1
    IdentitiesOnly yes

Host github.com-repo2
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_key_repo2
    IdentitiesOnly yes
```

### Clone Each Repository

```
git clone git@github.com-repo1:username/repository1.git
git clone git@github.com-repo2:username/repository2.git
```

## Remove Deploy Key

### On GitHub

1. Go to repository **Settings** → **Deploy keys**
2. Find the key you want to remove
3. Click **Delete**
4. Confirm deletion

### On Ubuntu VM

```
rm ~/.ssh/deploy_key_repo
rm ~/.ssh/deploy_key_repo.pub
```

Edit `~/.ssh/config` and remove the configuration block.

## Complete Example Workflow

```
# Generate key
ssh-keygen -t ed25519 -C "production-server" -f ~/.ssh/deploy_production

# Display public key
cat ~/.ssh/deploy_production.pub
# Copy the output and add to GitHub with write access

# Configure SSH
cat >> ~/.ssh/config << 'EOF'
Host github.com-prod
    HostName github.com
    User git
    IdentityFile ~/.ssh/deploy_production
    IdentitiesOnly yes
EOF

chmod 600 ~/.ssh/deploy_production ~/.ssh/config

# Test connection
ssh -T git@github.com-prod

# Clone repository
git clone git@github.com-prod:yourusername/your-repo.git

# Work with repository
cd your-repo
git pull origin main
# Make changes...
git add .
git commit -m "Update from production"
git push origin main
```

Done! Your Ubuntu VM can now push and pull from your GitHub repository securely.
