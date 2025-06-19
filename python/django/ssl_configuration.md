# SSL Setup for Django with Nginx (Internal Use)

## Prerequisites
- Root/sudo access
- Nginx installed and serving Django over HTTP
- Django properly configured (ALLOWED_HOSTS, etc.)
- Existing certificate files:
  - `yourdomain.com.key` (private key)
  - `yourdomain.com.txt.crt` (certificate)

## 1. Prepare Certificate Files
```bash
# Rename and organize certificates
mv tnpl.com.txt.crt tnpl.com.crt
sudo mkdir -p /etc/ssl/private/
sudo cp yourdomain.com.key /etc/ssl/private/
sudo cp yourdomain.com.crt /etc/ssl/private/

# Set permissions
sudo chmod 600 /etc/ssl/private/tnpl.com.key
sudo chmod 644 /etc/ssl/private/tnpl.com.crt
```
## 2.Configure Nginx

Edit /etc/nginx/sites-available/tnpl_epms

```nginx
server {
    listen 80;
    server_name epas.tnpl.com;
    return 301 https://$host$request_uri;
}

server {
       listen 443 ssl;
       server_name epas.tnpl.com;

       ssl_certificate /etc/ssl/private/tnpl.com.crt;
       ssl_certificate_key /etc/ssl/private/tnpl.com.key;

       # SSL configuration
       ssl_protocols TLSv1.2 TLSv1.3;
       ssl_prefer_server_ciphers on;
       ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
       ssl_ecdh_curve secp384r1;
       ssl_session_cache shared:SSL:10m;
       ssl_session_tickets off;
       ssl_stapling on;
       ssl_stapling_verify on;

       # Security headers
       add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload";
       add_header X-Content-Type-Options nosniff;
       add_header X-Frame-Options DENY;
       add_header X-XSS-Protection "1; mode=block";

       location = /favicon.ico {access_log off;log_not_found off;}

       location /static/ {
            alias /epas/gt-epms/static/;
       }

       location /media/ {
            alias /epas/gt-epms/media/;
       }
       location / {
            include proxy_params;
            proxy_pass http://unix:/epas/gt-epms/tnpl_epms.sock;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
        }
     }

```
## 3. Test and Restart Nginx

```bash
# Test Nginx configuration
sudo nginx -t

# If test succeeds, restart Nginx
sudo systemctl restart nginx```
