# Cookbook: User and Group Management

> [!NOTE]
> Optimized for Ubuntu 22.04 LTS and 24.04 LTS.
> Comprehensive guide for managing user accounts, groups, permissions, and security policies.

## 1. Managing User Accounts

### Creating a User Account
The `adduser` command is a high-level interactive script recommended for Ubuntu. It handles home directory creation and skeleton file population automatically.

```bash
# Interactive user creation
sudo adduser <username>

# Non-interactive (batch/scripting)
sudo adduser --disabled-password --gecos "" <username>
```

### Creating Users in Batch Mode
You can use `newusers` to create multiple accounts from a text file.

1. **Create the user list file:**
   ```bash
   # Format: username:password:UID:GID:gecos:home_dir:shell
   # Note: Leave UID/GID/gecos empty to use defaults
   nano users.txt
   ```
2. **Secure the file:**
   ```bash
   chmod 600 users.txt
   ```
3. **Import users:**
   ```bash
   sudo newusers users.txt
   ```

### Modifying and Deleting Users
```bash
# Change user shell
sudo usermod -s /bin/bash <username>

# Lock/Unlock account
sudo usermod -L <username>  # Lock
sudo usermod -U <username>  # Unlock

# Expire an account (disable on specific date)
sudo usermod --expiredate 2024-12-31 <username>

# Delete user (keeping home directory)
sudo deluser <username>

# Delete user AND remove home/mail spool
sudo deluser --remove-home <username>

# Delete user with backup of home directory
sudo deluser --backup --remove-home <username>
```

---

## 2. Group Management

Groups serve as the primary mechanism for collective permission management.

```bash
# Create a new group
sudo addgroup <groupname>

# Create a system group
sudo addgroup --system <groupname>

# Add existing user to a group
sudo adduser <username> <groupname>

# Alternative: Add user to multiple supplemental groups (append)
sudo usermod -a -G group1,group2 <username>

# Remove user from a specific group
sudo deluser <username> <groupname>

# Delete a group
sudo delgroup <groupname>
```

---

## 3. Permissions and Ownership

### Changing Ownership
```bash
# Change owner
sudo chown <username> <file>

# Change owner and group recursively
sudo chown -R <username>:<groupname> <directory>

# Change group only
sudo chgrp <groupname> <file>
```

### Setting Permissions (chmod)
| Octal | Symbolic | Description |
|---|---|---|
| 755 | `u=rwx,go=rx` | Owner: Full; Others: Read/Execute (Common for dirs) |
| 644 | `u=rw,go=r` | Owner: Read/Write; Others: Read (Common for files) |
| 600 | `u=rw,go=` | Owner Only: Read/Write (Secure files like keys) |
| 700 | `u=rwx,go=` | Owner Only: Full (Secure directories) |

```bash
# Recursive permission change
chmod -R 755 /path/to/directory

# Set Sticky Bit (Only owner/root can delete files in directory)
chmod +t /shared/directory
```

---

## 4. Privilege Escalation (Sudo)

### Granting Sudo Access
In Ubuntu, members of the `sudo` group have full administrative privileges.

```bash
# Add user to sudo group
sudo adduser <username> sudo
```

### Hardening Sudo (`visudo`)
Always use `visudo` to edit `/etc/sudoers` to prevent syntax errors that could lock you out.

```bash
sudo visudo
```

**Common Configurations:**
```text
# Allow passwordless sudo for specific group (Use with caution!)
%sudo ALL=(ALL:ALL) NOPASSWD: ALL

# Limit sudo to specific commands
<username> ALL=(ALL) /usr/bin/apt-get update, /usr/bin/apt-get upgrade
```

---

## 5. Resource Limits (`limits.conf`)

Control system resource consumption per user or group in `/etc/security/limits.conf`.

```text
# /etc/security/limits.conf
# Format: <domain> <type> <item> <value>

# Limit max open files for 'developer' user
developer    soft    nofile    4096
developer    hard    nofile    8192

# Limit max processes for 'guest' group
@guest       hard    nproc     20
```

---

## 6. Security Hardening

### Password Quality (PAM)
Ubuntu 22.04/24.04 uses `pam_pwquality`. Configure it in `/etc/security/pwquality.conf`.

```bash
sudo nano /etc/security/pwquality.conf
```

**Recommended Policy:**
- `minlen = 12` (Minimum length)
- `dcredit = -1` (At least one digit)
- `ucredit = -1` (At least one uppercase)
- `ocredit = -1` (At least one special character)

**Enforce in `/etc/pam.d/common-password`:**
```text
password requisite pam_pwquality.so retry=3
```

### SSH Security Basics
Refer to the [SSH Management Cookbook](file:///media/tofecha/Data/workspace/myGitHub/ops-handbook/docs/linux/cookbooks/ssh-management.md) for full hardening.

1. **Disable Root Login:** `PermitRootLogin no`
2. **Disable Password Auth:** `PasswordAuthentication no`
3. **Use Public Keys:** Store in `~/.ssh/authorized_keys` with `600` permissions.

---

### Verification Checklist
- [ ] Users created with `adduser` have correct home directory permissions (`750` or `700`).
- [ ] Sudo access is tested and restricted where possible.
- [ ] Resource limits verified with `ulimit -a`.
- [ ] Password complexity verified by attempting to set a weak password.
