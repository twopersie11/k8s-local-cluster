
# k8s-local-cluster

Ansible-driven local Kubernetes cluster (k3s) with PostgreSQL & Redis, plus daily backups and CLI test scripts.

## 📂 Repository layout

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



## 💡 Solution Idea

This project delivers a fully-Ansible-driven, k3s-based Kubernetes lab environment, with PostgreSQL and Redis managed by Helm and daily backups—all reproducible on fresh Ubuntu VMs.  

1. **Infrastructure as Code**  
   - Use a local Git repo to track everything: inventories, playbooks, roles, values files and test scripts.  
   - An `inventories/hosts.yml` defines master & worker nodes; for this initial proof-of-concept only the master node is actually used, but adding the worker is as simple as listing it in the inventory and rerunning the playbook.

2. **Modular Ansible Roles**  
   - **`roles/k3s`**: Install and configure k3s on master (& optional worker) (incl. linking kubeconfig for seamless `kubectl` & `helm`).  
   - **`roles/helm`**: Install the Helm CLI, register the Bitnami repo (with cache refresh), and prepare the system for chart deployments.  
   - **`roles/postgres-helm`**:  
     - Deploy PostgreSQL via `kubernetes.core.helm` with a PVC for data persistence.  
     - Create an in-cluster CronJob (or host-side cron) that dumps the database daily into a backup PVC.  
   - **`roles/redis-helm`**:  
     - Create a namespaced Secret (with vault-encrypted password) and deploy Redis via Helm with a PVC.  
   - **`roles/backups`** (optional): Push backups to S3 or another external store.

3. **Security & Secrets**  
   - No plaintext passwords in Git—use `ansible-vault` for both `vault_postgres_password` and `vault_redis_password`.  
   - Kubernetes Secrets are generated in the correct namespace with the exact key names the charts expect.

4. **Persistence & Resilience**  
   - PostgreSQL and Redis each mount a PersistentVolumeClaim, ensuring data survives pod restarts.  
   - The PostgreSQL CronJob runs at 02:00 daily, dumping to a backup PVC (or host path) for point-in-time recovery.

5. **External Connectivity**  
   - Expose both services as **NodePort** (configurable in `values-*.yaml`) to allow CLI clients outside the cluster.  
   - Provide simple shell scripts (`scripts/test-postgres.sh` and `scripts/test-redis.sh`) that validate connectivity and basic commands.

6. **Idempotence & Reproducibility**  
   - All roles are idempotent — running `ansible-playbook site.yml` multiple times will only apply changes when drift is detected.  
   - Clear README with step-by-step instructions means anyone can bring up the entire environment on new Ubuntu VMs with a single command.

By following this structure, the solution meets all Business & Product Requirements, and provides a clean, modular foundation for future enhancements—TLS, S3 backups, CI/CD integration, and more.  


