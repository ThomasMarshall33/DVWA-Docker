# Dockerized DVWA (Damn Vulnerable Web Application)

## Project Overview

This project deploys DVWA (Damn Vulnerable Web Application) using Docker Compose with a multi-container architecture. DVWA is a deliberately vulnerable web application used by security professionals and students to practice common web application attacks in a safe, legal environment.

### Architecture

The application uses three containers connected via a Docker bridge network:

- **Nginx** (reverse proxy) — Listens on port 8080 and forwards traffic to the DVWA container
- **DVWA** (web application) — PHP-based vulnerable web app for security testing
- **MariaDB** (database) — Stores DVWA's application data including user accounts and vulnerability configurations

```
Browser (mgmt01) → Nginx (port 8080) → DVWA (port 80) → MariaDB (port 3306)
```

All three containers communicate over a private Docker bridge network called `dvwa-network`.

---

## Environment

| System | Role | IP Address |
|--------|------|------------|
| docker01 | Docker host (Ubuntu Server) | 10.0.5.12 |
| mgmt01 | Management workstation / client | Used to access DVWA via browser |

---

## Prerequisites

- Docker and Docker Compose installed on the Docker host
- PuTTY or SSH access to the Docker VM

Verify Docker is installed:
```bash
docker --version
docker-compose --version
```

---

## Deployment Steps

### Step 1: Create the Project Directory

```bash
mkdir ~/dvwa-docker && cd ~/dvwa-docker
```

### Step 2: Create the Docker Compose File

```bash
nano docker-compose.yml
```

Paste the contents of [docker-compose.yml](docker-compose.yml) into the file and save (`Ctrl+X`, `Y`, `Enter`).

This file defines three services:
- **nginx** — Pulls the latest Nginx image, maps port 8080 on the host to port 80 in the container, and mounts our custom nginx.conf
- **dvwa** — Pulls the official DVWA image and connects to the MariaDB database using environment variables
- **db** — Runs MariaDB 10.6 with a persistent volume for database storage

### Step 3: Create the Nginx Configuration

```bash
nano nginx.conf
```

Paste the contents of [nginx.conf](nginx.conf) into the file and save.

This configures Nginx as a reverse proxy that forwards all incoming HTTP requests to the DVWA container.

### Step 4: Launch the Containers

```bash
docker-compose up -d
```

This pulls all three images and starts the containers in detached mode.

### Step 5: Verify Containers are Running

```bash
docker-compose ps
```

All three containers (`nginx-proxy`, `dvwa-app`, `dvwa-db`) should show a status of "Up."

### Step 6: Access DVWA

From mgmt01, open a web browser and navigate to:

```
http://10.0.5.12:8080
```

### Step 7: Initialize the Database

1. On the DVWA setup page, click **Create / Reset Database**
2. You will be redirected to the login page
3. Log in with default credentials: `admin` / `password`

### Step 8: Verify the Dashboard

After logging in, you should see the full DVWA dashboard with vulnerability modules listed on the left sidebar, including SQL Injection, XSS, Command Injection, File Upload, and more.

---

## Troubleshooting

### Issue 1: "no configuration file provided: not found"

**Problem:** Running `docker-compose up -d` returned "no configuration file provided: not found."

**Cause:** The docker-compose file was named `docer-compose.yml` (missing the "k" in docker) due to a typo when creating the file.

**Fix:**
```bash
mv docer-compose.yml docker-compose.yml
```

### Issue 2: "Bind for 0.0.0.0:80 failed: port is already allocated"

**Problem:** After fixing the filename, the containers were created but Nginx failed to start because port 80 was already in use on the Docker host.

**Cause:** Another service was occupying port 80. Running `ss -tlnp | grep :80` showed docker-proxy had partially claimed the port from a previous failed attempt.

**Fix:** Changed the Nginx port mapping from `"80:80"` to `"8080:80"` in docker-compose.yml. This maps host port 8080 to container port 80 so DVWA is accessible at `http://10.0.5.12:8080`.

```yaml
ports:
  - "8080:80"
```

After editing, tore down and restarted:
```bash
docker-compose down
docker-compose up -d
```

### Issue 3: MySQL SQL Syntax Error on Database Setup

**Problem:** Clicking "Create / Reset Database" in DVWA returned a fatal MySQL error: `Uncaught mysqli_sql_exception: You have an error in your SQL syntax... near 'IF NOT EXISTS role VARCHAR(20) DEFAULT 'user'`

**Cause:** The DVWA image uses SQL syntax that is not compatible with MySQL 5.7 or MySQL 8.0. This is a known compatibility issue.

**Fix:** Switched from MySQL to MariaDB 10.6 by updating docker-compose.yml:

Changed the db service image from `mysql:8.0` to `mariadb:10.6` and updated the environment variable prefixes from `MYSQL_` to `MARIADB_`:

```yaml
db:
  image: mariadb:10.6
  container_name: dvwa-db
  environment:
    - MARIADB_ROOT_PASSWORD=root_password
    - MARIADB_DATABASE=dvwa
    - MARIADB_USER=dvwa
    - MARIADB_PASSWORD=dvwa_password
```

Then removed the old database volume and restarted:
```bash
docker-compose down
docker volume rm dvwa-docker_dvwa-db-data
docker-compose up -d
```

---

## Configuration Files

- [docker-compose.yml](docker-compose.yml) — Defines the three-container architecture, networking, and volumes
- [nginx.conf](nginx.conf) — Nginx reverse proxy configuration that forwards traffic to DVWA

---

## Useful Docker Commands

| Command | Description |
|---------|-------------|
| `docker-compose up -d` | Start all containers in detached mode |
| `docker-compose down` | Stop and remove all containers and networks |
| `docker-compose ps` | Show status of all containers |
| `docker-compose logs -f` | Follow live logs from all containers |
| `docker exec -it dvwa-app bash` | Open a shell inside the DVWA container |

---



