# Linux Logs

## The Source of Truth
- `/var/log/syslog` or `/var/log/messages`: General system activity.
- `/var/log/auth.log` or `/var/log/secure`: Auth attempts (SSH, sudo).
- `dmesg`: Kernel ring buffer (hardware, drivers, OOM killer).

## Journalctl (Systemd)
Structured logging for services.

```bash
# Logs for a unit, follow live
journalctl -u nginx -f

# Logs since boot
journalctl -b

# Reverse order (newest first)
journalctl -r
```

## Log Rotation
Logs are rotated (compressed/archived) to prevent disk fill. Configured in `/etc/logrotate.conf`.
