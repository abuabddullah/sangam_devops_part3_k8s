# K8s Deployment & Service — Complete Guide

তোমার cluster: 1 master + 2 worker (AWS EC2)
তোমার project: MERN (sangam-server + sangam-client + MongoDB)
Docker images: `abuabddullah/sangam-server` + `abuabddullah/sangam-client` (Docker Hub)

---

## Mental Model

```
Docker Compose:  একটা machine, "services" দিয়ে container define করো
Kubernetes:      অনেক machine, "Deployment" দিয়ে container define করো

Docker Compose → Kubernetes mapping:
  services:                →  Deployment
    image:                 →    spec.containers.image
    ports:                 →  Service
    environment:           →  env / ConfigMap / Secret
    restart: always        →  (Deployment automatically করে)
    replicas:              →  spec.replicas
```

---

## Deployment.yaml — প্রতিটা line কি বলে

```yaml
apiVersion: apps/v1          # সবসময় এটাই Deployment এর জন্য
kind: Deployment             # এটা একটা Deployment object

metadata:
  name: sangam-server        # cluster এ unique নাম (kubectl দিয়ে এই নামে reference করবে)

spec:                        # "আমি কি চাই" এর বর্ণনা শুরু
  replicas: 2                # কতটা Pod চলবে একসাথে

  selector:                  # এই Deployment কোন Pod গুলো manage করবে?
    matchLabels:
      app: sangam-server     # যেসব Pod এ এই label আছে, সেগুলো manage করবে

  template:                  # নতুন Pod বানানোর blueprint
    metadata:
      labels:
        app: sangam-server   # ← এটা selector.matchLabels এর সাথে match করতে হবে

    spec:
      containers:
        - name: server                             # container এর নাম
          image: abuabddullah/sangam-server:latest # Docker Hub থেকে এই image নেবে
          ports:
            - containerPort: 5000                  # container এর ভেতরে কোন port এ শুনছে
          env:
            - name: PORT
              value: "5000"
            - name: NODE_ENV
              value: "production"
```

---

## Mindmap

```
Deployment.yaml
│
├── apiVersion: apps/v1         (Deployment এর জন্য সবসময় এটা)
├── kind: Deployment
│
├── metadata
│     └── name                  (যেকোনো unique নাম)
│
└── spec
      │
      ├── replicas              (কতটা Pod? default 1)
      │
      ├── selector              ┐
      │     └── matchLabels    │ এই দুটো label সবসময়
      │           app: xxx     │ same হতে হবে
      │                        │
      └── template             │
            ├── metadata       │
            │   └── labels    ┘
            │       app: xxx   ← selector এর মতোই
            │
            └── spec
                  └── containers[]
                        ├── name          (container এর নাম)
                        ├── image         (DockerHub image:tag)
                        ├── ports[]
                        │     containerPort   (app কোন port এ চলে)
                        └── env[]
                              ├── name + value          (সরাসরি value)
                              └── name + valueFrom      (Secret/ConfigMap থেকে)
```

---

## নিজে থেকে লেখার চিন্তা

```
প্রশ্ন ১: কোন app?
  → metadata.name: sangam-server

প্রশ্ন ২: কতটা কপি চলবে?
  → replicas: 2

প্রশ্ন ৩: কোন Docker image?
  → image: abuabddullah/sangam-server:latest

প্রশ্ন ৪: app কোন port এ চলে?
  → Express/Node:  containerPort: 5000
  → React + nginx: containerPort: 80
  → Next.js:       containerPort: 3000
  → MongoDB:       containerPort: 27017

প্রশ্ন ৫: কোন env variables লাগবে?
  → backend:  MONGO_URI, PORT, JWT_SECRET
  → frontend: VITE_API_URL বা NEXT_PUBLIC_API_URL
```

> **Rule:** selector.matchLabels এর label আর template.metadata.labels সবসময় same রাখো।
> এটা miss হলে Deployment কোন Pod manage করবে বুঝবে না।

---

## Frontend vs Backend — কি আলাদা কি same

```
                    Backend (Express)       Frontend (React+Nginx)    Frontend (Next.js)
─────────────────────────────────────────────────────────────────────────────────────────
containerPort       5000                    80                        3000
image               node app চলে           nginx build serve করে     node standalone
env variables       MONGO_URI, PORT         VITE_API_URL              NEXT_PUBLIC_API_URL
                    JWT_SECRET              (build time এ দিতে হয়)
replicas            2                       2                         2

same থাকে:
  ✓ apiVersion, kind, metadata structure
  ✓ selector + matchLabels pattern
  ✓ template.spec.containers structure
```

