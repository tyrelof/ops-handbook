# Docker Debugging

## Analyzing Crashes
1. **Logs**: `docker logs my-app`
2. **Exit Code**: `docker inspect my-app --format='{{.State.ExitCode}}'`
   - `0`: Normal exit
   - `137`: OOM Killed (Out of Memory)
   - `1`: App error

## Stats (Resource Usage)
See live CPU/Memory usage.
```bash
docker stats
```

## "It works on my machine"
If it fails in the container:
1. `docker run -it --entrypoint /bin/sh my-image`
2. Run the command manually inside to see the *real* error.

## Dangling Resources
If weird network errors occur:
```bash
docker network prune
```
