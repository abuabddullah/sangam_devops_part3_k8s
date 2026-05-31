# CI/CD with GitHub Actions — Docker Build & Push Guide

যেকোনো project এ GitHub Actions দিয়ে Docker image auto-build + push করার reference।

---

## Mental Model

```
তুমি শুধু:  git push

বাকি সব automatic:
  ↓ GitHub Actions চালু হয়
  ↓ code build হয় কিনা check করে  ← CI
  ↓ Docker image build করে          ← CD
  ↓ Docker Hub এ push করে           ← CD
```

```
CI = Continuous Integration  → code ঠিক আছে কিনা দেখো
CD = Continuous Deployment   → ঠিক থাকলে deploy করো
```

---

## Mindmap

```
git push (main)
    │
    ▼
GitHub Actions
    │
    ├── TRIGGER
    │     └── on: push → branches: [main]
    │
    ├── JOB
    │     └── runs-on: ubuntu-latest (GitHub এর VM)
    │
    └── STEPS
          │
          ├── 1. Checkout       → code আনো (সবসময় প্রথমে)
          │
          ├── 2. CI (build check)
          │     ├── npm ci
          │     └── npm run build
          │           ↓ fail হলে এখানেই থামে, Docker এ যায় না
          │
          ├── 3. Docker Login   → Secrets দিয়ে
          │
          └── 4. CD
                ├── docker build -t :latest -t :sha
                └── docker push :latest + :sha
```

---

## Step 1 — Docker Hub Access Token বানাও

`Docker Hub → Account Settings → Security → New Access Token`

```
Name:        github-actions
Permission:  Read & Write
```

Token copy করে রাখো — **একবারই দেখাবে।**

> ⚠️ Password দিও না — Access Token দাও। Token revoke করা যায়, password না।

---

## Step 2 — GitHub Secrets add করো

`GitHub Repo → Settings → Secrets and variables → Actions → New repository secret`

```
Name:   DOCKERHUB_USERNAME
Value:  তোমার Docker Hub username

Name:   DOCKERHUB_TOKEN
Value:  Step 1 এর token
```

> Secret name এ শুধু A-Z, a-z, 0-9, _ চলে। Space বা hyphen চলে না।

---

## Step 3 — Workflow file বানাও

Path: `.github/workflows/docker-build-push.yml`

```yaml
name: Docker Build & Push

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      # ── CI: code build check ──────────────────────────────
      - name: Install server dependencies
        working-directory: ./server
        run: npm ci

      - name: Build server code
        working-directory: ./server
        run: npm run build

      - name: Install client dependencies
        working-directory: ./client
        run: npm ci

      - name: Build client code
        working-directory: ./client
        run: npm run build

      # ── CD: Docker build + push ───────────────────────────
      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push server image
        run: |
          docker build -t YOUR_USERNAME/YOUR_SERVER_IMAGE:latest \
                       -t YOUR_USERNAME/YOUR_SERVER_IMAGE:${{ github.sha }} \
                       ./server
          docker push YOUR_USERNAME/YOUR_SERVER_IMAGE:latest
          docker push YOUR_USERNAME/YOUR_SERVER_IMAGE:${{ github.sha }}

      - name: Build and push client image
        run: |
          docker build -t YOUR_USERNAME/YOUR_CLIENT_IMAGE:latest \
                       -t YOUR_USERNAME/YOUR_CLIENT_IMAGE:${{ github.sha }} \
                       ./client
          docker push YOUR_USERNAME/YOUR_CLIENT_IMAGE:latest
          docker push YOUR_USERNAME/YOUR_CLIENT_IMAGE:${{ github.sha }}
```

---

## Step 4 — Verify করো

**GitHub Actions:**
`Repo → Actions tab → সবুজ ✓ দেখাবে`

**Docker Hub:**
`hub.docker.com/r/YOUR_USERNAME/YOUR_IMAGE/tags`

```
latest       → সবসময় সর্বশেষ
abc1234...   → এই specific commit এর image (rollback এর জন্য)
```

---

## নতুন project এ কি কি বদলাবে?

```
বদলাবে:
  ✎ YOUR_USERNAME        → তোমার Docker Hub username
  ✎ YOUR_SERVER_IMAGE    → তোমার image name (যেমন: sangam-server)
  ✎ YOUR_CLIENT_IMAGE    → তোমার image name (যেমন: sangam-client)
  ✎ ./server, ./client   → তোমার Dockerfile এর location
  ✎ npm run build        → তোমার language এর build command

Same থাকবে (copy-paste):
  ✓ Checkout step
  ✓ Docker login step
  ✓ docker build + push pattern
  ✓ secrets.DOCKERHUB_USERNAME, secrets.DOCKERHUB_TOKEN
  ✓ github.sha tag
  ✓ runs-on: ubuntu-latest
```

---

## Language অনুযায়ী build command

| Language | Install | Build |
|---|---|---|
| Node.js (JS) | `npm ci` | `npm run build` |
| Node.js (TS) | `npm ci` | `npm run build` |
| Python | `pip install -r requirements.txt` | (build নেই) |
| Go | — | `go build ./...` |

> Python এর build step নেই — CI check এ শুধু install করলেই যথেষ্ট।

---

## Key Rules

| Rule | কারণ |
|---|---|
| Build check আগে, Docker পরে | Broken code Docker Hub এ যাবে না |
| Access Token দাও, password না | Token revoke করা যায় |
| `:latest` + `:sha` দুটো tag | Rollback এর জন্য sha দরকার |
| Secrets এ credentials রাখো | Public repo হলেও safe |
| `docker/login-action` ব্যবহার করো | Raw `echo | docker login` TTY error দেয় |
