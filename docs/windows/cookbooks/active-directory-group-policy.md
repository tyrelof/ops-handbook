# Cookbook: Windows Server Active Directory & Group Policy

> [!NOTE]
> Enterprise-grade Active Directory administration and Group Policy management.
> Covers: Domain setup, user/computer management, GPO deployment, security policies, and troubleshooting.

## 1. Active Directory Fundamentals

### AD Structure
```
Forest (trust boundary)
└── Domain (example.com)
    ├── Organizational Units (OUs)
    │   ├── Users
    │   ├── Computers
    │   ├── Groups
    │   └── Service Accounts
    ├── Users
    ├── Computers
    └── Groups
```

### Check AD Forest & Domain Info
```powershell
# Get forest information
Get-ADForest

# Get domain information
Get-ADDomain

# Get domain controllers
Get-ADDomainController -Filter *

# Get current domain
[System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()

# List all domains in forest
Get-ADForest | Select-Object -ExpandProperty Domains
```

## 2. User & Group Management

### User Creation
```powershell
# Create new user with full details
$params = @{
    Name = "John Smith"
    SamAccountName = "jsmith"
    UserPrincipalName = "jsmith@example.com"
    GivenName = "John"
    Surname = "Smith"
    DisplayName = "John Smith"
    Department = "Engineering"
    Title = "Senior Engineer"
    Company = "Example Corp"
    OfficePhone = "+1-555-0123"
    MobilePhone = "+1-555-0456"
    EmailAddress = "jsmith@example.com"
    Path = "OU=Users,OU=Engineering,DC=example,DC=com"
    AccountPassword = (ConvertTo-SecureString -AsPlainText "TempPassword123!" -Force)
    Enabled = $true
    ChangePasswordAtLogon = $true
}
New-ADUser @params

# Verify user created
Get-ADUser -Identity jsmith -Properties *
```

### Bulk User Import
```powershell
# Import from CSV
$users = Import-Csv "C:\users.csv"

foreach ($user in $users) {
    $params = @{
        Name = $user.Name
        SamAccountName = $user.SamAccountName
        UserPrincipalName = "$($user.SamAccountName)@example.com"
        GivenName = $user.FirstName
        Surname = $user.LastName
        Path = $user.OU
        AccountPassword = (ConvertTo-SecureString -AsPlainText "TempPass123!" -Force)
        Enabled = $true
        ChangePasswordAtLogon = $true
    }
    New-ADUser @params
}
```

### User Management
```powershell
# Reset password (force change on next login)
Set-ADAccountPassword -Identity jsmith -Reset -NewPassword (ConvertTo-SecureString -AsPlainText "NewPassword123!" -Force)
Set-ADUser -Identity jsmith -ChangePasswordAtLogon $true

# Unlock account (after failed login attempts)
Unlock-ADAccount -Identity jsmith

# Enable/disable user
Enable-ADAccount -Identity jsmith
Disable-ADAccount -Identity jsmith

# Set password to never expire (service accounts)
Set-ADUser -Identity serviceaccount -PasswordNeverExpires $true

# Get user details
Get-ADUser -Identity jsmith -Properties *

# Find locked out users
Search-ADAccount -LockedOut | Select-Object Name, SamAccountName

# Find disabled users
Search-ADAccount -AccountDisabled | Select-Object Name, SamAccountName

# Find users with expired passwords
Search-ADAccount -PasswordExpired | Select-Object Name, EmailAddress
```

### Group Management
```powershell
# Create security group
New-ADGroup -Name "Engineering-Team" -SamAccountName "eng-team" `
    -GroupCategory Security -GroupScope Global `
    -Path "OU=Groups,OU=Engineering,DC=example,DC=com"

# Create distribution group (email)
New-ADGroup -Name "Finance-Distribution" -SamAccountName "finance-dist" `
    -GroupCategory Distribution -GroupScope Universal

# Add user to group
Add-ADGroupMember -Identity "Engineering-Team" -Members jsmith

# Add multiple users
$users = Get-ADUser -Filter "Department -eq 'Engineering'"
Add-ADGroupMember -Identity "Engineering-Team" -Members $users

# Remove from group
Remove-ADGroupMember -Identity "Engineering-Team" -Members jsmith -Confirm:$false

# Get group members
Get-ADGroupMember -Identity "Engineering-Team" -Recursive

