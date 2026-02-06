# Windows Event Logs

## Analyzing Events via CLI
Avoid the slow GUI.

### Get Recent Errors
```powershell
# Get last 10 Error events from System log
Get-WinEvent -LogName System -MaxEvents 10 | Where-Object LevelDisplayName -eq "Error"
```

### Filter by ID
```powershell
# Find specific Event ID (e.g., 4624 = Logon)
Get-WinEvent -FilterHashtable @{LogName='Security'; Id=4624} -MaxEvents 5
```

### Export Logs
```powershell
Get-WinEvent -LogName Application -MaxEvents 100 | Export-Csv C:\temp\logs.csv
```
