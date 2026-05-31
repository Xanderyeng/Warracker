# Warracker Deployment Guide

This guide covers two deployment paths:

- **[Option A — Coolify](#option-a-deploy-with-coolify)** — Recommended if you want a managed platform with automatic SSL, a GUI, and one-click redeployments from Git.
- **[Option B — Bare VPS with Docker Compose](#option-b-deploy-on-a-vps-with-docker-compose)** — Recommended if you want full control and a minimal setup (Contabo, Hetzner, DigitalOcean, etc.).

---

## Prerequisites (Both Options)

Before you begin, make sure you have:

- A **domain name** you control (e.g. `warracker.yourdomain.com`), with access to its DNS settings.
- A **VPS** running Ubuntu 22.04 LTS or 24.04 LTS (the guide uses Ubuntu; adjust package manager commands for other distros).
- For Option A: a running **Coolify instance** (v4) on that VPS or another server.
- For Option B: SSH access to your VPS.

---

## Option A — Deploy with Coolify

Coolify is a self-hosted platform that manages deployments, SSL certificates, and reverse proxying automatically on top of Docker.

### Step 1 — Install Coolify on Your VPS

Skip this step if Coolify is already running.

1. SSH into your VPS:
   ```bash
   ssh root@YOUR_SERVER_IP
   ```

2. Run the official Coolify installer:
   ```bash
   curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
   ```

3. The installer will:
   - Install Docker and Docker Compose
   - Pull and start all Coolify services
   - Print a URL and credentials when it finishes (usually `http://YOUR_SERVER_IP:8000`)

4. Open `http://YOUR_SERVER_IP:8000` in your browser and complete the initial setup wizard (create your admin account, set the instance URL).

> **Tip:** Coolify itself can be placed behind your own domain with SSL — the setup wizard guides you through this.

---

### Step 2 — Connect Your Git Repository

1. In the Coolify dashboard, go to **Sources** (left sidebar).
2. Click **Add** and choose your Git provider (GitHub, GitLab, Gitea, or Bitbucket).
3. Follow the OAuth flow or paste a Personal Access Token to grant Coolify read access to your repositories.
4. Confirm the source appears as **Connected** in the Sources list.

---

### Step 3 — Create a New Project

1. In Coolify, go to **Projects → New Project**.
2. Give it a name, e.g. `Warracker`.
3. Select (or create) an **Environment** — `production` is fine.

---

### Step 4 — Add a New Resource (Docker Compose)

1. Inside your project, click **Add Resource**.
2. Choose **Docker Compose**.
3. Select your connected Git source, then choose the Warracker repository.
4. Set the **Branch** to `deploy/coolify-vps`.
5. Set **Docker Compose Location** to `docker-compose.coolify.yml`.
6. Click **Continue**.

---

### Step 5 — Configure the Service and Domain

1. Coolify will parse the compose file and list the services. Select **`warracker`** as the service to expose publicly.
2. Set **Port** to `80` (this is the port Nginx listens on inside the container).
3. Under **Domains**, add your domain:
   ```
   https://warracker.yourdomain.com
   ```
   Coolify will automatically request a Let's Encrypt certificate for this domain.

   > **DNS requirement:** Before deploying, create an **A record** in your DNS provider pointing `warracker.yourdomain.com` to your VPS IP. Let's Encrypt needs to reach this domain to issue the certificate.

---

### Step 6 — Set Environment Variables

In Coolify, go to the **Environment Variables** tab for the resource. Add each variable below. Never leave the security-critical ones at their defaults.

#### Required — Security

| Variable | Description | Example |
|---|---|---|
| `SECRET_KEY` | Flask session & JWT signing key. Use a long random string. | `openssl rand -hex 32` output |
| `DB_PASSWORD` | PostgreSQL password for the app user | `a_very_strong_password` |
| `DB_ADMIN_PASSWORD` | PostgreSQL password for the admin user | `another_strong_password` |

#### Required — URLs

| Variable | Description | Example |
|---|---|---|
| `FRONTEND_URL` | Public URL of the app (with https) | `https://warracker.yourdomain.com` |
| `APP_BASE_URL` | Same as above — used for email links | `https://warracker.yourdomain.com` |

#### Optional — Email (SMTP)

Required if you want password-reset emails and notifications.

| Variable | Description | Example |
|---|---|---|
| `SMTP_HOST` | Your SMTP server hostname | `smtp.gmail.com` |
| `SMTP_PORT` | SMTP port | `587` |
| `SMTP_USERNAME` | SMTP login username | `you@gmail.com` |
| `SMTP_PASSWORD` | SMTP password or app password | `your_app_password` |
| `SMTP_FROM_ADDRESS` | From address in emails | `warracker@yourdomain.com` |

#### Optional — Apprise Push Notifications

| Variable | Description | Example |
|---|---|---|
| `APPRISE_ENABLED` | Turn on Apprise notifications | `true` |
| `APPRISE_URLS` | Comma-separated Apprise notification URLs | `discord://id/token` |
| `APPRISE_EXPIRATION_DAYS` | Days before expiry to notify | `7,30` |
| `APPRISE_NOTIFICATION_TIME` | Time of day to send (24h HH:MM) | `09:00` |

#### Optional — OIDC / SSO

| Variable | Description |
|---|---|
| `OIDC_CLIENT_ID` | Client ID from your OIDC provider |
| `OIDC_CLIENT_SECRET` | Client secret from your OIDC provider |
| `OIDC_ISSUER_URL` | OIDC discovery URL (e.g. Keycloak realm URL) |
| `OIDC_SCOPE` | Scopes to request (default: `openid email profile`) |
| `FLASK_ENV` | Set to `production` to force HTTPS redirects with OIDC |

---

### Step 7 — Deploy

1. Click **Deploy** (or **Save and Deploy**).
2. Coolify will:
   - Clone the repository
   - Build the Docker image using the `Dockerfile` in the repo root
   - Start the `warracker` and `warrackerdb` containers on an internal network
   - Obtain and apply the Let's Encrypt SSL certificate
   - Wire Traefik to route `https://warracker.yourdomain.com` → container port 80

3. Watch the **Deploy Logs** tab in real time. A successful deployment ends with supervisor starting and the health check passing.

---

### Step 8 — Verify the Deployment

1. Open `https://warracker.yourdomain.com` in your browser. You should see the Warracker login page with a valid SSL certificate.
2. Register your first account and log in.
3. In Coolify, check the **Logs** tab to confirm no errors are streaming from the app.

---

### Step 9 — Set Up Automatic Redeployments (Optional)

Coolify can redeploy automatically on every push to the branch.

1. Go to your resource's **Configuration** tab.
2. Enable **Auto Deploy** and select the `deploy/coolify-vps` branch.
3. Copy the **Webhook URL** Coolify provides.
4. In your Git provider (e.g. GitHub → repo → Settings → Webhooks), add that URL with content type `application/json` and the `push` event.

From now on, every push to `deploy/coolify-vps` triggers a rebuild and redeployment with zero downtime.

---

### Coolify — Useful Operations

**View live logs:**
Coolify dashboard → your resource → **Logs** tab.

**Restart the app:**
Coolify dashboard → your resource → **Restart** button.

**Run a one-off command inside the container:**
```bash
# SSH into your Coolify server, then:
docker exec -it <warracker_container_name> bash
```

**Back up the database:**
```bash
docker exec warrackerdb pg_dump -U warranty_user warranty_test > backup_$(date +%Y%m%d).sql
```

---

---

## Option B — Deploy on a VPS with Docker Compose

This section uses a plain Ubuntu VPS (Contabo, Hetzner, DigitalOcean, etc.) with Docker Compose and Nginx as the reverse proxy with Certbot for SSL.

### Step 1 — Connect to Your VPS

```bash
ssh root@YOUR_SERVER_IP
```

If you are a non-root sudo user, substitute `sudo` where needed.

---

### Step 2 — Update the System

```bash
apt update && apt upgrade -y
```

---

### Step 3 — Install Docker and Docker Compose

Docker's official convenience script installs both:

```bash
curl -fsSL https://get.docker.com | sh
```

Verify:
```bash
docker --version
docker compose version
```

You should see output like `Docker version 26.x.x` and `Docker Compose version v2.x.x`.

If you are running as a non-root user and want to run Docker without `sudo`:
```bash
usermod -aG docker $USER
newgrp docker
```

---

### Step 4 — Install Nginx and Certbot

Nginx will act as the reverse proxy in front of Docker. Certbot manages Let's Encrypt SSL certificates.

```bash
apt install -y nginx certbot python3-certbot-nginx
```

---

### Step 5 — Point Your Domain to the Server

In your domain registrar's DNS settings, create an **A record**:

| Type | Name | Value |
|---|---|---|
| A | `warracker` | `YOUR_SERVER_IP` |

This makes `warracker.yourdomain.com` resolve to your server. DNS propagation can take a few minutes to a few hours. Check with:
```bash
dig warracker.yourdomain.com +short
```

Wait until it returns your server IP before continuing.

---

### Step 6 — Clone the Repository

```bash
cd /opt
git clone https://github.com/sassanix/Warracker.git warracker
cd warracker
git checkout deploy/coolify-vps
```

---

### Step 7 — Create the Environment File

Copy the example file and edit it:

```bash
cp env.example .env
nano .env
```

Set at minimum the following values (change every placeholder):

```dotenv
# Database passwords — use strong unique passwords
DB_PASSWORD=a_very_strong_db_password
DB_ADMIN_PASSWORD=another_strong_admin_password

# Flask secret key — generate one with: openssl rand -hex 32
SECRET_KEY=paste_your_generated_key_here

# Public URLs — replace with your actual domain
FRONTEND_URL=https://warracker.yourdomain.com
APP_BASE_URL=https://warracker.yourdomain.com

# Email (optional — needed for password resets)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USERNAME=you@gmail.com
SMTP_PASSWORD=your_app_password
SMTP_FROM_ADDRESS=warracker@yourdomain.com
```

Save and close (`Ctrl+O`, `Enter`, `Ctrl+X` in nano).

Secure the file so only root can read it:
```bash
chmod 600 .env
```

---

### Step 8 — Build and Start the Containers

Use the standard `docker-compose.yml` (the original, which exposes port 8005):

```bash
docker compose up -d --build
```

This will:
1. Build the Warracker image from the `Dockerfile`
2. Pull the `postgres:15-alpine` image
3. Start both containers
4. Create the named volumes (`postgres_data`, `uploads`)

Watch the startup logs:
```bash
docker compose logs -f warracker
```

Wait until you see supervisor starting all services and the health check passing before continuing. Press `Ctrl+C` to stop following logs.

Verify both containers are running:
```bash
docker compose ps
```

Both `warracker` and `warrackerdb` should show `Up (healthy)`.

---

### Step 9 — Configure Nginx as a Reverse Proxy

Create a new Nginx site configuration:

```bash
nano /etc/nginx/sites-available/warracker
```

Paste the following (replace `warracker.yourdomain.com` with your actual domain):

```nginx
server {
    listen 80;
    server_name warracker.yourdomain.com;

    # Certbot will modify this block to add SSL — leave the rest as-is
    location / {
        proxy_pass http://127.0.0.1:8005;
        proxy_http_version 1.1;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Required for file uploads
        client_max_body_size 32M;

        # WebSocket support (if needed in future)
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

Enable the site and test the configuration:

```bash
ln -s /etc/nginx/sites-available/warracker /etc/nginx/sites-enabled/warracker
nginx -t
```

You should see `syntax is ok` and `test is successful`. If not, review the config for typos.

Reload Nginx:
```bash
systemctl reload nginx
```

---

### Step 10 — Obtain an SSL Certificate with Certbot

```bash
certbot --nginx -d warracker.yourdomain.com
```

Certbot will:
1. Verify you own the domain (via the HTTP-01 challenge — this is why Nginx must be running)
2. Download the certificate from Let's Encrypt
3. Automatically modify your Nginx config to enable HTTPS and redirect HTTP → HTTPS
4. Set up a cron job to auto-renew the certificate

When prompted:
- Enter your email address for renewal reminders
- Agree to the Terms of Service
- Choose whether to share your email with the EFF (optional)
- Select option **2** (Redirect) to force HTTPS

Test the renewal timer:
```bash
certbot renew --dry-run
```

---

### Step 11 — Verify the Deployment

1. Open `https://warracker.yourdomain.com` in your browser.
2. You should see the Warracker login page with a valid padlock (Let's Encrypt certificate).
3. Register your first account and confirm login works.

---

### Step 12 — Configure the Firewall

Allow only the ports you need:

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw enable
```

`Nginx Full` allows both ports 80 (HTTP → redirects to HTTPS) and 443 (HTTPS). Port 8005 is intentionally **not** opened — Docker listens on it only on localhost (`127.0.0.1:8005`), so it is not reachable from the internet.

Confirm the rules:
```bash
ufw status
```

---

### Step 13 — Enable Systemd Auto-Start (Optional but Recommended)

Create a systemd service so Warracker starts automatically on server reboot:

```bash
nano /etc/systemd/system/warracker.service
```

Paste:

```ini
[Unit]
Description=Warracker Docker Compose Application
Requires=docker.service
After=docker.service network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/opt/warracker
ExecStart=/usr/bin/docker compose up -d --build
ExecStop=/usr/bin/docker compose down
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

Enable and start it:
```bash
systemctl daemon-reload
systemctl enable warracker
systemctl start warracker
```

---

### Updating Warracker (VPS)

To pull new code and rebuild:

```bash
cd /opt/warracker
git pull
docker compose up -d --build
```

Docker Compose will rebuild only the changed layers and restart only the affected containers. The database and uploaded files are preserved in named volumes.

---

### VPS — Useful Operations

**View live app logs:**
```bash
docker compose logs -f warracker
```

**View database logs:**
```bash
docker compose logs -f warrackerdb
```

**Restart the app container only:**
```bash
docker compose restart warracker
```

**Open a shell inside the app container:**
```bash
docker exec -it warracker-warracker-1 bash
```

**Back up the database:**
```bash
docker exec warrackerdb pg_dump -U warranty_user warranty_test \
  > /opt/warracker/backups/backup_$(date +%Y%m%d_%H%M%S).sql
```

**Restore a database backup:**
```bash
docker exec -i warrackerdb psql -U warranty_user warranty_test \
  < /opt/warracker/backups/backup_20240101_120000.sql
```

**Stop everything:**
```bash
docker compose down
```

**Stop and delete all data (destructive — deletes database and uploads):**
```bash
docker compose down -v
```

---

## Troubleshooting

### Container won't start

Check logs for the specific container:
```bash
docker compose logs warracker
docker compose logs warrackerdb
```

Common causes:
- Missing or wrong `DB_PASSWORD` / `DB_NAME` mismatch between services
- Port 8005 already in use on the host — check with `ss -tlnp | grep 8005`

### SSL certificate fails

- Confirm DNS is fully propagated: `dig warracker.yourdomain.com +short` returns your server IP
- Confirm Nginx is running: `systemctl status nginx`
- Confirm port 80 is reachable from the internet (firewall rules)

### App loads but login/redirect is wrong

- Verify `FRONTEND_URL` and `APP_BASE_URL` are set to the correct HTTPS domain in your `.env`
- Restart the container after any `.env` change: `docker compose up -d`

### Uploads not persisting after restart

- VPS path: Named volume `uploads` is managed by Docker — data persists as long as you don't run `docker compose down -v`
- Coolify path: The `uploads` named volume is managed by Coolify — it persists across redeployments automatically

### Database migration errors on startup

Check the app logs for migration output:
```bash
docker compose logs warracker | grep -i migrat
```

Migrations run automatically at startup via the entrypoint script. If a migration fails, the app will not start. Examine the full log for the SQL error.
