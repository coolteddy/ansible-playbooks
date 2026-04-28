# ansible-playbooks

General-purpose Ansible playbook repo. Each subdirectory is a standalone playbook.

---

## Playbooks

| Playbook | Description |
|----------|-------------|
| [k8s-cluster](k8s-cluster/) | Spin up a kubeadm Kubernetes cluster on Multipass VMs |

---

## Ansible Concepts

### Entry point

Every playbook starts from `site.yml`. This is where you run:

```bash
ansible-playbook k8s-cluster/site.yml
```

Think of it like `main.tf` in Terraform — it's the starting point that orchestrates everything.

---

### Roles

A role is a reusable, self-contained unit of work — similar to a Terraform module. You write it once and reuse it across playbooks.

Ansible enforces a fixed folder structure inside each role. It looks for specific folder names automatically:

```
roles/
└── common/
    ├── tasks/
    │   └── main.yml      # the actual work — always the entry point for a role
    └── handlers/
        └── main.yml      # tasks that only run when notified (event-driven)
```

`tasks/main.yml` is always loaded automatically when a role is referenced. You never call it directly.

---

### Handlers

Handlers are tasks that only run **when notified** — like an event listener. If the triggering task didn't change anything, the handler never runs.

```yaml
# in tasks/main.yml — triggers the handler
- name: Write containerd config
  copy:
    dest: /etc/containerd/config.toml
  notify: Restart containerd      # <-- fires only if file actually changed

# in handlers/main.yml — what runs when notified
- name: Restart containerd
  systemd:
    name: containerd
    state: restarted
```

This prevents unnecessary service restarts on repeat runs.

---

### Variables

All configurable values live in `group_vars/all.yml`. Change them here — not inside roles.

```yaml
num_workers: 1      # set to 0 for control-plane only
node_cpu: 2
k8s_version: "1.33"
```

---

### How it all flows

```
ansible-playbook site.yml
  │
  ├── Play 1: Create Multipass VMs        runs on: localhost
  │     ├── Create master VM (multipass launch)
  │     ├── Create worker VMs (multipass launch × num_workers)
  │     ├── Get IPs from multipass info
  │     └── Add IPs to in-memory inventory (add_host)
  │
  ├── Play 2: Common node setup           runs on: ALL nodes (master + workers)
  │     └── role: common
  │           ├── Disable swap
  │           ├── Load kernel modules (overlay, br_netfilter)
  │           ├── Set sysctl params
  │           ├── Install containerd + runc + CNI plugins
  │           ├── Configure containerd (SystemdCgroup = true)
  │           └── Install kubeadm + kubelet + kubectl
  │
  ├── Play 3: Control plane init          runs on: master only
  │     └── role: control_plane
  │           ├── kubeadm init --pod-network-cidr
  │           ├── Copy admin.conf → ~/.kube/config
  │           ├── Install Calico CNI
  │           └── Generate join command (kubeadm token create)
  │
  ├── Play 4: Worker join                 runs on: workers only
  │     └── role: worker
  │           └── kubeadm join <master-ip>:6443 --token ...
  │
  └── Play 5: Fetch kubeconfig            runs on: master → saved to host
        └── Saves ~/.kube/<cluster_name>-config on your local machine
```

---

### Ansible vs Terraform

| Terraform | Ansible |
|-----------|---------|
| Module | Role |
| `main.tf` | `tasks/main.yml` |
| Variables (`variables.tf`) | `group_vars/all.yml` |
| `terraform apply` | `ansible-playbook site.yml` |
| Manages state | Runs tasks top to bottom (no state file) |

---

## k8s-cluster usage

### Prerequisites

```bash
# Ansible
sudo apt-get install -y ansible

# Multipass
sudo snap install multipass
```

### Run

Always pass all variables explicitly so resource sizes are visible:

```bash
cd k8s-cluster

# 1 master + 1 worker
ansible-playbook site.yml \
  -e cluster_name=k8s \
  -e num_workers=1 \
  -e node_cpu=2 \
  -e node_memory=2G \
  -e node_disk=20G

# control-plane only
ansible-playbook site.yml \
  -e cluster_name=k8s \
  -e num_workers=0 \
  -e node_cpu=2 \
  -e node_memory=2G \
  -e node_disk=20G

# 2 workers with more RAM
ansible-playbook site.yml \
  -e cluster_name=k8s \
  -e num_workers=2 \
  -e node_cpu=4 \
  -e node_memory=4G \
  -e node_disk=30G
```

### After run

```bash
export KUBECONFIG=~/.kube/k8s-config
kubectl get nodes
```

### Teardown

Always pass `-e cluster_name=<name>` to avoid deleting the wrong cluster.
The playbook shows what will be deleted and waits for confirmation before proceeding.

```bash
# tear down test cluster
ansible-playbook teardown.yml -e cluster_name=test

# tear down default k8s cluster
ansible-playbook teardown.yml
```
