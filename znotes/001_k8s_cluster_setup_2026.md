# Kubernetes Cluster Setup — 2026 Edition (v1.32)

> এই নোটটি আপনার পুরনো 2021 স্লাইডের ক্রম অনুসরণ করে বানানো, কিন্তু **2026 সালের জন্য আপডেট করা**।
> পুরনো স্লাইডের যেসব কমান্ড এখন আর কাজ করে না, সেগুলো ঠিক করে দেওয়া হয়েছে এবং ⚠️ দিয়ে চিহ্নিত করা হয়েছে।
> কমান্ডগুলো ধাপে ধাপে কপি-পেস্ট করলেই কাজ হবে।

---

## 🔴 2021 → 2026: যা যা বদলে গেছে (এক নজরে)

| জিনিস | 2021 (পুরনো স্লাইড) | 2026 (এখন যা লাগবে) |
|---|---|---|
| Container Runtime | `docker.io` install | **containerd** সরাসরি (Docker shim v1.24-এ বাদ) |
| K8s apt রিপো | `apt.kubernetes.io` / `packages.cloud.google.com` | **`pkgs.k8s.io`** (পুরনোটা ২০২৩-এ বন্ধ) ⚠️ |
| apt key যোগ করা | `apt-key add` | **`signed-by` + keyring** (`apt-key` deprecated) |
| Flannel লিঙ্ক | `coreos/flannel/master/...` | **`flannel-io/flannel/releases/latest`** (পুরনোটা 404) ⚠️ |
| Flannel namespace | `kube-system` | **`kube-flannel`** (আলাদা namespace) |
| cgroup driver | আলাদা সেট করা লাগত না | **`SystemdCgroup = true`** বাধ্যতামূলক |

> **মূল কথা:** পুরনো স্লাইড হুবহু ফলো করলে "Release file নেই" আর "404 Not Found" এরর পাবেন। নিচের কমান্ডগুলোই বর্তমানে কাজ করে।

---

## ভূমিকা: ক্লাস্টার সেটআপ ওভারভিউ

- ক্লাউড (AWS EC2) বা VirtualBox-এ VM স্পিন আপ করুন — কমপক্ষে ১টি master + ১টি worker।
- প্রতিটি মেশিনে container runtime (**containerd**) ইনস্টল করুন।
- প্রতিটি মেশিনে **kubeadm, kubelet, kubectl** ইনস্টল করুন।
- শুধু master-এ `kubeadm init` চালান।
- Pod network (**Flannel**) ইনস্টল করুন।
- worker নোডগুলো master-এর সাথে join করান।

> **AWS ব্যবহারকারীদের জন্য জরুরি:** master ও worker-এর **Security Group**-এ একে অপরের ট্রাফিক allow করুন। নাহলে join কমান্ড `[preflight] Running pre-flight checks`-এ আটকে যাবে। সহজ উপায়: VPC CIDR (যেমন `172.31.0.0/16`) থেকে আসা সব ট্রাফিক allow করুন, অথবা অন্তত পোর্ট `6443`, `10250`, `30000-32767`, এবং Flannel-এর জন্য UDP `8472` খুলুন।

---

## ধাপ ১: সব মেশিনে Prerequisites (master + প্রতিটি worker)

প্রথমে প্যাকেজ লিস্ট আপডেট করুন:

```bash
sudo apt-get update && sudo apt-get upgrade -y
```

### ১.১ — Swap বন্ধ করুন (এবং রিবুটেও বন্ধ রাখুন)

```bash
sudo swapoff -a
```

```bash
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
```

### ১.২ — প্রয়োজনীয় kernel মডিউল লোড করুন

```bash
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
```

```bash
sudo modprobe overlay && sudo modprobe br_netfilter
```

### ১.৩ — নেটওয়ার্কিং sysctl সেট করুন (Flannel ঠিকমতো চলার জন্য জরুরি)

```bash
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
```

```bash
sudo sysctl --system
```

---

## ধাপ ২: Container Runtime (containerd) — সব মেশিনে

> ⚠️ **পুরনো স্লাইডে `sudo apt-get install docker.io` ছিল।** Kubernetes v1.24 থেকে Docker shim বাদ দেওয়া হয়েছে — এখন **containerd** সরাসরি ব্যবহার হয়।

```bash
sudo apt-get install -y containerd
```

ডিফল্ট কনফিগ জেনারেট করুন:

```bash
sudo mkdir -p /etc/containerd && containerd config default | sudo tee /etc/containerd/config.toml > /dev/null
```

### ২.১ — `SystemdCgroup = true` সেট করুন (এটা না করলে kubelet ফেইল করবে)

```bash
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
```

