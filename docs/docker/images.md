# Docker Images

## Mental Model
Images are immutable, read-only templates built from layers.

## Managing Images
```bash
# List local images
docker images

# Pull from registry
docker pull postgres:15-alpine

# Remove image
docker rmi <image_id>
```

## Building
```bash
# Build from current directory (Dockerfile)
docker build -t my-app:v1 .

# Build with build args
docker build --build-arg VERSION=1.0 -t my-app .
```

## Clean Up (Pruning)
Disk full? Run this.
```bash
# Remove stopped containers, unused networks, and dangling images
docker system prune

# Remove EVERYTHING not currently running (including named volumes!)
docker system prune -a --volumes
```
