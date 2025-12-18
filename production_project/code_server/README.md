# How to Set Up code-server

code-server lets you run VS Code in your browser on any machine. Here's how to install and configure it on port 8080 with password authentication.[1]

## Installation

The quickest installation method uses the official install script:[2][1]

```bash
curl -fsSL https://code-server.dev/install.sh | sh
```

For manual installation on Linux:[1]

```bash
VERSION=$(curl -s https://api.github.com/repos/coder/code-server/releases/latest | grep "tag_name" | cut -d '"' -f 4 | sed 's/v//')
mkdir -p ~/.local/lib ~/.local/bin
curl -fL https://github.com/coder/code-server/releases/download/v$VERSION/code-server-$VERSION-linux-amd64.tar.gz | tar -C ~/.local/lib -xz
mv ~/.local/lib/code-server-$VERSION-linux-amd64 ~/.local/lib/code-server-$VERSION
ln -s ~/.local/lib/code-server-$VERSION/bin/code-server ~/.local/bin/code-server
PATH="~/.local/bin:$PATH"
```

On macOS, use Homebrew:[3]

```bash
brew install code-server
```

## Configuration

The configuration file is located at `~/.config/code-server/config.yaml`. To set up port 8080 with password authentication, edit this file:[4][1]

```yaml
bind-addr: 127.0.0.1:8080
auth: password
password: your-secure-password-here
cert: false
```

Replace `your-secure-password-here` with your desired password. If you want to access code-server from other machines on your network, change `127.0.0.1` to `0.0.0.0`.[4]

## Running code-server

Start code-server by running:[3]

```bash
code-server
```

To run it as a system service:[1]

```bash
sudo systemctl enable --now code-server@$USER
```

## Accessing in Browser

Open your browser and navigate to `http://127.0.0.1:8080`. Enter the password you configured in the config file when prompted. If you didn't set a custom password, the default password is auto-generated and stored in `~/.config/code-server/config.yaml`.[5][4][1]