ঠিকমতো সেট হয়েছে কিনা যাচাই করুন (`true` দেখা উচিত):

```bash
grep SystemdCgroup /etc/containerd/config.toml
```

containerd রিস্টার্ট ও enable করুন:

```bash
sudo systemctl restart containerd && sudo systemctl enable containerd
```

চালু আছে কিনা দেখুন:

```bash
sudo systemctl status containerd
```

---

## ধাপ ৩: Kubernetes ইনস্টল (kubeadm, kubelet, kubectl) — সব মেশিনে

> ⚠️ **এটাই পুরনো স্লাইডের সবচেয়ে বড় সমস্যা।** পুরনো `apt.kubernetes.io` রিপো ২০২৩ সালে বন্ধ হয়ে গেছে। এখন `pkgs.k8s.io` ব্যবহার করতে হবে, আর `apt-key` এর বদলে keyring।

প্রয়োজনীয় প্যাকেজ:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
```

keyring ডিরেক্টরি তৈরি করুন:

```bash
sudo mkdir -p -m 755 /etc/apt/keyrings
```

### ৩.১ — Signing key যোগ করুন (v1.32)

```bash
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.32/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
```

### ৩.২ — রিপো যোগ করুন (v1.32)

```bash
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.32/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
```

### ৩.৩ — ইনস্টল ও version lock করুন

```bash
sudo apt-get update && sudo apt-get install -y kubelet kubeadm kubectl
```

```bash
sudo apt-mark hold kubelet kubeadm kubectl
```

> `apt-mark hold` কেন? — যাতে `apt upgrade` করার সময় হঠাৎ Kubernetes আপগ্রেড হয়ে আপনার ক্লাস্টার ভেঙে না যায়।

ইনস্টল যাচাই করুন:

```bash
kubeadm version && kubelet --version && kubectl version --client
```

---

## ধাপ ৪: Master নোড সেটআপ (শুধু MASTER নোডে)

### ৪.১ — হোস্টনেম সেট করুন

প্রতিটি মেশিনে আলাদা হোস্টনেম (master নোডে):

```bash
sudo hostnamectl set-hostname master-node
```

worker নোডে (worker1-এ):

```bash
sudo hostnamectl set-hostname worker1
```

### ৪.২ — ক্লাস্টার initialize করুন (শুধু MASTER নোডে)

> ⚠️ **হার্ডওয়্যার রিকোয়ারমেন্ট:** master নোডে কমপক্ষে **২ CPU + ১৭০০ MB RAM** লাগে। AWS-এ `t2.micro`/`t3.micro` (1 CPU, ~1 GB) এই চেকে আটকাবে। স্থিতিশীল ক্লাস্টারের জন্য **`t3.small` (2 GB) সর্বনিম্ন, `t3.medium` (4 GB) আরামদায়ক।**

**সাধারণ কমান্ড (পর্যাপ্ত হার্ডওয়্যার থাকলে):**

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16
```

> `10.244.0.0/16` Flannel-এর ডিফল্ট CIDR — তাই এডিট ছাড়াই Flannel কাজ করবে।

**যদি ছোট ইনস্ট্যান্সে (লার্নিং/টেস্ট) চালাতে বাধ্য হন** — `NumCPU`/`Mem` এরর পেলে শুধু ওই দুটো বাইপাস করুন:

```bash
sudo kubeadm init --pod-network-cidr=10.244.0.0/16 --ignore-preflight-errors=NumCPU,Mem
```

> 🔴 **`all` দেবেন না, শুধু `NumCPU,Mem`।** পুরনো স্লাইডে `--ignore-preflight-errors=all` ছিল — কিন্তু `all` দিলে port conflict বা swap-এর মতো আসল সমস্যাও চাপা পড়ে যায়, যা পরে ভোগায়।
> ⚠️ **RAM কম থাকলে ঝুঁকি:** ১ GB RAM-এ pod গুলো `OOMKilled` হতে পারে বা ক্লাস্টার মাঝে মাঝে আটকে যেতে পারে। অদ্ভুত সমস্যা দেখলে RAM-ই কারণ — তখন ইনস্ট্যান্স বড় করুন।

> **ভুল করে আগে init চালিয়ে ব্যর্থ হলে**, আবার চেষ্টার আগে `sudo kubeadm reset -f` দিয়ে ক্লিন করে নিন (পরিশিষ্ট দেখুন)।

### ৪.৩ — kubectl কনফিগ রেডি করুন (MASTER নোডে)

