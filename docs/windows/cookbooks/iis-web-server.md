# Cookbook: IIS Web Server (PowerShell)

> [!NOTE]
> This guide installs and configures IIS completely via PowerShell.
> Run PowerShell as **Administrator**.

## 1. Install IIS

```powershell
# Install Web-Server role and management tools
Install-WindowsFeature -Name Web-Server -IncludeManagementTools

# Verify installation
Get-WindowsFeature Web-Server
Get-Service W3SVC
```

## 2. Site Management

### Step 2.1: Clean Defaults
Remove the default website to start fresh.

```powershell
Remove-Website -Name "Default Web Site"
```

### Step 2.2: Create Directory & Content
```powershell
# Create folder
New-Item -ItemType Directory -Path "C:\inetpub\MySite"

# Create sample page
Set-Content -Path "C:\inetpub\MySite\index.html" -Value "<h1>IIS Managed by PowerShell</h1>"
```

### Step 2.3: Create App Pool
Isolate the process.

```powershell
New-WebAppPool -Name "MySitePool"
```

### Step 2.4: Create the Site
Bind to Port 80.

```powershell
New-Website -Name "MySite" -Port 80 -PhysicalPath "C:\inetpub\MySite" -ApplicationPool "MySitePool"
```

## 3. SSL Configuration (Self-Signed)
For production, import a real PFX. Here we create a self-signed one for testing.

```powershell
# Create Cert
$cert = New-SelfSignedCertificate -DnsName "mysite.local" -CertStoreLocation "cert:\LocalMachine\My"

# Create HTTPS Binding
New-WebBinding -Name "MySite" -Protocol "https" -Port 443

# Attach Cert to Binding
# (Requires IIS Administration module or manual netsh/GUI for complex bindings, 
#  but below is the pure POSH way using the PSDrive)
$path = "IIS:\SslBindings\0.0.0.0!443"
New-Item -Path $path -Value $cert
```

## 4. Firewall
Ensure ports are open.

```powershell
New-NetFirewallRule -DisplayName "Allow HTTP" -Direction Inbound -LocalPort 80 -Protocol TCP -Action Allow
New-NetFirewallRule -DisplayName "Allow HTTPS" -Direction Inbound -LocalPort 443 -Protocol TCP -Action Allow
```
