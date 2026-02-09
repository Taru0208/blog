---
layout: post
title: "Deploy a Node.js App to a VPS from Scratch"
date: 2026-02-08
tags: [nodejs, devops, deployment, tutorial]
description: "A complete guide to deploying a Node.js application on a VPS — from SSH setup to pm2 process management, with nginx reverse proxy and SSL."
---

You've built a Node.js app locally. Now you want it running on the internet, 24/7. This guide covers the entire process — from a blank VPS to a production deployment with SSL, process management, and automatic restarts.

No Docker, no PaaS — just a Linux server and your code.

## What you'll need

- A VPS with Ubuntu 22+ (any provider works — 1 GB RAM is enough for most Node.js apps)
- A domain name (optional but recommended for SSL)
- SSH access to your server
- Your Node.js app with a `package.json`

## Step 1: Initial server setup

SSH into your fresh server:

```bash
ssh root@your-server-ip
```

Update packages and create a non-root user:

```bash
apt update && apt upgrade -y
adduser deploy
usermod -aG sudo deploy
```

Set up SSH key authentication for the deploy user:

```bash
# On your local machine
ssh-copy-id deploy@your-server-ip
```

Then disable password authentication:

```bash
sudo nano /etc/ssh/sshd_config
# Set: PasswordAuthentication no
sudo systemctl restart sshd
```

## Step 2: Install Node.js

Use NodeSource for the latest LTS:

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
node --version  # Should show v20.x
```

## Step 3: Deploy your code

Clone your repo (or upload via scp):

```bash
cd /home/deploy
git clone https://github.com/your-user/your-app.git
cd your-app
npm install --production
```

Test that it runs:

```bash
node server.js
# Should start on port 3000 (or whatever you configured)
```

## Step 4: pm2 — Keep it running

Your app dies when you close the SSH session. pm2 fixes this:

```bash
sudo npm install -g pm2
pm2 start server.js --name "my-app"
pm2 save
pm2 startup
```

What pm2 gives you:
- **Auto-restart** on crash
- **Survives reboots** (with `pm2 startup`)
- **Log management** — `pm2 logs my-app`
- **Monitoring** — `pm2 monit`

Check status anytime:

```bash
pm2 status
```

```
┌─────┬──────────┬─────────────┬─────────┬──────────┐
│ id  │ name     │ status      │ cpu     │ memory   │
├─────┼──────────┼─────────────┼─────────┼──────────┤
│ 0   │ my-app   │ online      │ 0.1%    │ 45.2mb   │
└─────┴──────────┴─────────────┴─────────┴──────────┘
```

### pm2 ecosystem file

For more complex setups, create `ecosystem.config.js`:

```javascript
module.exports = {
  apps: [{
    name: 'my-app',
    script: 'server.js',
    instances: 'max',      // Use all CPU cores
    exec_mode: 'cluster',  // Cluster mode for zero-downtime restarts
    env: {
      NODE_ENV: 'production',
      PORT: 3000,
    },
  }],
};
```

Then: `pm2 start ecosystem.config.js`

## Step 5: Nginx reverse proxy

Your app runs on port 3000, but users expect port 80/443. Nginx bridges this:

```bash
sudo apt install -y nginx
```

Create a site config:

```bash
sudo nano /etc/nginx/sites-available/my-app
```

```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_cache_bypass $http_upgrade;
    }
}
```

Enable it:

```bash
sudo ln -s /etc/nginx/sites-available/my-app /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

Your app is now accessible on port 80.

## Step 6: SSL with Let's Encrypt

Free SSL in two commands:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d your-domain.com
```

Certbot automatically:
- Gets a certificate from Let's Encrypt
- Configures nginx to use it
- Sets up auto-renewal via cron

Your app now runs on HTTPS.

## Step 7: Firewall

Lock down everything except what you need:

```bash
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw enable
sudo ufw status
```

## Step 8: Automatic deployments

Add a simple deploy script:

```bash
#!/bin/bash
# deploy.sh
cd /home/deploy/your-app
git pull origin main
npm install --production
pm2 restart my-app
echo "Deployed at $(date)"
```

For GitHub Actions automatic deployment, add this workflow:

```yaml
name: Deploy
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to VPS
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.HOST }}
          username: deploy
          key: ${{ secrets.SSH_KEY }}
          script: bash /home/deploy/deploy.sh
```

## Common issues

### Port already in use

```bash
# Find what's using the port
sudo lsof -i :3000
# Kill it
kill -9 <PID>
```

### App crashes on startup

Check logs:

```bash
pm2 logs my-app --lines 50
```

### Nginx 502 Bad Gateway

Your app isn't running. Check pm2:

```bash
pm2 status
pm2 restart my-app
```

### Out of memory

Node.js defaults to ~1.5GB heap. On a 1GB VPS, set limits:

```bash
pm2 start server.js --node-args="--max-old-space-size=512"
```

## The complete checklist

1. Server setup (user, SSH keys, firewall)
2. Node.js installed
3. Code deployed
4. pm2 running and configured for startup
5. Nginx reverse proxy configured
6. SSL certificate installed
7. Deploy script or CI/CD pipeline ready

Total time from blank server to production: about 30 minutes.

---

*This guide is based on real deployment experience. The server running this blog uses the same stack — pm2 for process management, nginx for reverse proxy, and Let's Encrypt for SSL.*