# Get groups for user
Get-ADPrincipalGroupMembership -Identity jsmith | Select-Object Name

# List all groups in OU
Get-ADGroup -SearchBase "OU=Groups,DC=example,DC=com" -Filter *
```

## 3. Organizational Units (OUs)

### OU Design Best Practice
```
example.com
├── Workstations
│   ├── Engineering
│   ├── Sales
│   ├── Support
│   └── Executive
├── Servers
│   ├── Production
│   ├── Staging
│   └── Development
├── Users
│   ├── Engineering
│   ├── Sales
│   └── Admin
├── Service Accounts
└── Groups
```

### Create OUs
```powershell
# Create OU structure
New-ADOrganizationalUnit -Name "Workstations" -Path "DC=example,DC=com"
New-ADOrganizationalUnit -Name "Engineering" -Path "OU=Workstations,DC=example,DC=com"
New-ADOrganizationalUnit -Name "Servers" -Path "DC=example,DC=com"
New-ADOrganizationalUnit -Name "Production" -Path "OU=Servers,DC=example,DC=com"

# Get all OUs
Get-ADOrganizationalUnit -Filter * | Select-Object Name, Path

# Move computer/user to OU
Move-ADObject -Identity (Get-ADComputer -Identity "WORKPC01").ObjectGUID `
    -TargetPath "OU=Engineering,OU=Workstations,DC=example,DC=com"
```

## 4. Group Policy (GPO) Management

### Create & Link GPO
```powershell
# Create new GPO
New-GPO -Name "Engineering-Workstations-Policy"

# Link GPO to OU
New-GPLink -Name "Engineering-Workstations-Policy" `
    -Target "OU=Engineering,OU=Workstations,DC=example,DC=com" `
    -LinkEnabled Yes

# Create and link in one step
New-GPO -Name "Security-Baseline" | New-GPLink -Target "OU=Workstations,DC=example,DC=com"

# List all GPOs
Get-GPO -All

# Get GPO details
Get-GPO -Name "Engineering-Workstations-Policy"

# Get GPOs linked to OU
Get-GPInheritance -Target "OU=Engineering,OU=Workstations,DC=example,DC=com"
```

### Common GPO Policies

#### Password Policy
```powershell
# Set password policy via GPO script
Set-GPRegistryValue -Name "Default Domain Policy" -Key "HKLM\System\CurrentControlSet\Services\Netlogon\Parameters" `
    -ValueName "MaximumPasswordAge" -Value 90 -Type DWord

# Or via command line on DC
secedit /export /cfg C:\temp\sec_export.inf
# Edit C:\temp\sec_export.inf
secedit /import /cfg C:\temp\sec_export.inf /db C:\temp\sec_import.sdb /overwrite
secedit /configure /db C:\temp\sec_import.sdb
```

#### Account Lockout Policy
```powershell
# Set lockout policy (DC only)
Set-ADDefaultDomainPasswordPolicy -Identity example.com -LockoutDuration 00:30:00 `
    -LockoutThreshold 5 -LockoutObservationWindow 00:30:00

# Get current policy
Get-ADDefaultDomainPasswordPolicy -Identity example.com
```

#### Firewall via GPO
```powershell
# Enable Windows Firewall via GPO
Set-GPRegistryValue -Name "Firewall-Policy" `
    -Key "HKLM\Software\Policies\Microsoft\WindowsFirewall\StandardProfile" `
    -ValueName "EnableFirewall" -Value 1 -Type DWord

# Set specific port rules
netsh advfirewall firewall add rule name="Allow RDP" dir=in action=allow protocol=tcp localport=3389
```

#### Disable USB Ports
```powershell
# Disable USB storage via GPO registry
Set-GPRegistryValue -Name "Security-Policy" `
    -Key "HKLM\System\CurrentControlSet\Services\USBSTOR" `
    -ValueName "Start" -Value 4 -Type DWord  # 4 = Disabled
```

## 5. Delegation

### Delegate Admin Rights
```powershell
# Grant user permission to reset passwords in OU
$OU = Get-ADOrganizationalUnit -Identity "OU=Engineering,OU=Workstations,DC=example,DC=com"
$user = Get-ADUser -Identity jsmith

# Using dsacls (command line)
dsacls "OU=Engineering,OU=Workstations,DC=example,DC=com" /G "EXAMPLE\jsmith:CA;Reset Password" /I:S

# Or in GUI: ADUC > right-click OU > Delegate Control

# Verify delegations
Get-Acl "AD:\OU=Engineering,OU=Workstations,DC=example,DC=com" | Select-Object Access
```

