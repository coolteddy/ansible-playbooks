# ansible-playbooks

General-purpose Ansible playbook repo. Each subdirectory is a standalone playbook.

## Repo structure

```
ansible-playbooks/
├── k8s-cluster/          # spin up kubeadm clusters on Multipass VMs
│   ├── site.yml          # entry point — provision cluster
│   ├── teardown.yml      # destroy cluster and remove kubeconfig
│   ├── group_vars/
│   │   └── all.yml       # default variables (always override via -e on CLI)
│   └── roles/
│       ├── common/       # runs on all nodes — containerd, kubeadm, kubelet
│       ├── control_plane/ # master only — kubeadm init, Calico, join command
│       └── worker/       # workers only — kubeadm join
```

## How to run

Always pass all variables explicitly on the command line:

```bash
# provision
ansible-playbook k8s-cluster/site.yml \
  -e cluster_name=test \
  -e num_workers=1 \
  -e node_cpu=2 \
  -e node_memory=2G \
  -e node_disk=20G

# teardown (shows confirmation prompt before deleting)
ansible-playbook k8s-cluster/teardown.yml -e cluster_name=test
```

## Conventions

- SSH key: `~/.ssh/id_ed25519` (injected into VMs via cloud-init)
- Kubeconfig saved to: `~/.kube/<cluster_name>-config`
- Use `kubectl --kubeconfig=/home/thet/.kube/<cluster_name>-config get nodes` (full path — kubectl does not expand ~)
- Always test locally before committing and pushing
- Each new playbook goes in its own subdirectory

## Gotchas

- `-e` variables come in as strings — always use `| int` filter for numeric comparisons and math
- Multipass snap cannot read files from `/tmp` — write cloud-init to `~/` instead
- Use `kubectl rollout status` not `kubectl wait` for Calico — pods may not exist yet when wait runs
