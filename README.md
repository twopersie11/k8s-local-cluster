
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

