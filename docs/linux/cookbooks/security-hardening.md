# Cookbook: Basic Server Hardening

> [!IMPORTANT]
> Perform these steps immediately after provisioning a new server.
> Tested on **Ubuntu 22.04 LTS**.

## 1. Create a Sudo User
**Never** operate as root for daily tasks.

```bash
# Create user 'adminUser'
adduser adminUser

# Add to sudo group
usermod -aG sudo adminUser

# Switch to new user
su - adminUser
```

## 2. SSH Hardening

Edit `/etc/ssh/sshd_config`.

```bash
sudo nano /etc/ssh/sshd_config
```

Change the following lines:

```config
# 1. Disable Root Login
PermitRootLogin no

# 2. Disable Password Auth (Force Application of Keys)
# Ensure you have copied your SSH key first! (ssh-copy-id)
PasswordAuthentication no

# 3. Change Port (Optional, reduces noise)
Port 2222
```

Restart SSH:
```bash
sudo systemctl restart ssh
```

## 3. Configure Firewall (UFW)
Deny everything incoming by default, allow only what's needed.

```bash
# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow SSH (If you changed port, use that port!)
sudo ufw allow 2222/tcp

# Allow Web
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Enable
sudo ufw enable
```

**Verify status:**
```bash
sudo ufw status verbose
```

## 4. Install Fail2Ban
Ban IPs that spam failed login attempts.

```bash
sudo apt install -y fail2ban
```

Copy config to local overrides:
```bash
sudo cp /etc/fail2ban/jail.conf /etc/fail2ban/jail.local
```

Restart service:
```bash
sudo systemctl restart fail2ban
sudo systemctl status fail2ban
```

Check banned IPs:
```bash
sudo fail2ban-client status sshd
```

## 5. Automatic Updates
Keep security patches automatic.

```bash
sudo apt install -y unattended-upgrades
sudo dpkg-reconfigure --priority=low unattended-upgrades
```
