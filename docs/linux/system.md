# Linux System Management

## CPU & Process Management
**Mental Model**: Processes have a PID, Parent PID (PPID), and State (Running, Sleeping, Zombie).

```bash
# Real-time view
htop  # (Better than top)

# Tree view of processes
pstree -p

# Kill a process
kill <PID>       # SIGTERM (Polite)
kill -9 <PID>    # SIGKILL (Force - No cleanup)
pkill -f <name>  # Kill by name pattern
```

## Service Management (systemd)
```bash
# Check status
systemctl status <service>

# Logs for a service
journalctl -u <service> -n 100 -f

# List all running services
systemctl list-units --type=service --state=running
```

## Disk Usage
```bash
# Free space (Human readable)
df -h

# Directory size (Depth 1)
du -h --max-depth=1 . | sort -hr
```
