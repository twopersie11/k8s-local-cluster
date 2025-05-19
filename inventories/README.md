

Ansible-driven local Kubernetes cluster (k3s) with PostgreSQL & Redis, plus daily backups and CLI test scripts.

├── inventories
│ └── hosts.yml # Defines k3s-master-1 & k3s-worker-1
├── playbooks
│ └── site.yml # Main playbook (roles: k3s, helm, postgres-helm, redis-helm, backups)
├── roles
│ ├── k3s # Installs and configures k3s
│ ├── helm # Installs Helm, registers repos
│ ├── postgres-helm # Deploys Postgres via Helm, sets up CronJob or host-cron backup
│ ├── redis-helm # Deploys Redis via Helm, manages secrets
│ └── backups # (Optional) Role for filesystem or S3 backups
├── scripts
│ ├── test-postgres.sh # CLI test for external PostgreSQL access
│ └── test-redis.sh # CLI test for external Redis access
├── values
│ ├── values-postgres.yaml # Helm values for PostgreSQL (PVC, NodePort, etc.)
│ └── values-redis.yaml # Helm values for Redis (PVC, passwords, NodePort, etc.)
└── README.md # ← you’re here

---

## 🛠 Prerequisites

- **Controller machine** (your laptop or jump box) with:
  - Ansible >= 2.14  
  - `kubectl` & `helm` CLIs (optional, for manual checks)  
  - SSH access to target Ubuntu VMs
- **Target VMs** (Ubuntu 20.04+):
  - Passwordless SSH for your Ansible user
  - Python 3 installed
  - (No manual Kubernetes, Docker, or Helm required)

---

## 🔧 Inventory

Edit `inventories/hosts.yml` to point at your master & worker:

```yaml
all:
  hosts:
    k3s-master-1:
      ansible_host: 192.168.157.129
    k3s-worker-1:
      ansible_host: 192.168.157.129


ansible-vault encrypt_string 'postgrestest' --name 'postgrestest'
ansible-vault encrypt_string 'redistest' --name 'redistest'
ansible-playbook -i inventories/hosts.yml playbooks/site.yml --ask-vault-pass

# 1. Clone repo
git clone git@github.com:<you>/k8s-local-cluster.git
cd k8s-local-cluster

# 2. Supply your vault password interactively:
ansible-playbook \
  -i inventories/hosts.yml \
  playbooks/site.yml \
  --ask-vault-pass
# Check nodes & pods
kubectl --kubeconfig=~/.kube/config get nodes
kubectl --kubeconfig=~/.kube/config get pods -A

# Check Helm releases
helm --kubeconfig=~/.kube/config list -A

# Test in-cluster services
kubectl -n database exec deploy/postgres -- psql -U postgres -c '\l'
kubectl -n cache   exec deploy/redis-master -- redis-cli ping

# Test Postgres
./scripts/test-postgres.sh   # Defaults to localhost:30007
# or
HOST=192.168.157.129 PG_PORT=30007 ./scripts/test-postgres.sh

# Test Redis
./scripts/test-redis.sh      # Defaults to localhost:30008
# or
HOST=192.168.157.129 REDIS_PORT=30008 ./scripts/test-redis.sh


