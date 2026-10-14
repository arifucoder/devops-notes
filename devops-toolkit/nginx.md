# Nginx Class Notes — SSH, Nginx, Reverse Proxy ও SSL

> এই নোটে সব ব্যাখ্যা বাংলায়, কিন্তু সব **command, config, file path ও technical term ইংরেজিতে** রাখা হয়েছে।
> ✅ চিহ্ন = সবচেয়ে গুরুত্বপূর্ণ / মনে রাখার মতো পয়েন্ট। ⚠️ চিহ্ন = সতর্কতা।

---

## সূচিপত্র

**PART 1 — Basics**
1. [SSH কী](#1-ssh-কী)
2. [Common Network Ports](#2-common-network-ports)
3. [Nginx পরিচিতি](#3-nginx-পরিচিতি)
4. [Nginx-এর প্রয়োজনীয় Command](#4-nginx-এর-প্রয়োজনীয়-command)
5. [Nginx-এর File Structure (conf.d vs sites-available)](#5-nginx-এর-file-structure)
6. [Server Block-এর প্রতিটি Directive](#6-server-block-এর-প্রতিটি-directive)

**PART 2 — Application Host করা (HTTP পর্যন্ত)**

7. [Static Site Host করা](#7-static-site-host-করা)
8. [Express.js Backend Host করা (PM2 + Nginx)](#8-expressjs-backend-host-করা-pm2--nginx)
9. [একই সার্ভারে একাধিক Application](#9-একই-সার্ভারে-একাধিক-application)
10. [Domain সেটআপ — A Record ও CNAME](#10-domain-সেটআপ--a-record-ও-cname)

**PART 3 — SSL কোনটা বেছে নেবেন?**

11. [SSL পদ্ধতিগুলোর তুলনা](#11-ssl-পদ্ধতিগুলোর-তুলনা)

**PART 4 — SSL Option A: Nginx + Certbot (Let's Encrypt)**

12. [`certbot --nginx` (সবচেয়ে সহজ)](#12-certbot---nginx-সবচেয়ে-সহজ)
13. [`certbot certonly --webroot` (zero downtime)](#13-certbot-certonly---webroot)
14. [`certbot certonly --standalone`](#14-certbot-certonly---standalone)
15. [Wildcard Certificate — DNS-01 Challenge](#15-wildcard-certificate--dns-01-challenge)
16. [Auto-Renewal](#16-auto-renewal)
17. [Production-ready Config (Hardening)](#17-production-ready-config-hardening)

**PART 5 — SSL Option B: Nginx + Cloudflare**

18. [Cloudflare কীভাবে কাজ করে](#18-cloudflare-কীভাবে-কাজ-করে)
19. [Cloudflare-এ Domain ও DNS সেটআপ](#19-cloudflare-এ-domain-ও-dns-সেটআপ)
20. [Cloudflare SSL Mode](#20-cloudflare-ssl-mode)
21. [Origin Certificate দিয়ে Nginx Config](#21-origin-certificate-দিয়ে-nginx-config)
22. [ব্যবহারকারীর আসল IP ফেরত আনা](#22-ব্যবহারকারীর-আসল-ip-ফেরত-আনা)
23. [Cloudflare-এর Extra Security](#23-cloudflare-এর-extra-security)

**PART 6 — SSL Option C: অন্যান্য পদ্ধতি**

24. [কেনা CA Certificate (DigiCert/Sectigo)](#24-কেনা-ca-certificate)
25. [Self-Signed Certificate (শুধু dev)](#25-self-signed-certificate)
26. [`acme.sh` (Certbot-এর বিকল্প)](#26-acmesh)

**PART 7 — Troubleshooting ও Reference**

27. [সাধারণ Error ও সমাধান](#27-সাধারণ-error-ও-সমাধান)
28. [এক নজরে মূল পয়েন্ট](#28-এক-নজরে-মূল-পয়েন্ট)
29. [Command Cheat Sheet](#29-command-cheat-sheet)

---
---

# PART 1 — Basics

## 1. SSH কী

**SSH = Secure Shell.** এটি একটি network protocol যার মাধ্যমে দূরের (remote) server-এ **encrypted ভাবে** connect করে command line access পাওয়া যায়।

**Definition:** SSH হলো client ও server-এর মধ্যে একটি নিরাপদ, encrypted tunnel — data plain text-এ যায় না, তাই মাঝপথে কেউ পড়তে পারে না।

```bash
# AWS EC2 — key file দিয়ে login
ssh -i my-key.pem ubuntu@13.229.45.10

# সাধারণ VPS — password দিয়ে login
ssh root@203.0.113.25
```

✅ Telnet বা plain FTP-এর বদলে সবসময় **SSH/SFTP** ব্যবহার করতে হবে।

---

## 2. Common Network Ports

| Port | Protocol | কাজ |
|------|----------|-----|
| **22** | SSH | Linux server-এ নিরাপদ command line access |
| **22** | SFTP | SSH-এর উপর দিয়ে encrypted file transfer |
| **21** | FTP | সাধারণ file transfer (unencrypted) |
| **80** | HTTP | অনিরাপদ web browsing |
| **443** | HTTPS | নিরাপদ ও encrypted web communication |

✅ **SFTP আলাদা port ব্যবহার করে না** — SSH-এর port **22**-ই ব্যবহার করে।

⚠️ FTP (21) password-সহ সব data plain text-এ পাঠায় — production-এ **SFTP** ব্যবহার করুন।

**বাস্তবে আরও যেসব port লাগে:**

| Port | Service |
|------|---------|
| 3306 | MySQL / MariaDB |
| 5432 | PostgreSQL |
| 27017 | MongoDB |
| 6379 | Redis |
| 3000 / 5000 / 8000 | Node.js, Django ইত্যাদির dev port |

✅ Database port **কখনোই public inbound rule-এ open করা যাবে না** — শুধু নিজের server বা VPC-এর ভেতর থেকে।

---

## 3. Nginx পরিচিতি

**Nginx** একই সাথে একটি **Web Server** এবং **Reverse Proxy**।

| ব্যবহার | ব্যাখ্যা |
|---------|----------|
| **Web Server** | HTML/CSS/JS-এর মতো static file serve করা |
| **Reverse Proxy** | বাইরের request ভেতরের app (Node/Django/Laravel)-এ পাঠানো |
| **Caching** | একই response জমিয়ে রেখে দ্রুত দেওয়া |
| **Multiple Site Hosting** | একই server-এ অনেক domain/site |
| **API Gateway** | একাধিক backend-কে একটি entry point-এর নিচে আনা |
| **Load Balancing** | Traffic একাধিক server-এ ভাগ করা |

**Definition — Reverse Proxy:** যে server client-এর request নিজে নেয়, পেছনের backend app-এ পাঠায়, উত্তর এনে client-কে দেয়। Client কখনো backend-কে সরাসরি দেখে না।

- `https://devsskills.com` → Nginx → `http://127.0.0.1:5000` (Express app)
- `https://devsskills.com/api` → Node backend, `https://devsskills.com/` → React build folder

✅ Reverse proxy থাকলে app-এর আসল port (5000, 3000) বাইরে থেকে ধরা যায় না — বড় security সুবিধা।

---

## 4. Nginx-এর প্রয়োজনীয় Command

```bash
sudo apt update
sudo apt install nginx -y          # install

systemctl status nginx             # status দেখা
sudo nginx -t                      # config test (reload-এর আগে বাধ্যতামূলক)
sudo systemctl reload nginx        # config update-এর পর
sudo systemctl restart nginx       # পুরো বন্ধ করে চালু
sudo systemctl enable nginx        # reboot-এর পর নিজে চালু হবে
```

✅ **`reload` vs `restart`:** `reload`-এ চলমান connection নষ্ট হয় না → **downtime নেই**। `restart`-এ কয়েক মুহূর্তের downtime হতে পারে।

✅ **নিয়ম:** আগে `sudo nginx -t`, তারপর `sudo systemctl reload nginx`। Test fail করলে **কখনোই reload নয়**।

---

## 5. Nginx-এর File Structure

| Path | কাজ |
|------|-----|
| `/etc/nginx/nginx.conf` | প্রধান config; এখান থেকে বাকি সব include হয় |
| `/etc/nginx/conf.d/` | নিজের `.conf` file রাখার folder (**আমরা এটি ব্যবহার করছি**) |
| `/etc/nginx/sites-available/` | Ubuntu পদ্ধতি — সব site-এর config |
| `/etc/nginx/sites-enabled/` | যে site গুলো **চালু**, তাদের symlink |
| `/var/www/` | Website ফাইল রাখার default জায়গা |
| `/var/log/nginx/access.log` | কে কখন কোন page চেয়েছে |
| `/var/log/nginx/error.log` | ✅ সমস্যা হলে **সবার আগে এখানে দেখুন** |

`nginx.conf`-এর ভেতরে Ubuntu-তে এই দুটি line থাকে:

```nginx
include /etc/nginx/conf.d/*.conf;
include /etc/nginx/sites-enabled/*;
```

### পদ্ধতি A — `conf.d` (আমাদের ক্লাসের পদ্ধতি)

```bash
sudo nano /etc/nginx/conf.d/devsskills.com.conf   # শেষে অবশ্যই .conf
sudo nginx -t
sudo systemctl reload nginx
```

### পদ্ধতি B — `sites-available` + symlink (Ubuntu পদ্ধতি)

```bash
sudo nano /etc/nginx/sites-available/devsskills.com
sudo ln -s /etc/nginx/sites-available/devsskills.com /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx

# site সাময়িক বন্ধ করতে
sudo unlink /etc/nginx/sites-enabled/devsskills.com
sudo nginx -t && sudo systemctl reload nginx
```

✅ দুটোর **যেকোনো একটি** বেছে নিন। মেশালে একই `server_name` দুবার আসে → `conflicting server name` warning।

### Default site সরানো

```bash
sudo rm /etc/nginx/sites-enabled/default
```

⚠️ Default site না সরালে server IP-তে গেলে **"Welcome to nginx!"** দেখাবে — কারণ এটি `default_server` হিসেবে সব request ধরে ফেলে।

(বিকল্প: `nginx.conf`-এ `include /etc/nginx/sites-enabled/*;` line টি `#` দিয়ে comment করা। তবে symlink মোছাই বেশি পরিষ্কার।)

---

## 6. Server Block-এর প্রতিটি Directive

```nginx
server {
    listen 80;
    server_name devsskills.com;
    root /var/www/site;
    index index.html;

    location / {
        proxy_pass http://127.0.0.1:5000;
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

| Directive | কাজ |
|-----------|-----|
| `server { }` | একটি website/domain-এর সব নিয়ম |
| `listen 80;` | কোন port-এ শুনবে (80 = HTTP, 443 = HTTPS) |
| `listen [::]:80;` | IPv6-এও শুনবে |
| `default_server` | কোনো `server_name` না মিললে এই block request ধরবে |
| `server_name` | কোন domain-এর জন্য; একাধিক হলে space দিয়ে। `_` = যেকোনো নাম |
| `root` | File কোন folder থেকে খুঁজবে |
| `index` | Folder চাইলে কোন file দেখাবে (বাম থেকে ডানে) |
| `location / { }` | কোন URL path-এ কোন নিয়ম |
| `try_files $uri $uri/ =404;` | আগে file, তারপর folder; না পেলে 404 |
| `proxy_pass` | ✅ Request backend-এ পাঠায় — reverse proxy-র মূল directive |
| `proxy_http_version 1.1;` | Backend-এর সাথে HTTP/1.1 (WebSocket-এর জন্য দরকার) |
| `Upgrade` / `Connection 'upgrade'` | WebSocket connection upgrade |
| `Host $host` | ✅ আসল domain নাম backend-কে জানায় |
| `X-Real-IP $remote_addr` | ✅ ব্যবহারকারীর আসল IP |
| `X-Forwarded-For` | একাধিক proxy পার হলে পুরো IP chain |
| `X-Forwarded-Proto $scheme` | ✅ Request http না https ছিল (redirect loop এড়াতে) |
| `proxy_cache_bypass $http_upgrade` | WebSocket request cache করবে না |

✅ `X-Real-IP` / `X-Forwarded-For` না দিলে backend log-এ সব IP দেখাবে `127.0.0.1` — rate limiting ও analytics ভুল হবে।

---
---

# PART 2 — Application Host করা (HTTP পর্যন্ত)

> এই অংশ শেষ হলে site **HTTP (port 80)**-এ চলবে। HTTPS-এর জন্য PART 4, 5 বা 6 দেখুন।

## 7. Static Site Host করা

**ধাপ ১ — ফাইল রাখা**

```bash
cd /var/www/
sudo git clone https://github.com/username/figmaland.git figmaland
```

✅ `/var/www/` ubuntu user-এর home নয়, তাই এখানে **`sudo` লাগে**।

**ধাপ ২ — Config লেখা**

```bash
sudo nano /etc/nginx/conf.d/figmaland.com.conf
```

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name figmaland.com www.figmaland.com;

    root /var/www/figmaland;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }

    # Image/CSS/JS browser-এ ১ বছর cache
    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
    }
}
```

> Domain না থাকলে শুধু IP দিয়ে test করতে: `listen 80 default_server;` এবং `server_name _;`

| Directive | ব্যাখ্যা |
|-----------|----------|
| `location ~* \.(...)$` | `~*` = case-insensitive regex; নির্দিষ্ট extension-এর জন্য নিয়ম |
| `expires 1y;` | Browser ১ বছর file জমা রাখবে |
| `Cache-Control "public, immutable"` | বারবার check করার দরকার নেই |

**ধাপ ৩ — Test, reload ও যাচাই**

```bash
sudo nginx -t
sudo systemctl reload nginx
curl -I http://localhost          # HTTP/1.1 200 OK আসা উচিত
```

✅ **403 Forbidden** এলে permission ঠিক করুন:

```bash
sudo chown -R www-data:www-data /var/www/figmaland
sudo chmod -R 755 /var/www/figmaland
```

---

## 8. Express.js Backend Host করা (PM2 + Nginx)

### ধাপ ১ — Node.js ও Yarn install

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
sudo npm install -g yarn

node -v && npm -v && yarn -v
```

### ধাপ ২ — Project clone ও install

```bash
cd ~
mkdir application && cd application
git clone https://github.com/username/my-api.git
cd my-api
yarn install          # অথবা npm install
```

✅ Home folder (`/home/ubuntu`)-এ **`sudo` লাগে না**।

### ধাপ ৩ — PM2 দিয়ে চালানো

**Definition — PM2:** Node.js-এর process manager। App background-এ চালায়, crash করলে restart করে, reboot-এর পরও চালু করতে পারে।

```bash
sudo npm install -g pm2

pm2 start index.js --name my-api
pm2 list
pm2 logs my-api

pm2 startup        # যে command দেখাবে সেটি copy করে চালান
pm2 save
```

✅ `pm2 save` না দিলে reboot-এর পর app চালু হবে না — খুব সাধারণ ভুল।

```bash
curl http://127.0.0.1:5000      # response এলে backend ঠিক আছে
```

### ধাপ ৪ — Reverse proxy config

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo nano /etc/nginx/conf.d/devsskills.com.conf
```

```nginx
server {
    listen 80;
    server_name devsskills.com www.devsskills.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    client_max_body_size 20M;    # upload limit (default মাত্র 1M)
}
```

✅ `server_name`-এ domain-এর বানান হুবহু ঠিক থাকতে হবে।

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### ধাপ ৫ — Firewall

- **AWS EC2** → Security Group → Inbound rules → **HTTP 80** ও **HTTPS 443**, Source: `0.0.0.0/0`
- **UFW** চালু থাকলে:

```bash
sudo ufw allow 'Nginx Full'     # 80 + 443
sudo ufw allow OpenSSH          # ⚠️ UFW চালুর আগে অবশ্যই, নাহলে SSH বন্ধ হয়ে যাবে
sudo ufw status
```

✅ Cloud Security Group আর UFW — **দুটো আলাদা firewall**, দুটোতেই খোলা থাকতে হবে।

---

## 9. একই সার্ভারে একাধিক Application

| পদ্ধতি | উদাহরণ | কখন |
|--------|---------|------|
| **Separate domain** | `sitea.com`, `siteb.com` | আলাদা client/product |
| **Subdomain** | `api.devsskills.com`, `admin.devsskills.com` | একই product-এর ভিন্ন অংশ |
| **Path-based** | `devsskills.com/api`, `/admin` | এক domain-এ সব |

ধরে নিই ৪টি app:

| # | App | ধরন | Port |
|---|-----|-----|------|
| 1 | Figmaland | Static HTML | — |
| 2 | Express API | Node backend | 5000 |
| 3 | Admin Panel | React SPA build | — |
| 4 | Next.js Blog | Node SSR | 3000 |

### 9.1 পদ্ধতি ১ — প্রতিটি domain/subdomain-এর আলাদা `.conf` file ✅

**App 1 — Static:** উপরের Section 7-এর config।

**App 2 — Express API** (`/etc/nginx/conf.d/api.devsskills.com.conf`):

```nginx
server {
    listen 80;
    server_name api.devsskills.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    client_max_body_size 20M;
}
```

**App 3 — React SPA** (`/etc/nginx/conf.d/admin.devsskills.com.conf`):

```nginx
server {
    listen 80;
    server_name admin.devsskills.com;

    root /var/www/admin/build;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;    # ✅ SPA-র সবচেয়ে জরুরি line
    }
}
```

✅ SPA-তে routing browser-এ হয়; server-এ `/dashboard` নামে কোনো file নেই। `/index.html` fallback না দিলে **refresh করলেই 404**।

**App 4 — Next.js** (`/etc/nginx/conf.d/blog.devsskills.com.conf`):

```nginx
server {
    listen 80;
    server_name blog.devsskills.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
    }

    location /_next/static/ {
        proxy_pass http://127.0.0.1:3000;
        expires 1y;
        access_log off;
    }
}
```

```bash
yarn build
pm2 start "yarn start" --name blog
pm2 save
```

### 9.2 পদ্ধতি ২ — একটি domain, আলাদা path

```nginx
server {
    listen 80;
    server_name devsskills.com www.devsskills.com;

    root /var/www/frontend/build;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location /api/ {
        proxy_pass http://127.0.0.1:5000/;      # ⚠️ শেষে / আছে
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /blog/ {
        proxy_pass http://127.0.0.1:3000;       # শেষে / নেই — ইচ্ছাকৃত
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

✅ **`proxy_pass`-এর শেষের `/` — সবচেয়ে বেশি ভুল এখানে:**

| Config | User চায় | Backend পায় |
|--------|-----------|--------------|
| `proxy_pass http://127.0.0.1:5000/;` (শেষে `/`) | `/api/users` | `/users` ← prefix কেটে যায় |
| `proxy_pass http://127.0.0.1:5000;` (`/` নেই) | `/api/users` | `/api/users` ← পুরো path |

Backend route `/users` হলে → `/` দিন। Backend route নিজেই `/api/users` হলে → `/` দেবেন না।

### 9.3 পদ্ধতি ৩ — Load Balancing

```nginx
upstream my_app {
    least_conn;
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
    server 127.0.0.1:3003;
}

server {
    listen 80;
    server_name app.devsskills.com;

    location / {
        proxy_pass http://my_app;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

| Directive | ব্যাখ্যা |
|-----------|----------|
| `upstream name { }` | একদল backend server-এর group |
| `least_conn;` | সবচেয়ে কম ব্যস্ত server-এ পাঠায় |
| `ip_hash;` | একই user সবসময় একই server-এ (session ধরে রাখতে) |

> PM2 cluster mode (`pm2 start index.js -i 3`) সব instance **একই port** share করে — তখন upstream লাগে না। আলাদা port-এ চালালে তবেই `upstream`।

### 9.4 সব app যোগের পর যাচাই

```bash
sudo nginx -t
sudo systemctl reload nginx

curl -I http://figmaland.com
curl -I http://api.devsskills.com
curl -I http://admin.devsskills.com
curl -I http://blog.devsskills.com
```

⚠️ একটি file-এ syntax error থাকলে `nginx -t` fail করে — তখন reload করলে **সব site একসাথে প্রভাবিত** হয়।

---

## 10. Domain সেটআপ — A Record ও CNAME

| Type | Host | Value | কাজ |
|------|------|-------|-----|
| **A** | `@` | `13.229.45.10` | মূল domain → server IP |
| **CNAME** | `www` | `devsskills.com` | `www` → মূল domain |
| **A** | `api` | `13.229.45.10` | subdomain → server IP |

**Definition — CNAME (Canonical Name):** একটি নামকে অন্য নামের **alias** বানায়, IP-তে নয়।

- `www.devsskills.com` → CNAME → `devsskills.com` → A → IP
- `blog.devsskills.com` → CNAME → `myblog.netlify.app`

✅ DNS propagation-এ কয়েক মিনিট থেকে কয়েক ঘণ্টা লাগতে পারে:

```bash
dig +short devsskills.com
nslookup devsskills.com
```

> Cloudflare ব্যবহার করলে DNS সেটআপ আলাদা — **Section 19** দেখুন।

---
---

# PART 3 — SSL কোনটা বেছে নেবেন?

## 11. SSL পদ্ধতিগুলোর তুলনা

| পদ্ধতি | খরচ | মেয়াদ | Renewal | কখন ব্যবহার করবেন |
|--------|------|--------|---------|-------------------|
| **A. Nginx + Certbot** (Let's Encrypt) | ফ্রি | ৯০ দিন | Automatic | ✅ সাধারণ VPS/EC2 — সবচেয়ে প্রচলিত |
| **B. Nginx + Cloudflare** (Origin Cert) | ফ্রি | ১৫ বছর পর্যন্ত | প্রায় লাগে না | ✅ CDN, DDoS protection, IP লুকাতে চাইলে |
| **C1. কেনা CA Certificate** | টাকা লাগে | ১ বছর-এর মতো | Manual | কোম্পানি policy / OV-EV cert দরকার হলে |
| **C2. Self-signed** | ফ্রি | যা খুশি | — | শুধু local/dev testing |
| **C3. acme.sh** | ফ্রি | ৯০ দিন | Automatic | Certbot-এর হালকা বিকল্প |

**দ্রুত সিদ্ধান্ত:**

- শুধু server আর domain আছে, সহজ চাই → **PART 4 (Certbot)**
- Domain-এর DNS Cloudflare-এ, orange cloud (proxy) চালু → **PART 5 (Cloudflare)**
- ⚠️ দুটো মেশানো যায়, কিন্তু Cloudflare proxy চালু থাকলে Certbot-এর `--nginx`/`--webroot` মাঝে মাঝে ঝামেলা করে — তখন Cloudflare Origin Cert বা DNS-01 challenge সহজ।

---
---

# PART 4 — SSL Option A: Nginx + Certbot (Let's Encrypt)

> **পূর্বশর্ত (সব Certbot পদ্ধতিতে):**
> 1. DNS-এর A record server-এর IP-তে point করছে (`dig +short domain.com` দিয়ে যাচাই)
> 2. Port **80** ও **443** inbound rules ও UFW-তে খোলা
> 3. Nginx-এ ওই domain-এর HTTP (port 80) config ঠিকমতো কাজ করছে
>
> ✅ Certbot port 80-এ validation file বসিয়ে domain-এর মালিকানা প্রমাণ করে — DNS ভুল বা port 80 বন্ধ থাকলে **fail** করবে (DNS-01 ছাড়া)।

## 12. `certbot --nginx` (সবচেয়ে সহজ)

### ধাপ ১ — Install

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### ধাপ ২ — Config-এর backup (Certbot নিজে file edit করে)

```bash
sudo cp /etc/nginx/conf.d/devsskills.com.conf ~/devsskills.com.conf.bak
```

### ধাপ ৩ — Certificate নেওয়া

```bash
sudo certbot --nginx -d devsskills.com -d www.devsskills.com
```

Certbot নিজে থেকে:
- Certificate আনে
- `.conf` file edit করে `listen 443 ssl;` ও certificate path যোগ করে
- HTTP → HTTPS redirect যোগ করে
- Auto-renewal timer চালু করে

### ধাপ ৪ — Certbot-এর পর config দেখতে যেমন হয়

```nginx
server {
    server_name devsskills.com www.devsskills.com;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    listen 443 ssl;                                                       # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/devsskills.com/fullchain.pem;   # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/devsskills.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf;                      # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;                        # managed by Certbot
}

server {
    if ($host = www.devsskills.com) {
        return 301 https://$host$request_uri;
    }
    if ($host = devsskills.com) {
        return 301 https://$host$request_uri;
    }
    listen 80;
    server_name devsskills.com www.devsskills.com;
    return 404;
}
```

| Directive | ব্যাখ্যা |
|-----------|----------|
| `listen 443 ssl;` | HTTPS port-এ SSL চালু |
| `ssl_certificate` | Public cert + intermediate chain (`fullchain.pem`) |
| `ssl_certificate_key` | ✅ Private key — **কখনো share বা git-এ commit নয়** |
| `include options-ssl-nginx.conf` | Let's Encrypt-এর নিরাপদ default settings |
| `ssl_dhparam` | Key exchange আরও শক্ত করে |
| `return 301 https://...` | ✅ HTTP-এর সব request স্থায়ীভাবে HTTPS-এ |

### ধাপ ৫ — যাচাই

```bash
curl -I https://devsskills.com
sudo certbot certificates
```

**একাধিক app-এর জন্য** — প্রতিটি domain আলাদা করে বা একসাথে:

```bash
sudo certbot --nginx -d figmaland.com -d www.figmaland.com
sudo certbot --nginx -d api.devsskills.com -d admin.devsskills.com -d blog.devsskills.com
```

---

## 13. `certbot certonly --webroot`

Nginx চলন্ত রেখে শুধু certificate আনে, **config-এ হাত দেয় না**। ✅ Live site-এর জন্য নিরাপদ — **কোনো downtime নেই**, আর config পুরোপুরি নিজের নিয়ন্ত্রণে থাকে।

### ধাপ ১ — Validation path যোগ করা

```nginx
server {
    listen 80;
    server_name devsskills.com www.devsskills.com;

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        proxy_pass http://127.0.0.1:5000;
    }
}
```

```bash
sudo mkdir -p /var/www/certbot
sudo nginx -t && sudo systemctl reload nginx
```

### ধাপ ২ — Certificate নেওয়া

```bash
sudo certbot certonly --webroot -w /var/www/certbot \
  -d devsskills.com -d www.devsskills.com
```

### ধাপ ৩ — Certificate path নিজে config-এ বসানো

Section 17-এর production config ব্যবহার করুন। Path হবে:

```
/etc/letsencrypt/live/devsskills.com/fullchain.pem
/etc/letsencrypt/live/devsskills.com/privkey.pem
```

✅ এই পদ্ধতিতে renewal-এর পর Nginx নিজে reload হয় না — Section 16-এর **deploy hook** অবশ্যই যোগ করুন।

---

## 14. `certbot certonly --standalone`

Certbot নিজেই সাময়িকভাবে port 80-এ ছোট web server চালায়।

```bash
sudo systemctl stop nginx           # ⚠️ port 80 খালি করতে হবে
sudo certbot certonly --standalone -d devsskills.com -d www.devsskills.com
sudo systemctl start nginx
```

⚠️ Nginx বন্ধ থাকায় **live site কিছুক্ষণ down থাকে**। শুধু প্রথমবার সেটআপে বা Nginx install-এর আগে ব্যবহার করুন। Renewal-এর সময়ও Nginx বন্ধ করতে হবে — তাই hook লাগবে:

```bash
sudo certbot renew --pre-hook "systemctl stop nginx" --post-hook "systemctl start nginx"
```

---

## 15. Wildcard Certificate — DNS-01 Challenge

**Definition — Wildcard Certificate:** একটি cert যা মূল domain-এর **সব subdomain** কভার করে (`*.devsskills.com`)।

- একটি cert দিয়েই `api.`, `admin.`, `blog.` সব কভার
- নতুন client-কে `client1.myapp.com` দিলে নতুন cert লাগে না

✅ Wildcard cert **শুধু DNS-01 challenge দিয়েই** পাওয়া যায়। এতে port 80 খোলা থাকারও দরকার নেই।

### 15.1 Manual পদ্ধতি

```bash
sudo certbot certonly --manual --preferred-challenges dns \
  -d "*.devsskills.com" -d devsskills.com
```

Certbot একটি value দেবে, domain panel-এ যোগ করুন:

| Type | Host | Value |
|------|------|-------|
| TXT | `_acme-challenge` | Certbot-এর দেওয়া string |

```bash
dig +short TXT _acme-challenge.devsskills.com    # record এসেছে কিনা দেখে তারপর Enter
```

⚠️ Manual পদ্ধতিতে **auto-renewal কাজ করে না** — প্রতি ৯০ দিনে হাতে করতে হয়।

### 15.2 Automatic পদ্ধতি (DNS যদি Cloudflare-এ থাকে) ✅

```bash
sudo apt install python3-certbot-dns-cloudflare -y

sudo mkdir -p /root/.secrets
sudo nano /root/.secrets/cloudflare.ini
```

```ini
dns_cloudflare_api_token = your_cloudflare_api_token_here
```

> Token বানানো: Cloudflare Dashboard → My Profile → API Tokens → Create Token → **"Edit zone DNS"** template → শুধু নিজের zone বেছে নিন।

```bash
sudo chmod 600 /root/.secrets/cloudflare.ini

sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  -d "*.devsskills.com" -d devsskills.com
```

✅ এভাবে নেওয়া wildcard cert **নিজে থেকেই renew** হয়।

> ✅ এই পদ্ধতি Cloudflare proxy (orange cloud) চালু থাকলেও কাজ করে — কারণ validation DNS দিয়ে হয়, HTTP দিয়ে নয়।

---

## 16. Auto-Renewal

Let's Encrypt cert **৯০ দিন** valid। Certbot install করলে Ubuntu-তে **systemd timer** নিজে তৈরি হয়।

```bash
systemctl status certbot.timer
systemctl list-timers | grep certbot

sudo certbot renew --dry-run      # আসল renew না করে পুরো প্রক্রিয়া পরীক্ষা
sudo certbot certificates         # মেয়াদ দেখা
sudo certbot renew                # হাতে renew
```

✅ SSL সেটআপের পর `--dry-run` অবশ্যই একবার চালান — নইলে ৯০ দিন পর হঠাৎ site down হতে পারে।

**Renew-এর পর Nginx reload করার deploy hook** (`--webroot` / `--dns-*` পদ্ধতিতে জরুরি):

```bash
sudo nano /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

```bash
#!/bin/bash
systemctl reload nginx
```

```bash
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

---

## 17. Production-ready Config (Hardening)

```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name devsskills.com www.devsskills.com;

    # renewal যেন redirect-এ আটকে না যায়
    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location / {
        return 301 https://$host$request_uri;
    }
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;                       # Nginx 1.25.1+; পুরনো হলে: listen 443 ssl http2;
    server_name devsskills.com www.devsskills.com;

    ssl_certificate     /etc/letsencrypt/live/devsskills.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/devsskills.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;

    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;
    ssl_session_tickets off;

    # Security headers
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;

    client_max_body_size 20M;

    location / {
        proxy_pass http://127.0.0.1:5000;
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

| Directive | কাজ |
|-----------|-----|
| `http2 on;` | HTTP/2 — একসাথে অনেক file দ্রুত load |
| `ssl_protocols TLSv1.2 TLSv1.3;` | ✅ পুরনো TLS 1.0/1.1 বন্ধ |
| `ssl_session_cache shared:SSL:10m;` | ~40,000 session জমা — বারবার handshake লাগে না |
| `ssl_session_tickets off;` | Forward secrecy আরও শক্ত |
| `Strict-Transport-Security` (HSTS) | ✅ Browser সবসময় HTTPS দিয়ে খুলবে |
| `X-Content-Type-Options: nosniff` | Browser file type অনুমান করবে না |
| `X-Frame-Options: SAMEORIGIN` | Clickjacking রোধ |
| `always` | 404/500 response-এও header পাঠাবে |

> **OCSP stapling নিয়ে:** পুরনো নোটে `ssl_stapling on;` ছিল। Let's Encrypt ২০২৫ সালে OCSP সাপোর্ট বন্ধ করে দিয়েছে, তাই Let's Encrypt cert-এ এটি আর কাজে আসে না (Nginx warning দিতে পারে)। কেনা CA cert-এ (Section 24) এখনো ব্যবহার করা যায়।

⚠️ **HSTS সতর্কতা:** একবার পাঠালে browser নির্দিষ্ট সময় ওই domain HTTP-তে খুলতেই দেবে না। SSL ঠিক আছে **নিশ্চিত হওয়ার পরেই** চালু করুন — প্রথমে `max-age=300` দিয়ে পরীক্ষা করে তারপর বাড়ান।

**যাচাই:**

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I https://devsskills.com
openssl s_client -connect devsskills.com:443 -servername devsskills.com
```

Online test: **SSL Labs** (`https://www.ssllabs.com/ssltest/`) — grade **A / A+** আসা উচিত।

---
---

# PART 5 — SSL Option B: Nginx + Cloudflare

## 18. Cloudflare কীভাবে কাজ করে

Cloudflare proxy চালু থাকলে traffic দুই ভাগে যায়:

```
User (Browser) ──[ভাগ ১: Edge SSL]──▶ Cloudflare ──[ভাগ ২: Origin SSL]──▶ আপনার Nginx server
```

- **ভাগ ১ (User → Cloudflare):** Cloudflare নিজেই ফ্রি SSL দেয় — আপনার কিছু করতে হয় না।
- **ভাগ ২ (Cloudflare → Server):** এটি আপনাকে encrypt করতে হবে — সেজন্যই **Origin Certificate** (বা Let's Encrypt cert)।

**সুবিধা:**
- ✅ Server-এর আসল IP লুকানো থাকে
- ✅ ফ্রি CDN, caching, DDoS protection
- ✅ Origin cert ১৫ বছর valid — renewal-এর ঝামেলা নেই

---

## 19. Cloudflare-এ Domain ও DNS সেটআপ

### ধাপ ১ — Domain Cloudflare-এ যোগ করা

Cloudflare Dashboard → **Add a site** → domain লিখুন → Free plan।

### ধাপ ২ — Nameserver বদলানো

Cloudflare দুটি nameserver দেবে (যেমন `xxx.ns.cloudflare.com`)। Domain যেখান থেকে কেনা (Namecheap/GoDaddy ইত্যাদি) সেখানে গিয়ে nameserver এই দুটি দিয়ে বদলান। Active হতে কয়েক মিনিট থেকে ২৪ ঘণ্টা লাগতে পারে।

### ধাপ ৩ — DNS record

| Type | Name | Content | Proxy status |
|------|------|---------|--------------|
| **A** | `@` | `13.229.45.10` | 🟠 **Proxied** |
| **CNAME** | `www` | `devsskills.com` | 🟠 **Proxied** |
| **A** | `api` | `13.229.45.10` | 🟠 **Proxied** |

| Proxy status | মানে |
|--------------|------|
| 🟠 **Proxied** (orange cloud) | Traffic Cloudflare হয়ে যায়; Cloudflare SSL, CDN, IP লুকানো — সব কাজ করে |
| ⚪ **DNS only** (grey cloud) | Cloudflare শুধু DNS; traffic সরাসরি server-এ যায় |

✅ Proxied থাকলে `dig +short devsskills.com` আপনার server IP নয়, **Cloudflare-এর IP** দেখাবে — এটা স্বাভাবিক।

⚠️ SSH-এর জন্য আলাদা `ssh.devsskills.com` রাখলে সেটি **DNS only** রাখুন — Cloudflare proxy port 22 পার করে না।

---

## 20. Cloudflare SSL Mode

Cloudflare Dashboard → **SSL/TLS → Overview**

| Mode | User→CF | CF→Server | মন্তব্য |
|------|---------|-----------|---------|
| **Off** | HTTP | HTTP | ব্যবহার করবেন না |
| **Flexible** | HTTPS | ⚠️ **HTTP** | অর্ধেক নিরাপদ; **redirect loop-এর প্রধান কারণ** |
| **Full** | HTTPS | HTTPS (cert যাচাই হয় না) | Self-signed-এও চলে |
| **Full (Strict)** | HTTPS | ✅ HTTPS (cert যাচাই হয়) | **এটিই ব্যবহার করা উচিত** |

✅ **Full (Strict)**-এর জন্য server-এ valid cert লাগবে — **Cloudflare Origin Certificate** বা **Let's Encrypt** cert, দুটোই চলে।

⚠️ **Flexible + Nginx-এ HTTP→HTTPS redirect = `ERR_TOO_MANY_REDIRECTS`**
কারণ: Cloudflare server-কে HTTP-তে request পাঠায় → Nginx বলে "HTTPS-এ যাও" → Cloudflare আবার HTTP-তে পাঠায় → অসীম loop।
সমাধান: mode **Full (Strict)** করুন।

**আরও যা চালু করবেন (SSL/TLS → Edge Certificates):**
- **Always Use HTTPS** → On (HTTP→HTTPS redirect Cloudflare-ই করবে)
- **Minimum TLS Version** → TLS 1.2
- **Automatic HTTPS Rewrites** → On

---

## 21. Origin Certificate দিয়ে Nginx Config

### ধাপ ১ — Certificate তৈরি

Cloudflare Dashboard → **SSL/TLS → Origin Server → Create Certificate**
- Key type: RSA (2048)
- Hostnames: `devsskills.com`, `*.devsskills.com` (wildcard ফ্রি পাওয়া যায় ✅)
- Validity: 15 years

Cloudflare দুটি জিনিস দেখাবে — **Origin Certificate** ও **Private Key**।

⚠️ Private key **শুধু একবারই** দেখায় — তখনই copy করে রাখুন।

### ধাপ ২ — Server-এ রাখা

```bash
sudo mkdir -p /etc/nginx/ssl
sudo nano /etc/nginx/ssl/cloudflare-origin.pem     # Origin Certificate paste
sudo nano /etc/nginx/ssl/cloudflare-origin.key     # Private Key paste

sudo chmod 600 /etc/nginx/ssl/cloudflare-origin.key
sudo chown root:root /etc/nginx/ssl/cloudflare-origin.key
```

### ধাপ ৩ — Nginx config

```nginx
# HTTP → HTTPS (Cloudflare-এর "Always Use HTTPS" চালু থাকলে এটি ঐচ্ছিক)
server {
    listen 80;
    listen [::]:80;
    server_name devsskills.com www.devsskills.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    http2 on;
    server_name devsskills.com www.devsskills.com;

    ssl_certificate     /etc/nginx/ssl/cloudflare-origin.pem;
    ssl_certificate_key /etc/nginx/ssl/cloudflare-origin.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    client_max_body_size 20M;

    location / {
        proxy_pass http://127.0.0.1:5000;
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

✅ একটি wildcard Origin cert (`*.devsskills.com`) দিয়ে `api.`, `admin.`, `blog.` — সব subdomain-এর `.conf` file-এ **একই cert path** ব্যবহার করা যায়।

### ধাপ ৪ — Test ও যাচাই

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -I https://devsskills.com        # header-এ "server: cloudflare" ও "cf-ray" দেখা যাবে
```

তারপর Cloudflare-এ SSL mode **Full (Strict)** করুন।

✅ Origin cert **শুধু Cloudflare-এর মাধ্যমে আসা traffic-এই trusted** — সরাসরি server IP-তে গেলে browser warning দেবে। এটা স্বাভাবিক।

---

## 22. ব্যবহারকারীর আসল IP ফেরত আনা

Cloudflare proxy চালু থাকলে Nginx ও backend-এর কাছে সব request-এর IP দেখায় **Cloudflare-এর IP** — ব্যবহারকারীর নয়। Cloudflare আসল IP পাঠায় `CF-Connecting-IP` header-এ।

Cloudflare-এর IP range গুলো থেকে একটি config file বানানো:

```bash
{
  echo "# Cloudflare IP ranges"
  for ip in $(curl -s https://www.cloudflare.com/ips-v4); do echo "set_real_ip_from $ip;"; done
  for ip in $(curl -s https://www.cloudflare.com/ips-v6); do echo "set_real_ip_from $ip;"; done
  echo "real_ip_header CF-Connecting-IP;"
} | sudo tee /etc/nginx/conf.d/cloudflare-realip.conf

sudo nginx -t && sudo systemctl reload nginx
```

| Directive | কাজ |
|-----------|-----|
| `set_real_ip_from` | এই IP গুলো থেকে আসা header বিশ্বাস করবে (শুধু Cloudflare) |
| `real_ip_header CF-Connecting-IP;` | এই header থেকে আসল IP নেবে |

✅ এরপর `$remote_addr` = ব্যবহারকারীর আসল IP — log, `X-Real-IP`, rate limiting সব ঠিক হয়ে যায়।

> Cloudflare মাঝে মাঝে IP range আপডেট করে — বছরে একবার এই command আবার চালানো ভালো।

---

## 23. Cloudflare-এর Extra Security

### 23.1 শুধু Cloudflare থেকে 80/443 খোলা রাখা

Proxy থাকলেও কেউ server-এর আসল IP জানলে Cloudflare এড়িয়ে সরাসরি আক্রমণ করতে পারে। তাই firewall-এ 80/443 শুধু Cloudflare-এর IP-র জন্য খুলুন:

```bash
for ip in $(curl -s https://www.cloudflare.com/ips-v4) $(curl -s https://www.cloudflare.com/ips-v6); do
  sudo ufw allow from $ip to any port 80,443 proto tcp
done
sudo ufw delete allow 'Nginx Full'     # আগের "সবার জন্য খোলা" rule সরানো
sudo ufw status
```

⚠️ এটি করার পর **Certbot HTTP challenge কাজ করবে না** — Cloudflare-এর সাথে Origin cert বা DNS-01 ব্যবহার করুন।

### 23.2 Authenticated Origin Pulls (ঐচ্ছিক, আরও কড়া)

Cloudflare প্রতিটি request-এ একটি client certificate পাঠায়; Nginx শুধু সেটি থাকলেই request নেয়।

Cloudflare Dashboard → SSL/TLS → Origin Server → **Authenticated Origin Pulls → On**, তারপর Cloudflare-এর docs থেকে origin pull CA cert download করে:

```nginx
ssl_client_certificate /etc/nginx/ssl/cloudflare-origin-pull-ca.pem;
ssl_verify_client on;
```

---
---

# PART 6 — SSL Option C: অন্যান্য পদ্ধতি

## 24. কেনা CA Certificate

কোম্পানি DigiCert / Sectigo / GoDaddy থেকে cert কিনলে:

### ধাপ ১ — Private key ও CSR তৈরি

```bash
sudo mkdir -p /etc/nginx/ssl
cd /etc/nginx/ssl

sudo openssl req -new -newkey rsa:2048 -nodes \
  -keyout devsskills.key -out devsskills.csr
```

**Definition — CSR (Certificate Signing Request):** domain ও প্রতিষ্ঠানের তথ্যসহ একটি file; CA-কে দিলে তারা cert বানিয়ে দেয়।

- `devsskills.csr`-এর content CA-এর website-এ paste করতে হয়
- "Common Name"-এ অবশ্যই domain লিখতে হবে (`devsskills.com`)

### ধাপ ২ — Chain বানানো

```bash
# নিজের cert আগে, intermediate/CA bundle পরে
sudo sh -c 'cat devsskills.crt intermediate.crt > devsskills-bundle.crt'
```

✅ **ক্রম উল্টে গেলে** "certificate chain incomplete" error — মোবাইলে site খুলবে না, desktop-এ হয়তো ঠিক দেখাবে।

### ধাপ ৩ — Nginx config

```nginx
server {
    listen 443 ssl;
    http2 on;
    server_name devsskills.com;

    ssl_certificate     /etc/nginx/ssl/devsskills-bundle.crt;
    ssl_certificate_key /etc/nginx/ssl/devsskills.key;

    ssl_protocols TLSv1.2 TLSv1.3;

    # কেনা CA cert-এ OCSP stapling এখনো কাজে আসে
    ssl_stapling on;
    ssl_stapling_verify on;
    resolver 8.8.8.8 1.1.1.1 valid=300s;

    location / {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### ধাপ ৪ — Key নিরাপদ করা

```bash
sudo chmod 600 /etc/nginx/ssl/devsskills.key
sudo chown root:root /etc/nginx/ssl/devsskills.key
sudo nginx -t && sudo systemctl reload nginx
```

⚠️ কেনা cert-এর renewal **হাতে** করতে হয় — মেয়াদ শেষের আগে calendar-এ reminder রাখুন।

---

## 25. Self-Signed Certificate

```bash
sudo mkdir -p /etc/nginx/ssl
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/nginx/ssl/selfsigned.key \
  -out /etc/nginx/ssl/selfsigned.crt
```

```nginx
server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/selfsigned.key;

    root /var/www/test;
    index index.html;
}
```

⚠️ কোনো CA sign করে না → browser **"Your connection is not private"** দেখাবে। ✅ **কখনো production-এ নয়** — শুধু local testing / internal tool। (Cloudflare **Full** mode-এ এটি চলে, কিন্তু **Full (Strict)**-এ চলে না।)

---

## 26. acme.sh

Pure shell script — Python লাগে না; Let's Encrypt ও ZeroSSL দুটোই সাপোর্ট করে।

```bash
curl https://get.acme.sh | sh -s email=you@example.com
source ~/.bashrc

# webroot পদ্ধতিতে issue
~/.acme.sh/acme.sh --issue -d devsskills.com -w /var/www/certbot

# cert নির্দিষ্ট জায়গায় install + renewal-এর পর reload
~/.acme.sh/acme.sh --install-cert -d devsskills.com \
  --key-file       /etc/nginx/ssl/devsskills.key \
  --fullchain-file /etc/nginx/ssl/devsskills.crt \
  --reloadcmd      "sudo systemctl reload nginx"
```

✅ acme.sh নিজে cron job বসায় — auto-renewal আলাদা করে সেট করতে হয় না।

> Cloudflare DNS দিয়ে wildcard: `export CF_Token="..."` তারপর `acme.sh --issue --dns dns_cf -d devsskills.com -d '*.devsskills.com'`

---
---

# PART 7 — Troubleshooting ও Reference

## 27. সাধারণ Error ও সমাধান

| Error | কারণ | সমাধান |
|-------|------|--------|
| **502 Bad Gateway** | Backend চলছে না / port ভুল | `pm2 list`, `curl http://127.0.0.1:5000` |
| **403 Forbidden** | Folder permission নেই | `sudo chown -R www-data:www-data /var/www/site` |
| **404** (SPA refresh-এ) | `try_files` ভুল | `try_files $uri $uri/ /index.html;` |
| **413 Request Entity Too Large** | Upload limit কম | `client_max_body_size 20M;` |
| **504 Gateway Timeout** | Backend দেরি করছে | `proxy_read_timeout 120s;` |
| **"Welcome to nginx!"** | Default site active | `sudo rm /etc/nginx/sites-enabled/default` |
| **`bind() to 0.0.0.0:80 failed`** | অন্য service port 80 দখল করেছে | `sudo lsof -i :80` |
| **`conflicting server name`** | একই `server_name` দুই file-এ | ডুপ্লিকেট `.conf` মুছুন |
| **পুরনো version দেখাচ্ছে** | Browser / Cloudflare cache | `Ctrl+Shift+R`; Cloudflare → Caching → Purge |

**Certbot-সংক্রান্ত:**

| Error | কারণ | সমাধান |
|-------|------|--------|
| **Timeout during connect** | Port 80 বন্ধ বা DNS ভুল | Inbound rule ও `dig +short domain.com` |
| **Unauthorized / 404 on challenge** | Validation path পৌঁছাচ্ছে না | `/.well-known/acme-challenge/` location ঠিক আছে কিনা |
| **too many certificates already issued** | Rate limit (একই domain-এ বারবার) | Test-এর সময় `--dry-run` / `--staging` ব্যবহার করুন |
| **৯০ দিন পর site down** | Renewal fail | `sudo certbot renew --dry-run`, deploy hook |

**Cloudflare-সংক্রান্ত:**

| Error | কারণ | সমাধান |
|-------|------|--------|
| **ERR_TOO_MANY_REDIRECTS** | Flexible mode + Nginx redirect | Mode → **Full (Strict)** |
| **526 Invalid SSL certificate** | Full (Strict) কিন্তু server-এ valid cert নেই | Origin cert / Let's Encrypt বসান |
| **525 SSL handshake failed** | Server-এ 443 চলছে না / cert config ভুল | `sudo nginx -t`, `listen 443 ssl;` আছে কিনা |
| **521 Web server is down** | Nginx বন্ধ বা firewall Cloudflare-কে আটকাচ্ছে | `systemctl status nginx`, UFW rule |
| **522 Connection timed out** | Server-এ পৌঁছানো যাচ্ছে না | Security Group-এ 80/443 খোলা কিনা |
| **Log-এ সব IP Cloudflare-এর** | Real IP সেট নেই | Section 22 |

**সমস্যা হলে সবার আগে:**

```bash
sudo nginx -t
sudo tail -f /var/log/nginx/error.log      # ✅ আসল কারণ প্রায় সবসময় এখানে
systemctl status nginx
pm2 logs
```

**Debugging checklist:**

```bash
ls -l /etc/nginx/sites-enabled/                 # ১. symlink ভাঙা কিনা
sudo nginx -T | grep "configuration file"       # ২. আপনার file load হচ্ছে কিনা
sudo nginx -T | grep server_name                # ৩. কোন server_name active
sudo tail -30 /var/log/nginx/error.log          # ৪. আসল error
curl -I http://127.0.0.1                        # ৫. server-এর ভেতর থেকে test
sudo nginx -T | grep -E "root|index|listen"     # ৬. config-এ root কী
ls -la /var/www/teahouse/                       # ৭. folder-এ আসলে কী আছে
sudo find /var/www -name "index.html"           # ৮. index.html কোথায়
```

---

## 28. এক নজরে মূল পয়েন্ট

**Basics ও Hosting**

| # | পয়েন্ট |
|---|--------|
| ✅ 1 | SSH ও SFTP দুটোই port **22**; HTTP **80**, HTTPS **443** |
| ✅ 2 | Nginx = **Web Server + Reverse Proxy** |
| ✅ 3 | সবসময় **`sudo nginx -t` → তারপর `reload`**; `reload`-এ downtime নেই |
| ✅ 4 | `/var/www/`-এ `sudo` লাগে, `/home/ubuntu`-তে লাগে না |
| ✅ 5 | Node app → **PM2**; `pm2 startup` + `pm2 save` বাধ্যতামূলক |
| ✅ 6 | Reverse proxy-র মূল directive **`proxy_pass`**; `Host`, `X-Real-IP`, `X-Forwarded-Proto` header দিন |
| ✅ 7 | `proxy_pass`-এর **শেষের `/`** backend-এর path বদলে দেয় |
| ✅ 8 | প্রতিটি domain-এর **আলাদা `.conf` file**; `conf.d` আর `sites-available` মেশাবেন না |
| ✅ 9 | SPA → **`try_files $uri $uri/ /index.html;`** |
| ✅ 10 | DNS: `@` → **A** (IP), `www` → **CNAME** |

**Certbot**

| # | পয়েন্ট |
|---|--------|
| ✅ 11 | Certbot-এর আগে **DNS ঠিক + port 80 খোলা** |
| ✅ 12 | সাধারণ ক্ষেত্রে **`--nginx`**; config নিজের হাতে রাখতে **`--webroot`** |
| ✅ 13 | **Wildcard** cert শুধু **DNS-01**; automatic-এর জন্য DNS plugin |
| ✅ 14 | Cert **৯০ দিন**; `sudo certbot renew --dry-run` দিয়ে যাচাই |

**Cloudflare**

| # | পয়েন্ট |
|---|--------|
| ✅ 15 | Proxy (🟠) চালু থাকলে SSL দুই ভাগে — Edge (Cloudflare দেয়) + Origin (আপনি দেবেন) |
| ✅ 16 | SSL mode সবসময় **Full (Strict)**; Flexible = redirect loop |
| ✅ 17 | Origin cert **১৫ বছর**, wildcard ফ্রি; private key একবারই দেখায় |
| ✅ 18 | আসল user IP-র জন্য **`real_ip_header CF-Connecting-IP`** |

**সাধারণ**

| # | পয়েন্ট |
|---|--------|
| ✅ 19 | **Private key কখনো share বা git-এ commit নয়**; `chmod 600` |
| ✅ 20 | Self-signed **production-এ নয়** |
| ✅ 21 | সমস্যা হলে সবার আগে **`/var/log/nginx/error.log`** |

---

## 29. Command Cheat Sheet

```bash
# ---------- Nginx ----------
sudo apt install nginx -y
sudo nginx -t
sudo systemctl reload nginx
sudo systemctl restart nginx
sudo systemctl enable nginx
systemctl status nginx
sudo nginx -T | grep server_name

# ---------- Logs ----------
sudo tail -f /var/log/nginx/error.log
sudo tail -f /var/log/nginx/access.log

# ---------- Firewall ----------
sudo ufw allow OpenSSH
sudo ufw allow 'Nginx Full'
sudo ufw status

# ---------- PM2 ----------
pm2 start index.js --name my-api
pm2 list
pm2 logs my-api
pm2 restart my-api
pm2 startup && pm2 save

# ---------- Certbot ----------
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d domain.com -d www.domain.com
sudo certbot certonly --webroot -w /var/www/certbot -d domain.com
sudo certbot certonly --dns-cloudflare \
  --dns-cloudflare-credentials /root/.secrets/cloudflare.ini \
  -d "*.domain.com" -d domain.com
sudo certbot certificates
sudo certbot renew --dry-run

# ---------- Cloudflare ----------
curl -s https://www.cloudflare.com/ips-v4      # Cloudflare IP range
curl -I https://domain.com                     # "server: cloudflare" দেখা যাবে

# ---------- SSL check ----------
openssl s_client -connect domain.com:443 -servername domain.com

# ---------- DNS ----------
dig +short domain.com
dig +short TXT _acme-challenge.domain.com
nslookup domain.com

# ---------- sites-available পদ্ধতি ----------
sudo ln -s /etc/nginx/sites-available/domain.com /etc/nginx/sites-enabled/
sudo unlink /etc/nginx/sites-enabled/domain.com
```