```bash
mkdir -p $HOME/.kube && sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config && sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

> 🔑 **`init` শেষে আউটপুটে যে `kubeadm join ...` কমান্ডটা আসে, সেটা কপি করে রাখুন।** হারিয়ে গেলে চিন্তা নেই — ধাপ ৬-এ নতুন করে বের করার উপায় আছে।

---

## ধাপ ৫: Pod Network (Flannel) ইনস্টল (শুধু MASTER নোডে)

> ⚠️ **পুরনো স্লাইডের `coreos/flannel/master/...` লিঙ্ক এখন 404।** সঠিক লিঙ্ক:

```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

Flannel pod চালু হচ্ছে কিনা দেখুন (⚠️ নতুন namespace `kube-flannel`):

```bash
kubectl get pods -n kube-flannel -w
```

> সবগুলো `kube-flannel-ds-*` pod `1/1 Running` হলে `Ctrl+C` দিয়ে বেরিয়ে আসুন।

CoreDNS সহ system pod চেক করুন:

```bash
kubectl get pods -n kube-system
```

### 🔴 Flannel `CrashLoopBackOff`/`Error`-এ আটকালে (খুব কমন)

লক্ষণ: `kube-flannel-ds-*` বারবার ক্র্যাশ করছে, আর CoreDNS `ContainerCreating`-এ ঝুলে আছে। আগে লগ দেখুন:

```bash
kubectl logs -n kube-flannel -l app=flannel
```

লগে যদি এই এররটা থাকে —
```
Failed to check br_netfilter: stat /proc/sys/net/bridge/bridge-nf-call-iptables: no such file or directory
```
— তার মানে এই নোডে **`br_netfilter` মডিউল লোড হয়নি** (ধাপ ১.২/১.৩ এই নোডে চালানো হয়নি)। ফিক্স:

```bash
sudo modprobe overlay && sudo modprobe br_netfilter
```

```bash
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
```

```bash
echo -e "net.bridge.bridge-nf-call-iptables = 1\nnet.bridge.bridge-nf-call-ip6tables = 1\nnet.ipv4.ip_forward = 1" | sudo tee /etc/sysctl.d/k8s.conf && sudo sysctl --system
```

যাচাই করুন (`1` আসা উচিত):

```bash
cat /proc/sys/net/bridge/bridge-nf-call-iptables
```

তারপর ক্র্যাশ করা pod delete করুন (DaemonSet নিজেই নতুন বানাবে):

```bash
kubectl delete pod -n kube-flannel --all
```

> Flannel `Running` হলে CoreDNS-ও কয়েক সেকেন্ডে `Running`-এ চলে যাবে।
> লগে যদি `br_netfilter` নয় বরং OOM/মেমোরি দেখেন — তাহলে RAM কম, ইনস্ট্যান্স বড় করুন।

---

## ধাপ ৬: Worker নোড Join (worker1-এ)

### ৬.১ — Join টোকেন বের করুন (MASTER নোডে)

পুরনো join কমান্ড হারিয়ে গেলে, master-এ এই এক লাইনেই নতুন সম্পূর্ণ কমান্ড পাবেন:

```bash
kubeadm token create --print-join-command
```

> টোকেন ডিফল্টভাবে **২৪ ঘণ্টা** পর এক্সপায়ার হয়। তাই বের করার পরপরই ব্যবহার করুন।

### ৬.২ — Join করুন (worker1-এ, sudo দিয়ে)

উপরের আউটপুট থেকে পাওয়া কমান্ডটি (উদাহরণ):

```bash
sudo kubeadm join <MASTER_IP>:6443 --token <token> --discovery-token-ca-cert-hash sha256:<hash>
```

> আটকে গেলে বিস্তারিত লগের জন্য শেষে `--v=5` যোগ করুন।
> `This node has joined the cluster` দেখলে — সফল! ✅

### ৬.৩ — Join আটকে গেলে দ্রুত চেক (worker1-এ)

master-এর API পোর্টে কানেকশন আছে কিনা:

```bash
nc -zv <MASTER_IP> 6443
```

> `timed out` দেখালে → AWS Security Group / firewall-এ পোর্ট `6443` ব্লকড। (ভূমিকা সেকশন দেখুন)

---

## ধাপ ৭: ক্লাস্টার যাচাই (MASTER নোডে)

নোডগুলো join হয়েছে ও `Ready` কিনা:

```bash
kubectl get nodes -o wide
```

> প্রথমে `NotReady` দেখানো স্বাভাবিক — Flannel pod চালু হতে ১-২ মিনিট লাগে। লাইভ দেখতে:

```bash
watch kubectl get nodes
```

---

## ধাপ ৮: টেস্ট Nginx Pod (MASTER নোডে)

```bash
kubectl create deployment nginx-test --image=nginx --replicas=2
```

```bash
kubectl expose deployment nginx-test --port=80 --type=NodePort
```

pod কোন নোডে, কী অবস্থায়:

