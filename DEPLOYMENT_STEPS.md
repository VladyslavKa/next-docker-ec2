# Deploy Next.js (Docker + Nginx) on AWS EC2 with GitHub Actions (self-hosted runner)

This document describes how the Next.js app was deployed to AWS EC2 using Docker Compose, an Nginx reverse proxy, and a self-hosted GitHub Actions runner.

---

## 0. Overview

- AWS EC2 (Ubuntu 24.04)
- Docker + Docker Compose (v2)
- Next.js (listens on port 3000 in the container)
- Nginx (listens on 80/443 and proxies to 3000)
- GitHub Actions self-hosted runner (on EC2)
- Automatic deploys on push to `main`

Flow:

Internet
-> 80/443
-> Nginx (container)
-> proxy
-> Next.js (container :3000)

---

## 1. AWS EC2 setup

### 1.1 Instance
- OS: Ubuntu 24.04 LTS
- Type: t3.micro (1 GB RAM for a small demo)
- Auto-assign Public IP: Enabled

### 1.2 Security Group (firewall)

Inbound rules:
- 22 / TCP -> YOUR_IP/32 (SSH)
- 80 / TCP -> 0.0.0.0/0 (HTTP)
- 443 / TCP -> 0.0.0.0/0 (HTTPS)

Outbound:
- All traffic -> 0.0.0.0/0

Path in AWS:
EC2 -> Instance -> Security -> Security Group

---

## 2. SSH from Windows and key permissions

Connect:
```bash
ssh -i next.pem ubuntu@PUBLIC_IP
```

If you see:
```
UNPROTECTED PRIVATE KEY FILE
bad permissions
```

Fix Windows ACL for the key:
- Properties -> Security -> Advanced
- Disable inheritance
- Remove everyone except your user with Read permission

Check:
```bash
icacls next.pem
```

---

## 3. Docker and Docker Compose

Verify:
```bash
docker --version
docker compose version
```

Note:
Ubuntu 24.04 uses `docker compose` (v2), not `docker-compose`.

To run Docker without sudo:
```bash
sudo usermod -aG docker ubuntu
```

Re-login after group change.

---

## 4. HTTP and Nginx basics

Ports:
- HTTP uses 80
- HTTPS uses 443

Nginx role:
- Accepts external requests
- Proxies to the app on localhost:3000
- Can terminate TLS when needed

---

## 5. App directory on the server

Example path:
```
/home/ubuntu/apps/next-docker-ec2
```

Create:
```bash
mkdir -p ~/apps
cd ~/apps
```

If you copied via scp (instead of git):
```bash
scp -i next.pem -r next-docker-ec2 ubuntu@IP:/home/ubuntu/apps/
```

---

## 6. Docker Compose build and run

```bash
cd /home/ubuntu/apps/next-docker-ec2
docker compose up -d --build
docker compose ps
```

Check:
```bash
curl -I http://localhost
```

---

## 7. Notes about Docker Compose

- `version: "3.9"` is obsolete in Compose v2.
- Avoid `container_name` unless required, it can cause name conflicts.

---

## 8. Git setup on the server

Install Git (Ubuntu):
```bash
sudo apt update
sudo apt install -y git
git --version
```

If you see:
```
fatal: not a git repository
```

Reason:
The project was copied via scp without `.git`.

Fix options:
- Re-clone from GitHub
- Or init and add remote manually

---

## 9. GitHub Actions self-hosted runner

Create runner user:
```bash
sudo adduser github-runner
```

Install runner:
```bash
sudo -i -u github-runner
mkdir ~/actions-runner
cd ~/actions-runner
./config.sh --url https://github.com/USER/REPO --token XXX
```

Check runner service:
```bash
sudo systemctl show -p User,WorkingDirectory actions.runner.*
```

---

## 10. Workflow permissions (groups and access)

If workflow fails with:
```
cd /home/ubuntu/apps/next-docker-ec2: Permission denied
```

Reason:
Runner user is `github-runner`, app files belong to `ubuntu`.

Fix:
```bash
sudo usermod -aG ubuntu github-runner

sudo chgrp -R ubuntu /home/ubuntu/apps/next-docker-ec2
sudo chmod -R g+rwX /home/ubuntu/apps/next-docker-ec2
sudo find /home/ubuntu/apps/next-docker-ec2 -type d -exec chmod g+s {} \;

sudo chmod 750 /home/ubuntu
sudo chmod 750 /home/ubuntu/apps
```

Restart runner:
```bash
sudo systemctl restart actions.runner.*
```

Verify:
```bash
sudo -u github-runner bash -lc "cd /home/ubuntu/apps/next-docker-ec2 && pwd"
```

---

## 11. Git "dubious ownership"

Error:
```
fatal: detected dubious ownership
```

Fix:
```bash
git config --global --add safe.directory /home/ubuntu/apps/next-docker-ec2
```

---

## 12. Low memory, runner offline, or SSH drops

Possible causes:
- 1 GB RAM is tight
- No swap
- Disk full
- OOM killer

Check:
```bash
df -h /
free -m
dmesg -T | grep -i oom
```

Cleanup:
```bash
docker system prune -af --volumes
sudo apt clean
```

Add swap:
```bash
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 13. Workflow example (minimal)

```yaml
name: Deploy to EC2

on:
  push:
    branches: [ main ]

concurrency:
  group: deploy-main
  cancel-in-progress: true

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - uses: actions/checkout@v4
      - run: |
          set -e
          git config --global --add safe.directory /home/ubuntu/apps/next-docker-ec2
          cd /home/ubuntu/apps/next-docker-ec2
          git fetch --all
          git reset --hard origin/main
          docker compose up -d --build
          docker image prune -f
```

---

## 14. Observations

- Next.js can be heavy for t3.micro.
- Image size affects deploy time.
- Docker cache can speed up builds.
- Host filesystem permissions are critical for the runner.
- Self-hosted runner is just a Linux user on the VM.

---

## 15. Quick checklist

- SSH key permissions fixed
- Security Group rules set
- Docker runs without sudo
- No `container_name` conflicts
- `safe.directory` configured
- Swap added if memory is low
- Runner has access to the app directory
