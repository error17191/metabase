# Metabase on Docker (with Postgres) and Nginx

This folder contains a secure-by-default Metabase deployment using Docker Compose and Postgres, designed to be reverse-proxied by an existing Nginx on the server.

## Prerequisites
- Docker and Docker Compose v2 installed
- Nginx already installed and running on the server
- A domain pointing to this server, e.g. `metabase.example.com`
- (Recommended) Certbot for automatic Let's Encrypt SSL

## 1) Configure environment

Copy the example environment file and edit values:

```bash
cd /home/error/apps/metabase
cp example.env .env
$EDITOR .env
```

Required keys in `.env`:
- `MB_SITE_URL` (e.g., `https://metabase.example.com` once SSL is set)
- `MB_DB_DBNAME`, `MB_DB_USER`, `MB_DB_PASS` for the internal Metabase metadata database

Optional auto-admin (uncomment in `docker-compose.yml` if used):
- `MB_ADMIN_EMAIL`, `MB_ADMIN_FIRST_NAME`, `MB_ADMIN_LAST_NAME`, `MB_ADMIN_PASSWORD`

## 2) Start Metabase and Postgres

```bash
cd /home/error/apps/metabase
# Pull images and start in background
docker compose pull
docker compose up -d

# Check health
docker compose ps
# Metabase listens on 127.0.0.1:3000 (local-only); exposed via Nginx
```

To view logs:
```bash
docker compose logs -f metabase
```

## 3) Configure Nginx reverse proxy

Use the sample server block and adjust the domain:

File: `/etc/nginx/sites-available/metabase`

```nginx
# Replace metabase.example.com with your domain
server {
    listen 80;
    listen [::]:80;
    server_name metabase.example.com;

    location = /healthz {
        return 200 'ok';
        add_header Content-Type text/plain;
    }

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        proxy_read_timeout 75s;
    }

    map $http_upgrade $connection_upgrade {
        default upgrade;
        ''      close;
    }
}
```

Enable and test:
```bash
sudo ln -s /etc/nginx/sites-available/metabase /etc/nginx/sites-enabled/metabase
sudo nginx -t
sudo systemctl reload nginx
```

At this point, HTTP should proxy to Metabase. Proceed to HTTPS.

## 4) Enable HTTPS with Let's Encrypt (Certbot)

Install certbot (if not already):
```bash
# Ubuntu/Debian example
sudo apt update
sudo apt install -y certbot python3-certbot-nginx
```

Issue certificate and auto-configure Nginx:
```bash
sudo certbot --nginx -d metabase.example.com --redirect -n --agree-tos -m you@example.com
```

This will:
- Obtain a certificate for your domain
- Add SSL blocks to your Nginx site
- Redirect HTTP to HTTPS automatically
- Set up auto-renewal (via systemd timer or cron)

Check renewal:
```bash
sudo certbot renew --dry-run
```

## 5) Initial Metabase setup

Open `https://metabase.example.com` in your browser and follow the onboarding wizard to:
- Create the admin user (unless auto-admin variables were provided)
- Connect your data sources
- Configure email/SMTP if needed

## Maintenance
- Update images: `docker compose pull && docker compose up -d`
- Backup Postgres volume `metabase_db_data` (e.g., via `pg_dump` from within the container)
- View service health: `docker compose ps` and `docker compose logs`

## Notes
- The Compose file binds Metabase to `127.0.0.1:3000` so it is not publicly reachable without Nginx.
- Postgres is on an internal Docker network only and not exposed to the host.
