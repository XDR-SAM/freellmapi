# Dokploy Deployment Guide

This guide explains how to deploy **FreeLLMAPI** using **Dokploy** (an open-source, self-hostable PaaS). Dokploy supports three deployment pathways:
1. **Dockerfile (Recommended)** - Builds using the optimized multi-stage `Dockerfile`.
2. **Nixpacks** - Builds dynamically using Dokploy's default Nixpacks build system.
3. **Docker Compose** - Deploys multi-container setups using `docker-compose.yml`.

---

## 🔐 Required Environment Variables

Before deploying, make sure you configure the following environment variables in Dokploy:

*   `ENCRYPTION_KEY` (Required): A 64-character hexadecimal key used to encrypt your provider API keys.
    *   *To generate one:* Run `node -e "console.log(require('crypto').randomBytes(32).toString('hex'))"` in your local terminal.
    *   **Keep this key stable.** If you change it or lose it, you won't be able to decrypt your saved provider keys.
*   `PORT` (Optional): The port the container listens on. Defaults to `3001`.
*   `HOST_BIND` (Optional): Docker host interface to publish on. (Only relevant for Docker Compose. Not needed for standalone applications where Dokploy manages internal proxy networks).

---

## 💾 Persistent Storage (SQLite)

FreeLLMAPI stores its data, keys, and history in a SQLite database inside the `/app/server/data` directory. You **must** attach a persistent volume to this path to avoid losing data during redeployments.

---

## 🚀 Option 1: Dockerfile Deployment (Recommended)

This is the most secure and resource-efficient method, as it runs as a non-root user (`node`) and has build dependencies isolated.

### Step 1: Create a new Application in Dokploy
1. Log in to your Dokploy dashboard.
2. Select your Project and Environment (or create them).
3. Click **Create Service** and select **Application**.
4. Give it a name (e.g., `freellmapi`).

### Step 2: Configure Git Repository
1. Select **Git** as the source type.
2. Select your provider (GitHub, GitLab, etc.) and repository.
3. Specify the branch (e.g., `main`).

### Step 3: Configure Build Settings
1. Set the **Build Type** to **Dockerfile**.
2. Keep the **Dockerfile Path** as `Dockerfile`.
3. Keep the **Context Path** as `./` or empty.

### Step 4: Configure Port and Domain
1. In the application settings, go to the **Domains** tab.
2. Add your desired domain name (e.g., `llm.yourdomain.com`).
3. Set the **Container Port** to `3001`. Dokploy's Traefik reverse proxy will automatically manage routing and generate SSL certificates.

### Step 5: Configure Environment Variables
1. Go to the **Environment** tab.
2. Add:
   * Key: `ENCRYPTION_KEY` | Value: `[your-64-character-hex-key]`

### Step 6: Configure Volumes
1. Go to the **Volumes** tab.
2. Add a new volume mapping:
   * **Host Path / Volume Name**: `freellmapi_data`
   * **Mount Path (inside container)**: `/app/server/data`
3. Save the volume configuration.

### Step 7: Deploy
1. Click **Deploy**. Dokploy will pull the code, build the multi-stage Docker image, mount the volume, and start the container.

---

## 📦 Option 2: Nixpacks Deployment

If you prefer using Nixpacks (Dokploy's default build engine), this project contains a custom `nixpacks.toml` configured to automatically compile native C++ packages like `better-sqlite3`.

### Step 1: Configure Service in Dokploy
1. Create a new **Application** service in Dokploy and connect your git repository.
2. Set **Build Type** to **Nixpacks**.

### Step 2: Configure Environment Variables & Volumes
1. Set the environment variable `ENCRYPTION_KEY` just like in Option 1.
2. Mount a persistent volume at `/app/server/data` just like in Option 1.

### Step 3: Deploy
1. Click **Deploy**. Nixpacks will read the root `nixpacks.toml`, install Node.js alongside `python3`, `gnumake`, and `gcc`, run `npm run build`, and then start the server.

---

## 🐳 Option 3: Docker Compose Deployment

If you want to manage the service as a Docker Compose stack:

### Step 1: Create a Compose Service
1. In Dokploy, click **Create Service** and select **Compose**.
2. Give it a name (e.g., `freellmapi-stack`).

### Step 2: Configure Source
1. Select Git or enter your repository credentials.
2. Point it to the directory containing `docker-compose.yml`.

### Step 3: Define Domain Routing
To map a domain via Traefik in a Docker Compose deployment, connect your service to the `dokploy-network` and configure the domain in Dokploy's UI, or write Traefik labels in `docker-compose.yml` as follows:

```yaml
services:
  freellmapi:
    image: ghcr.io/tashfeenahmed/freellmapi:latest
    build:
      context: .
      dockerfile: Dockerfile
    environment:
      NODE_ENV: production
      PORT: 3001
      ENCRYPTION_KEY: "${ENCRYPTION_KEY}"
    networks:
      - dokploy-network
    labels:
      - "traefik.enable=true"
      - "traefik.docker.network=dokploy-network"
      - "traefik.http.routers.freellmapi.rule=Host(`llm.yourdomain.com`)"
      - "traefik.http.routers.freellmapi.entrypoints=websecure"
      - "traefik.http.routers.freellmapi.tls=true"
      - "traefik.http.routers.freellmapi.tls.certresolver=letsencrypt"
      - "traefik.http.services.freellmapi.loadbalancer.server.port=3001"
    volumes:
      - freellmapi-data:/app/server/data
    restart: unless-stopped

networks:
  dokploy-network:
    external: true

volumes:
  freellmapi-data:
```

### Step 4: Deploy
1. Click **Deploy** to pull and launch the stack.
