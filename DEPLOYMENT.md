# Plane Deployment Documentation

This document provides a detailed overview of the Plane application deployment on Google Cloud Platform (GCP).

## 🚀 Infrastructure Details

- **Cloud Provider:** Google Cloud Platform (GCP)
- **VM Instance:** `plane-vm`
- **Region/Zone:** `us-central1-a`
- **External IP:** `34.10.3.146`
- **Deployment Method:** Docker Compose

## 🌐 Domain & HTTPS

The application is secured with HTTPS using automatic SSL certificate management via Caddy.

- **Primary URL:** [https://knbl-plane.giize.com](https://knbl-plane.giize.com)
- **SSL Provider:** Let's Encrypt (Automated via Caddy)
- **DNS Provider:** Dynu DNS

### Proxy Configuration

Caddy acts as a reverse proxy, handling SSL termination and routing traffic to the internal Docker services.

## 📦 Services Architecture

The following services are running as part of the Docker Compose stack:

| Service       | Description                            |
| :------------ | :------------------------------------- |
| `web`         | Main Plane frontend application        |
| `api`         | Core Django API backend                |
| `admin`       | Plane God-Mode instance administration |
| `space`       | Plane Spaces functionality             |
| `live`        | Real-time collaboration server         |
| `proxy`       | Caddy reverse proxy and SSL handler    |
| `plane-db`    | PostgreSQL 15 Database                 |
| `plane-redis` | Redis Cache (Valkey)                   |
| `plane-mq`    | RabbitMQ Message Broker                |
| `bgworker`    | Celery background worker for tasks     |
| `beatworker`  | Celery beat for scheduled tasks        |

## 💾 Storage Integration (GCS)

Local Minio has been replaced with **Google Cloud Storage (GCS)** for reliable, globally accessible file storage.

- **Bucket Name:** `plane-uploads-hazel-flag`
- **Region:** `us-central1`
- **Endpoint:** `https://storage.googleapis.com`
- **Auth:** HMAC keys used for S3-compatible API access.

> [!NOTE]
> CORS policies are applied to the GCS bucket to allow direct browser uploads from `knbl-plane.giize.com`.

## 📧 Email Configuration

Email is configured via **Google Workspace SMTP**.

- **SMTP Host:** `smtp.gmail.com`
- **Port:** 587 (TLS)
- **Auth:** App-specific password for `ai@kanibal.co.il`.

### Important Setting: `SKIP_ENV_VAR=1`

We have enabled `SKIP_ENV_VAR=1` in the backend configuration. This ensures that any email or SMTP settings changed via the **God-Mode UI** take priority over the `.env` file, allowing for easier administrative updates without container restarts.

## ⚙️ Maintenance & Operations

### Restarting Services

To restart the entire stack:

```bash
sudo docker compose down && sudo docker compose up -d
```

To restart a specific service (e.g., the API):

```bash
sudo docker compose restart api
```

### Checking Logs

To monitor real-time logs:

```bash
sudo docker compose logs -f [service_name]
```

### Updating Configuration

1. Modify the `.env` file in the root directory.
2. If changing core backend settings, also check `apps/api/.env`.
3. Restart the updated service:

```bash
sudo docker compose up -d [service_name]
```

---

_Created on January 11, 2026_
