---

# OpenVPN Client Setup (Ubuntu / Linux Mint)

This guide walks you through installing OpenVPN, adding a client configuration, optionally storing credentials, and testing the connection.

---

## 1. Install OpenVPN

Update your package list and install OpenVPN:

```bash
sudo apt update
sudo apt install -y openvpn
```

---

## 2. Place the OpenVPN Configuration File

Create the OpenVPN client directory (if it doesn’t already exist):

```bash
sudo mkdir -p /etc/openvpn/client
```

Copy your `.ovpn` configuration file into the directory and rename it to `.conf`:

```bash
sudo cp ~/Downloads/amh.ovpn /etc/openvpn/client/amh.conf
```

Restrict file permissions for security:

```bash
sudo chmod 600 /etc/openvpn/client/amh.conf
```

---

## 3. (Optional) Store Credentials for Automatic Login

If your VPN requires a username and password, you can store them securely to avoid entering them each time.

### 3.1 Create the Credentials File

Create a credentials file:

```bash
sudo nano /etc/openvpn/client/amh.auth
```

Add **two lines only**:

```
yourusername
yourpassword
```

Save and exit.

Secure the file:

```bash
sudo chmod 600 /etc/openvpn/client/amh.auth
```

---

### 3.2 Reference the Credentials File in the Config

Edit the OpenVPN config:

```bash
sudo nano /etc/openvpn/client/amh.conf
```

Add this line anywhere in the file:

```
auth-user-pass /etc/openvpn/client/amh.auth
```

Save and exit.

---

## 4. Test the VPN Connection (Manual)

Start the VPN manually to verify everything works:

```bash
sudo openvpn --config /etc/openvpn/client/amh.conf
```

If the connection is successful, you should see:

```
Initialization Sequence Completed
```

Press `Ctrl + C` to disconnect.

---

## (Optional) Run as a Systemd Service

To start the VPN using systemd:

```bash
sudo systemctl start openvpn-client@amh
```

Enable it to start automatically on boot:

```bash
sudo systemctl enable openvpn-client@amh
```

Check status:

```bash
sudo systemctl status openvpn-client@amh
```
