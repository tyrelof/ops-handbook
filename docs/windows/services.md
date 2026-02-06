# Windows Services

## Managing Services
```powershell
# Check status
Get-Service -Name "wuauserv"

# Restart a service
Restart-Service -Name "Spooler" -Force

# Change startup type
Set-Service -Name "Spooler" -StartupType Disabled
```

## Process Management
```powershell
# Find process by name
Get-Process -Name "notepad"

# Kill process
Stop-Process -Name "notepad" -Force

# Find top CPU users
Get-Process | Sort-Object CPU -Descending | Select-Object -First 5
```
