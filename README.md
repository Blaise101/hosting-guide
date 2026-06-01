# 📘 FULL STACK DEPLOYMENT GUIDE (React + Laravel + Namecheap + DigitalOcean + Vercel)

### 🚀 Overview

This guide explains how to deploy a full-stack application:

- Frontend: React (Vite / CRA)
- Backend: Laravel API
- Database: MySQL
- Domain: Namecheap
- Frontend Hosting: Vercel
- Backend Hosting: DigitalOcean (Ubuntu VPS)
- CI/CD: GitHub auto deployment (optional later)

## 🧱 ARCHITECTURE

```bash
  Frontend (React)
      ↓
  Vercel
      ↓
  yourdomain.com / www.yourdomain.com
  
  Backend (Laravel API)
      ↓
  DigitalOcean VPS
      ↓
  api.yourdomain.com
  
  Database
      ↓
  MySQL (on VPS)
```

## 🪜 STEP 1: DOMAIN SETUP (Namecheap)

### 1. Buy domain
- Go to: https://www.namecheap.com
- Purchase domain (e.g. example.com)

### 1. DNS Configuration

Go to: ```Domain List → Manage → Advanced DNS``` </br>
Add records: </br>
**Frontend (Vercel)** </br>
Vercel will provide DNS records like:
```bash
A / CNAME records for:
example.com
www.example.com
```
**Backend (DigitalOcean)**
```bash
Type: A
Host: api
Value: YOUR_DROPLET_IP
```

## STEP 2: FRONTEND DEPLOYMENT (React → Vercel)

### 1. Push frontend to GitHub
```bash
git add .
git commit -m "deploy frontend"
git push origin main
```

### 2. Deploy on Vercel
- Go to https://vercel.com
- Import GitHub repository
- Click Deploy
  
### 3. Add custom domain
Inside Vercel:
```bash
Settings → Domains
Add:
example.com
www.example.com
```

### 4. Result
```bash
https://example.com → React app live
```

## 🪜 STEP 3: CREATE DIGITALOCEAN SERVER (Backend)

### 1. Create Droplet
- Ubuntu 24.04
- 1GB RAM minimum
- Choose region closest to users

### 2. Connect via SSH
```bash
ssh root@YOUR_SERVER_IP
```

### 3. Update server 
```bash
apt update && apt upgrade -y
```
## 🪜 STEP 4: INSTALL SERVER DEPENDENCIES

```bash
apt install nginx -y
apt install mysql-server -y
apt install git -y
```

**Install PHP**
```bash
apt install software-properties-common -y
add-apt-repository ppa:ondrej/php -y
apt update

apt install php8.2 php8.2-fpm php8.2-mysql php8.2-cli php8.2-xml php8.2-mbstring php8.2-curl php8.2-zip unzip -y
```

**Install Composer**
```bash
curl -sS https://getcomposer.org/installer | php
mv composer.phar /usr/local/bin/composer
```

## 🪜 STEP 5: SETUP DATABASE
```bash
mysql
```

**Inside MySQL**

```SQL
CREATE DATABASE laravel_app;

CREATE USER 'laravel_user'@'localhost' IDENTIFIED BY 'password';

GRANT ALL PRIVILEGES ON laravel_app.* TO 'laravel_user'@'localhost';

FLUSH PRIVILEGES;
```

## 🪜 STEP 6: DEPLOY LARAVEL PROJECT

### 1. Clone repo 
```bash
cd /var/www
mkdir api
cd api

git clone git@github.com:USERNAME/REPO.git .
```

###  2. Install dependencies
```bash
composer install
```

###  3. Setup environment
```bash
cp .env.example .env
nano .env
```

**Update**
```bash
APP_URL=https://api.yourdomain.com

DB_DATABASE=laravel_app
DB_USERNAME=laravel_user
DB_PASSWORD=your_password
```

to go out of .env ```Ctrl+O  →  Enter  →  Ctrl+X```

###  3. Generate Key
```bash
php artisan key:generate
```

###  4. Run migrations
```bash
php artisan migrate
```

## 🪜 STEP 7: NGINX CONFIGURATION
```bash
nano /etc/nginx/sites-available/api
```

Paste: 
```Nginx
server {
    listen 80;
    server_name api.yourdomain.com;

    root /var/www/api/public;

    index index.php index.html;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php8.2-fpm.sock;
    }
}
```
Enable:
```bash
ln -s /etc/nginx/sites-available/api /etc/nginx/sites-enabled/
nginx -t
systemctl reload nginx
```

## 🪜 STEP 8: DOMAIN CONNECTION (API)

In Namecheap DNS:
```
Type: A
Host: api
Value: YOUR_SERVER_IP
```

Result:
```
http://api.example.com → Laravel API
```

## 🪜 STEP 9: SSL (HTTPS - IMPORTANT)

Install Certbot:
```Bash
apt install certbot python3-certbot-nginx -y
```

Run:
```Bash
certbot --nginx -d api.example.com
```

Now ```https://api.example.com``` is secure.

## 🪜 STEP 10: CONNECT FRONTEND TO BACKEND

In React:
```JavaScript
const API_URL = "https://api.example.com";
```

Then:
```Bash
git add .
git commit -m "update api url"
git push
```

Vercel auto deploys.

## ✅ FINAL RESULT

```bash
Frontend: https://example.com
Backend: https://api.example.com
Database: MySQL on VPS
Auto deploy: GitHub → Vercel (Frontend)
Manual/optional: GitHub → VPS (Backend later)
```

## 🧠 NOTES (IMPORTANT LESSONS)

- DNS propagation may take 1–48 hours
- SSL issues usually come from misconfigured domains or propagation delay
- Always separate:
   - frontend domain
   - API subdomain
- Use /var/www/api for backend root
- Never expose .env publicly

