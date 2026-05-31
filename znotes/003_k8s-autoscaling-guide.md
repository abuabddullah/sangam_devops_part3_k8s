# K8s Autoscaling — Complete Guide

তোমার cluster: 1 master + 2 worker (AWS EC2, kubeadm)
তোমার project: MERN (sangam-server + sangam-client + MongoDB)

---

## Mental Model — Autoscaling কি

```
সমস্যা: রাত ১২টায় ১০ জন user, দুপুরে ১০,০০০ জন user
         static replicas: 2 → দুপুরে slow/crash

Solution: traffic দেখে নিজে নিজে বাড়াও-কমাও
```

---

## ৩ ধরনের Autoscaling

```
HPA   Horizontal Pod Autoscaler
        → Pod এর সংখ্যা বাড়ায়/কমায়
        → CPU 80% হলে replica 2 → 5
        → metrics-server দরকার

VPA   Vertical Pod Autoscaler
        → একটা Pod কে বেশি CPU/RAM দেয়
        → production এ জটিল, কম ব্যবহার হয়

CA    Cluster Autoscaler
        → Node (EC2) বাড়ায়/কমায়
        → সব pod schedule হচ্ছে না → নতুন node চালু
        → cloud provider integration লাগে
```

---

## Analogy — Office & Desk

```
Node    = office এর desk
Pod     = desk এ বসা কর্মী
Traffic = কাজের চাপ

কাজ বাড়লে:

HPA              → একই ৩ desk এ বেশি কর্মী বসাও (pod বাড়াও)
Cluster Autoscaler → নতুন desk কিনে আনো (node বাড়াও)

HPA র limit:
  ৩ desk এ ১৫ জন বসানো সম্ভব না
  তখন Cluster Autoscaler দরকার
```

---

## কোনটা কখন

```
পরিস্থিতি                              কোনটা
──────────────────────────────────────────────────────────────
traffic spike, node এ resource আছে     HPA
node full, নতুন pod Pending            HPA + Cluster Autoscaler
AWS EKS managed cluster                CA সহজে setup হয়
kubeadm manual cluster (তোমারটা)       HPA সহজ, CA জটিল
cost control (idle node বন্ধ করা)      Cluster Autoscaler
production standard                    HPA + CA একসাথে
```

---

## Hybrid: HPA + CA একসাথে (Production Standard)

```
Traffic বাড়লে:
  HPA দেখে CPU বেশি → pod বাড়াও
    ↓
  node তে আর জায়গা নেই → নতুন pod Pending
    ↓
  CA দেখে pod Pending → নতুন EC2 চালু করো
    ↓
  নতুন node ready → pending pod সেখানে schedule হয়

Traffic কমলে:
  HPA দেখে CPU কম → pod কমাও
    ↓
  কিছু node idle → CA দেখে → সেই node বন্ধ করো
    ↓
  cost কমে
```

**এই দুটো একে অপরের পরিপূরক — একটা pod স্তরে, আরেকটা node স্তরে।**

---

## HPA — How It Works

```
metrics-server প্রতি 15 সেকেন্ডে CPU/memory collect করে
  ↓
HPA controller সেটা দেখে
  ↓
target threshold পার হলে → replicas বাড়াও
  ↓
Deployment এ replicas update করে
  ↓
নতুন Pod তৈরি হয়
```

### HPA yaml

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: sangam-server-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: sangam-server       # কোন Deployment scale করবে
  minReplicas: 2              # কমপক্ষে কতটা Pod থাকবে
  maxReplicas: 10             # বেশিতে কতটা Pod হবে
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70    # CPU 70% হলে scale up
```

### resources এর values এর মানে

**CPU — "m" মানে millicores:**

```
1 CPU core = 1000m

মনে করো ১টা পিৎজা = ১ CPU core
পিৎজাকে ১০০০ টুকরো করো → প্রতিটা টুকরো = 1m

requests: cpu: "100m"  → ১০০ টুকরো সবসময় দেওয়া থাকবে (guarantee)
limits:   cpu: "500m"  → ৫০০ টুকরোর বেশি নেওয়া যাবে না (cap)
```

**Memory — "Mi" মানে Mebibytes (≈ MB):**

```
128Mi ≈ 128 MB RAM
512Mi ≈ 512 MB RAM

requests: memory: "128Mi"  → এই pod এর জন্য 128MB সবসময় reserve
limits:   memory: "512Mi"  → এর বেশি নিলে pod kill হয় (OOMKilled)
```

**requests vs limits পার্থক্য:**

```
                requests                    limits
                ──────────────────────────────────────────
মানে            minimum guarantee           maximum allowed
না পেলে         pod schedule হবে না         cpu throttle / memory kill
HPA দেখে        শুধু requests দেখে %        requests এর উপর ভিত্তি করে
```

**HPA % হিসাব কিভাবে করে:**

```
requests cpu: 100m
averageUtilization: 70%

মানে: 100m এর 70% = 70m

সব pod এর গড় CPU > 70m → scale up
সব pod এর গড় CPU < 70m → scale down
```

**Project এর values কেন এগুলো:**

```
sangam-server (Express API):
  requests cpu: 100m   → db query, business logic — একটু বেশি লাগে
  limits   cpu: 500m   → traffic spike এ নিতে পারবে
  memory:  128Mi/512Mi → Node.js এ যথেষ্ট

