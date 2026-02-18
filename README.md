# K8S Fullstack Base

> Automated infrastructure platform for deploying a containerized web application on Kubernetes, provisioned entirely with Ansible and delivered through CI/CD.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Ansible](https://img.shields.io/badge/Ansible-2.15+-EE0000?logo=ansible&logoColor=white)](https://docs.ansible.com/)
[![Kubernetes](https://img.shields.io/badge/Kubernetes-1.28+-326CE5?logo=kubernetes&logoColor=white)](https://kubernetes.io/)
[![Docker](https://img.shields.io/badge/Docker-24+-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)

---

## Overview

This project is **not** about the application itself — it's about the **platform** that makes it run. The goal is to demonstrate a fully automated infrastructure pipeline:

1. **Provision** bare VMs into ready-to-use nodes (Ansible)
2. **Containerize** a sample application (Docker)
3. **Orchestrate** the deployment on a Kubernetes cluster (kubeadm)
4. **Route traffic** through an external Nginx reverse proxy into the cluster
5. **Automate** the entire build & deploy cycle (GitHub Actions)

```
┌───────────────────────────────────────────────────────────────────────────┐
│                          INFRASTRUCTURE FLOW                              │
│                                                                           │
│   ┌──────────┐    ┌──────────┐    ┌──────────┐    ┌───────┐   ┌───────┐ │
│   │  Ansible  │───▶│  Docker  │───▶│   K8S    │───▶│ Nginx │──▶│  App  │ │
│   │  (Nodes)  │    │ (Images) │    │(Cluster) │    │(Proxy)│   │ Live! │ │
│   └──────────┘    └──────────┘    └──────────┘    └───────┘   └───────┘ │
│        │                                                          │       │
│        └───────────────── GitHub Actions ─────────────────────────┘       │
└───────────────────────────────────────────────────────────────────────────┘
```

## Architecture

```
                    ┌─────────────────────────────────────────────────────┐
                    │                    NGINX VM                         │
  Internet ───────▶│  SSL Termination ─ Rate Limit ─ Security Headers   │
                    └──────────────────────┬──────────────────────────────┘
                                           │
                                      proxy_pass
                                           │
          ┌────────────────────────────────▼──────────────────────────────┐
          │                     KUBERNETES CLUSTER                        │
          │                                                               │
          │                    ┌───────────────────┐                     │
          │                    │ Ingress Controller │                     │
          │                    └────────┬──────────┘                     │
          │                             │                                │
          │     ┌────────────┐     ┌────▼───────┐     ┌────────────┐     │
          │     │  App Pod   │     │  App Pod   │     │  App Pod   │     │
          │     └─────┬──────┘     └─────┬──────┘     └─────┬──────┘     │
          │           └──────────────────┼──────────────────┘             │
          │                              │                                │
          │                    ┌─────────▼─────────┐                     │
          │                    │    PostgreSQL      │                     │
          │                    │   (StatefulSet)    │                     │
          │                    └───────────────────┘                      │
          │                                                               │
          │   Master (1) ──── Workers (N)  ──── Provisioned by Ansible   │
          └───────────────────────────────────────────────────────────────┘
```

The external Nginx acts as the **edge proxy** — handling SSL termination, rate limiting, and security hardening before traffic enters the cluster. Inside K8S, the **Ingress Controller** handles internal routing to the appropriate services.

| Layer               | Technology       | Purpose                                                        |
|---------------------|------------------|----------------------------------------------------------------|
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
├── site.yml                           # Main entry point (orchestrates all playbooks)
│
├── inventory/
│   ├── hosts.yml.example              # Inventory template (committed to Git)
│   └── hosts.yml                      # Real inventory (generated, gitignored)
│
├── playbooks/
│   ├── playbook-bootstrap.yml         # Node bootstrap (users, SSH, sudo)
│   ├── playbook-docker.yml            # Docker CE installation
│   ├── playbook-nginx.yml             # Nginx reverse proxy setup
│   ├── playbook-k8s-prereqs.yml       # K8S prerequisites (swap, kernel modules)
│   ├── playbook-k8s-master.yml        # Initialize control plane (kubeadm init)
│   └── playbook-k8s-workers.yml       # Join workers to cluster (kubeadm join)
│
├── templates/
│   └── nginx.conf.j2                  # Nginx reverse proxy configuration
│
├── variables/
│   ├── bootstrap.yml                  # Bootstrap variables
│   └── nginx.yml                      # Nginx variables (ports, SSL paths, etc.)
│
├── app/                               # Sample containerized application
│   ├── Dockerfile
│   └── src/
│
├── k8s/                               # Kubernetes manifests
│   ├── namespace.yml
│   ├── app-deployment.yml
│   ├── app-service.yml
│   ├── postgres-statefulset.yml
│   ├── postgres-service.yml
│   ├── postgres-pv.yml
│   ├── ingress.yml
│   └── secrets.yml
│
├── .github/                           # CI/CD pipeline
│   └── workflows/
│       └── deploy.yml
│
├── setup-inventory.sh                 # Inventory generator script (cloud-agnostic)
├── .gitignore
├── LICENSE
└── README.md
```

## Prerequisites

- [Ansible](https://docs.ansible.com/ansible/latest/installation_guide/) >= 2.15
- **Debian/Ubuntu-based** Linux VMs (any cloud provider or local) with SSH access
- SSH key pair (`~/.ssh/id_rsa` & `~/.ssh/id_rsa.pub`)

### VM Requirements

You need at least **4 VMs** running **Debian or Ubuntu**:

> **Why Debian/Ubuntu?** The Docker and Nginx playbooks use `apt` for package management and Debian-specific repository paths. RHEL/CentOS/Rocky would require separate playbooks using `dnf`.

| VM | Role | Suggested Size |
|----|------|----------------|
| `master1` | K8S control plane | 2 vCPU, 4 GB RAM |
| `worker1` | K8S worker node | 2 vCPU, 4 GB RAM |
| `worker2` | K8S worker node | 2 vCPU, 4 GB RAM |
| `nginx1` | Reverse proxy | 1 vCPU, 2 GB RAM |

These can be provisioned on **any platform**: GCP, AWS, Azure, Hetzner, VirtualBox, bare metal, etc.

<details>
<summary><b>Example: GCP setup</b></summary>

```bash
# Master node
gcloud compute instances create k8s-master \
  --zone=europe-west1-b \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud

# Worker nodes
gcloud compute instances create k8s-worker-1 k8s-worker-2 \
  --zone=europe-west1-b \
  --machine-type=e2-medium \
  --image-family=debian-12 \
  --image-project=debian-cloud

# Nginx reverse proxy
gcloud compute instances create nginx-proxy \
  --zone=europe-west1-b \
  --machine-type=e2-small \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

</details>

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
ansible-playbook playbooks/playbook-bootstrap.yml

# Install Docker
ansible-playbook playbooks/playbook-docker.yml

# Setup Nginx reverse proxy
ansible-playbook playbooks/playbook-nginx.yml

# Setup Kubernetes cluster
ansible-playbook playbooks/playbook-k8s-prereqs.yml
ansible-playbook playbooks/playbook-k8s-master.yml
ansible-playbook playbooks/playbook-k8s-workers.yml
```

## Playbooks

| Playbook | Target Hosts | Description |
|----------|-------------|-------------|
| `playbook-bootstrap.yml` | `all` | Creates service user, configures SSH keys, sets up passwordless sudo |
| `playbook-docker.yml` | `k8s_cluster` | Installs Docker CE, containerd, and Docker Compose plugin |
| `playbook-nginx.yml` | `proxy` | Installs and configures Nginx as a reverse proxy to the K8S cluster |
| `playbook-k8s-prereqs.yml` | `k8s_cluster` | Disables swap, loads kernel modules, configures sysctl for K8S networking |
| `playbook-k8s-master.yml` | `k8s_master` | Runs `kubeadm init`, installs CNI plugin, generates join token |
| `playbook-k8s-workers.yml` | `k8s_workers` | Joins worker nodes to the cluster using the master's token |

## Architecture Decisions

| Decision | Rationale |
|----------|-----------|
| **External Nginx instead of cloud LB** | Demonstrates manual Nginx configuration and Ansible provisioning skills. A cloud LB would abstract away the learning opportunity. |
| **Cloud-agnostic inventory** | A setup script generates the inventory from user input. No IPs are committed to Git, and the project works on any cloud provider or local VMs. For GCP users, Ansible's `google.cloud.gcp_compute` dynamic inventory plugin is also supported as an alternative. |
| **kubeadm for K8S setup** | Shows understanding of how a cluster is built from scratch, rather than using managed K8S (GKE). |
| **Dual Nginx architecture** | External Nginx handles edge concerns (SSL, rate limiting, DDoS protection). The Ingress Controller inside K8S handles internal service routing. Each layer has a distinct responsibility. |
| **App-agnostic design** | The platform is built to deploy any containerized application, not tied to a specific framework or language. |
| **Nginx version pinning** | Nginx version is held (`apt-mark hold`) after installation to prevent automatic upgrades from breaking dynamic modules or custom configurations. Manual upgrades are documented in the playbook. |
| **Separate playbooks, one `site.yml`** | Each playbook can run independently for debugging. `site.yml` orchestrates them all in the correct order for full provisioning. |

## Roadmap

- [x] Node provisioning with Ansible (bootstrap + Docker)
- [x] Nginx reverse proxy configuration template (in progress)
- [x] Cloud-agnostic inventory setup script
- [x] Nginx provisioning playbook
- [ ] Kubernetes cluster setup (kubeadm)
- [ ] Nginx Ingress Controller
- [ ] Sample containerized application + Dockerfile
- [ ] Kubernetes manifests (Deployments, Services, StatefulSet, Ingress)
- [ ] GitHub Actions CI/CD pipeline
- [ ] Ansible Vault for secrets management
- [ ] Monitoring & health checks

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
