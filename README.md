# Shared Infrastructure - Traefik Reverse Proxy

This project sets up a Traefik reverse proxy with automatic SSL certificate generation via Let's Encrypt for your server infrastructure.

## Prerequisites

- Docker and Docker Compose installed on your server
- A domain name pointed to your server's IP address
- Ports 80 and 443 open on your server's firewall

## Setup Instructions

### 1. Clone the Repository

```bash
git clone https://github.com/horlerdipo/shared-infra.git
cd shared-infra
```

### 2. Create Required Directories

Create the `letsencrypt` directory for SSL certificates:

```bash
mkdir -p letsencrypt
```

### 3. Configure Environment Variables

Copy the example environment file and edit it with your values:

```bash
cp .env.example .env
```

Edit `.env` and fill in the required values:

```env
# Database credentials (if using databases)
MYSQL_DB_USERNAME=your_mysql_username
MYSQL_DB_PASSWORD=your_mysql_password
MYSQL_DB_PORT=3306

POSTGRES_DB_USERNAME=your_postgres_username
POSTGRES_DB_PASSWORD=your_postgres_password
POSTGRES_DB_PORT=5432

REDIS_DB_USERNAME=your_redis_username
REDIS_DB_PASSWORD=your_redis_password
REDIS_DB_PORT=6379

# TLS (Let's Encrypt)
DOMAIN_NAME=example.com
TLS_EMAIL_ADDRESS=your-email@example.com
```

**Important:** 
- `DOMAIN_NAME` should be without prefix (e.g., `example.com` not `www.example.com` or `https://example.com`)
- `TLS_EMAIL_ADDRESS` is used for Let's Encrypt certificate notifications

### 4. Create the Proxy Network

Create the Docker network that Traefik uses to communicate with your services:

```bash
docker network create proxy
```

### 5. Set Proper Permissions

Ensure proper permissions for the letsencrypt directory:

```bash
chmod 700 letsencrypt
```

### 6. Secure the Traefik Dashboard (Required for Production)

The Traefik dashboard must be secured with authentication in production environments.

1. Generate a password hash using `htpasswd`:

```bash
# Install apache2-utils if not already installed
sudo apt-get install apache2-utils -y

# Generate the password hash (replace YOUR_PASSWORD with a strong password)
htpasswd -nbB admin YOUR_PASSWORD
```

2. Copy the output (it will look like `admin:$2y$05$...`). You'll need to escape any `$` characters by doubling them (e.g., `$` becomes `$$`).

3. Add the following environment variable to your `.env` file:

```env
# Traefik Dashboard Authentication
DASHBOARD_USER=admin
DASHBOARD_PASSWORD_HASH=$$2y$$05$$your_hash_here
```

4. The `docker-compose.yml` already includes the dashboard authentication middleware. Verify these labels are present:

```yaml
labels:
  - "traefik.http.middlewares.dashboard-auth.basicauth.users=${DASHBOARD_USER}:${DASHBOARD_PASSWORD_HASH}"
```

**Important:** Use a strong, unique password for the dashboard. Consider using a password manager to generate and store it.

### 7. Start Traefik

Launch the Traefik proxy:

```bash
docker compose up -d
```

### 8. Verify Installation

Check that Traefik is running:

```bash
docker compose ps
```

Test the whoami service (replace `example.com` with your domain):

```bash
curl -H "Host: whoami.example.com" http://localhost
```

## Accessing the Dashboard

The Traefik dashboard is secured with basic authentication. To access it:

1. Open `https://traefik.${DOMAIN_NAME}/dashboard`  in your browser
2. When prompted, enter the username and password you configured in `.env`
3. The dashboard shows all routed services, their status, and SSL certificates

**Note:** The dashboard is only accessible over HTTPS with valid credentials.

## Adding New Services

To add a new service to the Traefik proxy:

1. Add the service to `docker-compose.yml` or create a separate compose file
2. Connect the service to the `proxy` network
3. Add Traefik labels to configure routing

Example:

```yaml
services:
  myapp:
    image: myapp:latest
    restart: unless-stopped
    networks:
      - proxy
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.myapp.rule=Host(`myapp.${DOMAIN_NAME}`)"
      - "traefik.http.routers.myapp.entrypoints=websecure"
      - "traefik.http.routers.myapp.tls=true"
      - "traefik.http.routers.myapp.tls.certresolver=le"

networks:
  proxy:
    external: true
```

## SSL Certificates

Traefik automatically requests and renews SSL certificates from Let's Encrypt for all configured domains. Certificates are stored in `./letsencrypt/acme.json`.

### Certificate Storage

The `acme.json` file contains sensitive certificate data. Ensure:

- The file is included in `.gitignore` (already configured)
- Proper file permissions are set: `chmod 600 letsencrypt/acme.json`

## Useful Commands

### View Logs

```bash
docker compose logs -f traefik
```

### Restart Traefik

```bash
docker compose restart traefik
```

### Stop All Services

```bash
docker compose down
```

### Update Traefik

```bash
docker compose pull traefik
docker compose up -d traefik
```

## Troubleshooting

### Certificate Issues

If certificates aren't being generated:

1. Ensure port 80 is accessible from the internet (required for HTTP-01 challenge)
2. Check logs: `docker compose logs traefik`
3. Verify your domain DNS points to the correct server IP
4. Check rate limits at Let's Encrypt (max 5 certificates per domain per week)

### Service Not Reachable

If a service isn't accessible:

1. Verify the service is connected to the `proxy` network
2. Check Traefik labels are correctly formatted
3. Ensure `traefik.enable=true` is set
4. Verify the Host rule matches your domain

### Network Issues

If you see network errors:

```bash
# Recreate the proxy network
docker network rm proxy
docker network create proxy
docker compose up -d
```

## File Structure

```
shared-infra/
├── docker-compose.yml    # Main compose file
├── .env.example          # Environment template
├── .env                  # Your environment (gitignored)
├── .gitignore            # Git ignore rules
├── letsencrypt/          # SSL certificates (gitignored)
│   └── acme.json         # Let's Encrypt certificate storage
└── README.md             # This file
```

## License

MIT
