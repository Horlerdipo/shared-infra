# Shared Infrastructure

A production-ready Docker infrastructure stack with Traefik reverse proxy, automatic SSL certificates, and database services.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start](#quick-start)
- [Services](#services)
  - [Traefik (Reverse Proxy)](#traefik-reverse-proxy)
  - [PostgreSQL (Database)](#postgresql-database)
- [Environment Variables](#environment-variables)
- [Useful Commands](#useful-commands)
- [Troubleshooting](#troubleshooting)

---

## Prerequisites

- Docker and Docker Compose installed on your server
- A domain name pointed to your server's IP address
- Ports 80 and 443 open on your server's firewall

---

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/horlerdipo/shared-infra.git
cd shared-infra
```

### 2. Create Required Directories

```bash
mkdir -p letsencrypt
chmod 700 letsencrypt
```

### 3. Create the Proxy Network

```bash
docker network create proxy
```

### 4. Configure Environment Variables

```bash
cp .env.example .env
```

Edit `.env` with your values. See [Environment Variables](#environment-variables) for details.

### 5. Secure the Dashboard (Required)

Generate a password hash for the Traefik dashboard:

```bash
sudo apt-get install apache2-utils -y
htpasswd -nbB admin YOUR_PASSWORD
```

Add the output to your `.env` file (escape `$` by doubling them):

```env
TRAEFIK_DASHBOARD_USER=admin
TRAEFIK_DASHBOARD_PASSWORD_HASH=$$2y$$05$$your_hash_here
```

### 6. Start Services

```bash
docker compose up -d
```

### 7. Verify Installation

```bash
docker compose ps
```

---

## Services

### Traefik (Reverse Proxy)

Traefik serves as the entry point for all traffic, handling SSL termination and routing requests to appropriate services.

**Features:**
- Automatic HTTPS redirection
- Let's Encrypt SSL certificate generation and renewal
- Secure dashboard with basic authentication
- Docker provider for automatic service discovery

**Accessing the Dashboard:**
1. Open `https://traefik.<your-domain>/dashboard` in your browser
2. Enter the credentials configured in `.env`

**Configuration:**
- HTTP port: 80
- HTTPS port: 443
- Dashboard port: 8080
- SSL certificates stored in `./letsencrypt/acme.json`

**Start Traefik only:**
```bash
docker compose up -d traefik
```

**View Traefik logs:**
```bash
docker compose logs -f traefik
```

---

### PostgreSQL (Database)

PostgreSQL database service with persistent storage and health checks.

**Features:**
- Persistent data storage via Docker volume
- Health checks for container status
- Configurable version via environment variable

**Default Configuration:**
- Internal port: 5432
- External port: Configurable via `POSTGRES_DB_PORT`
- Volume: `postgres_data`

**Connecting to PostgreSQL:**

From another container on the `proxy` network:
```
Host: postgres
Port: 5432
User: <POSTGRES_DB_USERNAME>
Password: <POSTGRES_DB_PASSWORD>
```

From external applications:
```
Host: <your-server-ip>
Port: <POSTGRES_DB_PORT>
User: <POSTGRES_DB_USERNAME>
Password: <POSTGRES_DB_PASSWORD>
```

**Start PostgreSQL only:**
```bash
docker compose up -d postgres
```

**View PostgreSQL logs:**
```bash
docker compose logs -f postgres
```

**Access PostgreSQL CLI:**
```bash
docker exec -it postgres psql -U <POSTGRES_DB_USERNAME>
```

**Backup database:**
```bash
docker exec postgres pg_dump -U <POSTGRES_DB_USERNAME> <database_name> > backup.sql
```

**Restore database:**
```bash
cat backup.sql | docker exec -i postgres psql -U <POSTGRES_DB_USERNAME> <database_name>
```

---

## Environment Variables

Create a `.env` file based on `.env.example`:

```env
# PostgreSQL
POSTGRES_VERSION=18
POSTGRES_DB_USERNAME=your_username
POSTGRES_DB_PASSWORD=your_secure_password
POSTGRES_DB_PORT=5432
POSTGRES_DB_NAME=your_database

# MySQL (optional)
MYSQL_VERSION=8.0
MYSQL_DB_USERNAME=your_username
MYSQL_DB_PASSWORD=your_secure_password
MYSQL_DB_PORT=3306

# Redis (optional)
REDIS_VERSION=7
REDIS_DB_USERNAME=your_username
REDIS_DB_PASSWORD=your_secure_password
REDIS_DB_PORT=6379

# TLS (Let's Encrypt)
DOMAIN_NAME=example.com
TLS_EMAIL_ADDRESS=your-email@example.com

# Traefik Dashboard Authentication
TRAEFIK_DASHBOARD_USER=admin
TRAEFIK_DASHBOARD_PASSWORD_HASH=$$2y$$05$$your_hash_here
```

### Required Variables

| Variable | Description |
|----------|-------------|
| `DOMAIN_NAME` | Your domain without prefix (e.g., `example.com`) |
| `TLS_EMAIL_ADDRESS` | Email for Let's Encrypt notifications |
| `TRAEFIK_DASHBOARD_USER` | Dashboard username |
| `TRAEFIK_DASHBOARD_PASSWORD_HASH` | Hashed password (use `htpasswd -nbB`) |

### PostgreSQL Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `POSTGRES_VERSION` | PostgreSQL version | `18` |
| `POSTGRES_DB_USERNAME` | Database username | - |
| `POSTGRES_DB_PASSWORD` | Database password | - |
| `POSTGRES_DB_PORT` | External port | `5432` |
| `POSTGRES_DB_NAME` | Default database name | - |

---

## Useful Commands

### Service Management

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# Restart a specific service
docker compose restart <service_name>

# View logs for a service
docker compose logs -f <service_name>
```

### Updates

```bash
# Pull latest images
docker compose pull

# Recreate containers with new images
docker compose up -d
```

### Cleanup

```bash
# Remove stopped containers
docker compose down

# Remove volumes (WARNING: deletes all data)
docker compose down -v
```

---

## Troubleshooting

### Certificate Issues

If SSL certificates aren't being generated:

1. Ensure port 80 is accessible from the internet
2. Check logs: `docker compose logs traefik`
3. Verify DNS points to your server IP
4. Check Let's Encrypt rate limits (5 certificates per domain per week)

### Service Not Reachable

1. Verify the service is on the `proxy` network
2. Check Traefik labels are correctly formatted
3. Ensure `traefik.enable=true` is set
4. Verify the Host rule matches your domain

### PostgreSQL Connection Issues

1. Verify PostgreSQL is running: `docker compose ps postgres`
2. Check health status: `docker inspect postgres | grep -A 10 Health`
3. Verify credentials in `.env` match connection string
4. Check if port is exposed correctly

### Network Issues

```bash
# Recreate the proxy network
docker network rm proxy
docker network create proxy
docker compose up -d
```

---

## File Structure

```
shared-infra/
├── docker-compose.yml    # Main compose file
├── .env.example          # Environment template
├── .env                  # Your environment (gitignored)
├── .gitignore            # Git ignore rules
├── letsencrypt/          # SSL certificates (gitignored)
│   └── acme.json         # Let's Encrypt storage
└── README.md             # This file
```

---

## License

MIT
