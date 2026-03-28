# Server Git Access (Clone + Push Only)

## Overview

This setup ensures:

* The server can **clone and push to one repository only**
* Access is **isolated (no global account access)**
* Uses **SSH deploy key or machine user**
* Follows **least-privilege principles**

---

## 1. Create a Dedicated System User

```bash
sudo adduser gitdeploy
```

```bash
sudo su - gitdeploy
```

---

## 2. Generate SSH Key (On Server)

```bash
ssh-keygen -t ed25519 -C "repo-deploy-access"
```

* Save to default: `~/.ssh/id_ed25519`
* No passphrase (required for automation)

---

## 3. Copy Public Key

```bash
cat ~/.ssh/id_ed25519.pub
```

---

## 4. Grant Repo Access

### Option A — Deploy Key (Recommended for single repo)

In GitHub:

1. Go to the repository → **Settings → Deploy Keys**
2. Click **Add deploy key**
3. Paste the public key
4. Enable:

   * ✅ **Allow write access** (required for push)

✔ This key now:

* Can access **only this repo**
* Cannot access any other repos
* Cannot access account APIs

---

### Option B — Machine User (if multiple repos needed)

1. Create a dedicated account (e.g. `repo-bot`)
2. Add SSH key to that account
3. Grant access only to required repo(s)

---

## 5. Lock SSH to This Repo Only (Important)

Edit SSH config:

```bash
nano ~/.ssh/config
```

```bash
Host repo-access
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

---

## 6. Clone Repository

```bash
git clone git@repo-access:ORG/REPO.git
cd REPO
```

---

## 7. Enforce Repo-Only Usage (Hardening)

### Restrict SSH key usage (recommended advanced step)

If using self-hosted Git or stricter control:

In `authorized_keys` (server-side Git host):

```bash
command="git-shell -c \"$SSH_ORIGINAL_COMMAND\"",no-port-forwarding,no-agent-forwarding,no-pty ssh-ed25519 AAAA...
```

*(Skip if using GitHub deploy keys — already restricted)*

---

## 8. Verify Access

```bash
ssh -T git@repo-access
```

```bash
git pull
git push
```

---

## 9. Security Rules

* One key **per server per repo**
* Never reuse keys across environments
* Do NOT use personal SSH keys
* Rotate keys periodically
* Remove keys immediately if server is compromised

---

## 10. Expected Behavior

| Action                  | Allowed |
| ----------------------- | ------- |
| Clone repo              | ✅       |
| Pull changes            | ✅       |
| Push changes            | ✅       |
| Access other repos      | ❌       |
| Access account settings | ❌       |

---

## 11. Minimal Checklist

* [ ] SSH key generated on server
* [ ] Key added as **deploy key with write access**
* [ ] SSH config scoped to repo
* [ ] Clone works
* [ ] Push works
* [ ] No access to other repos confirmed
