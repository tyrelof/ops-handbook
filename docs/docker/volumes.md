# Docker Volumes & Persistence

## Mental Model
Containers are ephemeral. If you delete the container, the data is gone—UNLESS you mount a volume.

## Types of Mounts
1. **Bind Mounts**: Maps a path on Host -> Container.
   `-v /home/user/code:/app`
   *Good for development (live reload).*

2. **Named Volumes**: Managed by Docker in `/var/lib/docker/volumes`.
   `-v postgres-data:/var/lib/postgresql/data`
   *Good for databases and persistence.*

## Operations
```bash
# Create volume
docker volume create db-data

# Inspect volume (Find where it lives on host)
docker volume inspect db-data

# Prune unused volumes
docker volume prune
```

## Permission Issues
- If container user runs as `non-root`, ensure host directory privileges match.
- Use `chown -R 1000:1000 ./data` often fixes "Permission Denied".
