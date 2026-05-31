# ArgoCD — Complete Guide

তোমার cluster: 1 master + 2 worker (AWS EC2, kubeadm)
তোমার project: MERN (sangam-server + sangam-client + MongoDB)

---

## Mental Model — ArgoCD কি

```
এখন তোমার workflow:
  code change → GitHub push → GitHub Actions → Docker Hub (new image)
  কিন্তু K8s cluster জানে না নতুন image এসেছে
  manually: kubectl rollout restart করতে হয়

ArgoCD দিয়ে:
  code change → GitHub push → GitHub Actions → Docker Hub
                                                     ↓
                                              ArgoCD দেখে
                                                     ↓
                                         K8s cluster auto update ✓
```

**ArgoCD কি?** — GitOps tool। Git repo কে "source of truth" হিসেবে ধরে।
Git এ যা আছে সেটাই cluster এ থাকবে — ArgoCD নিশ্চিত করে।

---

## GitOps Mental Model

```
পুরনো way (push-based):
  CI/CD pipeline → directly cluster এ kubectl apply

GitOps way (pull-based):
  Git repo (k8s yamls) ← ArgoCD পড়ে → cluster sync করে

পার্থক্য:
  পুরনো:  CI pipeline কে cluster এ kubectl access দিতে হয় (risky)
  GitOps: cluster নিজেই Git থেকে pull করে (secure)
```

**Analogy:**
```
পুরনো way: তুমি নিজে গিয়ে office এ কাজ দিয়ে আসো
GitOps:    office এ একজন assistant আছে, সে নিজেই তোমার
           Google Drive দেখে কাজ নিয়ে করে

তুমি Drive এ file রাখো → assistant নিজে নেয়
তুমি file বদলাও → assistant বুঝে নেয়, আবার করে
```

---

## ArgoCD কিভাবে কাজ করে

```
১. তুমি k8s/server-deployment.yaml এ image tag বদলাও
২. GitHub এ push করো
৩. ArgoCD প্রতি 3 মিনিটে Git repo check করে
৪. Git vs cluster এ পার্থক্য (drift) দেখলে → apply করে
৫. cluster = Git repo → "in sync" state

ArgoCD এর ভাষায়:
  Synced   = Git আর cluster same
  OutOfSync = Git বদলেছে কিন্তু cluster এখনো পুরনো
```

---

## তোমার Project এ Workflow

```
এখন (without ArgoCD):
  GitHub Actions → Docker Hub push ✓
  K8s update: kubectl rollout restart (manual) ✗

ArgoCD এর পর:
  GitHub Actions → Docker Hub push
                 → k8s/server-deployment.yaml এ image tag update + commit
                 → ArgoCD দেখলো → kubectl apply করলো ✓

পুরো flow:
  git push → Actions build image → push :v1.2.3 to Docker Hub
           → Actions update deployment yaml: image: ...server:v1.2.3
           → Actions commit + push yaml to GitHub
           → ArgoCD detect change → apply to cluster ✓
```

**কেন :latest tag ব্যবহার করবো না ArgoCD তে:**
```
:latest tag বদলায় না → ArgoCD বুঝতে পারে না নতুন image এসেছে
solution: specific tag ব্যবহার করো (v1.0.1, git SHA, build number)
```

---

## Install করার Steps

### Step 1: ArgoCD install

```bash
# namespace তৈরি করো
kubectl create namespace argocd

# ArgoCD install করো
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# pods ready হওয়া দেখো
kubectl get pods -n argocd --watch
```

প্রতীক্ষিত output (সব Running হলে OK):
```
NAME                                                READY   STATUS    RESTARTS
argocd-application-controller-0                     1/1     Running   0
argocd-applicationset-controller-xxx                1/1     Running   0
argocd-dex-server-xxx                               1/1     Running   0
argocd-notifications-controller-xxx                 1/1     Running   0
argocd-redis-xxx                                    1/1     Running   0
argocd-repo-server-xxx                              1/1     Running   0
argocd-server-xxx                                   1/1     Running   0
```

### Step 2: ArgoCD UI access

```bash
# argocd-server কে NodePort এ expose করো
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

# NodePort নম্বর দেখো
kubectl get svc argocd-server -n argocd
```

Output দেখতে এরকম:
```
NAME            TYPE       CLUSTER-IP       EXTERNAL-IP   PORT(S)
argocd-server   NodePort   10.102.105.226   <none>        80:30646/TCP,443:30342/TCP
```

**Node public IP বের করো (IMDSv2):**
```bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/public-ipv4
```

Browser এ যাও: `https://<node-public-ip>:<NodePort>`
- NodePort যেকোনো node এর IP দিয়ে কাজ করে (master, worker1, worker2 সব থেকে)
- "Your connection is not private" warning → Advanced → Proceed ক্লিক করো

### Step 3: Initial password বের করো

```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

Login: username = `admin`, password = উপরের output

### Troubleshooting: Memory issue (1GB RAM node এ)

```
সমস্যা: API server crash করে, kubectl TLS timeout দেয়
কারণ:   ArgoCD 7টা pod — 1GB RAM এ memory শেষ হয়
fix:    Swap add করো

sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

### Step 4: ArgoCD CLI install (optional, master node এ)

```bash
curl -sSL -o /usr/local/bin/argocd \
  https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
chmod +x /usr/local/bin/argocd
```

---

## Application তৈরি করো

ArgoCD তে "Application" = একটা Git repo এর একটা path কে একটা K8s namespace এ deploy করার config।

### Application YAML বোঝার Mindmap

