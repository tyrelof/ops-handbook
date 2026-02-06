# Docker Container Lifecycle

## Mental Model
A container is a process with an isolated filesystem (image) and namespace.
States: Created → Running → Paused/Stopped → Exited → Deleted.

## Basic Operations
```bash
# Start a container (Background)
docker run -d --name web -p 80:80 nginx

# Start Interactive
docker run -it ubuntu /bin/bash

# Stop / Kill
docker stop web  # SIGTERM (Graceful)
docker kill web  # SIGKILL (Force)

# Remove
docker rm web       # Must be stopped
docker rm -f web    # Force remove running
```

## Executing Commands
Running a sidebar process inside an existing container (e.g., debugging).
```bash
docker exec -it web /bin/bash
```

## Inspecting
```bash
# View metadata (IP, mounts, env vars)
docker inspect web
```
