# K8S Fullstack Base

> Automated infrastructure platform for deploying a containerized web application on Kubernetes, provisioned entirely with Ansible and delivered through CI/CD.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Ansible](https://img.shields.io/badge/Ansible-2.14+-EE0000?logo=ansible&logoColor=white)](https://docs.ansible.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.35+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-29+-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

---

## Overview

This project is **not** about the application itself — it's about the **platform** that makes it run. The goal is to demonstrate a fully automated infrastructure pipeline:

1. **Provision** bare VMs into ready-to-use nodes (Ansible)
2. **Containerize** a sample application (Docker)
3. **Orchestrate** the deployment on a Kubernetes cluster (kubeadm)
4. **Route traffic** through an external Nginx reverse proxy into the cluster
5. **Automate** the entire build & deploy cycle (GitHub Actions)

```

                                    INFRASTRUCTURE FLOW
                                 (Ansible provisioning order)

  ┌───────────┐   ┌─────────┐   ┌─────────┐   ┌────────────┐   ┌────────────┐
  │ Bootstrap │──▶│  Nginx  │──▶│ Docker  │──▶│ K8S Prereqs│──▶│ K8S Master │
  │   (all)   │   │ (Proxy) │   │(Cluster)│   │  (Cluster) │   │ (+ Calico) │
  └───────────┘   └─────────┘   └─────────┘   └────────────┘   └─────┬──────┘
                                                                     │
                                                                     ▼
                                                              ┌────────────┐
                                                              │K8S Workers │
                                                              │  (Join)    │
                                                              └─────┬──────┘
                                                                    │
                                                                    ▼
                                                              ┌────────────┐
                                                              │  Ingress   │
                                                              │ Controller │
                                                              └────────────┘

```

## Architecture

```
                    ┌─────────────────────────────────────────────────────┐
                    │                    NGINX VM                         │
  Internet ───────▶ │  SSL Termination ─ Rate Limit ─ Security Headers    │
                    └──────────────────────┬──────────────────────────────┘
                                           │
                                      proxy_pass
                                           │
          ┌────────────────────────────────▼──────────────────────────────┐
          │                     KUBERNETES CLUSTER                        │
          │                                                               │
          │                    ┌───────────────────┐                      │
          │                    │ Ingress Controller│                      │
          │                    └────────┬──────────┘                      │
          │                             │                                 │
          │     ┌────────────┐     ┌────▼───────┐     ┌────────────┐      │
          │     │  App Pod   │     │  App Pod   │     │  App Pod   │      │
          │     └─────┬──────┘     └─────┬──────┘     └─────┬──────┘      │
          │           └──────────────────┼──────────────────┘             │
          │                              │                                │
          │                    ┌─────────▼─────────┐                      │
          │                    │    PostgreSQL     │                      │
          │                    │   (StatefulSet)   │                      │
          │                    └───────────────────┘                      │
          │                                                               │
          │   Master (1) ──── Workers (N)  ──── Provisioned by Ansible    │
          └───────────────────────────────────────────────────────────────┘
```

The external Nginx acts as the **edge proxy** — handling SSL termination, rate limiting, and security hardening before traffic enters the cluster. Inside K8S, the **Ingress Controller** handles internal routing to the appropriate services.

| Layer               | Technology       | Purpose                                                         |
|---------------------|------------------|-----------------------------------------------------------------|
| Infrastructure      | Ansible          | Provisions all nodes and installs dependencies                  |
| Containerization    | Docker           | Builds the application image                                    |
| Orchestration       | Kubernetes       | Manages pods, scaling, and self-healing                         |
| Database            | PostgreSQL       | Persistent storage via StatefulSet                              |
| Edge Proxy          | Nginx (VM)       | SSL termination, rate limiting, security headers                |
| Internal Routing    | Nginx Ingress    | Path-based routing inside the cluster                           |
| CI/CD               | GitHub Actions   | Automates build, push, and deploy                               |

## Project Structure

```
k8s-fullstack-base/
│
├── ansible.cfg                        # Ansible configuration
├── site.yml                           # Main entry point (orchestrates all roles)
├── Vagrantfile                        # Local development environment (5 VMs, VirtualBox)
│
├── inventory/
│   ├── hosts.yml.example              # Inventory template (committed to Git)
│   ├── hosts-vagrant.yml              # Inventory for Vagrant environment
│   └── hosts.yml                      # Real inventory (generated, gitignored)
│
├── group_vars/
│   └── all.yml                        # Shared variables across all roles
│
├── roles/                             # Each role follows: tasks/, defaults/, handlers/, templates/
│   ├── bootstrap/                     # Node provisioning (users, SSH, sudo)
│   ├── nginx/                         # Nginx reverse proxy (SSL, rate limiting, hardening)
│   ├── docker/                        # Docker CE + containerd installation
│   ├── k8s-prereqs/                   # K8S prerequisites (swap, sysctl, kubeadm/kubelet/kubectl)
│   ├── k8s-master/                    # Control plane init (kubeadm init, Calico CNI)
│   ├── k8s-workers/                   # Worker nodes join (kubeadm join)
│   ├── k8s-ingress/                   # Nginx Ingress Controller (bare metal, NodePort)
│   └── app-web/                       # K8S manifests deployment (Namespace, Deployment, Service, Ingress)
│
├── setup-inventory.sh                 # Inventory generator script (cloud-agnostic)
├── .gitignore
├── LICENSE
└── README.md
```

## Prerequisites

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) >= 2.14
- **Debian/Ubuntu-based** Linux VMs (any cloud provider or local) with SSH access
- SSH key pair (`~/.ssh/id_rsa` & `~/.ssh/id_rsa.pub`)