```
Application
│
├── metadata
│     ├── name      → ArgoCD UI তে কি নামে দেখাবে
│     └── namespace → সবসময় "argocd" (ArgoCD এর নিজের namespace)
│
├── spec.source    → "কোথা থেকে নেবো?"
│     ├── repoURL        → GitHub repo এর URL
│     ├── targetRevision → কোন branch? (HEAD = main/master)
│     └── path           → repo এর কোন folder এ yaml আছে?
│
├── spec.destination  → "কোথায় deploy করবো?"
│     ├── server    → কোন cluster? (নিজের cluster = kubernetes.default.svc)
│     └── namespace → K8s এর কোন namespace এ?
│
└── spec.syncPolicy  → "কিভাবে sync করবো?"
      └── automated
            ├── prune     → Git থেকে মুছলে cluster থেকেও মুছবে
            └── selfHeal  → কেউ manually বদলালে Git এ ফিরিয়ে আনবে
```

নিজে লেখার সময় ৩টা প্রশ্নের উত্তর দাও:
```
১. source:  আমার k8s yaml কোথায় আছে? (GitHub repo + folder path)
২. dest:    কোন cluster এ, কোন namespace এ deploy হবে?
৩. policy:  auto sync চাই নাকি manual?
```

### Option A: UI দিয়ে

```
ArgoCD UI → New App → fill:
  Application Name: sangam-app
  Project:          default
  Sync Policy:      Automatic
  Repository URL:   https://github.com/<your-user>/sangam_devops_part3_k8s
  Path:             k8s
  Cluster:          https://kubernetes.default.svc
  Namespace:        default
```

### Option B: yaml দিয়ে

```yaml
# k8s/argocd-application.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: sangam-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/<your-username>/sangam_devops_part3_k8s
    targetRevision: HEAD
    path: k8s
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  syncPolicy:
    automated:
      prune: true        # Git থেকে মুছলে cluster থেকেও মুছবে
      selfHeal: true     # cluster manually বদললে Git এ ফিরিয়ে আনবে
```

```bash
kubectl apply -f k8s/argocd-application.yaml
```

---

## Sync Policy

```
Manual:    তুমি বললে sync করবে (UI তে Sync বাটন)
Automatic: Git change detect করলে নিজেই sync করবে

selfHeal:
  কেউ cluster এ manually কিছু বদললে
  ArgoCD আবার Git version এ ফিরিয়ে আনবে
  "Git is the truth" — cluster না

prune:
  Git থেকে কোনো yaml মুছে দিলে
  ArgoCD cluster থেকেও সেই resource মুছবে
```

---

## পুরো GitOps Flow (ArgoCD + CI/CD একসাথে)

```
git push (code change)
      ↓
test-build job (npm build — কোড ঠিক আছে কিনা দেখো)
      ↓
docker-build-push job
  → Docker Hub এ push করো :latest + :sha দুটো tag
      ↓
update-k8s-manifests job
  → server-deployment.yaml এ image tag বদলাও (:sha দিয়ে)
  → GitHub এ commit করো
      ↓
ArgoCD দেখলো yaml বদলেছে (Git poll)
      ↓
cluster এ নতুন image deploy হলো ✓
```

---

## Image Tag Strategy (ArgoCD + CI/CD)

```
ভুল:  image: abuabddullah/sangam-server:latest
সঠিক: image: abuabddullah/sangam-server:v1.0.5
      অথবা: image: abuabddullah/sangam-server:abc123f  ← git SHA
```

### GitHub Actions এ automatic tag update (নতুন job):

```yaml
  update-k8s-manifests:
    runs-on: ubuntu-latest
    needs: [docker-build-push]

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Update image tags in deployment yamls
        run: |
          sed -i "s|abuabddullah/sangam-server:.*|abuabddullah/sangam-server:${{ github.sha }}|g" k8s/server-deployment.yaml
          sed -i "s|abuabddullah/sangam-client:.*|abuabddullah/sangam-client:${{ github.sha }}|g" k8s/client-deployment.yaml

      - name: Commit and push updated yamls
        run: |
          git config user.email "actions@github.com"
          git config user.name "GitHub Actions"
          git add k8s/server-deployment.yaml k8s/client-deployment.yaml
          git commit -m "ci: update image tags to ${{ github.sha }}"
          git push
```

**কিভাবে কাজ করে:**
```
sed -i "s|পুরনো pattern|নতুন value|g" file
→ deployment yaml এর image line খুঁজে বের করে
→ tag অংশটা github.sha দিয়ে replace করে
→ git commit + push → ArgoCD দেখে → deploy করে
```

**secrets.GITHUB_TOKEN:**
```
এটা GitHub নিজেই দেয় — আলাদা করে set করতে হয় না
Actions কে repo তে push করার permission দেয়
```

---

## ArgoCD Commands

```bash
# সব application দেখো
kubectl get applications -n argocd

# detail দেখো
kubectl describe application sangam-app -n argocd

# manually sync করো
argocd app sync sangam-app

# sync status দেখো
argocd app status sangam-app

# login (CLI)
argocd login <argocd-server-ip>:<port>
```

---

## Common Issues

```
"OutOfSync" কিন্তু sync হচ্ছে না:
  → automated syncPolicy নেই → UI তে manually Sync করো
  → অথবা yaml এ syncPolicy: automated add করো

"ImagePullBackOff" after sync:
  → Docker Hub এ image নেই (tag wrong)
  → Actions এ push হয়নি আগে

Private repo:
  → ArgoCD কে repo access দিতে হবে
  → Settings → Repositories → Connect Repo → SSH key বা token দাও

ArgoCD pod crash:
  → kubectl logs -n argocd deployment/argocd-server
```

---

## Related Notes

- znotes/cicd-github-actions-guide.md — CI/CD pipeline
- znotes/002_k8s-deployment-service-guide.md — Deployment/Service
- znotes/003_k8s-autoscaling-guide.md — HPA/CA
