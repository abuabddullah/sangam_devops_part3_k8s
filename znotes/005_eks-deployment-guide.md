# EKS Deployment Guide

তোমার project: MERN (sangam-server + sangam-client + MongoDB)
লক্ষ্য: kubeadm cluster থেকে AWS EKS এ migrate করা

---

## EKS vs kubeadm — পার্থক্য

```
kubeadm (manual):                    EKS (AWS managed):
  তুমি EC2 নাও                        AWS control plane manage করে
  তুমি K8s install করো               তুমি শুধু worker node দাও
  তুমি control plane fix করো         AWS নিজে fix করে
  free (EC2 cost ছাড়া)               $0.10/hour cluster fee + EC2 cost

মিল:
  kubectl একইভাবে কাজ করে
  তোমার k8s yaml হুবহু একই
  ArgoCD, HPA সব একইভাবে চলে
```

**Analogy:**
```
kubeadm = নিজের জমিতে বাড়ি বানাও (foundation থেকে সব নিজে)
EKS     = apartment কিনো (structure AWS এর, ভেতরে তুমি সাজাও)
```

---

## Step 0: IAM User তৈরি করো (Root account ব্যবহার করো না)

**কেন root account ব্যবহার করা উচিত না:**
```
root account = bank এর master key
সব কিছুর access আছে, কোনো restriction নেই
হ্যাক হলে সব শেষ

IAM user = নির্দিষ্ট কাজের জন্য আলাদা চাবি
শুধু যতটুকু দরকার ততটুকু permission
```

### IAM User তৈরির Steps

```
AWS Console → IAM → Users → Create user

Step 1: User details
  User name: sangam-devops  (যেকোনো নাম)

Step 2: Permissions
  "Attach policies directly" select করো
  এই policies গুলো দাও:

  ✓ AmazonEKSClusterPolicy
  ✓ AmazonEKSWorkerNodePolicy
  ✓ AmazonEC2FullAccess
  ✓ IAMFullAccess
  ✓ AmazonVPCFullAccess
  ✓ CloudFormationFullAccess
  ✓ AmazonEKSServicePolicy

  (অথবা শেখার জন্য সহজে: AdministratorAccess একটাই দাও)

Step 3: Create user → done
```

### Access Key তৈরি করো

```
IAM → Users → sangam-devops → Security credentials tab
→ Create access key
→ Use case: Command Line Interface (CLI)
→ Next → Create

দেখাবে:
  Access key ID:     AKIAIOSFODNN7EXAMPLE
  Secret access key: wJalrXUtnFEMI/K7MDENG/...  ← এটা একবারই দেখায়, save করো!
```

### aws configure আপডেট করো

```powershell
aws configure
# নতুন IAM user এর key দাও

# verify
aws sts get-caller-identity
# Arn তে "root" না দেখিয়ে user name দেখাবে:
# "Arn": "arn:aws:iam::820028473878:user/sangam-devops"
```

---

## সবার আগে বোঝো — কোথায় কাজ করবে

```
kubeadm (আগে করেছিলে):
  EC2 তে SSH করে master node এ কাজ করতে হতো
  kubectl ছিল EC2 এর ভেতরে

EKS (এখন):
  তোমার নিজের Windows laptop এ বসে কাজ করবে
  AWS CLI + eksctl + kubectl — সব laptop এ install করবে
  laptop থেকেই AWS এ EKS cluster তৈরি হবে
  EC2 তে SSH করতে হবে না
```

---

## Prerequisites — তোমার Windows Laptop এ

```
১. AWS Account + IAM user (Admin permission)
২. AWS CLI — laptop এ install
৩. eksctl  — laptop এ install
৪. kubectl — laptop এ install
```

### AWS CLI install (Windows)

```
১. এই URL থেকে installer download করো:
   https://awscli.amazonaws.com/AWSCLIV2.msi

২. .msi file run করো → Next → Install

৩. verify করো (PowerShell বা CMD):
   aws --version
```

### eksctl install (Windows)

```
Chocolatey দিয়ে (সহজ):
  choco install eksctl

অথবা manual:
  https://github.com/eksctl-io/eksctl/releases
  → eksctl_Windows_amd64.zip download করো
  → unzip করো
  → eksctl.exe কে C:\Windows\System32\ এ রাখো

verify:
  eksctl version
```

### kubectl install (Windows)

```
Chocolatey দিয়ে:
  choco install kubernetes-cli

অথবা:
  https://dl.k8s.io/release/v1.32.0/bin/windows/amd64/kubectl.exe
  → download করো
  → C:\Windows\System32\ এ রাখো

verify:
  kubectl version --client
```

### Chocolatey কি না থাকলে

```
PowerShell (Admin হিসেবে run করো):
  Set-ExecutionPolicy Bypass -Scope Process -Force
  [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
  iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))

তারপর নতুন PowerShell খুলে:
  choco install eksctl
  choco install kubernetes-cli
  choco install awscli
```

### AWS Configure

```powershell
aws configure
```

দেবে:
```
AWS Access Key ID:     → IAM user এর Access Key
AWS Secret Access Key: → IAM user এর Secret Key
Default region name:   ap-southeast-1  (বা তোমার region)
Default output format: json
```

**IAM user এর Access Key কোথায় পাবে:**
```
AWS Console → IAM → Users → তোমার user
→ Security credentials tab
→ Create access key → CLI select করো
→ Key ID + Secret দেখাবে (Secret একবারই দেখায়, save করো)
```

---

## Step 1: EKS Cluster তৈরি করো

