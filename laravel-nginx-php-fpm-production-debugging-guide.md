# Laravel + Nginx + PHP-FPM Production Debugging Guide
## Handling Multi-Version PHP Failures, 500 Errors, and 502 Bad Gateway Issues

---

## 1. Overview

Modern Laravel deployments on Ubuntu servers often fail not because of Laravel itself, but due to inconsistencies in the runtime stack:

- Nginx (web server)
- PHP-FPM (FastCGI process manager)
- PHP CLI (command-line interface)
- PHP extensions (mbstring, etc.)
- Multiple installed PHP versions

This document describes how to diagnose and resolve a real-world failure scenario involving:

- Laravel 500 Internal Server Error
- PHP fatal error: `Call to undefined function Illuminate\Support\mb_split()`
- Nginx 502 Bad Gateway errors
- Multi-version PHP-FPM conflict (8.2, 8.3, 8.4)

---

## 2. Core Problem Category

These failures typically fall into **three systemic classes**:

### 2.1 PHP Version Mismatch (CLI vs FPM)
PHP CLI may differ from PHP-FPM version serving web requests.

Example:
- CLI → PHP 8.4
- FPM → PHP 8.2 or 8.3

This causes:
- Missing extensions at runtime
- Inconsistent function availability
- Random Laravel bootstrap failures

---

### 2.2 Multi-PHP-FPM Conflict

Having multiple active PHP-FPM services leads to:

- Multiple sockets (`/run/php/php8.2-fpm.sock`, `php8.3-fpm.sock`)
- Nginx pointing to the wrong socket
- Unpredictable request routing

This results in:
- 502 Bad Gateway (broken socket)
- intermittent Laravel crashes

---

### 2.3 Nginx Configuration Drift

Nginx configurations often contain:

- multiple site blocks (`default`, `api`, etc.)
- outdated PHP socket references (8.2, 7.4)
- partially commented PHP location blocks

Even one wrong line causes full service failure.

---

## 3. Failure Symptoms and Interpretation

### 3.1 Laravel 500 Internal Server Error

Usually indicates:
- PHP fatal error during bootstrap
- missing extension in FPM
- inconsistent runtime environment

Common signature: Call to undefined function Illuminate\Support\mb_split()


This is often misinterpreted as missing mbstring, but is usually deeper:
- wrong PHP version
- cached vendor mismatch
- FPM inconsistency

---

### 3.2 Nginx 502 Bad Gateway

Means:
- Nginx cannot reach PHP-FPM socket

Root causes:
- PHP-FPM stopped
- wrong socket path in Nginx
- PHP-FPM version mismatch

Example: fastcgi_pass unix:/run/php/php8.2-fpm.sock;

while PHP 8.2 is not running.

---

### 3.3 Nginx "unexpected end of file"

Caused by:
- missing closing `}` in config
- commented-out structural blocks
- broken server block nesting

---

## 4. Diagnostic Strategy

### Step 1 — Identify Running PHP-FPM Versions

```bash
ps aux | grep php-fpm
```

Look for:
- multiple versions running simultaneously
- unintended active PHP-FPM pools

### Step 2 — Verify Active Socket

```bash
ls -l /run/php/
```
This shows which PHP-FPM versions are actually available.

### Step 3 — Validate Nginx Routing

```bash
grep -R "fastcgi_pass" -n /etc/nginx/sites-enabled/
```
Ensure all sites point to ONE correct socket.

### Step 4 — Confirm FPM Service Status

```bash
systemctl status php8.3-fpm
```

Ensure:
- Active: running
- No crashes or restart loops

### Step 5 — Validate Nginx Configuration
```bash
nginx -t
```
This catches:
- missing braces
- syntax errors
- broken includes

## 5. Resolution Principles
### 5.1 Single PHP Version Principle

Only ONE PHP-FPM version should be active:

Recommended:

- PHP 8.3 FPM (stable for Laravel 12)

Disable others:
```bash
systemctl stop php8.2-fpm
systemctl disable php8.2-fpm
```

### 5.2 Unified Socket Routing

Nginx must point to:
- /run/php/php8.3-fpm.sock

No exceptions.

### 5.3 Consistent Extension Layer

Ensure PHP-FPM has required extensions:
- mbstring
- openssl
- pdo
- tokenizer

Install per version:
```bash
apt install php8.3-mbstring
```

### 5.4 Clear Laravel Cache After Runtime Changes

Always clear cached state after system-level changes:

```bash
php artisan config:clear
php artisan cache:clear
php artisan route:clear
php artisan view:clear
rm -rf bootstrap/cache/*
```

## 6. Common Root Causes Summary

| Issue                | Root Cause                                 |
| -------------------- | ------------------------------------------ |
| 500 error            | PHP runtime mismatch or missing extension  |
| 502 error            | Wrong PHP-FPM socket or stopped service    |
| mb_split fatal error | version mismatch or broken runtime binding |
| nginx crash          | missing brace or malformed config          |
| random behavior      | multiple PHP-FPM versions active           |

## 7. Production Best Practices
- Use only one PHP version in production
- Disable unused PHP-FPM services
- Always verify socket path in Nginx
- Run nginx -t before reload
- Align CLI + FPM versions
- Avoid mixing PHP 8.2 / 8.3 / 8.4 on same app server
- Treat /run/php/*.sock as authoritative runtime mapping

## 8. Key Insight

Most Laravel production failures are NOT application bugs.
They are caused by: *Runtime inconsistency between Nginx, PHP-FPM, and installed PHP extensions.*
Fixing the runtime layer resolves most “Laravel issues” instantly.

## 9. Final Recommendation

For stability:
- Standardize on PHP 8.3
- Remove other PHP-FPM versions
- Keep a single source of truth for Nginx socket routing
- Treat server configuration as part of application integrity