### VM Requirements

You need at least **4 VMs** running **Debian or Ubuntu** plus a control node with Ansible 2.14+ installed.

> **Why Debian/Ubuntu?** It is the Linux distribution I am most familiar with. 
> RHEL/CentOS/Rocky would require separate playbooks with different package 
> management (`dnf`), GPG key handling, and additional steps not needed 
> in Debian such as SELinux configuration and `firewalld` rules.
> **Note:** On fresh VMs, run the bootstrap playbook with your cloud provider's 
> default SSH user:
> ```bash
> ansible-playbook site.yml --tags bootstrap -u <your_cloud_user>
> ```
> After bootstrap, all subsequent roles will use the `ansible` user automatically.

| VM | Role | Suggested Size |
|----|------|----------------|
| `master1` | K8S control plane | 2 vCPU, 4 GB RAM |
| `worker1` | K8S worker node | 2 vCPU, 4 GB RAM |
| `worker2` | K8S worker node | 2 vCPU, 4 GB RAM |
| `nginx1` | Reverse proxy | 1 vCPU, 2 GB RAM |

These can be provisioned on **any platform**: GCP, AWS, Azure, Hetzner, VirtualBox, bare metal, etc.

### Local Development with Vagrant

A `Vagrantfile` is included to spin up the full environment locally using VirtualBox. This creates 5 VMs (the 4 above + an Ansible control node) on a host-only network (`192.168.56.0/24`).

