# Backend App — Docker Complete Guide

যেকোনো backend project dockerize করার reference।

---

## Mental Model — সবার আগে এই প্রশ্নগুলো করো

```
প্রশ্ন ১: Node/Python/PHP সরাসরি source file চালাতে পারবে?
  হ্যাঁ → Single stage (Plain JS, Python, PHP)
  না  → Multistage (TypeScript, Go, Rust)

প্রশ্ন ২: Build output কোথায়?
  TypeScript → tsconfig.json এর outDir দেখো (সাধারণত dist/)
  Go         → go build এর output file নাম দেখো
  Plain JS   → build নেই, src/ সরাসরি চলে

প্রশ্ন ৩: Entry file কোনটা?
  package.json এর "start" script দেখো → node dist/???.js

প্রশ্ন ৪: Port কত?
  server file এ PORT বা default value দেখো
```

---

## Language Type — সবচেয়ে গুরুত্বপূর্ণ concept

```
Compiled (Go, Rust):
  source → binary বানাও → binary run করো
  Production এ language runtime লাগে না

Interpreted (JavaScript, Python, PHP):
  source → runtime সরাসরি পড়ে চালায়
  Production এ runtime থাকতে হবে

TypeScript:
  Interpreted কিন্তু translate লাগে
  .ts → tsc → .js → Node চালায়
  তাই Multistage লাগে
```

---

## Pattern — মাত্র ২টা

### Pattern 1 — Plain JS (Single stage)

```dockerfile
FROM node:20-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev
COPY src ./src
EXPOSE 3000
CMD ["node", "src/server.js"]
```

### Pattern 2 — TypeScript (Multistage)

```dockerfile
FROM node:20-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build          # tsc → dist/

FROM node:20-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --omit=dev      # devDeps বাদ (typescript, @types/*)
COPY --from=builder /app/dist ./dist
EXPOSE 5000
CMD ["node", "dist/server.js"]
```

**Rule: JS Framework যাই হোক — TypeScript থাকলে এই pattern। শুধু port আর entry file বদলায়।**

---

## Framework Comparison

| Framework | Language | Build output | Entry file | Default port |
|---|---|---|---|---|
| Express | JS/TS | dist/ (TS হলে) | dist/server.js | নিজে set করো |
| NestJS | TS (default) | dist/ | dist/main.js | 3000 |
| Fastify | JS/TS | dist/ (TS হলে) | dist/server.js | 3000 |
| Hono | TS | dist/ | dist/index.js | 3000 |

---

## Python (FastAPI / Django / Flask)

Interpreted → Single stage, build step নেই।

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt ./
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

> `uvicorn` = FastAPI এর server  
> `gunicorn` = Django/Flask এর production server

---

## Go

Compiled → Multistage। Production এ Go runtime লাগে না — শুধু binary।

```dockerfile
FROM golang:1.23-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o server .   # একটাই binary বানায়

FROM alpine:latest AS production
WORKDIR /app
COPY --from=builder /app/server ./server
EXPOSE 8080
CMD ["./server"]
```

Go image সবচেয়ে ছোট হয় (~10MB) — কারণ production এ শুধু একটা binary।

---

## Laravel (PHP)

Interpreted কিন্তু দুইটা deps: composer (PHP) + npm (JS assets)।

```dockerfile
FROM php:8.3-fpm-alpine
WORKDIR /app
RUN docker-php-ext-install pdo pdo_mysql
COPY --from=composer:latest /usr/bin/composer /usr/bin/composer
COPY composer.json composer.lock ./
RUN composer install --no-dev
COPY package*.json ./
RUN apk add --no-cache nodejs npm && npm ci && npm run build
COPY . .
EXPOSE 9000
CMD ["php-fpm"]
```

---

## Image Size Comparison

```
Go/Rust   → ~10-20MB    (binary only, no runtime)
Python    → ~100-150MB  (slim image)
Node/TS   → ~150-200MB  (node:alpine)
PHP       → ~200MB+     (extensions সহ)
```

Multistage ব্যবহার করলে সব ক্ষেত্রেই size কমে।

---

## `--omit=dev` কি করে?

```json
"devDependencies": {
  "typescript": "^5.6.2",    ← build এর পরে আর লাগে না
  "@types/express": "^5.0.0" ← শুধু TypeScript check এ লাগে
}
```

`npm ci --omit=dev` → devDependencies বাদ দিয়ে install করে।  
Image ছোট হয়, unnecessary packages production এ থাকে না।

---

## Multistage কেন?

```
Without multistage:
  node + devDeps (typescript, tsx) + source + dist = ~500MB

With multistage:
  Stage 1 (builder): সব install + build
  Stage 2 (production): runtime + dist only = ~150MB
```

---

## Key Rules

| Rule | কারণ |
|---|---|
| `npm ci` ব্যবহার করো | lock file থেকে exact install — reproducible |
| `package*.json` আলাদা COPY | layer cache — deps না বদলালে re-install হবে না |
| `dev` script কখনো না | dev server শুধু development এর জন্য |
| `--omit=dev` production এ | devDeps production এ লাগে না |
| TypeScript → Multistage | Node সরাসরি .ts চালাতে পারে না |
| Compiled lang → Multistage | Production এ runtime লাগে না |
