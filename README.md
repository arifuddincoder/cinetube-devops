# 🚀 Full Deployment Guide — CineTube

## Phase 1 — Project Structure

তিনটা আলাদা GitHub repo:

- `cinetube-backend` → Express + TypeScript + Prisma
- `cinetube-frontend` → Next.js + Bun
- `cinetube-devops` → docker-compose + nginx + CI/CD

---

## Phase 2 — Dockerfile লেখা

### Backend (`cinetube-backend/Dockerfile`) — 3 stage:

- `deps`: pnpm install
- `builder`: prisma generate + tsup build
- `runner`: production এ dist/ চালায়

```dockerfile
# ── Stage 1: deps ──
FROM node:20-alpine AS deps

RUN corepack enable && corepack prepare pnpm@10.33.0 --activate

WORKDIR /app

COPY package.json pnpm-lock.yaml ./
COPY prisma ./prisma/

RUN pnpm install --frozen-lockfile

# ── Stage 2: builder ──
FROM node:20-alpine AS builder

RUN corepack enable && corepack prepare pnpm@10.33.0 --activate

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

RUN pnpm generate
RUN pnpm build

# ── Stage 3: runner ──
FROM node:20-alpine AS runner

RUN corepack enable && corepack prepare pnpm@10.33.0 --activate

WORKDIR /app

ENV NODE_ENV=production

COPY package.json pnpm-lock.yaml ./
COPY prisma ./prisma/
RUN pnpm install --frozen-lockfile
RUN ./node_modules/.bin/prisma generate --schema=./prisma/schema

COPY --from=builder /app/dist ./dist

EXPOSE 5000

CMD ["node", "dist/server.js"]
```

### Frontend (`cinetube-frontend/Dockerfile`) — 3 stage:

- `deps`: bun install
- `builder`: next build
- `runner`: standalone output serve করে

⚠️ `next.config.ts` এ `output: 'standalone'` লাগবে।

```dockerfile
# ── Stage 1: deps ──
FROM oven/bun:1 AS deps

WORKDIR /app

COPY package.json bun.lockb* ./

RUN bun install --frozen-lockfile

# ── Stage 2: builder ──
FROM oven/bun:1 AS builder

WORKDIR /app

COPY --from=deps /app/node_modules ./node_modules
COPY . .

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV NEXT_PUBLIC_API_BASE_URL=https://cinetube.arifuddincoder.site/api/v1
ENV NEXT_PUBLIC_BACKEND_URL=https://cinetube.arifuddincoder.site

RUN bun --bun next build

# ── Stage 3: runner ──
FROM oven/bun:1 AS runner

WORKDIR /app

ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1

COPY --from=builder /app/public ./public
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static

EXPOSE 3000

ENV PORT=3000
ENV HOSTNAME="0.0.0.0"

CMD ["bun", "server.js"]
```

---

## Phase 3 — cinetube-devops এ তিনটা file

### `docker-compose.yml`

```yaml
services:

  cinetube-frontend:
    build:
      context: https://github.com/arifuddincoder/cinetube-frontend.git
      dockerfile: Dockerfile
    container_name: cinetube-frontend
    restart: unless-stopped
    networks:
      - cinetube-network

  cinetube-backend:
    build:
      context: https://github.com/arifuddincoder/cinetube-backend.git
      dockerfile: Dockerfile
    container_name: cinetube-backend
    restart: unless-stopped
    env_file:
      - .env
    networks:
      - cinetube-network

  nginx:
    image: nginx:alpine
    container_name: cinetube-nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - certbot-certs:/etc/letsencrypt
      - certbot-www:/var/www/certbot
    depends_on:
      - cinetube-frontend
      - cinetube-backend
    networks:
      - cinetube-network

  certbot:
    image: certbot/certbot
    container_name: cinetube-certbot
    volumes:
      - certbot-certs:/etc/letsencrypt
      - certbot-www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

volumes:
  certbot-certs:
  certbot-www:

networks:
  cinetube-network:
    driver: bridge
```

### `nginx/nginx.conf`

```nginx
events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    server {
        listen 80;
        server_name cinetube.arifuddincoder.site;

        location /.well-known/acme-challenge/ {
            root /var/www/certbot;
        }

        location / {
            return 301 https://$host$request_uri;
        }
    }

    server {
        listen 443 ssl;
        server_name cinetube.arifuddincoder.site;

        ssl_certificate     /etc/letsencrypt/live/cinetube.arifuddincoder.site/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/cinetube.arifuddincoder.site/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;

        client_max_body_size 50M;

        location / {
            proxy_pass http://cinetube-frontend:3000;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /api/ {
            proxy_pass http://cinetube-backend:5000/api/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

### `.github/workflows/deploy.yml`

```yaml
name: Deploy to VPS

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Deploy to Contabo VPS
        uses: appleboy/ssh-action@v1.0.3
        with:
          host: ${{ secrets.VPS_HOST }}
          username: ${{ secrets.VPS_USER }}
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            cd /opt/cinetube
            docker compose up --build -d
            docker image prune -f
```

---

## Phase 4 — VPS Setup

```bash
# 1. Docker install
curl -fsSL https://get.docker.com | sh

# 2. folder বানাও
mkdir -p /opt/cinetube
cd /opt/cinetube

# 3. cinetube-devops repo clone করো
git clone https://github.com/arifuddincoder/cinetube-devops.git .

# 4. .env file upload করো (PC থেকে)
scp .env root@VPS_IP:/opt/cinetube/.env
```

---

## Phase 5 — SSL Certificate

প্রথমে HTTP-only nginx দিয়ে চালু করো (443 block ছাড়া):

```bash
docker compose up -d
```

তারপর certificate নাও:

```bash
docker compose run --rm --entrypoint certbot certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email তোমার@email.com \
  --agree-tos \
  --no-eff-email \
  -d তোমার-domain.com
```

Certificate পাওয়ার পর nginx.conf এ 443 block add করো, তারপর:

```bash
docker compose restart nginx
```

---

## Phase 6 — GitHub Actions Secrets

`cinetube-devops` repo → Settings → Secrets → Actions এ তিনটা secret:

| Secret | Value |
|--------|-------|
| `VPS_HOST` | VPS IP |
| `VPS_USER` | `root` |
| `VPS_SSH_KEY` | Private SSH key |

VPS এ SSH key বানাতে:

```bash
ssh-keygen -t ed25519 -C "github-actions"
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_ed25519   # এটা GitHub Secret এ দাও
```

---

## Phase 7 — Auto-renew Cron Job

```bash
crontab -e
```

নিচে add করো:

```
0 0 * * * docker compose -f /opt/cinetube/docker-compose.yml run --rm --entrypoint certbot certbot renew && docker compose -f /opt/cinetube/docker-compose.yml restart nginx
```

---

## এরপর থেকে Workflow

```
Code লিখবে
    ↓
GitHub এ push করবে
    ↓
GitHub Actions trigger হবে
    ↓
VPS এ SSH → docker compose up --build -d
    ↓
নতুন version live!
```