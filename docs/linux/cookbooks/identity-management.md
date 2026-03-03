# Cookbook: Identity Management (LDAP & SSSD)

> [!IMPORTANT]
> This guide modernizes centralized authentication for Ubuntu 22.04/24.04 LTS. 
> Key updates include **SSSD** for robust client authentication and **LDAP Account Manager (LAM)** for a modern management UI.

## 1. OpenLDAP: The Backend

OpenLDAP (`slapd`) remains the standard for open-source directory services. Modern Ubuntu uses **OLC (Online Configuration)** or `cn=config`.

### Installation
```bash
sudo apt update
sudo apt install slapd ldap-utils -y
```
During installation, you will be prompted for an administrator password.

### Initial Configuration
If you need to change the base DN or organization name:
```bash
sudo dpkg-reconfigure slapd
```
*Tip: Omit LDAP server configuration? NO. DNS domain name? e.g., `example.com`.*

---

## 2. LDAP Account Manager (LAM)

`phpLDAPadmin` is largely deprecated and buggy on modern PHP. **LDAP Account Manager (LAM)** is the modern, stable replacement.

### Installation
```bash
sudo apt install ldap-account-manager -y
```

### Accessing LAM
1.  Navigate to `http://your-server-ip/lam`.
2.  **Configuration**: Click on "LAM configuration" -> "Edit server profiles".
3.  **Login**: Use the default profile (`lam`) and your LDAP admin password.
4.  **Manage**: Create Organizational Units (OUs) like `People` and `Groups` to organize your users.

---

## 3. SSSD: Modern Client Authentication

The legacy `ldap-auth-client` (libnss-ldap) is fragile. **SSSD** (System Security Services Daemon) is the modern standard for joining Linux systems to LDAP or Active Directory.

### Installation (LDAP Client)
```bash
sudo apt install sssd sssd-ldap -y
```

### Configuration (`/etc/sssd/sssd.conf`)
Create the file with strict permissions:
```bash
sudo touch /etc/sssd/sssd.conf
sudo chmod 600 /etc/sssd/sssd.conf
```
Add the following configuration:
```ini
[sssd]
services = nss, pam
config_file_version = 2
domains = default

[domain/default]
id_provider = ldap
auth_provider = ldap
ldap_uri = ldap://your-ldap-server-ip
ldap_search_base = dc=example,dc=com
ldap_id_use_start_tls = false
cache_credentials = True
ldap_tls_reqcert = allow
```
*Note: Enable `ldaps://` or `STARTTLS` for production.*

### Activation
```bash
sudo systemctl restart sssd
# Ensure pam-auth-update includes SSSD
sudo pam-auth-update --enable sssd
```

---

## 4. Modern Alternative: Lldap

For home servers or simple Docker environments, Full OpenLDAP is often overkill. **Lldap** (Lightweight LDAP) is a modern, high-performance LDAP server with a built-in web UI.

- **Why Lldap?**: Tiny footprint, easy to manage, specific for user authentication.
- **Setup**: Best deployed via Docker.
- **Compatibility**: Works perfectly with SSSD and Jellyfin/Nextcloud.

---

## 5. Troubleshooting

- **LDAP Search**: `ldapsearch -x -b "dc=example,dc=com"`
- **SSSD Logs**: `sudo journalctl -u sssd`
- **User Discovery**: `id ldapuser` (Should return user details if SSSD is working).
- **LAM Issues**: Check Apache logs at `/var/log/apache2/error.log`.
