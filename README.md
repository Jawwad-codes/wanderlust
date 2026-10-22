# Wanderlust: DevSecOps + GitOps Pipeline on a Self-Hosted Kubernetes Cluster (KVM)

Wanderlust is an open-source MERN travel blog. This repository deploys it through a **DevSecOps CI pipeline (Jenkins)** and a **GitOps CD flow (ArgoCD)** onto a **self-hosted Kubernetes cluster running on 5 KVM virtual machines**, with no managed cloud services involved.

> **Credits:** the application is [krishnaacharyaa/wanderlust](https://github.com/krishnaacharyaa/wanderlust). The pipeline design follows the community "Wanderlust Mega Project" by DevMadhup. This repo adapts that design from AWS EKS to an on-prem KVM environment.

> Replace every value in `<angle brackets>` with your own (IPs, usernames, tokens).

---

## Architecture

### Virtual machines (KVM / libvirt)

| VM | Role | Runs |
|----|------|------|
| `k8s-master` | Kubernetes control plane | kube-apiserver, etcd, scheduler, controller-manager |
| `k8s-worker-1` | Kubernetes worker | Wanderlust pods, ArgoCD, monitoring |
| `k8s-worker-2` | Kubernetes worker | Wanderlust pods, ArgoCD, monitoring |
| `jenkins-master` | CI server | Jenkins controller, SonarQube (Docker) |
| `jenkins-agent` | CI build node | Docker, Trivy, OWASP Dependency-Check, build jobs |

All VMs sit on one libvirt network (example: `192.168.122.0/24`).

### Delivery flow

```
Developer push
      |
      v
  GitHub repo ---> Jenkins master ---> Jenkins agent (build node)
                                          | 1. Checkout
                                          | 2. SonarQube code analysis
                                          | 3. OWASP dependency check
                                          | 4. Trivy filesystem scan
                                          | 5. Docker build + push
                                          | 6. Update image tag in k8s manifests (Git commit)
                                          v
                                   GitHub repo (manifests)
                                          |
                                          v
                              ArgoCD (inside the cluster) detects change
                                          |
                                          v
                        Rolling update on kubeadm cluster (2 workers)
```

## Tech stack

| Area | Tool |
|------|------|
| Virtualization | KVM, libvirt |
| Container runtime | Docker (CI), containerd (cluster) |
| Orchestration | Kubernetes (kubeadm), Flannel CNI |
| CI | Jenkins (master + SSH agent) |
| Code quality | SonarQube |
| Security | OWASP Dependency-Check, Trivy |
| CD / GitOps | ArgoCD |
| Monitoring | Prometheus + Grafana (Helm, kube-prometheus-stack) |
| App | MERN (MongoDB, Express, React, Node.js) + Redis |

---

## Prerequisites

- Linux host with virtualization enabled (`egrep -c '(vmx|svm)' /proc/cpuinfo` > 0)
- `qemu-kvm`, `libvirt`, `virt-manager` installed
- Ubuntu Server ISO (22.04 LTS recommended) for all VMs
- Suggested sizing (adjust to your host): 2 vCPU / 4 GB RAM / 30 GB disk per VM
- A GitHub account, a Docker Hub account, and a GitHub Personal Access Token

---

## 1. Create the VMs

Create the five VMs with `virt-manager` or `virt-install`, for example:

```bash
virt-install \
  --name k8s-master \
  --memory 4096 --vcpus 2 \
  --disk size=30 \
  --cdrom /path/to/ubuntu-22.04-live-server-amd64.iso \
  --os-variant ubuntu22.04 \
  --network network=default
```

Repeat for `k8s-worker-1`, `k8s-worker-2`, `jenkins-master`, `jenkins-agent`.

Give every VM a **static IP** (netplan or a libvirt DHCP reservation) and add all hosts to `/etc/hosts` on each VM:

```
<192.168.56.10> k8s-master
<192.168.56.11> k8s-worker-1
<192.168.56.12> k8s-worker-2
<192.168.56.15> jenkins-master
<192.168.56.16> jenkins-agent
```

---

## 2. Build the Kubernetes cluster (kubeadm)

### 2.1 On `k8s-master`, `k8s-worker-1`, `k8s-worker-2`

Disable swap and enable required kernel modules:

```bash
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF
sudo modprobe overlay && sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF
sudo sysctl --system
```

Install containerd:

```bash
sudo apt-get update
sudo apt-get install -y containerd
sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' /etc/containerd/config.toml
sudo systemctl restart containerd && sudo systemctl enable containerd
```

Install kubeadm, kubelet, kubectl:

```bash
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.30/deb/Release.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.30/deb/ /' | \
  sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

### 2.2 Initialize the control plane (`k8s-master` only)

> The pod CIDR must **not** overlap with the libvirt network (`192.168.122.0/24`), so Flannel's `10.244.0.0/16` is used.

```bash
sudo kubeadm init \
  --apiserver-advertise-address=<k8s-master-ip> \
  --pod-network-cidr=10.244.0.0/16

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 2.3 Join the workers

Run the `kubeadm join ...` command printed by `kubeadm init` on `k8s-worker-1` and `k8s-worker-2` (with `sudo`). If you lost it:

```bash
kubeadm token create --print-join-command
```

Verify from the master:

```bash
kubectl get nodes -o wide
```

All three nodes should be `Ready`.

---

## 3. Jenkins master (`jenkins-master`)

```bash
sudo apt update -y
sudo apt install -y fontconfig openjdk-17-jre

sudo wget -O /usr/share/keyrings/jenkins-keyring.asc \
  https://pkg.jenkins.io/debian-stable/jenkins.io-2023.key
echo "deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian-stable binary/" | \
  sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install -y jenkins
```

Open `http://<jenkins-master-ip>:8080` and finish the setup wizard.

### SonarQube (on `jenkins-master`)

SonarQube needs a higher `vm.max_map_count`:

```bash
sudo sysctl -w vm.max_map_count=262144
echo "vm.max_map_count=262144" | sudo tee -a /etc/sysctl.conf

sudo apt-get install -y docker.io
docker run -itd --name SonarQube-Server -p 9000:9000 sonarqube:lts-community
```

---

## 4. Jenkins agent (`jenkins-agent`)

```bash
sudo apt update -y
sudo apt install -y fontconfig openjdk-17-jre docker.io wget gnupg lsb-release apt-transport-https

# dedicated CI user (instead of root)
sudo useradd -m -s /bin/bash jenkins
sudo usermod -aG docker jenkins
```

Install Trivy:

```bash
wget -qO - https://aquasecurity.github.io/trivy-repo/deb/public.key | sudo apt-key add -
echo deb https://aquasecurity.github.io/trivy-repo/deb $(lsb_release -sc) main | \
  sudo tee /etc/apt/sources.list.d/trivy.list
sudo apt-get update -y && sudo apt-get install -y trivy
```

Connect the agent over SSH:

1. On `jenkins-master`: `sudo -u jenkins ssh-keygen`
2. Copy `~jenkins/.ssh/id_rsa.pub` into `/home/jenkins/.ssh/authorized_keys` on `jenkins-agent`.
3. In Jenkins: **Manage Jenkins → Nodes → New Node**
   - Name: `agent`, Type: Permanent Agent, Executors: 2, Label: `agent`
   - Usage: *Only build jobs with label expressions matching this node*
   - Launch method: *Via SSH*, Host: `<jenkins-agent-ip>`
   - Credentials: SSH username with private key (user `jenkins`)
   - Host key verification: choose a strategy that fits your lab (known-hosts is preferred)

Using the `docker` group (instead of `chmod 777 /var/run/docker.sock`) gives the `jenkins` user Docker access without opening the socket to every user on the VM.

---

## 5. Configure Jenkins

1. **Plugins** (Manage Jenkins → Plugins): OWASP Dependency-Check, SonarQube Scanner, Docker, Pipeline: Stage View.
2. **Tools**: add OWASP Dependency-Check and SonarQube Scanner installations.
3. **SonarQube token**: SonarQube → Administration → Security → Users → Tokens. Add it to Jenkins credentials.
4. **SonarQube server**: Manage Jenkins → System → SonarQube installations (`http://<jenkins-master-ip>:9000`).
5. **SonarQube webhook**: SonarQube → Administration → Webhooks → `http://<jenkins-master-ip>:8080/sonarqube-webhook/`
6. **Credentials**: GitHub (username + Personal Access Token), Docker Hub login.
7. **Global Trusted Pipeline Library**: Manage Jenkins → System → Global Trusted Pipeline Libraries (point to the shared library repo used by the Jenkinsfile).

---

## 6. Pipelines

Create two pipeline jobs:

- **Wanderlust-CI**: checkout, SonarQube analysis, OWASP dependency check, Trivy filesystem scan, Docker build and push.
- **Wanderlust-CD**: updates the image tag in the Kubernetes manifests and pushes the commit that ArgoCD watches.

Before the first run, update the scripts in the repo's `Automations/` directory (`updatefrontendnew.sh`, `updatebackendnew.sh`). The original versions look up an AWS EC2 instance ID. In this on-prem setup, replace that lookup with the **IP of a Kubernetes worker node** (`<k8s-worker-ip>`), since the frontend reaches the backend through the worker's NodePort.

---

## 7. ArgoCD (inside the cluster)

Run on `k8s-master`:

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
watch kubectl get pods -n argocd

# expose the UI with NodePort (lab environment)
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'
kubectl get svc -n argocd

# initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

Open `https://<k8s-worker-ip>:<nodeport>`, log in as `admin`, and change the password.

Because ArgoCD runs **in the same cluster** it deploys to, there is no need to register an external cluster. Use the in-cluster destination `https://kubernetes.default.svc`.

Then in the UI:

1. **Settings → Repositories → Connect Repo** (HTTPS + Personal Access Token).
2. **Applications → New App**: point it at the manifests path, destination `https://kubernetes.default.svc`, enable **Auto-Create Namespace** and automated sync.

Access the app:

```
http://<k8s-worker-ip>:31000   # frontend
http://<k8s-worker-ip>:31100   # backend API
```

If `ufw` is enabled on the VMs, allow these NodePorts.

---

## 8. Email notifications (optional)

1. Enable 2-step verification on the Gmail account and create an **App Password**.
2. Add it to Jenkins credentials.
3. Manage Jenkins → System → *Extended E-mail Notification* and *E-mail Notification* → SMTP `smtp.gmail.com`, port `465`, SSL enabled.

---

## 9. Monitoring (Prometheus + Grafana via Helm)

On `k8s-master`:

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh && ./get_helm.sh

helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create namespace prometheus
helm install stable prometheus-community/kube-prometheus-stack -n prometheus
kubectl get pods -n prometheus
```

Expose Grafana (and Prometheus) with NodePort:

```bash
kubectl edit svc stable-grafana -n prometheus                       # type: NodePort
kubectl edit svc stable-kube-prometheus-sta-prometheus -n prometheus
kubectl get svc -n prometheus
```

Grafana admin password:

```bash
kubectl get secret --namespace prometheus stable-grafana \
  -o jsonpath="{.data.admin-password}" | base64 --decode; echo
```

Log in as `admin` and open the built-in Kubernetes dashboards.

---

## Differences from a managed-cloud (EKS) setup

| Topic | Managed cloud | This project |
|-------|---------------|--------------|
| Control plane | Managed by provider | Self-built with kubeadm, operated by me |
| Networking | VPC, cloud load balancers | libvirt network, Flannel CNI, NodePort |
| Access control | IAM roles | Linux users, SSH keys, Kubernetes RBAC |
| Machines | EC2 instances | KVM virtual machines |
| Cost | Pay per hour | Runs on local hardware |

## Known limitations and improvements

- NodePort exposure is for a lab. A production setup would use an Ingress controller (and MetalLB for LoadBalancer services) with TLS.
- Secrets live in Jenkins credentials and Kubernetes Secrets (base64, not encrypted by default). Enabling etcd encryption or an external secret store would harden this.
- Single control-plane node means no control-plane high availability.
- No persistent storage class is configured, so monitoring data is not persisted across pod rescheduling.

## Clean up

```bash
# on each Kubernetes node
sudo kubeadm reset

# on the KVM host
virsh destroy <vm-name>
virsh undefine <vm-name> --remove-all-storage
```

## Screenshots

Add screenshots to a `docs/` folder and link them here: Jenkins pipeline stage view, SonarQube dashboard, ArgoCD application, `kubectl get nodes`, Grafana dashboard.

## License

Application code belongs to its original authors under their license. Pipeline documentation and automation in this repository are provided for learning purposes.
