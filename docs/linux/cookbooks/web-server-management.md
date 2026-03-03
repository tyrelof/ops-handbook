# Cookbook: Web Server Management

> [!NOTE]
> Optimized for Ubuntu 22.04 LTS (PHP 8.1) and 24.04 LTS (PHP 8.3).
> Focuses on **PHP-FPM**, **Nginx Proxying**, and **Automated SSL**.

## 1. Apache Web Server with PHP 8.3

Modern Apache on Ubuntu uses the **Event MPM** and **PHP-FPM** for better performance vs legacy `mod_php`.

### Installation
```bash
sudo apt update
sudo apt install apache2 php8.3-fpm libapache2-mod-fcgid
```

### Configure Virtual Host (`/etc/apache2/sites-available/example.com.conf`)
```apache
<VirtualHost *:80>
    ServerName example.com
    DocumentRoot /var/www/example.com/public_html

    # Proxy PHP requests to FPM socket
    <FilesMatch \.php$>
        SetHandler "proxy:unix:/run/php/php8.3-fpm.sock|fcgi://localhost"
    </FilesMatch>

    ErrorLog ${APACHE_LOG_DIR}/example.com_error.log
    CustomLog ${APACHE_LOG_DIR}/example.com_access.log combined
</VirtualHost>
```

### Enable Site and Necessary Modules
```bash
sudo a2enmod proxy_fcgi setenvif
sudo a2enconf php8.3-fpm
sudo a2ensite example.com.conf
sudo systemctl restart apache2
```

---

## 2. Nginx with PHP-FPM

Nginx is the standard for high-concurrency environments.

### Installation
```bash
sudo apt install nginx php8.3-fpm
```

### Configure Server Block (`/etc/nginx/sites-available/example.com`)
```nginx
server {
    listen 80;
    server_name example.com;
    root /var/www/example.com/public_html;
    index index.php index.html;

    location / {
        try_files $uri $uri/ =404;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }
}
```

---

## 3. Nginx as a Reverse Proxy for Apache

Use Nginx to serve static files and terminate SSL, while Apache handles legacy `.htaccess` or specific modules.

### Nginx Config (`/etc/nginx/sites-available/proxy`)
```nginx
server {
    listen 80;
    server_name example.com;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
*Note: Ensure Apache listens on port 8080 in `/etc/apache2/ports.conf`.*

---

## 4. Modern SSL with Let's Encrypt (Certbot)

Stop using self-signed certificates for public sites. Use [Certbot](https://certbot.eff.org/).

### Installation & Setup
```bash
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot

# For Nginx
sudo certbot --nginx -d example.com

# For Apache
sudo certbot --apache -d example.com
```
*Certbot automatically adds a renewal timer to systemd.*

---

## 5. Hardening & Performance

### Security Headers
Add these to your Nginx `server` block:
```nginx
add_header X-Frame-Options "SAMEORIGIN";
add_header X-Content-Type-Options "nosniff";
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
```

### Benchmarking with `wrk`
A modern alternative to `ab`:
```bash
# Install
sudo apt install wrk

# Run test: 12 threads, 400 connections, for 30 seconds
wrk -t12 -c400 -d30s http://127.0.0.1/
```

---

## 6. Troubleshooting

### Check Configuration Syntax (The Golden Rule)
- **Nginx**: `sudo nginx -t`
- **Apache**: `sudo apache2ctl configtest`

### Log Locations
- **Nginx**: `/var/log/nginx/error.log`
- **Apache**: `/var/log/apache2/error.log`
- **PHP-FPM**: `/var/log/php8.3-fpm.log`

### Process/Socket Check
```bash
sudo systemctl status php8.3-fpm
ls -l /run/php/php8.3-fpm.sock
ss -tulpn | grep -E '80|443'
```