---

## তোমার project এর Deployment files

### server-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sangam-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sangam-server
  template:
    metadata:
      labels:
        app: sangam-server
    spec:
      containers:
        - name: sangam-server
          image: abuabddullah/sangam-server:latest
          ports:
            - containerPort: 5000
          env:
            - name: PORT
              value: "5000"
            - name: NODE_ENV
              value: "production"
            - name: MONGO_URI
              value: "mongodb://sangam-mongo-service:27017/sangamdb"
```

### client-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sangam-client
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sangam-client
  template:
    metadata:
      labels:
        app: sangam-client
    spec:
      containers:
        - name: sangam-client
          image: abuabddullah/sangam-client:latest
          ports:
            - containerPort: 80
```

### mongo-deployment.yaml

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sangam-mongo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: sangam-mongo
  template:
    metadata:
      labels:
        app: sangam-mongo
    spec:
      containers:
        - name: sangam-mongo
          image: mongo:6
          ports:
            - containerPort: 27017
```

---

## Service.yaml — কেন লাগে

```
Pod এর সমস্যা:
  Pod restart হলে নতুন IP পায়
  Backend জানে না MongoDB কোথায়

Service সমাধান:
  Service এর IP কখনো বদলায় না
  Service → সঠিক Pod এ traffic route করে
```

## Service Types

```
ClusterIP (default)
  → শুধু cluster এর ভেতরে accessible
  → Use: backend → mongodb, server → client internal
  → বাইরে থেকে access নেই

NodePort
  → Worker node এর IP + port (30000-32767) এ accessible
  → Use: তোমার cluster এ testing, bare-metal K8s
  → http://worker-node-ip:31000

LoadBalancer
  → Cloud (AWS/GCP) নতুন external IP দেয়
  → Use: production, AWS EKS
  → Bare-metal cluster এ কাজ করে না
```

## Service.yaml — প্রতিটা line কি বলে

```yaml
apiVersion: v1          # Service এর জন্য সবসময় v1
kind: Service

metadata:
  name: sangam-server-service   # অন্য pod এই নামে এই service কে খুঁজে পাবে

spec:
  selector:
    app: sangam-server          # কোন Pod এ traffic পাঠাবে (Deployment label এর সাথে match)

  ports:
    - port: 80                  # Service কোন port এ শুনবে (outside থেকে)
      targetPort: 5000          # Pod এর কোন port এ forward করবে (containerPort)

  type: NodePort                # ClusterIP / NodePort / LoadBalancer
  # nodePort: 31000             # (optional) নির্দিষ্ট port দিতে চাইলে, না দিলে auto assign
```

## Service Mindmap

```
Service.yaml
│
├── apiVersion: v1
├── kind: Service
│
├── metadata
│     └── name              (DNS নাম — অন্য pod এই নামে call করবে)
│
└── spec
      ├── selector
      │     app: xxx         ← Deployment এর label এর সাথে match করে pod খোঁজে
      │
      ├── ports[]
      │     ├── port         (service কোন port এ শুনবে)
      │     └── targetPort   (pod এর containerPort)
      │
      └── type
            ├── ClusterIP    (internal only — default)
            ├── NodePort     (node IP + port — তোমার cluster)
            └── LoadBalancer (cloud only)
```

---

## তোমার project এর Service files

### server-service.yaml — NodePort (বাইরে থেকে access)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sangam-server-service
spec:
  selector:
    app: sangam-server
  ports:
    - port: 80
      targetPort: 5000
      nodePort: 31000
  type: NodePort
```

### client-service.yaml — NodePort (browser এ দেখতে)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sangam-client-service
spec:
  selector:
    app: sangam-client
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30000
  type: NodePort
```

### mongo-service.yaml — ClusterIP (শুধু internal)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sangam-mongo-service
spec:
  selector:
    app: sangam-mongo
  ports:
    - port: 27017
      targetPort: 27017
  type: ClusterIP
```

> MongoDB বাইরে expose করার দরকার নেই — শুধু server Pod এ দরকার।
> server এর MONGO_URI: `mongodb://sangam-mongo-service:27017/sangamdb`
> Service এর name-ই DNS হিসেবে কাজ করে cluster এর ভেতরে।

