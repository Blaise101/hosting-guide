# Changing a Laravel API Domain on DigitalOcean Without Affecting the Existing Server

## Overview

This guide explains how to change the domain name of a Laravel API hosted on a DigitalOcean droplet without creating a new server, moving databases, redeploying the application, or affecting any existing functionality.

This procedure is useful when:

* The project is being rebranded.
* The client purchased a new domain.
* The application, database, and infrastructure remain unchanged.
* Only the domain name changes.

### Example

Old API Domain:

```text
api.old.com
```

New API Domain:

```text
api.new.com
```

The Laravel application, database, droplet, and source code remain exactly the same.

---

# Architecture Before Change

```text
Namecheap Domain
        |
        v
api.old.com
        |
        v
DigitalOcean Droplet
        |
        v
Laravel API
        |
        v
MySQL Database
```

---

# Architecture After Change

```text
Namecheap Domain
        |
        v
api.new.com
        |
        v
DigitalOcean Droplet
        |
        v
Laravel API
        |
        v
MySQL Database
```

No changes are made to:

* Laravel codebase
* Database
* Storage
* Queues
* Cron jobs
* GitHub repository
* Droplet configuration

---

# Step 1: Purchase the New Domain

Purchase the new domain from your preferred registrar (e.g., Namecheap).

Example:

```text
new.com
```

---

# Step 2: Create DNS Records

Open the DNS settings for the new domain.

Create an A Record for the API subdomain:

```text
Type: A
Host: api
Value: <DROPLET_PUBLIC_IP>
TTL: Automatic
```

Example:

```text
Type: A
Host: api
Value: 167.172.191.72
```

Save changes.

Allow DNS propagation.

---

# Step 3: Verify DNS Resolution

On your local machine:

```bash
nslookup api.new.com
```

or

```bash
ping api.new.com
```

The domain should resolve to the droplet's IP address.

Example:

```text
167.172.191.72
```

---

# Step 4: Update Laravel Environment

SSH into the server:

```bash
ssh root@SERVER_IP
```

Navigate to the Laravel project:

```bash
cd /var/www/api
```

Open the environment file:

```bash
nano .env
```

Find:

```env
APP_URL=https://api.old.com
```

Replace with:

```env
APP_URL=https://api.new.com
```

Save the file.

---

# Step 5: Update Nginx Configuration

Open the Nginx configuration:

```bash
nano /etc/nginx/sites-available/api
```

Replace the old domain:

```nginx
server_name api.pld.com;
```

with:

```nginx
server_name api.new.com;
```

---

# Recommended Nginx Configuration

```nginx
server {
    listen 80;
    server_name api.new.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name api.new.com;

    root /var/www/api/public;
    index index.php index.html;

    ssl_certificate /etc/letsencrypt/live/api.new.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.new.com/privkey.pem;

    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
    }

    location ~ /\.ht {
        deny all;
    }
}
```

---

# Step 6: Test Nginx

Before reloading Nginx:

```bash
nginx -t
```

Expected output:

```text
syntax is ok
test is successful
```

If errors appear, fix them before continuing.

---

# Step 7: Reload Nginx

Apply the configuration:

```bash
systemctl reload nginx
```

---

# Step 8: Generate SSL Certificate

Request a certificate for the new domain:

```bash
certbot --nginx -d api.new.com
```

Certbot will:

* Generate SSL certificates
* Configure HTTPS
* Configure automatic renewal

Verify:

```bash
https://api.new.com
```

---

# Step 9: Clear Laravel Cache

Navigate to the Laravel project:

```bash
cd /var/www/api
```

Run:

```bash
php artisan optimize:clear
php artisan optimize
```

This ensures Laravel uses the updated environment variables.

---

# Step 10: Update Frontend API URLs

If the frontend communicates with the API using a hardcoded URL, update it.

Old:

```javascript
https://api.old.com/api
```

New:

```javascript
https://api.new.com/api
```

Redeploy the frontend after the change.

---

# Verification Checklist

Verify the following:

* DNS resolves correctly.
* SSL certificate is valid.
* API endpoints return expected responses.
* Authentication works.
* Database operations work.
* File uploads work.
* Frontend communicates with API successfully.
* No browser certificate warnings appear.

---

# Common Mistakes

## Wrong SSL Certificate Path

Incorrect:

```nginx
ssl_certificate /etc/letsencrypt/live/api.old.com/fullchain.pem;
```

Correct:

```nginx
ssl_certificate /etc/letsencrypt/live/api.new.com/fullchain.pem;
```

---

## Forgetting to Update APP_URL

Incorrect:

```env
APP_URL=https://api.old.com
```

Correct:

```env
APP_URL=https://api.new.com
```

---

## Forgetting to Clear Laravel Cache

Always run:

```bash
php artisan optimize:clear
php artisan optimize
```

after modifying `.env`.

---

# Conclusion

Changing the API domain does not require:

* A new DigitalOcean droplet
* A new database
* A new Laravel deployment
* Code migration

Only DNS records, Nginx configuration, SSL certificates, and application URLs need to be updated. The entire Laravel application continues running on the same infrastructure with no changes to its data or functionality.
