# Docker Networking

## Mental Model
- **Bridge (default)**: NATs traffic out. Containers on the same bridge can talk by name.
- **Host**: Container shares host's network stack (no isolation).
- **None**: No network.

## Common Commands
```bash
# List networks
docker network ls

# Create a custom bridge (Better DNS resolution)
docker network create my-net

# Connect container to network
docker network connect my-net web
```

## Debugging Connectivity
From inside a container:
```bash
# Install tools (if missing)
apt-get update && apt-get install -y iputils-ping dnsutils

# Resolve another container by name
ping app-backend
```

## Port Mapping
Has format `HOST:CONTAINER`.
`-p 8080:80` maps Host 8080 -> Container 80.
