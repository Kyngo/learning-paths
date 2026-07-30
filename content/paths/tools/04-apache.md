---
title: "Apache HTTP Server"
weight: 4
---

## What Is Apache?

**Apache HTTP Server** (commonly "Apache" or "httpd") is the world's most widely-used web server, maintained by the Apache Software Foundation since 1995. It serves static and dynamic content via a modular architecture with support for virtual hosts, URL rewriting, authentication, proxying, and more.

### Apache vs Nginx

| Aspect | Apache | Nginx |
|--------|--------|-------|
| Architecture | Process/thread per connection | Event-driven, async |
| Config style | `.htaccess` per-directory (flexible) | Centralized config (performant) |
| Dynamic content | `mod_php`, `mod_python` (in-process) | Proxies to external process (PHP-FPM) |
| Best for | Shared hosting, `.htaccess` flexibility | High concurrency, reverse proxying |
| Performance (static) | Good | Excellent |
| Performance (dynamic) | Excellent (mod_php) | Excellent (via proxy) |

---

## Installation

```bash
# Debian/Ubuntu
sudo apt update && sudo apt install apache2

# RHEL/CentOS/Amazon Linux
sudo yum install httpd        # or: sudo dnf install httpd

# macOS (built-in, or via Homebrew)
brew install httpd

# Start and enable
sudo systemctl start apache2       # Debian
sudo systemctl start httpd         # RHEL
sudo systemctl enable apache2
```

---

## Directory Structure

### Debian/Ubuntu

```
/etc/apache2/
├── apache2.conf          # Main config
├── ports.conf            # Listen directives
├── sites-available/      # Virtual host configs (inactive)
├── sites-enabled/        # Symlinks to active vhosts
├── mods-available/       # Available modules
├── mods-enabled/         # Active modules (symlinks)
├── conf-available/       # Additional config fragments
└── conf-enabled/         # Active config fragments
```

### RHEL/CentOS

```
/etc/httpd/
├── conf/httpd.conf       # Main config
├── conf.d/               # Additional configs (*.conf auto-loaded)
└── modules/              # Module .so files
```

### Content and Logs

```
/var/www/html/            # Default document root
/var/log/apache2/         # Debian logs
/var/log/httpd/           # RHEL logs
```

---

## Virtual Hosts

Virtual hosts let one Apache instance serve multiple websites:

### Name-Based Virtual Host

```apache
# /etc/apache2/sites-available/mysite.conf

<VirtualHost *:80>
    ServerName mysite.com
    ServerAlias www.mysite.com
    DocumentRoot /var/www/mysite
    
    <Directory /var/www/mysite>
        AllowOverride All
        Require all granted
    </Directory>
    
    ErrorLog ${APACHE_LOG_DIR}/mysite-error.log
    CustomLog ${APACHE_LOG_DIR}/mysite-access.log combined
</VirtualHost>
```

Enable and reload:
```bash
sudo a2ensite mysite.conf
sudo systemctl reload apache2
```

### HTTPS Virtual Host

```apache
<VirtualHost *:443>
    ServerName mysite.com
    DocumentRoot /var/www/mysite
    
    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/mysite.com/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/mysite.com/privkey.pem
    
    <Directory /var/www/mysite>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>

# Redirect HTTP to HTTPS
<VirtualHost *:80>
    ServerName mysite.com
    Redirect permanent / https://mysite.com/
</VirtualHost>
```

---

## Modules

Apache's power comes from its modular architecture:

```bash
# List loaded modules
apache2ctl -M

# Enable/disable modules (Debian)
sudo a2enmod rewrite
sudo a2enmod ssl
sudo a2enmod proxy proxy_http
sudo a2dismod autoindex
sudo systemctl restart apache2
```

### Essential Modules

| Module | Purpose |
|--------|---------|
| `mod_rewrite` | URL rewriting (clean URLs, redirects) |
| `mod_ssl` | HTTPS/TLS support |
| `mod_proxy` | Reverse proxy and load balancing |
| `mod_headers` | Set/modify HTTP headers |
| `mod_deflate` | Gzip compression |
| `mod_expires` | Cache control headers |
| `mod_security` | Web application firewall (WAF) |
| `mod_php` | In-process PHP execution |

