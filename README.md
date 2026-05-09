# 🚀 Full Deployment Guide — CineTube

## Phase 1 — Project Structure

তিনটা আলাদা GitHub repo:

- `cinetube-backend` → Express + TypeScript + Prisma
- `cinetube-frontend` → Next.js + Bun
- `cinetube-devops` → docker-compose + nginx + CI/CD

---

## Phase 2 — Dockerfile লেখা

### Backend (`cinetube-backend/Dockerfile`) — 3 stage:

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

⚠️ `next.config.ts` এ `output: 'standalone'` থাকতে হবে।

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

প্রথমে HTTP-only দিয়ে চালু করতে হবে (SSL certificate নেওয়ার আগে):

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
            proxy_pass http://cinetube-frontend:3000;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }

        location /api/ {
            proxy_pass http://cinetube-backend:5000/api/;
            proxy_http_version 1.1;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
    }
}
```

SSL certificate নেওয়ার পর এই final config দাও:

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

### `.github/workflows/deploy.yml` (তিনটা repo তেই একই)

`cinetube-devops`, `cinetube-backend`, `cinetube-frontend` — তিনটাতেই এই file দাও:

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
            docker compose restart nginx
            docker image prune -f
```

---

## Phase 4 — VPS Setup

```bash
# 1. Docker install করো
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

## Phase 5 — .env file

`cinetube-devops` folder এ `.env` বানাও (PC তে), তারপর VPS এ upload করো।

```dotenv
# Backend
NODE_ENV=production
PORT=5000

DATABASE_URL=your_database_url_here

BETTER_AUTH_SECRET=your_better_auth_secret_here
BETTER_AUTH_URL=https://cinetube.arifuddincoder.site

ACCESS_TOKEN_SECRET=your_access_token_secret_here
REFRESH_TOKEN_SECRET=your_refresh_token_secret_here
ACCESS_TOKEN_EXPIRES_IN="1d"
REFRESH_TOKEN_EXPIRES_IN="7d"

CLOUDINARY_CLOUD_NAME=your_cloudinary_cloud_name_here
CLOUDINARY_API_KEY=your_cloudinary_api_key_here
CLOUDINARY_API_SECRET=your_cloudinary_api_secret_here

FRONTEND_URL=https://cinetube.arifuddincoder.site

EMAIL_SENDER_SMTP_USER=your_email_here
EMAIL_SENDER_SMTP_PASS=your_email_password_here
EMAIL_SENDER_SMTP_HOST=smtp.gmail.com
EMAIL_SENDER_SMTP_PORT=465
EMAIL_SENDER_SMTP_FROM=your_email_here

GOOGLE_CLIENT_ID=your_google_client_id_here
GOOGLE_CLIENT_SECRET=your_google_client_secret_here
GOOGLE_CALLBACK_URL=https://cinetube.arifuddincoder.site/api/auth/callback/google

SUPER_ADMIN_EMAIL=your_super_admin_email_here
SUPER_ADMIN_PASSWORD=your_super_admin_password_here

STRIPE_SECRET_KEY=your_stripe_secret_key_here
STRIPE_WEBHOOK_SECRET=your_stripe_webhook_secret_here
STRIPE_MONTHLY_PRICE_ID=your_stripe_monthly_price_id_here
STRIPE_YEARLY_PRICE_ID=your_stripe_yearly_price_id_here

# Frontend
NEXT_PUBLIC_API_BASE_URL=https://cinetube.arifuddincoder.site/api/v1
NEXT_PUBLIC_BACKEND_URL=https://cinetube.arifuddincoder.site
NEXT_PUBLIC_CLOUDINARY_CLOUD_NAME=your_cloudinary_cloud_name_here
NEXT_PUBLIC_CLOUDINARY_UPLOAD_PRESET=your_cloudinary_upload_preset_here
```

⚠️ `.env` কখনো GitHub এ push করবে না। `cinetube-devops/.gitignore` এ `.env` add করো।

---

## Phase 6 — SSL Certificate

### Step 1: HTTP-only দিয়ে চালু করো

```bash
docker compose up -d
```

### Step 2: Certificate নাও

```bash
docker compose run --rm --entrypoint certbot certbot certonly \
  --webroot \
  --webroot-path=/var/www/certbot \
  --email তোমার@email.com \
  --agree-tos \
  --no-eff-email \
  -d cinetube.arifuddincoder.site
```

### Step 3: nginx.conf এ SSL config দাও

উপরের final nginx.conf টা VPS এ দাও:

```bash
cat > /opt/cinetube/nginx/nginx.conf << 'EOF'
# উপরের final nginx.conf content এখানে
EOF
```

### Step 4: nginx restart করো

```bash
docker compose restart nginx
```

---

## Phase 7 — GitHub Actions Secrets

তিনটা repo তেই (`cinetube-devops`, `cinetube-backend`, `cinetube-frontend`) এই secrets add করো:

**Settings → Secrets and variables → Actions → New repository secret**

| Secret | Value |
|--------|-------|
| `VPS_HOST` | VPS IP (যেমন: 81.17.101.34) |
| `VPS_USER` | `root` |
| `VPS_SSH_KEY` | Private SSH key |

### SSH Key বানাতে (VPS এ):

```bash
ssh-keygen -t ed25519 -C "github-actions"
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
cat ~/.ssh/id_ed25519   # এই output টা VPS_SSH_KEY তে দাও
```

---

## Phase 8 — Auto-renew Cron Job

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
Code লিখবে (backend বা frontend)
        ↓
git push origin main
        ↓
GitHub Actions trigger হবে
        ↓
VPS এ SSH → docker compose up --build -d → nginx restart
        ↓
নতুন version live! (~3-5 মিনিট)
```

---

## Useful Commands

```bash
# সব container দেখো
docker compose ps

# logs দেখো
docker compose logs -f

# একটা service এর log দেখো
docker compose logs -f cinetube-frontend

# manually deploy করো
docker compose up --build -d

# nginx restart করো
docker compose restart nginx

# সব বন্ধ করো
docker compose down
```