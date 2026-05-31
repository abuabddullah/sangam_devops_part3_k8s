# CI/CD with GitHub Actions — Docker Build & Push Guide

যেকোনো project এ GitHub Actions দিয়ে Docker image auto-build + push করার reference।

---

## Mental Model

```
তুমি শুধু:  git push

বাকি সব automatic:
  ↓ GitHub Actions চালু হয়
  ↓ code build হয় কিনা check করে  ← CI (test job)
  ↓ Docker image build করে          ← CD (docker-build-push job)
  ↓ Docker Hub এ push করে
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
    ├── JOB 1: test
    │     runs-on: ubuntu-latest
    │     │
    │     ├── Checkout
    │     ├── npm ci + npm run build (server)
    │     └── npm ci + npm run build (client)
    │               ↓ fail হলে এখানেই থামে
    │
    └── JOB 2: docker-build-push
          needs: [test]   ← test pass হলেই চলে
          runs-on: ubuntu-latest
          │
          ├── Checkout
          ├── Docker Login (Secrets দিয়ে)
          ├── docker build + push (server)
          └── docker build + push (client)
```

---

## Single Job vs Multi Job

```
Single Job:
  সব steps একটা VM এ চলে
  Simple, ছোট project এ ঠিক আছে

Multi Job (production-friendly):
  প্রতিটা job আলাদা VM এ চলে
  needs: [test] → test fail হলে docker job চলেই না
  GitHub UI তে clearly দেখা যায় কোথায় fail হলো
```

```
GitHub Actions UI:

  ✓ test ──→ ✓ docker-build-push   (সব ঠিক)
  ✗ test ──→ ✗ docker-build-push   (test fail, docker skipped)
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

  test:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

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

  docker-build-push:
    runs-on: ubuntu-latest
    needs: [test]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

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
`Repo → Actions tab → দুটো job দেখাবে: test → docker-build-push`

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
  ✎ YOUR_SERVER_IMAGE    → তোমার image name (যেমন: my-server)
  ✎ YOUR_CLIENT_IMAGE    → তোমার image name (যেমন: my-client)
  ✎ ./server, ./client   → তোমার Dockerfile এর location
  ✎ npm run build        → তোমার language এর build command

Same থাকবে (copy-paste):
  ✓ Checkout step
  ✓ Docker login step (docker/login-action@v3)
  ✓ docker build + push pattern
  ✓ secrets.DOCKERHUB_USERNAME, secrets.DOCKERHUB_TOKEN
  ✓ github.sha tag
  ✓ runs-on: ubuntu-latest
  ✓ needs: [test]
```

---

## Artifact কি এবং কখন লাগে?

```
প্রতিটা job আলাদা fresh VM এ চলে।
Job 1 এ যা build হলো → Job 2 সেটা দেখতে পায় না।

Artifact = Job 1 এর file save করো → Job 2 তে download করো
```

```
Artifact দরকার আছে:
  ✓ test report share করতে (coverage.xml, junit.xml)
  ✓ compiled binary একটা job এ build করে অন্য job এ deploy করতে
  ✓ build output একাধিক job এ use করতে

Artifact দরকার নেই:
  ✗ Docker build এ → Docker নিজেই ভেতরে npm run build করে
     তাই আলাদা artifact save করার দরকার নেই
```

---

## Language অনুযায়ী build command

| Language | Install | Build |
|---|---|---|
| Node.js (JS/TS) | `npm ci` | `npm run build` |
| Python | `pip install -r requirements.txt` | (build নেই) |
| Go | — | `go build ./...` |

---

## Key Rules

| Rule | কারণ |
|---|---|
| Multi job ব্যবহার করো | test fail হলে docker job চলবেই না |
| `needs: [test]` দাও | Job dependency declare করে |
| Build check আগে, Docker পরে | Broken image Docker Hub এ যাবে না |
| Access Token দাও, password না | Token revoke করা যায় |
| `:latest` + `:sha` দুটো tag | Rollback এর জন্য sha দরকার |
| `docker/login-action` ব্যবহার করো | Raw echo pipe TTY error দেয় |
| Secrets এ credentials রাখো | Public repo হলেও safe |
