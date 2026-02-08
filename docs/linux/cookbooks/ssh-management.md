# Cookbook: SSH Configuration & Key Management

> [!NOTE]
> Best practices for SSH access, key rotation, and hardening.
> Covers: User management, key generation, config files, and security patterns.

## 1. SSH Key Generation (Ed25519)

### On Your Local Machine
```bash
# Generate a new key (modern, secure)
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "your-email@example.com"

# Or RSA (if Ed25519 not available)
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -C "your-email@example.com"

# Set strong passphrase when prompted (use password manager)
```

### Key Storage
```bash
# Keys are created in ~/.ssh/
ls -la ~/.ssh/

# Public key (share this)
cat ~/.ssh/id_ed25519.pub

# Private key (NEVER share - protect with 0600)
chmod 600 ~/.ssh/id_ed25519

# ssh dir permissions
chmod 700 ~/.ssh/
```

## 2. Adding Keys to Remote Servers

### Manually (First Time)
```bash
# Copy your public key to server
scp ~/.ssh/id_ed25519.pub user@server.com:~/.ssh/

# SSH in and authorize it
ssh user@server.com
mkdir -p ~/.ssh
chmod 700 ~/.ssh
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
exit
```

### Automated
```bash
# ssh-copy-id (does the above automatically)
ssh-copy-id -i ~/.ssh/id_ed25519.pub user@server.com

# For custom SSH port
ssh-copy-id -i ~/.ssh/id_ed25519.pub -p 2222 user@server.com
```

### For Multiple Servers
```bash
# Script to deploy to many servers
SERVERS=(
    "user@server1.com"
    "user@server2.com"
    "user@server3.com"
)

for server in "${SERVERS[@]}"; do
    ssh-copy-id -i ~/.ssh/id_ed25519.pub "$server"
done
```

## 3. SSH Config File

### Create ~/.ssh/config
```bash
# ~/.ssh/config
# Centralize host configurations

Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519
    AddKeysToAgent yes

Host prod
    HostName prod.example.com
    User ubuntu
    Port 2222
    IdentityFile ~/.ssh/prod_key
    ServerAliveInterval 60
    StrictHostKeyChecking accept-new

Host staging
    HostName staging.example.com
    User ubuntu
    IdentityFile ~/.ssh/staging_key

Host *.example.com
    User ubuntu
    IdentityFile ~/.ssh/id_ed25519
    StrictHostKeyChecking accept-new
    UserKnownHostsFile ~/.ssh/known_hosts
```

### Usage
```bash
# Instead of: ssh -i ~/.ssh/prod_key -p 2222 ubuntu@prod.example.com
ssh prod

# Or just the domain
ssh staging

# SCP becomes simpler
scp file.txt prod:/tmp/
scp prod:/var/log/app.log ./
```

## 4. Server-Side Configuration (/etc/ssh/sshd_config)

### Hardening SSH
```bash
# Connect as root first
sudo -i

# Backup config
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup

# Edit config
nano /etc/ssh/sshd_config
```

### Recommended Settings
```bash
# /etc/ssh/sshd_config

# Only allow SSH protocol 2 (not 1)
Protocol 2

# Disable root login
PermitRootLogin no

# Disable password auth (keys only)
PasswordAuthentication no
PubkeyAuthentication yes

# Disable empty passwords
PermitEmptyPasswords no

# Restrict to specific users
AllowUsers ubuntu@10.0.0.0/8 ubuntu@203.0.113.0
# Or just
AllowUsers ubuntu

# Change port (security by obscurity, weak but adds noise)
Port 2222

# Only listen on IPv4 (or specify addresses)
ListenAddress 0.0.0.0
AddressFamily inet

# Disconnect after 3 failed logins
MaxAuthTries 3

# Disconnect idle connections after 5 mins
ClientAliveInterval 300
ClientAliveCountMax 0

# Disable X11 forwarding (if not needed)
X11Forwarding no

# Disable tunnel forwarding
PermitTunnel no

# Limit concurrent sessions
MaxSessions 10

# Use strong ciphers (modern)
Ciphers chacha20-poly1305@openssh.com,aes256-gcm@openssh.com,aes256-ctr
MACs hmac-sha2-512-etm@openssh.com,hmac-sha2-256-etm@openssh.com
KexAlgorithms curve25519-sha256,curve25519-sha256@libssh.org
HostKeyAlgorithms ssh-ed25519

# Require all auth methods to succeed
AuthenticationMethods publickey,password

# Log verbosely
LogLevel VERBOSE
```

### Apply Changes
```bash
# Validate config (won't lock you out)
sshd -t

# Restart SSH
systemctl restart ssh
# or
service sshd restart

# DON'T CLOSE YOUR SESSION - test in another terminal first!
ssh prod "echo success"
```

## 5. Key Management

### Rotating Keys (Annual)
```bash
# Generate new key
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_new -C "your-email@example.com"

# Copy to all servers
ssh-copy-id -i ~/.ssh/id_ed25519_new.pub user@server.com

# Verify new key works
ssh -i ~/.ssh/id_ed25519_new user@server.com "echo success"

# Once verified on all servers, remove old key from authorized_keys
ssh user@server.com
nano ~/.ssh/authorized_keys  # Delete old key line
exit

# Remove old key locally
rm ~/.ssh/id_ed25519
mv ~/.ssh/id_ed25519_new ~/.ssh/id_ed25519
```

