# Cookbook: Production Full Stack (Node + Redis + Postgres)

> [!NOTE]
> This guide builds a production-ready container stack using `docker compose`.
> Includes: Multi-stage builds, Healthchecks, Restart policies, and Network isolation.

## 1. Directory Structure

```text
my-app/
├── server/
│   ├── Dockerfile
│   ├── package.json
│   └── index.js
├── docker-compose.yml
└── .env
```

## 2. The Application (Server)

### Step 2.1: Multi-Stage Dockerfile
Create `server/Dockerfile`. Optimized for size and security.

```dockerfile
# --- Stage 1: Builder ---
FROM node:18-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build # (If using TS/Build step)

# --- Stage 2: Production ---
FROM node:18-alpine
WORKDIR /app
ENV NODE_ENV=production

# Create non-root user
USER node

# Copy only necessary files
COPY --from=builder /app/package*.json ./
COPY --from=builder /app/node_modules ./node_modules
COPY --from=builder /app/dist ./dist  
# (Or just COPY . . if no build step)

EXPOSE 3000
CMD ["node", "dist/index.js"]
```

## 3. Environment Variables (.env)
Create `.env` in root. **Add this to .gitignore!**

```ini
POSTGRES_USER=myuser
POSTGRES_PASSWORD=securepassword123
POSTGRES_DB=mydb
REDIS_PASSWORD=secure_redis_pass
NODE_ENV=production
```

## 4. Docker Compose
Create `docker-compose.yml`.

```yaml
version: '3.8'

services:
  # --- Backend API ---
  api:
    build: ./server
    container_name: my-api
    restart: always
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@db:5432/${POSTGRES_DB}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - app-net

  # --- Postgres Database ---
  db:
    image: postgres:15-alpine
    container_name: my-db
    restart: always
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - app-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s
      timeout: 5s
      retries: 5

  # --- Redis Cache ---
  cache:
    image: redis:alpine
    container_name: my-cache
    restart: always
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    networks:
      - app-net
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

# --- Networks & Volumes ---
networks:
  app-net:
    driver: bridge

volumes:
  pg-data:
  redis-data:
```

## 5. Deployment

```bash
# Start in background
docker compose up -d

# View logs
docker compose logs -f api

# Scale API (Load Balancing)
# Requires removing 'container_name' from api service first
docker compose up -d --scale api=3
```
