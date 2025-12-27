# HTTPS Deployment Guide

Production-ready HTTPS configuration patterns for Node.js applications.

## Table of Contents

- [About the "Not Secure" Warning](#about-the-not-secure-warning)
- [Solution 1: Reverse Proxy with SSL (Recommended)](#solution-1-reverse-proxy-with-ssl-recommended)
- [Solution 2: Node.js Direct SSL](#solution-2-nodejs-direct-ssl)
- [Let's Encrypt with Certbot](#lets-encrypt-with-certbot)
- [Development Environment](#development-environment)
- [Troubleshooting](#troubleshooting)

---

## About the "Not Secure" Warning

The **"Not secure"** warning in your browser appears because the application is running on HTTP (not HTTPS). This is a **deployment configuration issue, not a code issue**.

### Why This Happens

- Browser requests go to `http://` instead of `https://`
- No SSL/TLS certificate is configured for the domain
- The connection is unencrypted, so browsers warn users

### Key Understanding

**The application code itself doesn't handle HTTPS** - this is typically done at the infrastructure level with:

- A **load balancer** (AWS ALB, GCP Load Balancer, etc.)
- A **reverse proxy** (nginx, Traefik, Caddy)
- A **PaaS platform** (Heroku, Render, Railway - handle it automatically)

### Solutions Overview

| Approach | Best For | Complexity |
|----------|----------|------------|
| Reverse Proxy (nginx) | Production self-hosted | Medium |
| Reverse Proxy (Caddy) | Simple auto-HTTPS | Low |
| Reverse Proxy (Traefik) | Docker/Kubernetes | Medium |
| Cloud Load Balancer | AWS/GCP/Azure | Low-Medium |
| Node.js Direct SSL | Edge cases only | High |

---

## Solution 1: Reverse Proxy with SSL (Recommended)

### Why Reverse Proxy?

- **Separation of concerns**: App handles business logic, proxy handles TLS
- **Certificate management**: Easier renewal without app restarts
- **Performance**: Optimized TLS termination
- **Security**: Additional layer of protection

### nginx Configuration

```nginx
# /etc/nginx/sites-available/myapp
server {
    listen 80;
    server_name example.com www.example.com;

    # Redirect all HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name example.com www.example.com;

    # SSL Certificate paths (Let's Encrypt)
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_stapling on;
    ssl_stapling_verify on;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options nosniff;
    add_header X-Frame-Options DENY;

    # Proxy to Node.js app
    location / {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }
}
```

### Traefik Configuration (Docker)

```yaml
# docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v2.10
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - letsencrypt:/letsencrypt

  app:
    image: myapp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.app.rule=Host(`example.com`)"
      - "traefik.http.routers.app.entrypoints=websecure"
      - "traefik.http.routers.app.tls.certresolver=letsencrypt"
      # HTTP to HTTPS redirect
      - "traefik.http.routers.app-http.rule=Host(`example.com`)"
      - "traefik.http.routers.app-http.entrypoints=web"
      - "traefik.http.routers.app-http.middlewares=redirect-to-https"
      - "traefik.http.middlewares.redirect-to-https.redirectscheme.scheme=https"

volumes:
  letsencrypt:
```

### Caddy (Automatic HTTPS)

Caddy automatically obtains and renews SSL certificates:

```
# Caddyfile
example.com {
    reverse_proxy localhost:3000
}
```

That's it! Caddy handles everything automatically.

---

## Solution 2: Node.js Direct SSL

**Not recommended for production** - use only when you cannot use a reverse proxy.

### Express.js with HTTPS

```javascript
const https = require('https');
const fs = require('fs');
const express = require('express');

const app = express();

const options = {
  key: fs.readFileSync('/path/to/private.key'),
  cert: fs.readFileSync('/path/to/certificate.crt'),
  ca: fs.readFileSync('/path/to/ca-bundle.crt') // if applicable
};

// Your Express routes
app.get('/', (req, res) => {
  res.send('Hello HTTPS!');
});

https.createServer(options, app).listen(443, () => {
  console.log('HTTPS server running on port 443');
});

// Optional: Redirect HTTP to HTTPS
const http = require('http');
http.createServer((req, res) => {
  res.writeHead(301, { Location: `https://${req.headers.host}${req.url}` });
  res.end();
}).listen(80);
```

### Fastify with HTTPS

```javascript
const fastify = require('fastify')({
  https: {
    key: fs.readFileSync('/path/to/private.key'),
    cert: fs.readFileSync('/path/to/certificate.crt')
  }
});

fastify.get('/', async (request, reply) => {
  return { hello: 'world' };
});

fastify.listen({ port: 443, host: '0.0.0.0' });
```

### Why This Is Not Recommended

- **Certificate renewal** requires app restart
- **Port 443** requires root privileges (or capabilities)
- **No separation of concerns** - app handles both business logic and TLS
- **Performance** - Node.js TLS is slower than nginx/proxy

---

## Let's Encrypt with Certbot

### Initial Certificate Setup

```bash
# Install certbot
sudo apt update
sudo apt install certbot python3-certbot-nginx

# Obtain certificate (nginx plugin)
sudo certbot --nginx -d example.com -d www.example.com

# Or standalone (stops web server temporarily)
sudo certbot certonly --standalone -d example.com
```

### Automatic Renewal

```bash
# Test renewal
sudo certbot renew --dry-run

# Certbot automatically adds a cron job/systemd timer
# Check with:
systemctl list-timers | grep certbot
```

### Docker with Certbot

```yaml
# docker-compose.yml
services:
  certbot:
    image: certbot/certbot
    volumes:
      - ./certbot/conf:/etc/letsencrypt
      - ./certbot/www:/var/www/certbot
    entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./certbot/conf:/etc/letsencrypt:ro
      - ./certbot/www:/var/www/certbot:ro
    depends_on:
      - app
```

### Common Certbot Issues

**"No renewals were attempted"**
```bash
# Usually means entrypoint is overridden in Docker
# Fix: Use proper entrypoint that runs certbot
```

**Rate Limits**
- 50 certificates per domain per week
- Use `--staging` flag for testing

---

## Development Environment

### Self-Signed Certificates (Local Dev Only)

```bash
# Generate self-signed certificate
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout localhost.key \
  -out localhost.crt \
  -subj "/CN=localhost"
```

### mkcert (Recommended for Local Dev)

```bash
# Install mkcert
brew install mkcert  # macOS
# or
sudo apt install mkcert  # Linux

# Install local CA
mkcert -install

# Generate certificates
mkcert localhost 127.0.0.1 ::1
# Creates localhost+2.pem and localhost+2-key.pem
```

### Vite/Webpack Dev Server

```javascript
// vite.config.js
import fs from 'fs';

export default {
  server: {
    https: {
      key: fs.readFileSync('./localhost-key.pem'),
      cert: fs.readFileSync('./localhost.pem'),
    },
  },
};
```

---

## Troubleshooting

### Browser Still Shows "Not Secure"

1. **Check URL**: Ensure you're accessing `https://` not `http://`
2. **Mixed Content**: Check browser console for HTTP resources loaded on HTTPS page
3. **Certificate Issues**: Click the warning to see certificate details
4. **HSTS**: If previously visited over HTTP, clear browser cache

### Certificate Errors

| Error | Cause | Solution |
|-------|-------|----------|
| `ERR_CERT_AUTHORITY_INVALID` | Self-signed or untrusted CA | Use Let's Encrypt or install CA |
| `ERR_CERT_DATE_INVALID` | Expired certificate | Renew with certbot |
| `ERR_CERT_COMMON_NAME_INVALID` | Wrong domain in cert | Reissue certificate for correct domain |
| `SSL_ERROR_RX_RECORD_TOO_LONG` | HTTPS on HTTP port | Check port configuration |

### nginx SSL Issues

```bash
# Test nginx configuration
sudo nginx -t

# Check certificate dates
openssl x509 -in /etc/letsencrypt/live/example.com/cert.pem -noout -dates

# Test SSL configuration
openssl s_client -connect example.com:443 -servername example.com
```

### Force HTTPS in Application

Even with proxy handling TLS, ensure your app knows it's behind HTTPS:

```javascript
// Express - trust proxy
app.set('trust proxy', 1);

// Check X-Forwarded-Proto header
app.use((req, res, next) => {
  if (req.headers['x-forwarded-proto'] !== 'https') {
    return res.redirect(`https://${req.headers.host}${req.url}`);
  }
  next();
});
```

---

## Quick Reference

### Minimum Production Checklist

- [ ] SSL certificate installed and valid
- [ ] HTTP redirects to HTTPS (301)
- [ ] HSTS header enabled
- [ ] TLS 1.2+ only (disable TLS 1.0/1.1)
- [ ] Strong cipher suites configured
- [ ] Certificate auto-renewal configured
- [ ] Mixed content eliminated

### Recommended Architecture

```
Internet → Load Balancer/CDN (TLS termination)
                    ↓
              nginx/Traefik (optional second proxy)
                    ↓
              Node.js App (HTTP internally)
```

The application runs on HTTP internally while all external traffic is encrypted.
