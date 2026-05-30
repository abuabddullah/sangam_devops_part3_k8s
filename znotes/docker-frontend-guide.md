# Frontend App — Docker Complete Guide

যেকোনো frontend project dockerize করার complete reference।  
Plain HTML থেকে শুরু করে Next.js পর্যন্ত সব covered।

---

## সবার আগে — Mental Model

Dockerfile লেখার আগে নিজেকে মাত্র ২টা প্রশ্ন করো:

```
প্রশ্ন ১: Production এ কি Node server চলতে থাকে?
  হ্যাঁ → Node runtime লাগবে
  না  → nginx যথেষ্ট

প্রশ্ন ২: Build step আছে? (npm run build)
  না  → সরাসরি nginx তে COPY করো
  হ্যাঁ → Multistage Dockerfile লাগবে
```

**যা manually terminal এ করো, সেটাই Dockerfile এ লেখো।**

---

## Framework Comparison

| App | Build command | Output folder | Runtime | Port |
|---|---|---|---|---|
| Plain HTML/CSS/JS | নেই | . (root) | nginx | 80 |
| React + Vite | `npm run build` | `dist/` | nginx | 80 |
| Vue + Vite | `npm run build` | `dist/` | nginx | 80 |
| Vue CLI | `npm run build` | `dist/` | nginx | 80 |
| Angular | `npm run build` | `dist/<name>/browser/` | nginx | 80 |
| Next.js (SSR) | `npm run build` | `.next/` | Node | 3000 |
| Next.js (standalone) | `npm run build` | `.next/standalone/` | Node | 3000 |
| Next.js (static export) | `npm run build` | `out/` | nginx | 80 |

---

## Decision Tree

```
Frontend app dockerize করবো
│
├── Build step আছে?
│   │
│   ├── না → Plain HTML/CSS/JS
│   │         FROM nginx:alpine
│   │         COPY . /usr/share/nginx/html
│   │         (4 lines, done!)
│   │
│   └── হ্যাঁ → npm run build করলে output কোথায়?
│               │
│               ├── dist/ বা build/ বা out/
│               │   └── Multistage: node → nginx
│               │       (React, Vue, Angular, Next.js export)
│               │
│               └── .next/ এবং SSR আছে
│                   └── Multistage: node → node
│                       (Next.js default/standalone)
```

---

## Dockerfile Templates

### 1. Plain HTML/CSS/JS (build step নেই)

```dockerfile
FROM nginx:alpine

COPY . /usr/share/nginx/html

EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 2. Plain HTML/CSS/JS (build step আছে — SCSS, minify)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine AS production
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 3. React + Vite

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine AS production
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 4. Vue.js (Vite বা Vue CLI) — React এর মতোই

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine AS production
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

### 5. Angular

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM nginx:alpine AS production
COPY nginx.conf /etc/nginx/conf.d/default.conf
# angular.json এ "projects" এর নাম দেখো
COPY --from=builder /app/dist/<project-name>/browser /usr/share/nginx/html
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

> Project name কোথায় পাবে? `angular.json` খোলো → `"projects": { "your-app-name": ...`

### 6. Next.js — Standalone (recommended)

প্রথমে `next.config.js` তে যোগ করো:
```js
const nextConfig = {
  output: "standalone",
};
export default nextConfig;
```

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

FROM node:20-alpine AS production
WORKDIR /app
COPY --from=builder /app/.next/standalone ./
COPY --from=builder /app/.next/static ./.next/static
COPY --from=builder /app/public ./public
EXPOSE 3000
CMD ["node", "server.js"]
```

---

## nginx.conf Templates

### Backend API আছে (React/Vue/Angular + Express/Node)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    # SPA routing — React/Vue/Angular এ লাগবেই
    location / {
        try_files $uri $uri/ /index.html;
    }

    # Backend proxy — /api request গুলো backend container এ পাঠাও
    location /api {
        proxy_pass http://<service-name>:<port>;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

> `<service-name>` = docker-compose এর service name (যেমন: `server`, `backend`, `api`)

### proxy block এর প্রতিটা লাইন কি করে?

```nginx
location /api {
    proxy_pass http://server:5000;
    # /api request → backend container এ forward করো

    proxy_http_version 1.1;
    # HTTP 1.1 দিয়ে কথা বলো (1.0 না) → connection reuse হয়, faster

    proxy_set_header Host $host;
    # চিরকুট: "user আসলে example.com এ গিয়েছিল"

    proxy_set_header X-Real-IP $remote_addr;
    # চিরকুট: "user এর আসল IP এটা" → backend এ req.ip কাজ করবে

    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    # চিরকুট: "request এর পথ: user IP → nginx → তুমি"
}
```

**শেষ ৩টা লাইন কেন লাগে?**  
nginx মাঝখানে থাকায় backend সরাসরি user দেখতে পায় না — শুধু nginx দেখে।  
এই headers গুলো backend কে জানায় আসল user কে ছিল।

**Mandatory?** না। শুধু `proxy_pass` দিলেও কাজ করবে।  
কিন্তু logging, rate limiting, বা security এর জন্য এগুলো best practice।  
**Rule:** প্রায় সব nginx config এ এই ৩ লাইন copy-paste করা হয় — boilerplate মনে করো।

### Backend নেই (শুধু frontend)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Plain HTML (SPA routing নেই)

```nginx
server {
    listen 80;
    root /usr/share/nginx/html;
    index index.html;
}
```

---

## Key Rules (সবসময় মনে রাখো)

| Rule | কারণ |
|---|---|
| `npm ci` ব্যবহার করো, `npm install` না | `package-lock.json` থেকে exact versions — reproducible build |
| `package*.json` আলাদা COPY করো | Docker layer cache — deps change না হলে re-install হবে না |
| `dev` script কখনো Dockerfile এ না | Dev server শুধু development এর জন্য |
| Static app → nginx, Node app → node | Static files এ runtime লাগে না |
| `npm ci --omit=dev` production stage এ | devDependencies বাদ → image ছোট |
| `daemon off` nginx এ | Docker container এ foreground process রাখতে হয় |

---

## Multistage কেন?

```
Without multistage:
  node:20-alpine + node_modules (devDeps সহ) + source + dist = ~500MB

With multistage:
  Stage 1 (builder): সব কিছু install + build করো
  Stage 2 (production): শুধু output copy করো
  Final image = nginx:alpine + dist files = ~25MB
```

Image ছোট = faster pull, faster deploy, কম storage।

---

## Quick Reference

```bash
# Build এবং run করো
docker build -t my-app .
docker run -p 80:80 my-app

# Docker Compose দিয়ে
docker compose up --build
docker compose up --build -d    # background এ
docker compose down
```