sangam-client (nginx static):
  requests cpu: 50m    → শুধু file serve — খুব কম লাগে
  limits   cpu: 200m   → কখনো বেশি লাগে না
  memory:  64Mi/256Mi  → nginx অনেক কম খায়
```

### HPA কাজ করার শর্ত

```
Deployment এ resources দিতে হবে:

containers:
  - name: sangam-server
    resources:
      requests:
        cpu: "100m"       # ← এটা না থাকলে HPA কাজ করে না
        memory: "128Mi"
      limits:
        cpu: "500m"
        memory: "512Mi"
```

### kubectl commands

```bash
kubectl get hpa                          # HPA status দেখো
kubectl describe hpa sangam-server-hpa   # detail দেখো
kubectl top pods                         # Pod এর CPU/memory usage দেখো
kubectl top nodes                        # Node এর usage দেখো
```

---

## Cluster Autoscaler — Mindmap (নিজে শেখার জন্য)

### CA কিভাবে কাজ করে

```
Cluster Autoscaler
│
├── Watch করে: Pending pods (schedule হচ্ছে না কারণ node full)
│
├── Scale UP:
│     Pending pod দেখলে → AWS কে বলো নতুন EC2 চালু করো
│     নতুন node join করলে → pod সেখানে schedule হয়
│
└── Scale DOWN:
      কোনো node এ 10 মিনিট ধরে কম কাজ → node drain করো
      pod অন্য node এ সরাও → EC2 terminate করো
```

### CA এর Prerequisites

```
1. AWS Auto Scaling Group (ASG)
     → EC2 গুলো ASG এর ভেতরে থাকতে হবে
     → min/max capacity set করতে হবে

2. IAM Role/Policy
     → CA কে AWS API call করার permission দিতে হবে
     → autoscaling:DescribeAutoScalingGroups
     → autoscaling:SetDesiredCapacity
     → ec2:DescribeLaunchTemplateVersions

3. Node Labels
     → ASG এর node এ cluster-autoscaler label থাকতে হবে

4. CA কোথায় চলবে
     → master node এ একটা Deployment হিসেবে
```

### CA Install করার Steps (AWS kubeadm cluster)

```
Step 1: AWS Console এ Auto Scaling Group check করো
        worker node গুলো ASG এ আছে কিনা দেখো

Step 2: IAM Policy তৈরি করো
        CA এর জন্য নতুন policy:
        {
          "Version": "2012-10-17",
          "Statement": [{
            "Effect": "Allow",
            "Action": [
              "autoscaling:DescribeAutoScalingGroups",
              "autoscaling:DescribeAutoScalingInstances",
              "autoscaling:DescribeLaunchConfigurations",
              "autoscaling:SetDesiredCapacity",
              "autoscaling:TerminateInstanceInAutoScalingGroup",
              "ec2:DescribeLaunchTemplateVersions"
            ],
            "Resource": "*"
          }]
        }

Step 3: IAM Role তৈরি করো
        → EC2 instance role (worker node এর role)
        → উপরের policy attach করো

Step 4: ASG এ tag দাও (CA এটা দেখে চেনে)
        Key:   k8s.io/cluster-autoscaler/<cluster-name>
        Value: owned
        Key:   k8s.io/cluster-autoscaler/enabled
        Value: true

Step 5: CA Deployment apply করো
        Official yaml:
        https://github.com/kubernetes/autoscaler/blob/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

        বদলাতে হবে:
          - <YOUR CLUSTER NAME> → তোমার cluster name
          - image version → তোমার K8s version এর সাথে match করাও

Step 6: Verify করো
        kubectl get pods -n kube-system | grep cluster-autoscaler
        kubectl logs -n kube-system deployment/cluster-autoscaler
```

### CA Deployment এর key parts

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cluster-autoscaler
  namespace: kube-system
spec:
  template:
    spec:
      containers:
        - name: cluster-autoscaler
          image: registry.k8s.io/autoscaling/cluster-autoscaler:v1.32.0
          command:
            - ./cluster-autoscaler
            - --cloud-provider=aws
            - --nodes=1:10:<ASG-NAME>    # min:max:asg-name
            - --scale-down-enabled=true
```

### CA এর Common Issues

```
"No candidates for scale-up"
  → ASG এর max capacity reached
  → AWS Console এ ASG এর max বাড়াও

"Scale-down blocked"
  → Pod এ PodDisruptionBudget আছে
  → অথবা pod annotation: cluster-autoscaler.kubernetes.io/safe-to-evict: "false"

CA pod নেই বা Crash
  → IAM permission ঠিক নেই
  → kubectl logs দেখো
```

---

## তোমার Project এর Autoscaling Plan

```
এখন:     HPA শেখো + implement করো (metrics-server আছে)
পরে EKS:  Cluster Autoscaler add করো (AWS managed হলে সহজ)
```

---

## Related Notes

- znotes/002_k8s-deployment-service-guide.md — Deployment/Service basics
- znotes/001_k8s_cluster_setup_2026.md — Cluster setup
