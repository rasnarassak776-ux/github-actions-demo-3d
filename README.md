# Nova 3D — GitHub Actions CI/CD Demo

A simple **3D animation website** (Three.js) served by **Nginx** on an **Azure VM**, deployed automatically via **GitHub Actions** on every push to `main`.

This is a teaching demo: no build step, no framework complexity — just the cleanest possible path from `git push` to a live site.

---

## What you're building

```
  You: git push  ──►  GitHub  ──►  GitHub Actions  ──►  SSH/rsync  ──►  Azure VM (Nginx)  ──►  Live site
```

| Piece | Choice | Why |
|-------|--------|-----|
| 3D site | Plain HTML + Three.js (vendored locally) | No build step; files you commit = files served |
| Web server | Nginx serving static files | Simple, fast, standard |
| CI | Validate files + HTML on every push/PR | Catch mistakes before deploy |
| CD | SSH + `rsync` to the VM | Simplest deploy; no agent/service on the VM |

---

## Project structure

```
threed-demo/
├── index.html                 # the page + import map + overlay UI
├── css/
│   └── style.css              # styling for the overlay
├── js/
│   └── main.js                # the Three.js scene
├── vendor/
│   └── three/                 # Three.js vendored locally (no CDN needed)
│       ├── three.module.js
│       └── addons/controls/OrbitControls.js
├── .github/
│   └── workflows/
│       └── deploy.yml         # the CI/CD pipeline
└── README.md                  # this file
```

---

## Part 1 — Test the site locally (2 minutes)

Because it uses ES modules, you can't just double-click `index.html` — you need a tiny local server.

```bash
cd threed-demo
python3 -m http.server 8000
# open http://localhost:8000 in your browser
```

You should see a spinning 3D shape with **Change Color**, **Next Shape**, and **Toggle Spin** buttons. Drag to orbit, scroll to zoom.

---

## Part 2 — Set up the Azure VM (one time)

### 2.1 Create the VM

```bash
az vm create \
  --resource-group my-rg \
  --name threed-vm \
  --image Ubuntu2204 \
  --size Standard_B1s \
  --admin-username azureuser \
  --generate-ssh-keys

# open HTTP (port 80) to the internet
az vm open-port --resource-group my-rg --name threed-vm --port 80
```

Note the **public IP** it prints — that's your `VM_HOST`.

### 2.2 Install Nginx on the VM

SSH in and install Nginx:

```bash
ssh azureuser@<VM_PUBLIC_IP>

sudo apt-get update
sudo apt-get install -y nginx

# create the folder our site will live in
sudo mkdir -p /var/www/threed-demo
sudo chown -R azureuser:azureuser /var/www/threed-demo
```

### 2.3 Point Nginx at our folder

Create a site config:

```bash
sudo tee /etc/nginx/sites-available/threed-demo > /dev/null << 'EOF'
server {
    listen 80 default_server;
    server_name _;
    root /var/www/threed-demo;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF

# enable it, disable the default, reload
sudo ln -sf /etc/nginx/sites-available/threed-demo /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t          # test config
sudo systemctl reload nginx
```

Visit `http://<VM_PUBLIC_IP>` — you'll see the Nginx welcome or a 404 until the first deploy. That's expected.

---

## Part 3 — Create the deploy SSH key (one time)

GitHub Actions needs its own key to log into the VM. **Generate a dedicated key pair** (don't reuse your personal one).

On your **local machine**:

```bash
ssh-keygen -t ed25519 -C "github-actions-deploy" -f gha_deploy_key -N ""
# creates two files: gha_deploy_key (private) and gha_deploy_key.pub (public)
```

Add the **public** key to the VM so Actions can log in:

```bash
ssh-copy-id -i gha_deploy_key.pub azureuser@<VM_PUBLIC_IP>
# or manually append gha_deploy_key.pub to ~/.ssh/authorized_keys on the VM
```

---

## Part 4 — Add GitHub Secrets (one time)

In your GitHub repo: **Settings → Secrets and variables → Actions → New repository secret**.

Add these four:

| Secret name | Value |
|-------------|-------|
| `VM_HOST` | The VM's public IP, e.g. `20.1.2.3` |
| `VM_USER` | `azureuser` |
| `VM_SSH_KEY` | The **entire contents** of the **private** key file `gha_deploy_key` (including the `BEGIN`/`END` lines) |

> ⚠️ Paste the **private** key (`gha_deploy_key`), not the `.pub`. Copy the whole file exactly.

---

## Part 5 — Push and watch it deploy

```bash
# in your project folder, initialize git if you haven't
git init
git add .
git commit -m "Initial 3D site + CI/CD"
git branch -M main
git remote add origin https://github.com/<you>/<repo>.git
git push -u origin main
```

Now open the repo's **Actions** tab. You'll see the workflow run:

1. **Validate site** — checks the files exist and the HTML is sane.
2. **Deploy to VM** — creates `build.json`, sets up SSH, and `rsync`s the files to `/var/www/threed-demo/`.

When it goes green ✅, visit `http://<VM_PUBLIC_IP>` — your 3D site is live. The footer shows the deployed commit hash (from `build.json`).

**From now on, every `git push` to `main` redeploys automatically.** That's the whole CI/CD loop.

---

## How the pipeline works (for teaching)

The workflow (`.github/workflows/deploy.yml`) has **two jobs**:

```yaml
on:
  push:            { branches: [main] }   # deploy
  pull_request:    { branches: [main] }   # validate only
  workflow_dispatch:                       # manual button

jobs:
  validate:        # runs for BOTH pushes and PRs
    ...
  deploy:
    needs: validate                        # waits for validate to pass
    if: github.ref == 'refs/heads/main'    # only deploys from main
    ...
```

Key teaching points:

- **`needs: validate`** — the deploy job won't run unless validation passes first.
- **`if:` guard** — pull requests validate but never deploy (safe to review untrusted PRs).
- **Secrets** — credentials never appear in the code; they're injected as `${{ secrets.* }}`.
- **`rsync --delete`** — mirrors the repo to the VM, removing files that were deleted from git.

---

## Troubleshooting

| Symptom | Fix |
|---------|-----|
| Actions fails at "Set up SSH key" | Check `VM_SSH_KEY` is the full **private** key, no missing lines |
| "Permission denied (publickey)" | The public key isn't in the VM's `~/.ssh/authorized_keys` |
| Site loads but 3D is blank | Check the browser console; ensure `vendor/three/` files deployed |
| 404 on the site | Nginx `root` must be `/var/www/threed-demo` and files must be there |
| Can't reach the site at all | Port 80 not open — rerun `az vm open-port ... --port 80` |
| Blank page, module errors | You opened `index.html` directly; use a local server (Part 1) |

---

## Optional next steps

- **HTTPS**: add a domain and run `sudo certbot --nginx` for a free TLS cert.
- **Self-hosted runner**: instead of SSH, register the VM as a runner and use `runs-on: self-hosted`.
- **Staging + production**: add a second VM and use GitHub Environments with approvals.
- **Matrix**: not needed here (static site), but a great next lesson.

---

> This project pairs with the **GitHub Actions chapter guide** — Chapters 03 (syntax), 04 (triggers), 06 (secrets), 08 (deploy), and 09 (self-hosted) all show up in this one demo.

ThankYou>>!
