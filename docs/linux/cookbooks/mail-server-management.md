# Cookbook: Mail Server Management

> [!IMPORTANT]
> Self-hosting mail in the cloud era is challenging due to strict IP reputation checks. 
> This guide focuses on **Postfix**, **Dovecot**, and **Deliverability** (SPF/DKIM/DMARC) for Ubuntu 22.04/24.04.

## 1. Postfix: Sending Mail (MTA)

### Installation
During installation, select **Internet Site** and enter your FQDN (e.g., `mail.example.com`).
```bash
sudo apt update
sudo apt install postfix mailutils -y
```

### Modern Submission (Port 587)
Edit `/etc/postfix/master.cf` to enable the submission port with STARTTLS (uncomment the following lines):
```text
submission inet n       -       y       -       -       smtpd
  -o syslog_name=postfix/submission
  -o smtpd_tls_security_level=encrypt
  -o smtpd_sasl_auth_enable=yes
  -o smtpd_relay_restrictions=permit_sasl_authenticated,reject
```

### Basic Security (`/etc/postfix/main.cf`)
```text
myhostname = mail.example.com
mydestination = $myhostname, example.com, localhost.localdomain, localhost
mynetworks = 127.0.0.0/8 [::ffff:127.0.0.0]/104 [::1]/128
home_mailbox = Maildir/
```
*Reload:* `sudo systemctl reload postfix`

---

## 2. Dovecot: Receiving Mail (MDA)

### Installation
```bash
sudo apt install dovecot-imapd dovecot-pop3d -y
```

### Configuration Highlights
- **Mail Location** (`/etc/dovecot/conf.d/10-mail.conf`):
  `mail_location = maildir:~/Maildir`
- **SSL Enforcement** (`/etc/dovecot/conf.d/10-ssl.conf`):
  `ssl = required`

---

## 3. Deliverability: SPF, DKIM, and DMARC

Modern mail providers (Gmail, Outlook) will reject mail without these DNS records.

### SPF (Sender Policy Framework)
Add a TXT record for your domain:
`v=spf1 mx ip4:YOUR_SERVER_IP ~all`

### DKIM (DomainKeys Identified Mail)
1. **Install OpenDKIM**: `sudo apt install opendkim opendkim-tools`
2. **Generate Keys**: 
   ```bash
   sudo opendkim-genkey -s mail -d example.com
   ```
3. **Publish DNS TXT**: Add the content of `mail.txt` to your DNS records for `mail._domainkey`.

### DMARC
Add a TXT record for `_dmarc.example.com`:
`v=DMARC1; p=quarantine; adkim=s; aspf=s;`

---

## 4. SSL with Certbot

Automate your certificates for both Postfix and Dovecot.

```bash
sudo snap install --classic certbot
sudo certbot certonly --standalone -d mail.example.com
```

**Point Postfix/Dovecot to Certbot paths:**
- **Postfix**: `smtpd_tls_cert_file=/etc/letsencrypt/live/mail.example.com/fullchain.pem`
- **Dovecot**: `ssl_cert = </etc/letsencrypt/live/mail.example.com/fullchain.pem`

---

## 5. Virtual Users (Database)

Instead of system accounts, use a shared database (or simple file) for mail users.
Create `/etc/postfix/virtual_mailbox_maps`:
```text
user@example.com example.com/user/
```
*Hash and Link:* `sudo postmap /etc/postfix/virtual_mailbox_maps`

---

## 6. Modern All-in-One Stacks (Docker)

If you need a full suite (Webmail, Antispam, Admin UI) and prefer not to configure Postfix/Dovecot from scratch, consider **Mailcow** or **Docker Mailserver**.

- **Mailcow (Dockerized)**: Includes SOGo (Webmail), Rspamd (Antispam), and a beautiful Admin UI.
- **Why?**: Handles DKIM/SSL/Antispam automatically.

---

## 7. Troubleshooting

### Essential Commands
- **Check Logs**: `tail -f /var/log/mail.log`
- **Postfix Config Test**: `sudo postconf -n`
- **Dovecot Config Test**: `sudo doveconf -n`
- **Test SMTP Port 587**: `telnet localhost 587`
- **MX Record Check**: `dig MX example.com`

### Common Errors
- **Connection Timed Out**: Port 25 is often blocked by cloud providers (AWS, GCP). Use an external relay (like Amazon SES or SendGrid) if needed.
- **Permission Denied**: Ensure `Maildir` is owned by the user (`chown -R user:user ~/Maildir`).