**Requirements:**
- [VirtualBox](https://www.virtualbox.org/)
- [Vagrant](https://www.vagrantup.com/)
- 16 GB RAM minimum (the VMs use ~15 GB total)

| VM | IP | RAM | vCPU |
|-----|-----|-----|------|
| `master1` | 192.168.56.10 | 4 GB | 4 |
| `worker1` | 192.168.56.11 | 3 GB | 1 |
| `worker2` | 192.168.56.12 | 3 GB | 1 |
| `nginx1` | 192.168.56.20 | 1 GB | 1 |
| `ansible1` | 192.168.56.30 | 1 GB | 1 |

```bash
# 1. Start all VMs
vagrant up

# 2. SSH into the Ansible control node
vagrant ssh ansible1

# 3. Run the full provisioning
cd ~/project
ansible-playbook -i inventory/hosts-vagrant.yml site.yml

# 4. After editing files on the host, sync changes to the VM
vagrant rsync ansible1
```

> **Note:** The Vagrantfile automatically generates an SSH key pair, creates the `ansible` user on all VMs, installs Ansible on the control node, and syncs the project via rsync. No manual setup is needed beyond `vagrant up`.


## Quick Start

```bash
# 1. Clone the repository
git clone https://github.com/OrangeVice94/k8s-fullstack-base.git
cd k8s-fullstack-base

# 2. Generate your inventory (interactive — enter your VM IPs)
chmod +x setup-inventory.sh
./setup-inventory.sh

# 3. Verify inventory is correct
ansible-inventory --list

# 4. Run the full provisioning
ansible-playbook site.yml
```

### Run Individual Phases

```bash
# Bootstrap nodes (SSH, users, sudo)
ansible-playbook site.yml --tags bootstrap

# Install Docker
ansible-playbook site.yml --tags docker

# Setup Nginx reverse proxy
ansible-playbook site.yml --tags nginx

# Setup Kubernetes cluster
ansible-playbook site.yml --tags k8s-prereqs
ansible-playbook site.yml --tags k8s-master
ansible-playbook site.yml --tags k8s-workers

# Install Ingress Controller
ansible-playbook site.yml --tags k8s-ingress

# Deploy application manifests
ansible-playbook site.yml --tags app-web
```

## Roles

| Role | Target Hosts | Description |
|----------|-------------|-------------|
| `bootstrap` | `all` | Creates service user, configures SSH keys, sets up passwordless sudo |
| `docker` | `k8s-cluster` | Installs Docker CE, containerd, and Docker Compose plugin |
| `nginx` | `proxy` | Installs and configures Nginx as a reverse proxy to the K8S cluster |
| `k8s-prereqs` | `k8s-cluster` | Disables swap, configures sysctl, sets up containerd with systemd cgroup, installs kubeadm/kubelet/kubectl |
| `k8s-master` | `k8s-master` | Initializes control plane with `kubeadm init`, configures kubeconfig, installs Calico CNI |
| `k8s-workers` | `k8s-workers` | Generates a fresh join token from the master and joins worker nodes to the cluster |
| `k8s-ingress` | `k8s-master` | Installs Nginx Ingress Controller (bare metal, NodePort 30080/30443) |
| `app-web` | `k8s-master` | Deploys K8S manifests: Namespace, Deployment, Service, Ingress |


## Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **External Nginx instead of cloud LB** | Demonstrates manual Nginx configuration and Ansible provisioning skills. A cloud LB would abstract away the learning opportunity. |
| **Cloud-agnostic inventory** | A setup script generates the inventory from user input. No IPs are committed to Git, and the project works on any cloud provider or local VMs. For GCP users, Ansible's `google.cloud.gcp_compute` dynamic inventory plugin is also supported as an alternative. |
| **kubeadm for K8S setup** | Shows understanding of how a cluster is built from scratch, rather than using managed K8S (GKE). |
| **Dual Nginx architecture** | External Nginx handles edge concerns (SSL, rate limiting, DDoS protection). The Ingress Controller inside K8S handles internal service routing. Each layer has a distinct responsibility. |
| **App-agnostic design** | The platform is built to deploy any containerized application, not tied to a specific framework or language. |
| **Nginx version pinning** | Nginx version is held (`apt-mark hold`) after installation to prevent automatic upgrades that could affect configuration compatibility. Manual upgrades are documented in the playbook. |
| **Kubernetes version pinning** | Kubernetes version is held (`apt-mark hold`) after installation to prevent automatic upgrades to avoid breaking compatibility between your versions of kubeadm, kubectl, and kubelet. |
| **Usage of roles, over Playbooks** | It is the industry standard making possible a greater understanding of the project structure. Allows the separation of variables with default and vars. Handlers with scope. Isolated Testability.|
| **Calico as CNI** | Supports network policies out of the box, widely adopted in production, and uses standard BGP — making it a solid choice for learning real-world K8S networking. Flannel is simpler but lacks network policies. Cilium uses eBPF (more modern but more complex). |
| **NodePort** | Deterministic port assignment for Nginx proxy_pass connectivity, no external load balancer required in bare metal environments. |

## Roadmap

- [x] Node provisioning with Ansible (bootstrap + Docker)
- [x] Nginx reverse proxy with upstream load balancing
- [x] Cloud-agnostic inventory setup script
- [x] Local development environment (Vagrant + VirtualBox)
- [x] Kubernetes cluster setup (kubeadm + Calico CNI)
- [x] Nginx Ingress Controller (bare metal, NodePort)
- [x] Kubernetes manifests (Namespace, Deployment, Service, Ingress)
- [ ] Sample containerized application (FastAPI) + Dockerfile
- [ ] PostgreSQL persistence (StatefulSet)
- [ ] Terraform IaC for cloud provisioning
- [ ] GitHub Actions CI/CD pipeline
- [ ] Ansible Vault for secrets management
- [ ] Monitoring & health checks


## Security Considerations

This project is designed for learning and portfolio purposes. The following improvements would be recommended for a production environment:

| Area | Current State | Production Recommendation |
|------|--------------|--------------------------|
| **NodePort access** | Open on all nodes (30000-32767) | Firewall rules (iptables/ufw) to restrict access to the reverse proxy IP only |
| **SSL certificates** | Self-signed (development) | Let's Encrypt or CA-signed certificates |
| **Secrets management** | Plaintext in inventory/vars | Ansible Vault or external secrets manager |
| **Internal traffic** | Plain HTTP between proxy and cluster | mTLS for regulated environments (banking, government) |

## Tech Stack

<p>
  <img src="https://img.shields.io/badge/Ansible-EE0000?style=for-the-badge&logo=ansible&logoColor=white" />
  <img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white" />
  <img src="https://img.shields.io/badge/Kubernetes-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" />
  <img src="https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white" />
  <img src="https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white" />
  <img src="https://img.shields.io/badge/GitHub_Actions-2088FF?style=for-the-badge&logo=github-actions&logoColor=white" />
</p>

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

---

<p align="center">
  <sub>Built as a DevOps infrastructure portfolio project.</sub>
</p>
