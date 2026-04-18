# BMI Health Tracker — Deployment Guide

> **Single source of truth** for deploying on AWS EC2 Ubuntu.
> `IMPLEMENTATION_AUTO.sh` automates every step in this guide.
> For post-deploy updates use `AppUpdate_AUTO.sh`.

---

## Table of Contents

1. [Application Overview](#1-application-overview)
2. [Prerequisites](#2-prerequisites)
3. [Part A — AWS Console Setup](#part-a--aws-console-setup)
4. [Part B — Server Bootstrap](#part-b--server-bootstrap)
5. [Part C — Run the Deployment Script](#part-c--run-the-deployment-script)
6. [What the Script Does — Step by Step](#what-the-script-does--step-by-step)
7. [Script Options Reference](#script-options-reference)
8. [Post-Deployment Verification](#post-deployment-verification)
9. [SSL / HTTPS Setup](#ssl--https-setup)
10. [Updating the Application](#updating-the-application)
11. [Useful Day-2 Commands](#useful-day-2-commands)
12. [Troubleshooting](#troubleshooting)

---

## 1. Application Overview

**BMI Health Tracker** — a 3-tier full-stack web application.

| Layer    | Technology                     | Runs on        |
|----------|--------------------------------|----------------|
| Frontend | React 18 + Vite 5 + Chart.js 4 | Nginx (static) |
| Backend  | Node.js 18 LTS + Express 4     | PM2 port 3000  |
| Database | PostgreSQL 14                  | localhost:5432 |

Traffic flow: `Browser → Nginx :80/:443 → /api/* proxy → Node :3000 → PostgreSQL`

---

## 2. Prerequisites

### What you need before starting

| Item | Detail |
|------|--------|
| AWS account | EC2 launch permissions |
| EC2 key pair | `.pem` file on your local machine |
| Ubuntu 22.04 LTS instance | `t2.micro` or larger |
| Security group rules | Ports 22, 80, 443 open inbound |
| Domain (optional) | Required only for SSL — A record pointing to EC2 IP |

### What the script installs automatically (if missing)

- Node.js LTS via NVM
- npm
- PostgreSQL + postgresql-contrib
- Nginx
- PM2 (global npm package)
- Certbot + python3-certbot-nginx (SSL only)

---

## Part A — AWS Console Setup

*Do this once in the AWS Console before SSHing to the server.*

### A.1 Launch EC2 Instance

1. Go to **EC2 → Launch Instance**
2. Name: `bmi-health-tracker`
3. AMI: **Ubuntu Server 22.04 LTS (HVM), SSD Volume Type**
4. Instance type: `t2.micro` (free tier) or `t3.small`
5. Key pair: select or create a `.pem` key pair — save it securely
6. Storage: 20 GB gp3
7. Click **Launch Instance**

### A.2 Configure Security Group

| Type  | Protocol | Port | Source    | Purpose         |
|-------|----------|------|-----------|-----------------|
| SSH   | TCP      | 22   | Your IP   | Admin access    |
| HTTP  | TCP      | 80   | 0.0.0.0/0 | Web traffic     |
| HTTPS | TCP      | 443  | 0.0.0.0/0 | SSL web traffic |

### A.3 (Optional) Elastic IP

Assign a static IP so it does not change after reboots:
**EC2 → Elastic IPs → Allocate → Associate → select your instance**

### A.4 Connect via SSH

```bash
# Set key permissions (first time only)
chmod 400 your-key.pem

# Connect
ssh -i your-key.pem ubuntu@<EC2-PUBLIC-IP>
```

---

## Part B — Server Bootstrap

*Run once on the server immediately after first SSH login.*

```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install essential tools
sudo apt install -y git curl wget unzip build-essential

# Clone the repository
cd ~
git clone https://github.com/sarowar-alam/single-server-3tier-webapp.git
cd single-server-3tier-webapp

# Make scripts executable
chmod +x IMPLEMENTATION_AUTO.sh AppUpdate_AUTO.sh
```

---

## Part C — Run the Deployment Script

### Standard deploy (no SSL)

```bash
./IMPLEMENTATION_AUTO.sh
```

### Fresh / clean install

```bash
./IMPLEMENTATION_AUTO.sh --fresh
```

### Deploy with SSL in one command

```bash
./IMPLEMENTATION_AUTO.sh --with-ssl --domain=bmi.yourdomain.com
```

### Skip backup, no SSL

```bash
./IMPLEMENTATION_AUTO.sh --skip-backup --skip-ssl
```

The script interactively prompts for:

1. Database name (default: `bmidb`)
2. Database user (default: `bmi_user`)
3. Database password (must not be empty)
4. Confirm password
5. `Continue with deployment? (y/n)`
6. Domain name + Let's Encrypt email *(only if `--with-ssl` and no `--domain=` flag)*

---

## What the Script Does — Step by Step

| Step | Function | What happens |
|------|----------|-------------|
| 1 | `collect_database_credentials` | Prompts for DB name, user, password |
| 2 | `check_prerequisites` | Loads NVM; installs Node.js, PostgreSQL, Nginx, PM2 if missing |
| 3 | `setup_database` | Creates DB + user, grants privileges, adds md5 auth to `pg_hba.conf`, tests connection |
| 4 | `create_backend_env` | Writes `backend/.env` with DB credentials and `NODE_ENV=production`; `chmod 600` |
| 5 | `backup_current_deployment` | Backs up `.env`, deployed frontend, Nginx config, DB dump to `~/bmi_deployments_backup/` |
| 6 | `deploy_backend` | `npm install --production`; runs all `migrations/*.sql` in order |
| 7 | `deploy_frontend` | `npm install`; `npm run build`; copies `dist/` to `/var/www/bmi-health-tracker/`; sets `www-data` ownership |
| 8 | `setup_pm2` | Stops any existing process; starts `src/server.js` as `bmi-backend`; `pm2 save`; configures systemd startup |
| 9 | `configure_nginx` | Auto-detects EC2 public IP via IMDSv2; writes Nginx config with API proxy, gzip, security headers; reloads |
| 10 | `setup_ssl_certificate` | *(if `--with-ssl`)* Installs Certbot; validates domain; runs `certbot --nginx`; tests auto-renewal |
| 11 | `run_health_checks` | Tests backend API, PM2 status, frontend file presence, DB row count |
| 12 | `display_summary` | Prints access URL, useful commands, backup path |

### Files created by the script

| Path | Purpose |
|------|---------|
| `/opt/bmi-app/backend/` | Backend runtime directory — source synced here from the git repo |
| `/opt/bmi-app/backend/.env` | DB credentials, PORT, NODE_ENV — permissions `600` |
| `/etc/nginx/sites-available/bmi-health-tracker` | Nginx virtual host config |
| `/var/www/bmi-health-tracker/` | Built frontend static files served by Nginx |
| `/etc/nginx/sites-enabled/bmi-health-tracker` | Symlink enabling the site |
| `~/bmi_deployments_backup/deployment_TIMESTAMP/` | Timestamped backup (last 5 kept) |

---

## Script Options Reference

| Flag | Effect |
|------|--------|
| *(none)* | Standard interactive deploy |
| `--fresh` | Removes `node_modules`, `package-lock.json`, `dist` before reinstalling |
| `--skip-nginx` | Skips Nginx config (use when Nginx is already configured) |
| `--skip-backup` | Skips creating a backup before deploy |
| `--with-ssl` | Enables Let's Encrypt SSL certificate installation |
| `--skip-ssl` | Suppresses SSL prompt entirely |
| `--domain=example.com` | Pre-sets domain for SSL, avoids interactive prompt |
| `--help` | Prints usage and exits |

---

## Post-Deployment Verification

```bash
# Backend process is online
pm2 status

# Backend responds directly
curl -s http://localhost:3000/api/measurements | head -c 100

# Nginx is serving frontend (expect: 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost/

# API proxy through Nginx works (expect: 200)
curl -s -o /dev/null -w "%{http_code}" http://localhost/api/measurements

# Database schema
PGPASSWORD=<your_password> psql -U bmi_user -d bmidb -h localhost \
  -c "\d measurements"

# Open in browser
# http://<EC2-PUBLIC-IP>
```

---

## SSL / HTTPS Setup

### Option 1 — during initial deployment

```bash
./IMPLEMENTATION_AUTO.sh --with-ssl --domain=bmi.yourdomain.com
```

### Option 2 — after initial deployment

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Update Nginx config with your domain
sudo sed -i "s/server_name .*/server_name bmi.yourdomain.com;/" \
  /etc/nginx/sites-available/bmi-health-tracker
sudo nginx -t && sudo systemctl reload nginx

# Request certificate (handles Nginx config update + HTTP redirect automatically)
sudo certbot --nginx -d bmi.yourdomain.com
```

### Auto-renewal

Certbot installs a systemd timer automatically. Verify it works:

```bash
sudo certbot renew --dry-run
sudo systemctl status certbot.timer
```

> **DNS requirement**: domain A record must resolve to the EC2 public IP before running Certbot. Let's Encrypt validation will fail otherwise.

---

## Updating the Application

Use `AppUpdate_AUTO.sh` for all subsequent code updates.
**Do not re-run** `IMPLEMENTATION_AUTO.sh` — it will overwrite your `.env` and Nginx config.

```bash
cd ~/single-server-3tier-webapp

# Update both frontend + backend (default)
./AppUpdate_AUTO.sh

# Backend only
./AppUpdate_AUTO.sh --backend-only

# Frontend only
./AppUpdate_AUTO.sh --frontend-only

# Skip backup (faster)
./AppUpdate_AUTO.sh --no-backup
```

The update script: pulls from GitHub → backs up → reinstalls dependencies → restarts PM2 → rebuilds frontend → deploys to Nginx → runs health checks.

---

## Useful Day-2 Commands

```bash
# --- PM2 / Backend ---
pm2 status                           # Show all processes
pm2 logs bmi-backend                 # Stream logs live
pm2 logs bmi-backend --lines 50      # Last 50 lines
pm2 restart bmi-backend              # Restart
pm2 reload bmi-backend               # Zero-downtime reload
pm2 stop bmi-backend                 # Stop

# --- Nginx ---
sudo nginx -t                        # Test config syntax
sudo systemctl reload nginx          # Reload config (no downtime)
sudo systemctl restart nginx         # Full restart
sudo tail -f /var/log/nginx/bmi-error.log
sudo tail -f /var/log/nginx/bmi-access.log

# --- PostgreSQL ---
sudo systemctl status postgresql
psql -U bmi_user -d bmidb -h localhost   # App user shell (prompts password)
sudo -u postgres psql                    # Admin shell

# --- SSL ---
sudo certbot certificates            # Show cert status and expiry
sudo certbot renew                   # Force renewal
sudo certbot renew --dry-run         # Test renewal without changes

# --- Firewall ---
sudo ufw status
sudo ufw allow 'Nginx Full'
sudo ufw allow OpenSSH

# --- System ---
df -h                                # Disk usage
free -h                              # Memory usage
```

---

## Troubleshooting

### `SASL: client password must be a string`

`DB_PASSWORD` in `backend/.env` is empty or missing.

```bash
nano /opt/bmi-app/backend/.env
# Set: DB_PASSWORD=yourpassword
pm2 restart bmi-backend
```

---

### `ECONNREFUSED 127.0.0.1:5432`

PostgreSQL is not running.

```bash
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

---

### Nginx 502 Bad Gateway

Backend is down. Check:

```bash
pm2 status
pm2 logs bmi-backend --lines 20
```

---

### Nginx 404 on React frontend routes

The `try_files` directive is missing. Verify:

```bash
sudo grep "try_files" /etc/nginx/sites-available/bmi-health-tracker
```

---

### Re-run migrations (idempotent — safe to re-apply)

```bash
PGPASSWORD=<pw> psql -U bmi_user -d bmidb -h localhost \
  -f /opt/bmi-app/backend/migrations/001_create_measurements.sql
PGPASSWORD=<pw> psql -U bmi_user -d bmidb -h localhost \
  -f /opt/bmi-app/backend/migrations/002_add_measurement_date.sql
```

---

### PM2 not auto-starting after reboot

```bash
pm2 startup systemd -u ubuntu --hp /home/ubuntu
# Copy and run the command it prints, then:
pm2 save
```

---

### SSL certificate request fails

- Verify DNS resolves: `nslookup yourdomain.com` → must return EC2 public IP
- Verify ports 80 and 443 are open in the EC2 Security Group
- Check Certbot logs: `sudo journalctl -u certbot`

---

**Last Updated**: April 18, 2026
**Version**: 4.0
**Replaces**: IMPLEMENTATION_GUIDE.md (v2.1) + IMPLEMENTATION_GUIDE_ORDER.md (v3.0)

---

*MD Sarowar Alam*
Lead DevOps Engineer, WPP Production
📧 Email: sarowar@hotmail.com
🔗 LinkedIn: https://www.linkedin.com/in/sarowar/