# MEAN Stack CRUD Application — DevOps Setup Guide

A full-stack MEAN (MongoDB, Express, Angular, Node.js) application for managing tutorials, fully containerized and deployable via Docker Compose with a CI/CD pipeline.

---

## Project Structure

```
.
├── backend/                  # Node.js + Express REST API
│   ├── app/
│   │   ├── config/db.config.js
│   │   ├── models/
│   │   └── routes/
│   ├── server.js
│   ├── package.json
│   └── Dockerfile
├── frontend/                 # Angular 15 SPA
│   ├── src/
│   ├── nginx.conf            # Nginx config inside frontend container
│   ├── package.json
│   └── Dockerfile
├── nginx/
│   └── nginx.conf            # Main reverse proxy config
├── .github/
│   └── workflows/
│       └── ci-cd.yml         # GitHub Actions pipeline
├── docker-compose.yml
└── DEPLOYMENT.md
```

---

## Architecture

```
Internet → Port 80 → Nginx (reverse proxy)
                        ├── /api/*  → backend:8080 (Node.js/Express)
                        └── /*      → frontend:80  (Angular served by Nginx)
                                           MongoDB ← backend
```

---

## 1. Local Development

### Prerequisites
- Node.js 18+
- MongoDB running locally

### Backend
```bash
cd backend
npm install
node server.js
# Runs on http://localhost:8080
```

### Frontend
```bash
cd frontend
npm install
ng serve --port 8081
# Runs on http://localhost:8081
```

---

## 2. Docker Build & Push

### Prerequisites
- Docker installed
- Docker Hub account

```bash
# Log in to Docker Hub
docker login

# Build and push backend
docker build -t <your-dockerhub-username>/mean-backend:latest ./backend
docker push <your-dockerhub-username>/mean-backend:latest

# Build and push frontend
docker build -t <your-dockerhub-username>/mean-frontend:latest ./frontend
docker push <your-dockerhub-username>/mean-frontend:latest
```

---

## 3. VM Setup (Ubuntu)

### Install Docker & Docker Compose
```bash
# Update packages
sudo apt update && sudo apt upgrade -y

# Install Docker
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker compose version
```

### Clone the repository & deploy
```bash
git clone https://github.com/<your-username>/<your-repo>.git ~/mean-app
cd ~/mean-app

# Set your Docker Hub username
export DOCKER_USERNAME=<your-dockerhub-username>

# Pull images and start all services
docker compose pull
docker compose up -d

# View running containers
docker compose ps

# View logs
docker compose logs -f
```

The application will be accessible at **http://<your-vm-ip>**

---

## 4. GitHub Actions CI/CD Setup

### Required GitHub Secrets

Go to your GitHub repo → **Settings → Secrets and variables → Actions** and add:

| Secret Name     | Description                              |
|-----------------|------------------------------------------|
| `DOCKER_USERNAME` | Your Docker Hub username               |
| `DOCKER_PASSWORD` | Your Docker Hub password or access token |
| `VM_HOST`       | Public IP address of your Ubuntu VM     |
| `VM_USER`       | SSH username (e.g., `ubuntu`)           |
| `VM_SSH_KEY`    | Private SSH key for VM access           |

### Setting up SSH access for the VM

On your local machine:
```bash
# Generate a key pair (skip if you already have one)
ssh-keygen -t ed25519 -C "github-actions-deploy"

# Copy the public key to your VM
ssh-copy-id -i ~/.ssh/id_ed25519.pub <vm-user>@<vm-ip>

# Copy the private key content → paste into VM_SSH_KEY secret
cat ~/.ssh/id_ed25519
```

### How the pipeline works

On every push to `main`:
1. **Build** — Docker images for both frontend and backend are built
2. **Push** — Images are pushed to Docker Hub (tagged `latest` and with the commit SHA)
3. **Deploy** — The pipeline SSHs into the VM, pulls the latest images, and restarts containers

---

## 5. Nginx Reverse Proxy

The `nginx/nginx.conf` file configures Nginx to:
- Listen on **port 80**
- Route `/api/*` requests → backend container (port 8080)
- Route all other requests → frontend container (port 80)

No domain or HTTPS configuration is required for this setup.

---

## 6. Environment Variables

| Variable       | Service   | Default                          | Description          |
|----------------|-----------|----------------------------------|----------------------|
| `MONGODB_URL`  | backend   | `mongodb://localhost:27017/dd_db`| MongoDB connection   |
| `PORT`         | backend   | `8080`                           | Backend listen port  |
| `DOCKER_USERNAME` | compose | `yourdockerhubuser`            | Docker Hub username  |

---

## 7. Useful Commands

```bash
# Start all services
docker compose up -d

# Stop all services
docker compose down

# View logs for a specific service
docker compose logs -f backend

# Restart a single service
docker compose restart backend

# Pull latest images and redeploy
docker compose pull && docker compose up -d --remove-orphans

# Remove all containers, volumes, and images
docker compose down -v --rmi all
```

---

## Application Features

- Create, read, update, and delete (CRUD) tutorials
- Each tutorial has: title, description, published status
- Search tutorials by title
- REST API at `/api/tutorials`
