```markdown
# SSH Key Authentication Setup Guide

## 1. On Ubuntu VM (Server)

### A. Install and Configure SSH

1. **Install OpenSSH Server:**
```
sudo apt update
sudo apt install openssh-server -y
```

2. **Enable and Start SSH Service:**
```
sudo systemctl enable ssh
sudo systemctl start ssh
```

3. **Verify SSH is Running:**
```
sudo systemctl status ssh
```

4. **Allow SSH Through Firewall (if enabled):**
```
sudo ufw allow ssh
```

5. **Enable Password Authentication (if needed):**
```
sudo nano /etc/ssh/sshd_config
```

Find and ensure these lines are set:
```
PasswordAuthentication yes
PubkeyAuthentication yes
```

Press `Ctrl+W` to search for "PasswordAuthentication"

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`

6. **Restart SSH Service:**
```
sudo systemctl restart ssh
```

### B. Key Configuration

1. **Generate SSH Key Pair:**
```
ssh-keygen -t rsa -b 4096
```

Press `Enter` to accept default location (`~/.ssh/id_rsa`)
Optionally enter a passphrase or press `Enter` for no passphrase

2. **Verify Keys Were Created:**
```
ls -la ~/.ssh/
```

You should see `id_rsa` (private key) and `id_rsa.pub` (public key)

3. **Add Public Key to authorized_keys:**
```
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
```

4. **Set Correct Permissions:**
```
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

5. **Display Public Key (for reference):**
```
cat ~/.ssh/id_rsa.pub
```

6. **Display Private Key (to copy to host machine):**
```
cat ~/.ssh/id_rsa
```

Copy the entire output including `-----BEGIN OPENSSH PRIVATE KEY-----` and `-----END OPENSSH PRIVATE KEY-----` lines

## 2. On Host Machine (Linux)

### A. Copy the Private Key from VM

**Method 1: Using SCP (from host machine):**
```
# Create .ssh directory if it doesn't exist
mkdir -p ~/.ssh

# Copy the private key from VM
scp -o HostKeyAlgorithms=ssh-ed25519 arc-webapp-01@10.25.10.50:~/.ssh/id_rsa ~/.ssh/arc-webapp-01
```

**Method 2: Manual Copy:**
1. On VM, display the private key:
```
cat ~/.ssh/id_rsa
```

2. On host machine, create the key file:
```
nano ~/.ssh/arc-webapp-01
```

3. Paste the private key content, save (`Ctrl+O`, `Enter`) and exit (`Ctrl+X`)

### B. Configure the Private Key

1. **Set Correct Permissions:**
```
chmod 600 ~/.ssh/arc-webapp-01
```

2. **Verify Permissions:**
```
ls -l ~/.ssh/arc-webapp-01
```

Should show: `-rw------- 1 username username`

### C. Connect Using SSH Key

**Basic Connection:**
```
ssh -i ~/.ssh/arc-webapp-01 -o HostKeyAlgorithms=ssh-ed25519 arc-webapp-01@10.25.10.50
```

**Alternative Host Key Algorithms:**
```
# Using rsa-sha2-512
ssh -i ~/.ssh/arc-webapp-01 -o HostKeyAlgorithms=rsa-sha2-512 arc-webapp-01@10.25.10.50

# Using rsa-sha2-256
ssh -i ~/.ssh/arc-webapp-01 -o HostKeyAlgorithms=rsa-sha2-256 arc-webapp-01@10.25.10.50
```

### D. Configure SSH for Easier Access (Optional)

1. **Create/Edit SSH Config File:**
```
nano ~/.ssh/config
```

2. **Add Configuration:**
```
Host arc-vm
    HostName 10.25.10.50
    User arc-webapp-01
    IdentityFile ~/.ssh/arc-webapp-01
    HostKeyAlgorithms +ssh-ed25519,rsa-sha2-512,rsa-sha2-256
```

3. **Save and Exit** (`Ctrl+O`, `Enter`, `Ctrl+X`)

4. **Set Config Permissions:**
```
chmod 600 ~/.ssh/config
```

5. **Connect Using Alias:**
```
ssh arc-vm
```

## Troubleshooting

### Error: "Unable to negotiate with host"
**Solution:** Add `-o HostKeyAlgorithms=ssh-ed25519` to your ssh command

### Error: "WARNING: UNPROTECTED PRIVATE KEY FILE"
**Solution:** Fix permissions with `chmod 600 ~/.ssh/arc-webapp-01`

### Error: "Permission denied (publickey)"
**Solution:** Verify that:
- Public key is in `~/.ssh/authorized_keys` on the VM
- `authorized_keys` has permissions `600`
- `~/.ssh` directory has permissions `700`
- `PubkeyAuthentication yes` is set in `/etc/ssh/sshd_config` on VM

### Disable Password Authentication (After Key Auth Works)
On the VM:
```
sudo nano /etc/ssh/sshd_config
```

Change to:
```
PasswordAuthentication no
```

Restart SSH:
```
sudo systemctl restart ssh
```

## Additional Tips

### Copy Files Using SCP
```
# From host to VM
scp -i ~/.ssh/arc-webapp-01 -o HostKeyAlgorithms=ssh-ed25519 /local/file arc-webapp-01@10.25.10.50:/remote/path/

# From VM to host
scp -i ~/.ssh/arc-webapp-01 -o HostKeyAlgorithms=ssh-ed25519 arc-webapp-01@10.25.10.50:/remote/file /local/path/
```

### Find VM IP Address
On the VM:
```
ip addr show
# or
hostname -I
```

### Test SSH Connection
```
ssh -v -i ~/.ssh/arc-webapp-01 -o HostKeyAlgorithms=ssh-ed25519 arc-webapp-01@10.25.10.50
```

The `-v` flag provides verbose output for debugging
```
