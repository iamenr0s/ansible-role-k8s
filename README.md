[![Molecule](https://github.com/iamenr0s/ansible-role-k8s/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-k8s/actions/workflows/molecule.yml) [![Release](https://github.com/iamenr0s/ansible-role-k8s/actions/workflows/release.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-k8s/actions/workflows/release.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/ansible_role_k8s) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-k8s/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-k8s)

# Ansible Role: Kubernetes
Manage Kubernetes installation and cluster bootstrap across Debian/Ubuntu and RHEL/Fedora.

## Features

- Configure pkgs.k8s.io repositories for Debian/Ubuntu and RHEL/Fedora.
- Install `kubelet`, `kubeadm`, `kubectl`.
- Enable and start `kubelet` service.
- Initialize control plane with `kubeadm init` (idempotent check).
- Generate and expose `kubeadm join` command to workers; join workers automatically.
- Optional Flannel CNI installation with correct pod CIDR (`10.244.0.0/16`).
- Cilium-friendly configuration: toggle `--pod-network-cidr` and documented IPAM modes.
- Configurable version channel (`v1.32` by default) and repo management.

## Requirements

- Ansible 2.9 or higher
- Install role dependencies: `ansible-galaxy install -r requirements.yml`

## Supported Platforms

- Ubuntu 20.04, 22.04, 24.04
- Debian 10, 11, 12
- RHEL 8, 9, 10
- Rocky Linux 8, 9, 10
- AlmaLinux 8, 9, 10
- Fedora 39, 40, 41, 42

## Role Variables

### Basic Configuration

- `k8s_version_channel` (string, default: `v1.32`): Repository channel version.
- `k8s_packages` (list, default: `[kubelet, kubeadm, kubectl]`): Packages to install.
- `k8s_manage_repos` (bool, default: `true`): Manage repos inside this role.
- `k8s_enable_kubelet` (bool, default: `true`): Enable and start kubelet.
- `k8s_use_pod_cidr` (bool, default: `false`): Include `--pod-network-cidr` in `kubeadm init`.
- `k8s_apt_key_dest` (string): Path to apt keyring.
- `k8s_apt_repo_filename` (string): Apt sources filename.
- `k8s_yum_repo_name` (string): YUM/DNF repo name.
- `k8s_disable_excludes` (string, default: `all`): Pass-through to disable repo excludes for DNF/YUM operations.

## Dependencies

- Role: `iamenr0s.ansible_role_pkg_management` (used to manage repositories and packages)

## Example Playbook

### Basic Usage

Install Kubernetes using pkgs.k8s.io repositories (stable `v1.32`):

Debian/Ubuntu:
```
- hosts: debian_hosts
  gather_facts: true
  become: true

  roles:
    - role: ansible-role-k8s
      vars:
        k8s_version_channel: v1.32
        k8s_manage_repos: true
        k8s_packages:
          - kubelet
          - kubeadm
          - kubectl
```

RHEL/Rocky/AlmaLinux/Fedora:
```
- hosts: rhel_hosts
  gather_facts: true
  become: true

  roles:
    - role: ansible-role-k8s
      vars:
        k8s_version_channel: v1.32
        k8s_manage_repos: true
        k8s_disable_excludes: all
        k8s_packages:
          - kubelet
          - kubeadm
          - kubectl
```

This role enables and starts `kubelet` automatically when `k8s_enable_kubelet` is `true`.

### Cluster Init and Join

Inventory grouping (single control plane and multiple workers):
```
all:
  children:
    k8s_control_plane:
      hosts:
        master-01:
          ansible_host: 10.0.0.10
    k8s_workers:
      hosts:
        worker-01:
          ansible_host: 10.0.0.11
        worker-02:
          ansible_host: 10.0.0.12
  vars:
    ansible_user: ansible
    ansible_ssh_private_key_file: ~/.ssh/ansible_key
```

Playbook that installs, initializes the control plane, and joins workers:
```
- hosts: k8s_control_plane:k8s_workers
  gather_facts: true
  become: true

  roles:
    - role: ansible-role-k8s
      vars:
        k8s_version_channel: v1.32
        k8s_manage_repos: true
        k8s_control_plane_group: k8s_control_plane
        k8s_workers_group: k8s_workers
        # Flannel CNI
        k8s_install_flannel: true
        k8s_use_pod_cidr: true
        k8s_pod_network_cidr: 10.244.0.0/16
```

Notes:
- Control plane runs `kubeadm init` if not already initialized and exposes a join command.
- Workers consume the join command and run `kubeadm join` if not already part of the cluster.
- You can add extra flags via `k8s_init_extra_args` and `k8s_join_extra_args`.

### Cilium CIDR Guidance

- Cilium IPAM options:
  - Cluster-pool IPAM (default): Cilium allocates pod CIDRs internally. In this mode, keep `k8s_use_pod_cidr: false` and configure Cilium’s pool (e.g., via Helm values) to match your network plan.
  - Kubernetes IPAM: If you prefer, set `k8s_use_pod_cidr: true` and choose `k8s_pod_network_cidr` to match Cilium’s CIDR settings.

- Recommended approach for most setups:
  - Set `k8s_use_pod_cidr: false` and manage Cilium’s `cluster-pool-ipv4-cidr` and `cluster-pool-ipv4-mask-size` through Cilium’s install.

- If you choose to set kubeadm’s CIDR:
  - Example: `k8s_use_pod_cidr: true` and `k8s_pod_network_cidr: 10.244.0.0/16`.
  - Ensure Cilium configuration uses the same CIDR to avoid IPAM conflicts.

### Common Issues

- Repo not found or signature errors: ensure `k8s_version_channel` exists (e.g., `v1.32`) and that apt uses the `signed-by` keyring prepared by the role on Debian/Ubuntu.
- Missing dependencies: run `ansible-galaxy install -r requirements.yml` before applying this role.

## License

MIT

## Author Information

Author: iamenr0s
Galaxy: `iamenr0s.ansible_role_k8s`

## Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Add tests if applicable
5. Submit a pull request

## Changelog

See `CHANGELOG.md` for version history and release notes.