```bash
kubectl get pods -o wide
```

NodePort পোর্ট নম্বর (30000-32767):

```bash
kubectl get svc nginx-test
```

টেস্ট করুন:

```bash
curl http://<যেকোনো-নোডের-IP>:<NodePort>
```

> Nginx welcome পেজের HTML এলে — ক্লাস্টার সম্পূর্ণ সচল। ✅

টেস্ট শেষে ক্লিনআপ:

```bash
kubectl delete deployment nginx-test && kubectl delete svc nginx-test
```

---

## 🔧 পরিশিষ্ট: `sudo kubeadm reset -f` কখন ও কেন?

`kubeadm reset` একটা নোডকে তার Kubernetes সেটআপ থেকে একদম **ক্লিন (factory reset)** অবস্থায় ফিরিয়ে আনে — etcd ডেটা, কনফিগ ফাইল, certificate, manifest সব মুছে দেয়। `-f` মানে confirmation ছাড়াই force করে দেয়।

### কখন ব্যবহার করবেন:

1. **Join আটকে যাচ্ছে / অর্ধেক হয়ে ফেইল করেছে** — আগের ব্যর্থ চেষ্টার আবর্জনা (`/etc/kubernetes/`, certificate) থেকে গেলে নতুন join ব্যর্থ হয়। reset দিয়ে ক্লিন করে আবার join করলে কাজ হয়।

2. **ভুল করে worker-এ `kubeadm init` চালিয়ে ফেলেছেন** — তখন worker-এ control-plane কম্পোনেন্ট (etcd, admin.conf, pki) তৈরি হয়ে যায়, যা worker হিসেবে join আটকায়। reset সেগুলো মুছে ক্লিন worker বানায়।
   > 💡 **চিনবেন কীভাবে?** reset আউটপুটে যদি `etcd data directory`, `admin.conf`, `controller-manager.conf`, `scheduler.conf` মুছতে দেখেন — তার মানে ওই নোডে আসলে init চালানো হয়েছিল।

3. **নতুন করে শুরু করতে চান** — পুরো ক্লাস্টার ভেঙে ফেলে আবার গোড়া থেকে সেটআপ করতে।

### ⚠️ সতর্কতা:
- **MASTER নোডে `reset` মানে আপনার পুরো ক্লাস্টার মুছে যাওয়া।** শুধু worker রিসেট করতে চাইলে নিশ্চিত হোন আপনি worker নোডেই আছেন।
- reset সব কিছু পরিষ্কার করে না — কিছু ম্যানুয়ালি করতে হয়।

### reset-এর পর পূর্ণ ক্লিনআপ:

CNI কনফিগ:
```bash
sudo rm -rf /etc/cni/net.d
```

iptables রুল:
```bash
sudo iptables -F && sudo iptables -t nat -F && sudo iptables -t mangle -F && sudo iptables -X
```

পুরনো kubeconfig:
```bash
rm -rf $HOME/.kube
```

kubelet রিস্টার্ট:
```bash
sudo systemctl restart kubelet
```

---

## 🩺 দ্রুত ট্রাবলশুটিং রেফারেন্স

| লক্ষণ | সম্ভাব্য কারণ | সমাধান |
|---|---|---|
| `apt update`-এ "Release file নেই" | পুরনো `apt.kubernetes.io` রিপো | ধাপ ৩-এর `pkgs.k8s.io` ব্যবহার করুন |
| Flannel লিঙ্কে 404 | `coreos/flannel` সরানো হয়েছে | ধাপ ৫-এর `flannel-io` লিঙ্ক |
| join `preflight checks`-এ আটকে | পোর্ট `6443` ব্লকড | Security Group/firewall; `nc -zv` দিয়ে চেক |
| নোড অনেকক্ষণ `NotReady` | Flannel pod ওঠেনি | `kubectl get pods -n kube-flannel` |
| Flannel `CrashLoopBackOff` + লগে `br_netfilter` | মডিউল লোড হয়নি | ধাপ ৫-এর br_netfilter ফিক্স |
| `init`-এ `NumCPU`/`Mem` এরর | ইনস্ট্যান্স ছোট (১ CPU/<1.7GB) | ইনস্ট্যান্স বড় করুন; বা `--ignore-preflight-errors=NumCPU,Mem` |
| kubelet ফেইল করছে | swap চালু / cgroup ভুল | `swapoff -a`; `SystemdCgroup = true` |
| worker join বারবার ব্যর্থ | আগের আবর্জনা / ভুল init | `sudo kubeadm reset -f` দিয়ে ক্লিন |

worker-এ kubelet লগ দেখতে:
```bash
sudo journalctl -u kubelet -f --no-pager
```
