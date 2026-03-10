# k8s-observability-platform

A production-grade Kubernetes observability platform built entirely on local VMs — no cloud account, no cost. Provisions a multi-node K8s cluster from scratch using Vagrant and Ansible, then layers on GitOps-based app deployment and a full Prometheus + Grafana monitoring stack.

**[Full Documentation →](https://imsiddhant.github.io/k8s-observability-platform/)**

---

## What You Get

| URL | What it is |
|---|---|
| `http://app.lab.local` | Google Online Boutique — 11-microservice e-commerce demo |
| `http://argocd.lab.local` | ArgoCD — GitOps dashboard, synced to GitHub |
| `http://grafana.lab.local` | Grafana — pre-built K8s dashboards, 30+ panels |
| `http://prometheus.lab.local` | Prometheus — live metrics, PromQL queries |
| `http://alertmanager.lab.local` | Alertmanager — alert routing and silences |

All accessible from your browser. All running locally. Total cost: **$0**.

---

## Architecture

```
Your machine (Windows / macOS / Linux)
│
├── VirtualBox
│   ├── k8s-master    192.168.56.10   3 CPU  3 GB RAM
│   ├── k8s-worker-1  192.168.56.11   3 CPU  3 GB RAM
│   └── k8s-worker-2  192.168.56.12   3 CPU  3 GB RAM
│
└── MetalLB LoadBalancer IP: 192.168.56.200
    └── NGINX Ingress → routes *.lab.local hostnames to services
```

**Stack per phase:**

```
Phase 01 — Foundation      Vagrant · Ansible · kubeadm · Calico CNI
Phase 02 — Networking      MetalLB · NGINX Ingress Controller
Phase 03 — GitOps          ArgoCD · Google Online Boutique (11 microservices)
Phase 04 — Observability   Prometheus · Grafana · Alertmanager · node-exporter
```

---

## Prerequisites

| Tool | Version | Purpose |
|---|---|---|
| [VirtualBox](https://www.virtualbox.org/wiki/Downloads) | 7.x | VM hypervisor |
| [Vagrant](https://developer.hashicorp.com/vagrant/downloads) | 2.x | VM lifecycle management |
| Ansible | 2.15+ | Cluster provisioning (via WSL on Windows) |

**Minimum host machine specs:** 16 GB RAM, 8-core CPU, 50 GB free disk.
The 3 VMs together use 9 GB RAM and ~30 GB disk.

**Windows users:** Install [WSL 2](https://learn.microsoft.com/en-us/windows/wsl/install) and run all Ansible commands from inside WSL.

```bash
# Install Ansible inside WSL (Ubuntu)
sudo apt update && sudo apt install -y ansible
```

---

## Quick Start

### 1. Clone the repo

```bash
git clone https://github.com/imsiddhant/k8s-observability-platform.git
cd k8s-observability-platform
```

### 2. Start the VMs

```bash
cd vagrant
vagrant up
```

This provisions 3 Ubuntu 25.04 VMs with all kernel prerequisites for Kubernetes (swap disabled, required modules loaded). Takes ~5–10 minutes depending on download speed.

### 3. Run Phase 01 — Kubernetes cluster

```bash
cd ../ansible
ansible-playbook site.yml
```

Installs containerd, kubeadm, kubectl on all nodes, bootstraps the master, joins workers, and deploys Calico CNI. Takes ~10 minutes.

### 4. Run Phase 02 — Load balancer + Ingress

```bash
ansible-playbook phase-02.yml
```

Installs MetalLB (assigns real IPs to LoadBalancer services) and the NGINX Ingress Controller. At the end, Ansible prints the LoadBalancer IP — add it to your hosts file.

**Add to your hosts file** (`C:\Windows\System32\drivers\etc\hosts` on Windows, `/etc/hosts` on Linux/macOS):

```
192.168.56.200  argocd.lab.local
192.168.56.200  app.lab.local
192.168.56.200  grafana.lab.local
192.168.56.200  prometheus.lab.local
192.168.56.200  alertmanager.lab.local
```

### 5. Run Phase 03 — GitOps + Online Boutique

```bash
ansible-playbook phase-03.yml
```

Deploys ArgoCD and creates an Application that syncs Google's [Online Boutique](https://github.com/GoogleCloudPlatform/microservices-demo) directly from GitHub. ArgoCD continuously reconciles the cluster state with the repo. Takes ~15 minutes (ArgoCD needs to pull and deploy 11 microservices).

### 6. Run Phase 04 — Monitoring stack

```bash
ansible-playbook phase-04.yml
```

Deploys the full `kube-prometheus-stack` via Helm — Prometheus, Grafana, Alertmanager, node-exporter, and kube-state-metrics. Chart version is resolved automatically at deploy time. Takes ~10–15 minutes.

---

## Access

Once all 4 phases are complete:

| Service | URL | Credentials |
|---|---|---|
| Online Boutique | http://app.lab.local | — |
| ArgoCD | http://argocd.lab.local | `admin` / printed by Ansible |
| Grafana | http://grafana.lab.local | `admin` / `grafana` |
| Prometheus | http://prometheus.lab.local | — |
| Alertmanager | http://alertmanager.lab.local | — |

---

## Repo Structure

```
k8s-observability-platform/
├── vagrant/
│   └── Vagrantfile               # 3-node VM topology
├── ansible/
│   ├── site.yml                  # Phase 01 — cluster bootstrap
│   ├── phase-02.yml              # Phase 02 — MetalLB + NGINX Ingress
│   ├── phase-03.yml              # Phase 03 — ArgoCD + Online Boutique
│   ├── phase-04.yml              # Phase 04 — kube-prometheus-stack
│   ├── group_vars/all.yml        # All version pins and shared vars
│   ├── inventory/hosts.ini       # VM IP + SSH key mapping
│   └── roles/
│       ├── common/               # containerd, kubeadm, kubectl
│       ├── master/               # kubeadm init, Calico, kubeconfig
│       ├── worker/               # kubeadm join
│       ├── metallb/              # MetalLB + IP pool
│       ├── nginx-ingress/        # Helm install, wait for IP
│       ├── argocd/               # Install, insecure mode, Ingress
│       ├── online-boutique/      # ArgoCD Application CRD
│       └── monitoring/           # kube-prometheus-stack Helm chart
└── docs/                         # GitHub Pages documentation
```

---

## Tearing Down

```bash
cd vagrant
vagrant destroy -f
```

Deletes all 3 VMs and frees disk space. Run `vagrant up` again to start fresh.

---

## Documentation

Every design decision, configuration file, and "why" behind each component is documented at:

**https://imsiddhant.github.io/k8s-observability-platform/**

Covers:
- [Vagrantfile deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/vagrantfile.html) — VM topology, kernel prerequisites, bootstrap script
- [Ansible playbooks deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/ansible.html) — role structure, group_vars, idempotency patterns
- [MetalLB deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/metallb.html) — Layer 2 mode, IP pool, why LoadBalancer services need it locally
- [NGINX Ingress deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/nginx-ingress.html) — Ingress vs LoadBalancer, host-based routing, /etc/hosts
- [ArgoCD deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/argocd.html) — GitOps model, Application CRD, insecure mode, five components
- [Online Boutique deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/online-boutique.html) — 11 microservices, Istio exclusion fix, traffic flow
- [Prometheus & Alertmanager deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/prometheus.html) — pull model, kube-prometheus-stack, ServiceMonitor CRD, alert routing
- [Grafana deep dive](https://imsiddhant.github.io/k8s-observability-platform/docs/grafana.html) — pre-built dashboards, PromQL basics, auto-provisioned data source

---

## License

MIT
