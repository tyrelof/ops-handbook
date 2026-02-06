# Linux Troubleshooting Scenarios

## 1. "Server is Slow"
**Triage steps:**
1. `top` or `htop`: Check Load Average. If > CPU cores, systems is overloaded.
2. `iostat -x 1`: Check `%iowait`. High iowait = Disk bottleneck.
3. `free -m`: Check Swap usage. High swap activity = Thrashing (RAM mismatch).

## 2. "Disk Full"
**Triage steps:**
1. `df -h`: Find the full partition.
2. `du -h --max-depth=1 / | sort -hr`: Drill down to huge directories.
3. **Trap**: Deleted files held open by processes don't free space.
   - Check: `lsof | grep deleted`
   - Fix: Restart the process holding the file.

## 3. "OOM Killer"
Process suddenly disappears?
- Check: `dmesg | grep -i "killed process"`
- Cause: System ran out of RAM, kernel sacrificed a child.

## 4. "Connection Refused" vs "Timed Out"
- **Refused**: Packet reached server, but no process listening on port (or firewall rejected actively).
- **Timed Out**: Packet dropped (firewall DROP) or routing blackhole.