---

## URL Rewriting (mod_rewrite)

### In Virtual Host or .htaccess

```apache
RewriteEngine On

# Force HTTPS
RewriteCond %{HTTPS} off
RewriteRule ^ https://%{HTTP_HOST}%{REQUEST_URI} [L,R=301]

# Remove trailing slash
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^(.+)/$ /$1 [L,R=301]

# Clean URLs (front controller pattern)
RewriteCond %{REQUEST_FILENAME} !-f
RewriteCond %{REQUEST_FILENAME} !-d
RewriteRule ^ index.php [L]

# Redirect old URL to new
RewriteRule ^old-page$ /new-page [R=301,L]
```

### RewriteRule Flags

| Flag | Meaning |
|------|---------|
| `[L]` | Last rule — stop processing |
| `[R=301]` | Redirect with status code |
| `[NC]` | Case-insensitive |
| `[QSA]` | Append query string |
| `[P]` | Proxy (internal redirect to another server) |
| `[F]` | Forbidden (403) |

---

## .htaccess

Per-directory configuration file (requires `AllowOverride All`):

```apache
# Enable rewrite
RewriteEngine On

# Custom error pages
ErrorDocument 404 /404.html
ErrorDocument 500 /500.html

# Block IP addresses
Require not ip 192.168.1.100
Require not ip 10.0.0.0/8

# Password protection
AuthType Basic
AuthName "Restricted Area"
AuthUserFile /etc/apache2/.htpasswd
Require valid-user

# Disable directory listing
Options -Indexes

# Set headers
Header set X-Content-Type-Options "nosniff"
Header set X-Frame-Options "SAMEORIGIN"
```

> **Note:** `.htaccess` is read on every request — for performance-critical setups, put directives in the virtual host config instead.

---

## Reverse Proxy

```apache
# Enable modules
# sudo a2enmod proxy proxy_http proxy_balancer lbmethod_byrequests

<VirtualHost *:80>
    ServerName app.example.com
    
    ProxyPreserveHost On
    ProxyPass / http://localhost:3000/
    ProxyPassReverse / http://localhost:3000/
    
    # WebSocket support
    RewriteEngine On
    RewriteCond %{HTTP:Upgrade} websocket [NC]
    RewriteCond %{HTTP:Connection} upgrade [NC]
    RewriteRule ^/?(.*) "ws://localhost:3000/$1" [P,L]
</VirtualHost>

# Load balancer
<Proxy "balancer://backend">
    BalancerMember http://127.0.0.1:3001
    BalancerMember http://127.0.0.1:3002
    BalancerMember http://127.0.0.1:3003
    ProxySet lbmethod=byrequests
</Proxy>

ProxyPass / balancer://backend/
ProxyPassReverse / balancer://backend/
```

---

## Security Hardening

```apache
# Hide server version
ServerTokens Prod
ServerSignature Off

# Security headers
Header always set X-Content-Type-Options "nosniff"
Header always set X-Frame-Options "DENY"
Header always set X-XSS-Protection "1; mode=block"
Header always set Strict-Transport-Security "max-age=31536000; includeSubDomains"
Header always set Referrer-Policy "strict-origin-when-cross-origin"

# Disable TRACE method
TraceEnable Off

# Restrict request size
LimitRequestBody 10485760    # 10MB max upload
```

---

## Management Commands

```bash
# Test configuration
apache2ctl configtest          # or: httpd -t

# Graceful restart (no dropped connections)
sudo systemctl reload apache2

# Full restart
sudo systemctl restart apache2

# View active virtual hosts
apache2ctl -S

# View loaded modules
apache2ctl -M
```

---

## Key Takeaways

1. **Apache is modular** — enable only the modules you need
2. **Virtual hosts** let one server handle multiple domains
3. **`mod_rewrite`** is powerful but complex — test rules carefully
4. **`.htaccess` is flexible but slow** — prefer virtual host config for production
5. **Always test config** with `apache2ctl configtest` before reloading
6. **Harden by default** — hide version, set security headers, disable TRACE
7. **Use `mod_proxy`** for reverse proxying to application servers
