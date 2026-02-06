# Cookbook: Nginx Web Server from Scratch

> [!NOTE]
> This guide covers installation, configuration, and SSL setup on **Ubuntu 22.04 LTS**.
> Commands should be run as `root` or via `sudo`.

## 1. Installation

```bash
# Update repositories
apt update

# Install Nginx
apt install -y nginx

# Verify installation
nginx -v
systemctl status nginx
```

Expected output:
> Active: active (running)

## 2. Firewall Configuration (UFW)
Ensure HTTP (80) and HTTPS (443) are allowed.

```bash
ufw allow 'Nginx Full'
ufw reload
ufw status
```

## 3. Creating a Virtual Host (Server Block)

We will setup a site for `example.com`.

### Step 3.1: Create Web Root
```bash
# Create directory
mkdir -p /var/www/example.com/html

# Assign ownership to current user (for editing)
chown -R $USER:$USER /var/www/example.com/html

# Create a sample index.html
echo '<h1>Success! The example.com server block is working!</h1>' > /var/www/example.com/html/index.html
```

### Step 3.2: Create Config File
Create `/etc/nginx/sites-available/example.com`:

```nginx
server {
    listen 80;
    listen [::]:80;

    root /var/www/example.com/html;
    index index.html index.htm index.nginx-debian.html;

    server_name example.com www.example.com;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Step 3.3: Enable the Site
Link to `sites-enabled`.

```bash
ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
```

### Step 3.4: Test & Restart
```bash
# Check for syntax errors (CRITICAL STEP)
nginx -t

# If successful, restart
systemctl restart nginx
```

## 4. SSL Certificates (Let's Encrypt)

Install Certbot.

```bash
apt install -y certbot python3-certbot-nginx
```

Obtain certificate and automatically configure Nginx (Redirect HTTP -> HTTPS).

```bash
certbot --nginx -d example.com -d www.example.com
```

## 5. Hardening (Security Headers)

Edit `/etc/nginx/conf.d/security.conf`:

```nginx
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header X-Content-Type-Options "nosniff" always;
add_header Referrer-Policy "no-referrer-when-downgrade" always;
add_header Content-Security-Policy "default-src 'self' http: https: data: blob: 'unsafe-inline'" always;
```

Include this in your `nginx.conf` or sites if not automatic.
