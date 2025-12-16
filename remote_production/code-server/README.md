```markdown
# VS Code Server (code-server) Installation Guide

Access VS Code through your web browser from any device on the network.

## Installation

### 1. Install code-server on Ubuntu

```
curl -fsSL https://code-server.dev/install.sh | sh
```

### 2. Stop the Service (before configuration)

```
sudo systemctl stop code-server@$USER
```

## Configuration

### 1. Edit Configuration File

```
nano ~/.config/code-server/config.yaml
```

### 2. Update the Configuration

Change the content to:

```
bind-addr: 0.0.0.0:8080
auth: password
password: your-secure-password-here
cert: false
```

**Important:** Change `127.0.0.1:8080` to `0.0.0.0:8080` to allow external connections.

Save with `Ctrl+O`, `Enter`, then exit with `Ctrl+X`

### 3. Enable and Start the Service

```
sudo systemctl enable code-server@$USER
sudo systemctl start code-server@$USER
```

### 4. Verify Service is Running

```
sudo systemctl status code-server@$USER
```

Should show "active (running)"

### 5. Check the Logs

```
journalctl -u code-server@$USER -f
```

You should see:
```
HTTP server listening on http://0.0.0.0:8080/
```

Press `Ctrl+C` to exit the log viewer.

## Firewall Configuration

### Allow Port 8080

```
sudo ufw allow 8080
sudo ufw reload
```

### Verify Firewall Status

```
sudo ufw status
```

## Access code-server

### From Your Browser

Open your web browser and navigate to:

```
http://SERVER-IP:8080
```

Example:
```
http://10.25.10.50:8080
```

### Login

Enter the password you set in the `config.yaml` file.

## Useful Commands

### Restart code-server

```
sudo systemctl restart code-server@$USER
```

### Stop code-server

```
sudo systemctl stop code-server@$USER
```

### View Logs in Real-time

```
journalctl -u code-server@$USER -f
```

### Check if Port is Listening

```
sudo ss -tulpn | grep 8080
```

Should show: `0.0.0.0:8080`

### View Configuration

```
cat ~/.config/code-server/config.yaml
```

## Troubleshooting

### "Unable to connect" Error

**Check if service is running:**
```
sudo systemctl status code-server@$USER
```

**Verify bind address:**
```
cat ~/.config/code-server/config.yaml
```

Must show `bind-addr: 0.0.0.0:8080` (NOT `127.0.0.1:8080`)

**Check logs for errors:**
```
journalctl -u code-server@$USER -n 50
```

### Port Already in Use

```
sudo lsof -i :8080
```

Kill the process using the port if needed.

### Firewall Blocking Connection

```
sudo ufw status
sudo ufw allow 8080
```

### Change Password

1. Edit config:
```
nano ~/.config/code-server/config.yaml
```

2. Change the password line

3. Restart service:
```
sudo systemctl restart code-server@$USER
```

## Security Recommendations

### Use Strong Password

Replace `your-secure-password-here` with a strong password containing:
- Uppercase and lowercase letters
- Numbers
- Special characters
- At least 12 characters long

### Enable HTTPS (Optional)

For production use, consider setting up HTTPS with a reverse proxy (nginx or Apache).

### Restrict Access by IP (Optional)

If you only need access from specific IPs, configure firewall rules:

```
sudo ufw delete allow 8080
sudo ufw allow from YOUR_IP to any port 8080
```

Replace `YOUR_IP` with your actual IP address.

## Advanced: Run as Systemd Service (Already Configured)

The installation script automatically creates a systemd service. You can check it with:

```
systemctl cat code-server@$USER
```

## Uninstall code-server

If you need to remove code-server:

```
sudo systemctl stop code-server@$USER
sudo systemctl disable code-server@$USER
sudo rm -rf ~/.local/share/code-server
sudo rm -rf ~/.config/code-server
```

To completely remove the package:
```
sudo apt remove code-server
```

## Additional Tips

### Access from Outside Your Network

If you need to access code-server from outside your local network, you'll need to:

1. Configure port forwarding on your router (forward port 8080 to your Ubuntu server IP)
2. Use your public IP address
3. **Strongly recommended:** Set up HTTPS and use a strong password

### Use with Docker Projects

code-server works great with Docker. You can:
- Open terminal in code-server
- Run Docker commands directly
- Edit docker-compose.yml files
- View container logs

### Install Extensions

Once logged in, you can install VS Code extensions just like in desktop VS Code:
- Click Extensions icon in sidebar
- Search and install your preferred extensions

### Sync Settings

You can sign in with your GitHub/Microsoft account to sync your VS Code settings across devices.
```

Copy everything above (including the outer triple backticks with "markdown") and paste it into your README.md file!