---

## Access করার Pattern (তোমার cluster)

```
Browser → http://worker-node-ip:30000  → client Pod (React app)
Browser → http://worker-node-ip:31000  → server Pod (API)

Client Pod → http://sangam-server-service/api/...  (ClusterIP হলেও ভেতর থেকে)
Server Pod → mongodb://sangam-mongo-service:27017  (ClusterIP, internal only)
```

---

## kubectl commands

```bash
# Apply করো
kubectl apply -f k8s/

# কি চলছে দেখো
kubectl get pods
kubectl get deployments
kubectl get services

# Pod এর log দেখো
kubectl logs -f deployment/sangam-server

# Pod এ ঢোকো
kubectl exec -it <pod-name> -- sh

# Delete করো
kubectl delete -f k8s/
```

---

## Debugging — নিজে সমস্যা ধরার workflow

```
Pod সমস্যা হলে সবসময় এই ৩টা command:

1. kubectl get pods
   → STATUS দেখো

2. kubectl logs deployment/<name>
   → error message পড়ো (সবচেয়ে গুরুত্বপূর্ণ)

3. kubectl describe pod <pod-name>
   → Events section দেখো (image pull হলো? কোথায় schedule হলো?)
```

### STATUS দেখে কি বুঝবে

```
STATUS               মানে                          পরের কাজ
──────────────────────────────────────────────────────────────
Running              ঠিকঠাক চলছে                  কিছু করতে হবে না
CrashLoopBackOff     চালু হয়েই crash করছে         kubectl logs দেখো
Error                একবার crash করেছে             kubectl logs দেখো
ImagePullBackOff     Docker Hub থেকে image নামেনি  image name ঠিক আছে?
Pending              কোনো node এ schedule হয়নি    kubectl describe দেখো
OOMKilled            RAM শেষ হয়ে গেছে             node এর RAM বাড়াও
```

### Common Error Messages

```
"host not found in upstream X"
  → nginx.conf এ proxy_pass এ X নামে K8s Service নেই
  → Service এর সঠিক name দাও (docker-compose name না)

"MONGODB_URI is missing"
  → Deployment এর env section এ নেই
  → server-deployment.yaml এ env add করো

"connection refused"
  → target Service/Pod চলছে না
  → আগে সেই Pod এর status দেখো

"ImagePullBackOff"
  → image name ভুল, বা Docker Hub এ image নেই
  → Docker Hub এ গিয়ে image আছে কিনা confirm করো
```

### Docker Compose vs K8s — Name এর পার্থক্য

```
যেখানে চলছে    hostname কোথা থেকে আসে
────────────────────────────────────────────────────────
docker-compose  services: এর নাম → mongodb, server
kubernetes      Service object এর metadata.name → sangam-mongo-service

তাই nginx.conf, MONGODB_URI যেখানেই hostname লেখো:
  K8s তে চালাবে → K8s Service object এর name লেখো
  docker-compose → docker-compose service name লেখো
```

### Real Example (sangam project)

```
Problem:  sangam-client CrashLoopBackOff
Command:  kubectl logs deployment/sangam-client
Error:    "host not found in upstream server"
Reason:   nginx.conf এ proxy_pass http://server:5000 — docker-compose এর name
Fix:      proxy_pass http://sangam-server-service:80 — K8s Service name
```

### Fix করার পর workflow

```
1. code fix করো (nginx.conf / deployment.yaml / যেটা লাগে)
2. git add + commit + push
3. GitHub Actions নতুন image build করে Docker Hub এ push করবে
4. Master node এ:
     kubectl rollout restart deployment/<name>
5. kubectl get pods দিয়ে verify করো
```

---

## নতুন project এ কি বদলাবে

```
বদলাবে:
  ✎ metadata.name          → তোমার app এর নাম
  ✎ image                  → তোমার Docker Hub image
  ✎ containerPort          → তোমার app এর port
  ✎ env variables          → তোমার app এর variables
  ✎ selector/labels এর app → তোমার app এর নাম (সব জায়গায় same)

Same থাকবে:
  ✓ apiVersion: apps/v1 (Deployment)
  ✓ apiVersion: v1 (Service)
  ✓ পুরো yaml structure
  ✓ selector + matchLabels pattern
  ✓ MongoDB → ClusterIP, frontend/backend → NodePort
```
