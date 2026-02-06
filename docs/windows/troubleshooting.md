# Windows Troubleshooting Scenarios

## 1. "RDP Refused"
**Triage steps:**
1. **Service Running?**: `Get-Service TermService`
2. **Port Open?**: `Test-NetConnection localhost -Port 3389`
3. **Firewall?**: `Get-NetFirewallRule -DisplayName "Remote Desktop*"`

## 2. "Disk Full"
**Triage steps:**
1. **Find Volumes**: `Get-Volume`
2. **Find Large Files**:
   ```powershell
   Get-ChildItem C:\ -Recurse -ErrorAction SilentlyContinue | Sort-Object Length -Descending | Select-Object -First 10
   ```
   *(Note: This is slow on C:\ root, prefer targeting subfolders)*

## 3. "High CPU"
**Triage steps:**
1. `Get-Process | Sort-Object CPU -Descending | Select-Object -First 5`
2. If `System Idle Process` is high, that means CPU is free.

## 4. "Locked File"
Windows does not have `lsof` natively.
- Use **Resource Monitor** (resmon) -> CPU Tab -> Search Handles.
- Or Sysinternals **Handle.exe**.
