# 🧪 Online Study Infrastructure Template

A modular boilerplate repository designed to help researchers build and deploy online behavioral experiments, dynamic agent steering studies, and human-AI interaction evaluations.

This template provides the folder structure, basic configurations, and security practices to build a robust, containerized study environment.

---

## 🏗️ System Architecture

This template is pre-configured for a full-stack containerized deployment:

- **Frontend (Gateway):** Uses Nginx as a reverse proxy to serve static UI assets and route `/api/` traffic to the backend.
- **Backend (API):** A framework to handle study logic and API calls.
- **Database:** A PostgreSQL database to store participant logs securely.
- **Metrics:** Grafana monitoring to track active sessions.

---

## 🚀 Getting Started

### Prerequisites
* [Docker](https://docs.docker.com/get-docker/) & Docker Compose
* Git

### 1. Review Pre-configured Services (`docker-compose.yml`)
The `docker-compose.yml` file is fully populated and ready to run. It orchestrates the following services:
* **api:** Built from the `./backend/` directory.
* **postgres:** Uses `bitnami/postgresql:latest` with built-in health checks.
* **frontend:** Built from the `./frontend/` directory and handles port 80/443 mapping.
* **metrics:** A Grafana instance for monitoring. 

### 2. Configure Environment Variables
Copy the example `.env` file and set your local credentials to match the variables required by the compose file:
```bash
cp .env.example .env
```

### 3. Build and Launch
Once your `.env` is configured:
```bash
docker compose up --build -d
```

---

## ⚙️ Understanding Environment Variables (`.env`)

This boilerplate uses a single, global `.env` file at the root of the project to manage configuration across all containers. This keeps sensitive credentials out of version control and makes it easy to switch between local development and production.

**How Variables Propagate:**
1.  **Global `.env` File:** You define all variables here. (Always ensure `.env` is in your `.gitignore`!).
2.  **Backend & Database (Runtime):** Services like FastAPI and PostgreSQL load these variables directly at runtime via the `env_file:` directive in `docker-compose.yml`. They remain securely hidden inside the server environment.
3.  **Frontend (Build-time Injection):** React/Vite applications run in the user's browser, so they *cannot* access the `.env` file directly. To pass a variable to the frontend:
    *   The variable must be prefixed with `VITE_` (e.g., `VITE_API_BASE_URL`).
    *   It must be explicitly passed as a build argument (`args:`) in the `docker-compose.yml` under the frontend service.
    *   **⚠️ Security Warning:** Never put API keys or database passwords in a `VITE_` variable, as they will be publicly visible in the browser's source code!

**Essential Infrastructure Variables:**
To run the base infrastructure (Database, Gateway, and Metrics), your `.env` should at least include:

| Variable | Target | Description |
|---|---|---|
| `POSTGRES_USER` / `PASSWORD` / `DB` | Postgres, API | Credentials and database name for PostgreSQL. |
| `VITE_API_BASE_URL` | Frontend | Tells the React app where to send API requests (e.g., `/api`). |
| `SERVER_DOMAIN` | Metrics | Your server IP or domain name for secure Grafana routing. |
| `GRAFANA_ADMIN_PASSWORD` | Metrics | Secure password for the Grafana monitoring dashboard. |

*(Note: You can add any custom variables your specific study requires to the `.env` file and map them in the `docker-compose.yml`)*

---

## 🔒 Production Setup & SSL Certification (Certbot)

When you are ready to deploy your study securely over `https://` on a public server, follow this guide to generate SSL certificates. 

> **ℹ️ Note on IP Addresses vs. Domain Names:** If you are securing a raw IP address (a non-domain name) instead of a standard registered domain, be aware that these SSL certificates typically have a much shorter lifespan and require renewal **every 3 days**. The automated cron job in Step 5 is configured to run daily to safely handle these aggressive renewal cycles.

### 1. Initial Domain Setup
Ensure your domain name (e.g., `study.yourdomain.com`) or public IP address is accessible.

### 2. Generate SSL Certificates with Certbot
Run Certbot in standalone mode on your server host to request your initial certificate:

```bash
sudo certbot certonly --standalone -d study.yourdomain.com --preferred-challenges http
```

### 3. Mount Certificates in Docker Compose
Your `docker-compose.yml` is already set up to mount these certificates in the `frontend` service:

```yaml
  frontend:
    # ...
    volumes:
      - ./frontend/nginx.conf:/etc/nginx/conf.d/default.conf:z
      - /etc/letsencrypt:/etc/letsencrypt:ro
      - /var/www/certbot:/var/www/certbot:ro
```

### 4. Configure HTTPS in Nginx
Update your `./frontend/nginx.conf` file to enforce SSL:

```nginx
server {
    listen 80;
    server_name study.yourdomain.com;

    # Redirect all HTTP traffic to HTTPS
    location / {
        return 301 https://$host$request_uri;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }
}

server {
    listen 443 ssl;
    server_name study.yourdomain.com;

    ssl_certificate /etc/letsencrypt/live/[study.yourdomain.com/fullchain.pem](https://study.yourdomain.com/fullchain.pem);
    ssl_certificate_key /etc/letsencrypt/live/[study.yourdomain.com/privkey.pem](https://study.yourdomain.com/privkey.pem);

    location / {
        root /usr/share/nginx/html;
        index index.html;
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://api:8000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

### 5. Automated SSL Certificate Renewal
Set up a daily cron job on your server to automatically renew certificates. Because it runs daily, it will successfully catch standard 90-day domain certificates as well as strict 3-day non-domain (IP) certificates.

Since Certbot requires port 80 for standalone renewals, this cron job will briefly pause the Nginx container (causing a few seconds of downtime at 3 AM) to complete the check:

```bash
sudo crontab -e
# Add this line to attempt renewal daily at 3 AM. 
# It temporarily stops the frontend to free up port 80, renews, and restarts it.
0 3 * * * certbot renew --quiet --pre-hook "docker compose -f /path/to/online-study-template/docker-compose.yml stop frontend" --post-hook "docker compose -f /path/to/online-study-template/docker-compose.yml start frontend"
```

---

## 🛟 Data Management & Troubleshooting

When actively running a study, keep these critical operational details in mind:

*   **🛡️ Secure Database Access (SSH Tunneling):** For security reasons, the PostgreSQL database is bound strictly to `127.0.0.1` (localhost). This means you **cannot** connect to it directly from the outside world using tools like DBeaver or pgAdmin. To access the database remotely, you must open an SSH tunnel to your server:
    ```bash
    ssh -L 5432:127.0.0.1:5432 username@your-server-ip
    ```
    *Once the tunnel is active, you can connect your local database client directly to `localhost:5432`.*
*   **⚠️ Danger - Deleting Volumes:** Never run `docker compose down -v`. The `-v` flag deletes all associated Docker volumes, including `postgres_data`. This will permanently erase all collected participant data and interaction logs. To safely stop the stack without losing data, just run `docker compose down`.
*   **💾 Exporting Data:** Once your study concludes, you can safely extract your database contents for analysis (e.g., in Python or SPSS) by dumping the Postgres container data:
    ```bash
    docker exec -t <postgres_container_name> pg_dump -U <postgres_user> -d <postgres_db> -F c > study_backup.dump
    ```
*   **🐧 SELinux (Fedora/RHEL Users):** If you deploy this on a system with enforcing SELinux (like Fedora), standard volume mounts will be blocked. Ensure the Nginx configuration mount in `docker-compose.yml` uses the `:z` flag (`./frontend/nginx.conf:/etc/nginx/conf.d/default.conf:z`) to grant the container proper read permissions.

---

## 📄 License
MIT
