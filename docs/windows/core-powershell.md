# PowerShell Core Concepts

## Mental Model
PowerShell passes **objects**, not text. You filter properties, not strings.

## Verbs
- `Get`: Retrieve resource.
- `Set`: Modify resource.
- `New`: Create resource.
- `Remove`: Delete resource.

## Piping & Filtering
```powershell
# Get all processes using > 100 handles
Get-Process | Where-Object {$_.Handles -gt 100}

# Select specific properties
Get-Service | Select-Object Name, Status, StartType

# Sort
Get-ChildItem | Sort-Object Length -Descending
```

## Comparisons
- `-eq`: Equal
- `-ne`: Not Equal
- `-gt`: Greater Than
- `-lt`: Less Than
- `-like`: Wildcard match (`"Hello" -like "*ell*"`)
- `-match`: Regex match
