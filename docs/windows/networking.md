# Windows Networking

## Connectivity Checks
```powershell
# Ping & Port Check (The modern Telnet)
Test-NetConnection -ComputerName google.com -Port 443
# Alias: tnc
```

## Socket Statistics
```powershell
# List listening TCP ports
Get-NetTCPConnection -State Listen

# Find process owning a port
Get-NetTCPConnection -LocalPort 3389 | Select-Object OwningProcess, State
```

## DNS
```powershell
# Resolve name
Resolve-DnsName google.com
```

## Firewall
```powershell
# Check profile status
Get-NetFirewallProfile

# Allow a port (Advanced)
New-NetFirewallRule -DisplayName "Allow Web" -Direction Inbound -LocalPort 80 -Protocol TCP -Action Allow
```