### Delegate Computer Management
```powershell
# Grant right to join computers to domain
# Use: Active Directory Users and Computers > Delegate Control wizard
# Or via dsacls:
dsacls "OU=Engineering,OU=Workstations,DC=example,DC=com" `
    /G "EXAMPLE\CompAdmins:CA;Create All Child Objects;computer"
```

## 6. Computer Management

### Join Computer to Domain
```powershell
# Join domain (requires restart)
Add-Computer -DomainName example.com -Credential (Get-Credential) -Restart

# Or offline domain join (for servers without network access)
djoin.exe /provision /domain example.com /machine SERVERNAME /savefile C:\temp\file.txt
# Then restore on target: djoin.exe /requestodj /loadfile C:\temp\file.txt
```

### Manage Computer Objects
```powershell
# Get all computers in OU
Get-ADComputer -SearchBase "OU=Engineering,OU=Workstations,DC=example,DC=com" -Filter *

# Get computers by OS
Get-ADComputer -Filter "OperatingSystem -like '*Windows 10*'" | Select-Object Name, OperatingSystem

# Move computer to OU
Get-ADComputer -Identity "WORKPC01" | Move-ADObject `
    -TargetPath "OU=Engineering,OU=Workstations,DC=example,DC=com"

# Disable computer
Disable-ADAccount -Identity "WORKPC01"

# Remove computer from domain
Remove-ADComputer -Identity "WORKPC01" -Confirm:$false

# Reset computer account password (fixes replication issues)
Reset-ComputerMachinePassword -Server WORKPC01
```

## 7. Replication & Health

### Monitor AD Replication
```powershell
# Check replication status between DCs
Get-ADReplicationPartnerMetadata -Target "DC=example,DC=com" -Partition "DC=example,DC=com"

# Force replication
Sync-ADObject -Identity (Get-ADDomainController -Filter * | Select-Object -First 1)

# Check for replication errors
repadmin /showrepl

# Detailed replication status
repadmin /replsum

# Fix replication (manual)
repadmin /syncall /AdeP
```

### AD Database Health
```powershell
# Check tombstone lifetime (default 180 days)
Get-ADObject -SearchBase "CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=example,DC=com" `
    -Filter * -Properties tombstoneLifetime

# Check for lingering objects
repadmin /removelingeringobjects

# Rebuild AD indexes (DC only, offline)
# Stop AD, then: esentutl /d "C:\Windows\NTDS\ntds.dit"
```

## 8. Service Accounts

### Create Service Account
```powershell
# Create service account
New-ADUser -Name "SQL-Service" -SamAccountName "sql-svc" `
    -UserPrincipalName "sql-svc@example.com" `
    -Path "OU=Service Accounts,DC=example,DC=com" `
    -AccountPassword (ConvertTo-SecureString "LongSecurePassword123!" -AsPlainText -Force) `
    -PasswordNeverExpires $true `
    -Enabled $true

# Grant SpN (Service Principal Name) for SQL Server
setspn -A MSSQLSvc/sqlserver.example.com sql-svc
setspn -A MSSQLSvc/sqlserver.example.com:1433 sql-svc

# Verify SPN
setspn -L sql-svc
```

## 9. Troubleshooting

### Find Locked Out Users
```powershell
# Search for locked out users
Search-ADAccount -LockedOut

# Unlock specific user
Unlock-ADAccount -Identity jsmith

# Unlock all locked users
Search-ADAccount -LockedOut | Unlock-ADAccount
```

### Password Issues
```powershell
# Users with expired passwords
Search-ADAccount -PasswordExpired

# Users where password never expires
Get-ADUser -Filter 'PasswordNeverExpires -eq $true' | Select-Object Name, SamAccountName

# Users not logged in for 90 days
Search-ADAccount -AccountInactive -TimeSpan 90.00:00:00 | Select-Object Name, LastLogonDate
```

### Group Membership Issues
```powershell
# User is member of group but can't access resource?
# 1. Check if user is ACTUALLY in group:
Get-ADGroupMember "GroupName" | Select-Object Name, SamAccountName

# 2. Check nested groups:
Get-ADPrincipalGroupMembership -Identity jsmith -ResourceContextServer "DC.example.com"

# 3. User needs to logout and login to refresh token
# 4. Or use "klist purge" to clear Kerberos ticket cache
```

