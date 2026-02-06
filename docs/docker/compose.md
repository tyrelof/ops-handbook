# Docker Compose

## Mental Model
Define a multi-container application in a single YAML file (`docker-compose.yml`).

## Basic Commands
```bash
# Start everything in background
docker compose up -d

# View logs of all services
docker compose logs -f

# Stop and remove containers + networks
docker compose down

# Rebuild images
docker compose up -d --build
```

## Sample docker-compose.yml
```yaml
version: '3.8'
services:
  web:
    build: .
    ports:
      - "8080:80"
    volumes:
      - .:/app
  redis:
    image: redis:alpine
```