### Using Multiple Keys
```bash
# ~/.ssh/config can specify different keys per host
Host github.com
    IdentityFile ~/.ssh/github_key

Host gitlab.com
    IdentityFile ~/.ssh/gitlab_key

Host prod
    IdentityFile ~/.ssh/prod_key

# SSH tries keys in order from ~/.ssh/
# Or explicitly:
ssh -i ~/.ssh/prod_key ubuntu@prod.example.com
```

### SSH Agent (Avoid Repeated Passphrases)

```bash
# Start agent
eval "$(ssh-agent -s)"

# Add key (will prompt for passphrase once)
ssh-add ~/.ssh/id_ed25519

# List loaded keys
ssh-add -l

# Remove from agent
ssh-add -d ~/.ssh/id_ed25519

# Remove all keys
ssh-add -D
```

### Automatic SSH Agent (Add to ~/.bashrc or ~/.zshrc)
```bash
# Auto-start SSH agent on login
if [ -z "$SSH_AUTH_SOCK" ]; then
    eval "$(ssh-agent -s)"
    ssh-add ~/.ssh/id_ed25519
fi
```

## 6. Bastion/Jump Host Setup

### Connect via Jump Server
```bash
# ~/.ssh/config
Host bastion
    HostName bastion.example.com
    User ubuntu
    IdentityFile ~/.ssh/bastion_key

Host internal
    HostName 10.0.1.5
    User ubuntu
    IdentityFile ~/.ssh/internal_key
    ProxyJump bastion
```

### Usage
```bash
# Connects via bastion automatically
ssh internal

# Or manually
ssh -J ubuntu@bastion.example.com ubuntu@10.0.1.5

# SCP through bastion
scp -J bastion file.txt internal:/tmp/
```

### Without Config (Old Style)
```bash
# Using SSH tunneling
ssh -L 9999:internal:22 ubuntu@bastion &
ssh -p 9999 ubuntu@localhost
```

## 7. Kubernetes/Cloud Server Access

### AWS EC2 Key Management
```bash
# List your key pairs
aws ec2 describe-key-pairs --region us-east-1

# Create new key pair
aws ec2 create-key-pair --key-name my-key --region us-east-1 --query 'KeyMaterial' --output text > ~/.ssh/my-key.pem
chmod 600 ~/.ssh/my-key.pem

# Connect to instance
ssh -i ~/.ssh/my-key.pem ubuntu@ec2-instance-ip
```

### EKS/Kubernetes Node Access
```bash
# Add SSH key to node (via Systems Manager or EC2 user data)
# In ~/.ssh/config
Host eks-node
    HostName 10.0.1.100
    User ec2-user
    IdentityFile ~/.ssh/eks_key
    ProxyJump bastion

# Or use AWS Session Manager (no SSH needed)
aws ssm start-session --target i-0123456789abcdef0
```

## 8. Troubleshooting SSH Issues

### Connection Refused
```bash
# Check if SSH is running
sudo systemctl status ssh

# Check if port is listening
netstat -tulpn | grep :22

# Try verbose mode to see what's happening
ssh -vv user@server.com
```

### Permission Denied (Public Key)
```bash
# Verify key permissions (must be 600)
chmod 600 ~/.ssh/id_ed25519
chmod 700 ~/.ssh/

# Verify remote authorized_keys
ssh user@server.com "cat ~/.ssh/authorized_keys"

# Try with specific key
ssh -i ~/.ssh/id_ed25519 user@server.com

# Check SSH debug output
ssh -vvv user@server.com
```

### Host Key Verification Failed
```bash
# First time connecting, add to known_hosts
ssh-keyscan -H server.com >> ~/.ssh/known_hosts

# Or accept it automatically (less secure)
ssh -o StrictHostKeyChecking=accept-new user@server.com

# Remove old entry (if key changed)
ssh-keygen -R server.com
```

### Too Many Authentication Failures
```bash
# Means server tried too many keys
# Explicitly specify which key to use
ssh -i ~/.ssh/id_ed25519 user@server.com

# Or limit key identities in config
Host server.com
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

## 9. Security Best Practices

| Practice | Why | How |
|----------|-----|-----|
| Use Ed25519 keys | Modern, secure, smaller | Use `-t ed25519` when generating |
| Passphrase-protect keys | If key is stolen, still secure | Set passphrase in `ssh-keygen` |
| Disable password auth | Prevents brute force | Set `PasswordAuthentication no` in sshd_config |
| Use non-standard port | Reduces automated attacks | Change `Port` in sshd_config |
| Disable root login | Prevents root account compromise | Set `PermitRootLogin no` |
| Rotate keys annually | Limits exposure of old keys | Generate new key, deploy, remove old |
| Use jump hosts | Protects internal network | Use `ProxyJump` in config |
| Monitor failed logins | Detect attacks | Check `/var/log/auth.log` |

## 10. Monitoring SSH Access

### View Login History
```bash
# Failed logins
grep "Failed password" /var/log/auth.log | tail -10

# Successful logins
grep "Accepted publickey" /var/log/auth.log | tail -10

# All SSH events
journalctl -u ssh -n 50
```

### Alert on Suspicious Activity
```bash
# Script to monitor failed logins
watch -n 5 'grep "Failed password" /var/log/auth.log | tail -5'

# Or real-time
tail -f /var/log/auth.log | grep "Failed\|Accepted"
```

---

**Checklist**:
- Generate keys with Ed25519
- Add keys to servers with `ssh-copy-id`
- Harden sshd_config (disable password auth, root login)
- Use SSH config for host profiles
- Rotate keys annually
- Monitor failed login attempts
- Use jump hosts for internal servers
