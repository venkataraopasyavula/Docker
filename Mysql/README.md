Here’s a practical guide to **running MySQL in Docker** on **Windows**, **Linux**, and **macOS** (they are almost identical once Docker is installed).

The **recommended way in 2025–2026** is **using the official image directly** (`mysql:8.0` or `mysql:latest`) — **most people do not need a custom Dockerfile** for everyday development/testing.

### 1. Quick & Recommended Way (No custom Dockerfile needed)

**Prerequisites** (same on Win/Linux/mac)

- Docker Desktop (Windows / macOS) or Docker Engine + docker compose plugin (Linux) is installed and running
- At least 2–4 GB RAM free for comfortable usage

**One-line start (most common pattern):**

```bash
docker run -d \
  --name mysql8 \
  -e MYSQL_ROOT_PASSWORD=YourStrongPass123 \
  -e MYSQL_DATABASE=shop \
  -e MYSQL_USER=appuser \
  -e MYSQL_PASSWORD=apppass123 \
  -p 3306:3306 \
  -v mysql-data:/var/lib/mysql \
  --restart unless-stopped \
  mysql:8.0
```

Explanation of flags:

| Flag                        | Meaning                                                                 |
|:----------------------------|:------------------------------------------------------------------------|
| `-d`                        | detached (run in background)                                            |
| `--name mysql8`             | container name                                                          |
| `-e MYSQL_ROOT_PASSWORD=…`  | **required** — root password                                            |
| `-e MYSQL_DATABASE=…`       | optional — creates this database on first start                         |
| `-e MYSQL_USER=…`           | optional — creates non-root user                                        |
| `-e MYSQL_PASSWORD=…`       | password for the above user                                             |
| `-p 3306:3306`              | expose MySQL port to host                                               |
| `-v mysql-data:/var/lib/mysql` | **persistent data** (named volume — recommended)                     |
| `--restart unless-stopped`  | auto-start after reboot (unless you manually stop it)                   |
| `mysql:8.0`                 | use stable 8.0 series (or `mysql:8.4`, `mysql:9.0`, `mysql:latest`)     |

Check it's running:

```bash
docker ps
docker logs mysql8
```

Connect from host:

```bash
# Using mysql client (if installed)
mysql -h 127.0.0.1 -u root -p

# or using Docker (no client needed on host)
docker exec -it mysql8 mysql -uroot -p
```

### 2. When You Actually Need a Custom Dockerfile

Use cases:

- You want to bake in **initial schema/data** (`.sql` files)
- You want custom `my.cnf` settings baked in
- You want a reproducible image for CI/CD

**Minimal useful custom Dockerfile** (2025–2026 style)

```dockerfile
# Use explicit version - avoids surprises when mysql:latest changes
FROM mysql:8.0

# Optional: set timezone (very common need)
ENV TZ=Asia/Kolkata

# These can also be set at runtime with -e, but baking them is fine for dev images
# ENV MYSQL_ROOT_PASSWORD=dev123          # ← usually set at runtime instead
# ENV MYSQL_DATABASE=app
# ENV MYSQL_USER=app
# ENV MYSQL_PASSWORD=apppass

# Copy initialization scripts (they run exactly once on first start)
# Files are executed in alphabetical order
COPY 01-schema.sql  /docker-entrypoint-initdb.d/
COPY 02-seed.sql    /docker-entrypoint-initdb.d/

# Optional: custom config (small tweaks only — big changes better via config file mount)
# COPY my-custom.cnf /etc/mysql/conf.d/
```

**Example folder structure**

```
mysql-docker/
├── Dockerfile
├── 01-schema.sql
└── 02-seed.sql
```

**Build & run it**

```bash
# Build
docker build -t my-mysql:dev .

# Run (still need to provide root password!)
docker run -d \
  --name my-mysql \
  -e MYSQL_ROOT_PASSWORD=YourStrongPass123 \
  -p 3306:3306 \
  -v mysql-data:/var/lib/mysql \
  my-mysql:dev
```

### 3. docker-compose.yml (Very Popular & Recommended)

Much cleaner for development:

```yaml
services:
  mysql:
    image: mysql:8.0
    # build: .               # ← uncomment if using custom Dockerfile
    container_name: mysql8
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: YourStrongPass123
      MYSQL_DATABASE: shop
      MYSQL_USER: appuser
      MYSQL_PASSWORD: secret123
      TZ: Asia/Kolkata
    ports:
      - "3306:3306"
    volumes:
      - mysql-data:/var/lib/mysql
      # Optional: custom config
      # - ./my.cnf:/etc/mysql/conf.d/custom.cnf
      # Optional: init scripts
      # - ./initdb/:/docker-entrypoint-initdb.d/

volumes:
  mysql-data:
```

Start → `docker compose up -d`

### Platform-Specific Notes (Windows / macOS / Linux)

| Platform   | Docker             | Port 3306 conflict?          | Volume path hint                              | Performance tip                     |
|:-----------|:-------------------|:-----------------------------|-----------------------------------------------|:------------------------------------|
| **Windows** | Docker Desktop     | Yes — often used by WAMP/XAMPP | Use named volume (not bind mount to C:)       | Use WSL 2 backend                   |
| **macOS**   | Docker Desktop     | Rare                         | Named volume is best                          | Increase disk/CPU in Docker settings |
| **Linux**   | Docker Engine      | Possible (if mysql installed)| `/var/lib/docker/volumes/` or bind mount      | Best performance — use `--network host` if needed |

### Quick Troubleshooting Commands

```bash
# See logs
docker logs -f mysql8

# Connect from inside container
docker exec -it mysql8 mysql -uroot -p

# Stop & remove (if testing)
docker stop mysql8
docker rm mysql8
docker volume rm mysql-data   # ← careful — deletes all data!
```

That should cover most real-world usage patterns in 2026.

Let me know if you want a version with **healthcheck**, **multiple databases on startup**, **specific MySQL 8.4/9.0** settings, or **replication** example!
