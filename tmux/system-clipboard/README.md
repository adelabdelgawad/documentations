````md
# tmux → System Clipboard (Linux Mint)

This setup makes copying text in **tmux** automatically available to the **system clipboard**, so you can paste outside tmux with **Ctrl + V**.

---

## Recommended Permanent Fix (Best Practice)

### 1. Install `xclip`
```bash
sudo apt install xclip
````

---

### 2. Update `~/.tmux.conf`

Add the following configuration:

```tmux
# Enable mouse support (optional but convenient)
set -g mouse on

# Use vi-style keys in copy mode
set -g mode-keys vi

# Copy selection directly to system clipboard
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -selection clipboard"
bind -T copy-mode-vi Enter send-keys -X copy-pipe-and-cancel "xclip -selection clipboard"
```

---

### 3. Reload tmux Configuration

Inside tmux, press:

```
Ctrl + b  then  :
```

Then run:

```tmux
source-file ~/.tmux.conf
```

---

## How It Works After Setup

### Enter copy mode

```
Ctrl + b  [
```

### Start selection

```
Space
```

### Finish selection (copy to system clipboard)

```
Enter
```

---

## Result

* ✅ Text is copied to the **system clipboard**
* ✅ Paste anywhere outside tmux using:

  ```
  Ctrl + V
  ```

---

## Notes

* This setup works on **X11** (default on Linux Mint).
* If you are using **Wayland**, replace `xclip` with `wl-copy` from `wl-clipboard`.

```bash
sudo apt install wl-clipboard
```

And update bindings to:

```tmux
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "wl-copy"
bind -T copy-mode-vi Enter send-keys -X copy-pipe-and-cancel "wl-copy"
```

```
```