### Replication Issues
```powershell
# DC not replicating
repadmin /showrepl <SERVERNAME>

# Fix replication delay
repadmin /syncall <DC_NAME> /A

# Check last logon (helps identify which DC is working)
Get-ADUser jsmith -Properties LastLogonDate
```

## 10. Security Best Practices

| Practice | Implementation | Why |
|----------|----------------|-----|
| Password policy | 90-day expiration, 14+ chars, complexity | Prevent brute force |
| Account lockout | 5 attempts, 30 min lockout | Prevent account takeover |
| Service accounts | Long random passwords, never expire | Service continuity |
| Delegation | Least privilege, use groups | Reduce admin rights |
| Audit logging | Enable AD audit, monitor events | Detect compromise |
| MFA for admins | 2FA on admin accounts | Prevent admin compromise |
| Privileged groups | Separate admin OU, GPO restrictions | Reduce attack surface |
| DNS security | Secure DNSSEC updates | Prevent DNS spoofing |

## 11. Common Scenarios

### Onboard New Department (50 users)
```powershell
# 1. Create OUs
New-ADOrganizationalUnit -Name "Marketing" -Path "OU=Users,DC=example,DC=com"

# 2. Create security group
New-ADGroup -Name "Marketing-Team" -SamAccountName "mkt-team" `
    -GroupScope Global -Path "OU=Groups,DC=example,DC=com"

# 3. Bulk import users from CSV
$users = Import-Csv "marketing-users.csv"
foreach ($user in $users) {
    New-ADUser -Name $user.Name -SamAccountName $user.Username `
        -UserPrincipalName "$($user.Username)@example.com" `
        -Path "OU=Marketing,OU=Users,DC=example,DC=com" `
        -AccountPassword (ConvertTo-SecureString "TempPass123!" -AsPlainText -Force) `
        -Enabled $true -ChangePasswordAtLogon $true
    Add-ADGroupMember -Identity "Marketing-Team" -Members $user.Username
}

# 4. Create GPO for department
New-GPO -Name "Marketing-Policy" | New-GPLink -Target "OU=Marketing,OU=Users,DC=example,DC=com"
```

### Audit Inactive Users
```powershell
# Find users inactive for 90+ days
$inactiveThreshold = (Get-Date).AddDays(-90)
Get-ADUser -Filter {LastLogonDate -lt $inactiveThreshold} -Properties LastLogonDate |
    Select-Object Name, SamAccountName, LastLogonDate

# Disable inactive users
Get-ADUser -Filter {LastLogonDate -lt $inactiveThreshold} -Properties LastLogonDate |
    Disable-ADAccount
```

### Enforce Password Change
```powershell
# Force all users to change password at next logon
Get-ADUser -Filter * -SearchBase "OU=Users,DC=example,DC=com" |
    Set-ADUser -ChangePasswordAtLogon $true

# Notify users (via script/email)
# Inform them of deadline, provide password reset portal
```

## 12. Disaster Recovery

### Backup AD
```powershell
# Windows Server Backup (full system backup)
# Includes: System State, which includes AD database

# Or System State only:
wbadmin start systemstatebackup -backupTarget:D: -quiet

# Restore (DC restart required)
wbadmin start systemstaterecovery -version:<version> -backupTarget:D:
```

### Restore AD Object
```powershell
# If object deleted accidentally:
Get-ADObject -Filter "Name -eq 'UserName'" -IncludeDeletedObjects

# Restore deleted object
Restore-ADObject -Identity "CN=UserName\0ADEL:xxxxx,CN=Deleted Objects,DC=example,DC=com"
```

---

**Essential Commands Cheat Sheet**:
- `Get-ADUser`, `New-ADUser`, `Set-ADUser`, `Remove-ADUser`
- `Get-ADGroup`, `New-ADGroup`, `Add-ADGroupMember`
- `Get-GPO`, `New-GPO`, `Set-GPRegistryValue`
- `Get-ADDomain`, `Get-ADForest`, `Get-ADReplicationPartnerMetadata`
- `repadmin /showrepl` - Check replication
- `dsacls` - Manage ACLs
- `setspn` - Manage SPNs

**This is the foundation of enterprise Windows infrastructure.**