```bash
eksctl create cluster \
  --name sangam-cluster \
  --region ap-southeast-1 \
  --nodegroup-name sangam-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 1 \
  --nodes-max 4 \
  --managed
```

**এই command কি করে:**
```
--name          → cluster এর নাম
--region        → AWS region (তোমার কাছের region দাও)
--node-type     → worker node এর EC2 type
                  t3.medium = 2 CPU, 4GB RAM (minimum recommended)
--nodes         → শুরুতে কতটা worker
--nodes-min/max → Cluster Autoscaler এর জন্য range
--managed       → AWS নিজে node manage করবে (Node Group)

সময় লাগবে: ~15-20 মিনিট
```

**verify:**
```bash
kubectl get nodes
# দেখাবে 2টা worker node Ready
```

---

## Step 2: kubectl কে EKS এ connect করো

```bash
aws eks update-kubeconfig \
  --region ap-southeast-1 \
  --name sangam-cluster

# verify
kubectl cluster-info
kubectl get nodes
```

**এটা কি করে:**
```
~/.kube/config file এ EKS cluster এর credentials যোগ করে
এরপর kubectl command EKS cluster এ কাজ করবে
```

---

## Step 3: তোমার App Deploy করো

তোমার k8s/ yaml গুলো হুবহু একই কাজ করবে।

```bash
# repo clone করো (নতুন machine হলে)
git clone https://github.com/abuabddullah/sangam_devops_part3_k8s
cd sangam_devops_part3_k8s

# সব yaml apply করো
kubectl apply -f k8s/

# verify
kubectl get pods
kubectl get svc
```

**NodePort vs LoadBalancer:**
```
kubeadm এ: NodePort ব্যবহার করেছিলে (manual IP:port)
EKS এ:     LoadBalancer type ব্যবহার করো
           AWS নিজে একটা ALB/NLB তৈরি করবে
           public URL পাবে automatically

service yaml বদলাও:
  type: NodePort    → type: LoadBalancer
```

**LoadBalancer URL পাবে:**
```bash
kubectl get svc sangam-client-service
# EXTERNAL-IP column এ AWS এর URL আসবে
# e.g: abc123.ap-southeast-1.elb.amazonaws.com
```

---

## Step 4: ArgoCD Install করো (EKS এ)

kubeadm এর মতোই — হুবহু একই command:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f \
  https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# wait for pods
kubectl get pods -n argocd --watch
```

**EKS এ UI access (LoadBalancer দিয়ে):**
```bash
kubectl patch svc argocd-server -n argocd \
  -p '{"spec": {"type": "LoadBalancer"}}'

kubectl get svc argocd-server -n argocd
# EXTERNAL-IP এ URL আসবে
```

Browser: `https://<EXTERNAL-IP>`

**Password:**
```bash
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d && echo
```

**ArgoCD Application:**
```bash
kubectl apply -f k8s/argocd-application.yaml
# এটা হুবহু আগেরটাই কাজ করবে
```

---

## Step 5: Cluster Autoscaler (EKS এ সহজ)

kubeadm এ CA জটিল ছিল — EKS এ অনেক সহজ।

```bash
# OIDC provider enable করো (একবার)
eksctl utils associate-iam-oidc-provider \
  --region ap-southeast-1 \
  --cluster sangam-cluster \
  --approve

# CA এর IAM policy তৈরি করো
eksctl create iamserviceaccount \
  --cluster sangam-cluster \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::aws:policy/AutoScalingFullAccess \
  --approve

# CA install করো
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# cluster name set করো
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"

kubectl -n kube-system edit deployment.apps/cluster-autoscaler
# command section এ যোগ করো:
# - --node-group-auto-discovery=asg:tag=k8s.io/cluster-autoscaler/enabled,k8s.io/cluster-autoscaler/sangam-cluster
```

---

## Step 6: HPA (EKS এ একই)

metrics-server install করো (EKS এ আলাদা করে লাগে):

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# verify
kubectl get deployment metrics-server -n kube-system
```

তোমার HPA yaml হুবহু একই কাজ করবে:
```bash
kubectl apply -f k8s/server-hpa.yaml
kubectl apply -f k8s/client-hpa.yaml
kubectl get hpa
```

---

## Cost সম্পর্কে

```
EKS cluster fee:  $0.10/hour = ~$72/month
Worker nodes:     t3.medium × 2 = ~$60/month
Total:            ~$130/month (minimum)

বন্ধ করতে চাইলে:
  eksctl delete cluster --name sangam-cluster
  → সব কিছু delete হবে (EC2, Load Balancer, সব)

শুধু node বন্ধ রাখতে:
  eksctl scale nodegroup --cluster sangam-cluster \
    --name sangam-workers --nodes 0
  → EC2 বন্ধ কিন্তু cluster থাকবে ($0.10/hour চলবে)
```

---

## Quick Reference — kubeadm থেকে EKS পার্থক্য

```
                  kubeadm              EKS
──────────────────────────────────────────────────────
Cluster তৈরি     manual               eksctl create cluster
kubectl connect  ~/.kube/config       aws eks update-kubeconfig
Service access   NodePort (IP:port)   LoadBalancer (AWS URL)
CA setup         জটিল (manual IAM)   সহজ (eksctl iamserviceaccount)
metrics-server   আগে থেকে ছিল        আলাদা install লাগে
App yaml         একই                  একই
ArgoCD           একই                  একই
HPA yaml         একই                  একই
```

---

## Related Notes

- znotes/003_k8s-autoscaling-guide.md — HPA/CA details
- znotes/004_argocd-guide.md — ArgoCD setup
- znotes/002_k8s-deployment-service-guide.md — Deployment/Service basics
