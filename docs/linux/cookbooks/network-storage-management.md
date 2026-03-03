# Cookbook: Network Storage Management

> [!IMPORTANT]
> This guide covers modern network storage for Ubuntu 22.04/24.04 LTS. 
> Key modernizations include **Samba 4 (SMBv2/3)**, **NFSv4.2**, and **Secure SFTP** via vsftpd.

## 1. Samba: Cross-Platform File Sharing

Samba 4 is the standard for sharing files with Windows and Mac clients. Modern versions disable the insecure SMBv1 protocol by default.

### Installation
```bash
sudo apt update
sudo apt install samba -y
```

### Basic Public Share (No Auth)
Edit `/etc/samba/smb.conf`:
```text
[Public]
path = /var/samba/public
browsable = yes
writable = yes
guest ok = yes
read only = no
force user = nobody
```
*Permissions:* `sudo mkdir -p /var/samba/public && sudo chmod 777 /var/samba/public`

### Secure Private Share (Authenticated)
Add a user and set a Samba-specific password:
```bash
sudo useradd -s /sbin/nologin smbuser
sudo smbpasswd -a smbuser
```
Edit `/etc/samba/smb.conf`:
```text
[Private]
path = /var/samba/private
valid users = smbuser
browsable = yes
writable = yes
read only = no
```
*Reload:* `sudo systemctl restart smbd`

---

## 2. NFS: Linux-to-Linux Storage

NFSv4.2 provides high-performance storage sharing for Linux servers.

### Installation (Server)
```bash
sudo apt install nfs-kernel-server -y
```

### Exporting a Directory
Edit `/etc/exports`:
```text
/var/nfs_share  10.0.0.0/24(rw,sync,no_subtree_check)
```
*Apply:* `sudo exportfs -ra`

### Mounting (Client)
```bash
sudo apt install nfs-common -y
sudo mkdir /mnt/nas
sudo mount -t nfs 10.0.0.10:/var/nfs_share /mnt/nas
```

### Persistent Mount (`/etc/fstab`)
Add this line for persistent mounting on reboot:
```text
10.0.0.10:/var/nfs_share  /mnt/nas  nfs  defaults,_netdev,nfsvers=4.2  0  0
```

---

## 3. Secure FTP (vsftpd with TLS)

Legacy FTP is insecure. Always wrap it in TLS/SSL.

### Installation
```bash
sudo apt install vsftpd -y
```

### Configuration (`/etc/vsftpd.conf`)
Modernize security settings:
```text
anonymous_enable=NO
local_enable=YES
write_enable=YES
chroot_local_user=YES
ssl_enable=YES
allow_anon_ssl=NO
force_local_data_ssl=YES
force_local_logins_ssl=YES
ssl_tlsv1_2=YES
ssl_tlsv1_3=YES
```
*Certificates:* By default, Ubuntu provides a "snakeoil" cert at `/etc/ssl/certs/ssl-cert-snakeoil.pem`. Replace with a Let's Encrypt cert for production.

---

## 4. Rsync: Efficient Synchronization

Rsync is the gold standard for file mirroring and backups.

### Modern Best Practices
- **Archive Mode (`-a`)**: Preserves permissions, symlinks, and timestamps.
- **Compression (`-z`)**: Compresses data during transfer.
- **Partial/Progress (`-P`)**: Resumes interrupted transfers and shows progress.

### Sync Local to Remote (via SSH)
```bash
rsync -avzP /local/path/ user@remote_ip:/remote/path/
```

### Dry Run (Safety First)
Always dry run destructive commands:
```bash
rsync -avzP --dry-run --delete /src/ /dest/
```

---

## 5. Troubleshooting

### Samba
- **Syntax Check**: `testparm`
- **Check Connections**: `smbstatus`
- **Logs**: `/var/log/samba/log.smbd`

### NFS
- **List Exports**: `showmount -e [server_ip]`
- **Check RPC**: `rpcinfo -p`

### FTP
- **Logs**: `/var/log/vsftpd.log`
- **Port Check**: `ss -tpln | grep 21`
