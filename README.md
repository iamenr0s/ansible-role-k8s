[![Molecule](https://github.com/iamenr0s/ansible-role-k8s/actions/workflows/molecule.yml/badge.svg)](https://github.com/iamenr0s/ansible-role-k8s/actions/workflows/molecule.yml) ![Ansible Role](https://img.shields.io/ansible/role/d/iamenr0s/ansible_role_k8s) [![CodeFactor](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-k8s/badge)](https://www.codefactor.io/repository/github/iamenr0s/ansible-role-k8s)

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

- Ubuntu 22.04, 24.04
- Debian 12, 13
- RHEL/AlmaLinux/Rocky Linux 8, 9, 10
- Fedora 42, 43, 44

Note: platforms in `meta/main.yml` reflect Galaxy metadata; this list is kept in sync with the CI matrix in `.github/workflows/molecule.yml`.

## Role Variables

Defined in `defaults/main.yml`:

- `k8s_version_channel` (string, default: `v1.32`): Repository channel version.
- `k8s_packages` (list, default: `[kubelet, kubeadm, kubectl]`): Packages to install.
- `k8s_manage_repos` (bool, default: `true`): Manage repos inside this role.
- `k8s_enable_kubelet` (bool, default: `true`): Enable and start kubelet.
- `k8s_apt_key_dest` (string, default: `/etc/apt/keyrings/kubernetes-apt-keyring.gpg`): Path to apt keyring (Debian/Ubuntu).
- `k8s_apt_repo_filename` (string, default: `kubernetes.list`): Apt sources filename (Debian/Ubuntu).
- `k8s_yum_repo_name` (string, default: `kubernetes`): YUM/DNF repo name (RHEL/Fedora).
- `k8s_disable_excludes` (string, default: `all`): Pass-through to disable repo excludes for DNF operations.
- `k8s_control_plane_group` (string, default: `masters`): Inventory group identifying control-plane hosts.
- `k8s_workers_group` (string, default: `workers`): Inventory group identifying worker hosts.
- `k8s_control_plane_endpoint` (string, default: empty): Load-balancer endpoint for HA control planes, e.g. `lb.example.com:6443`.
- `k8s_use_pod_cidr` (bool, default: `false`): Include `--pod-network-cidr` in `kubeadm init` (usually keep `false` for Cilium).
- `k8s_pod_network_cidr` (string, default: `10.244.0.0/16`): Pod network CIDR used when `k8s_use_pod_cidr` is `true`.
- `k8s_init_extra_args` (string, default: empty): Additional arguments appended to `kubeadm init`.
- `k8s_join_extra_args` (string, default: empty): Additional arguments appended to `kubeadm join`.
- `k8s_kubeconfig_setup` (bool, default: `true`): Copy `admin.conf` to root's `.kube/config`.
- `k8s_configure_kernel_networking` (bool, default: `false`): Configure `br_netfilter`/`overlay`/sysctl kernel prerequisites (set `true` when not managed by a separate role or host setup).
- `k8s_install_flannel` (bool, default: `true`): Install the Flannel CNI on the control plane.
- `k8s_flannel_manifest_url` (string, default: Flannel v0.25.5 manifest): URL of the Flannel manifest to apply.
- `k8s_flannel_namespace` (string, default: `kube-flannel`): Namespace Flannel is installed into.
- `k8s_flannel_ds_name` (string, default: `kube-flannel-ds`): Name of the Flannel DaemonSet.
- `k8s_disable_swap` (bool, default: `true`): Disable swap and update fstab before init.
- `k8s_ignore_preflight_errors` (string, default: empty): Add to kubeadm init via `--ignore-preflight-errors`.
- `k8s_upgrade_enabled` (bool, default: `false`): Enable the rolling kubeadm/kubelet/kubectl upgrade path (opt-in).
- `k8s_upgrade_version` (string, default: empty): Target Kubernetes version for the rolling upgrade, e.g. `1.32.5` (no leading `v`). Required when `k8s_upgrade_enabled` is `true`.
- `k8s_upgrade_drain_timeout` (string, default: `300s`): Timeout passed to `kubectl drain --timeout` during the rolling upgrade.

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

### Rolling Upgrade

Upgrade an already-bootstrapped cluster to a new Kubernetes version, one
node at a time (control plane first, then each worker). This is opt-in and
never runs during a normal bootstrap:

```
- hosts: masters:workers
  gather_facts: true
  become: true

  roles:
    - role: ansible-role-k8s
      vars:
        k8s_version_channel: v1.33   # bump if crossing a minor version
        k8s_upgrade_enabled: true
        k8s_upgrade_version: "1.33.1"
```

Run with `--tags k8s-upgrade` to skip the (idempotent, but unnecessary)
repo/package-install tasks and go straight to the upgrade:

```
ansible-playbook site.yml --tags k8s-upgrade
```

**This shortcut only works for a patch-level upgrade within the same minor
version** (e.g. `1.33.0` -> `1.33.1`), where the repo/keyring config already
points at the right package repo. Crossing a minor version (as in the
`k8s_version_channel: v1.33` example above) requires the repo config to be
updated first — the `k8s-upgrade` tag alone never touches it, so
`kubeadm={{ k8s_upgrade_version }}-*` would 404 against the old repo. For a
minor-version bump, either run the full untagged playbook once first, or
include the repo tasks explicitly: `--tags k8s,k8s-upgrade`.

Nodes already running `k8s_upgrade_version` are skipped automatically, so a
partially-completed rolling upgrade can be safely re-run. A failed drain or
`kubeadm upgrade` stops the rollout at that node — this role does not
attempt automatic rollback; fix the underlying issue and re-run.

## Firewall and SELinux
- SELinux: set to `permissive` on all nodes.
- Control-plane firewall ports (zone `public`):
  - `6443/tcp`, `2379/tcp`, `2380/tcp`, `10250/tcp`, `10251/tcp`, `10252/tcp`, `10257/tcp`, `10259/tcp`, `179/tcp`, `4789/udp`
- Worker firewall ports (zone `public`):
  - `179/tcp`, `10250/tcp`, `30000-32767/tcp`, `4789/udp`

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

## CI & Release (maintainers)

A single workflow (`.github/workflows/molecule.yml`) runs lint and the full Molecule distro matrix on pushes to `main`, PRs, and `v*` tags. On `v*` tags, a `release` job publishes to Ansible Galaxy after all tests pass.

The Galaxy API key lives in the `galaxy` GitHub environment, which only `v*` tags may target. One-time setup:

```bash
# Galaxy publishing key (environment-scoped, get it from galaxy.ansible.com/ui/token)
gh secret set GALAXY_API_KEY --env galaxy --repo iamenr0s/ansible-role-k8s

# Code scanning notifications (Slack webhook URL; for Discord append /slack to the webhook URL)
gh secret set SECURITY_ALERT_WEBHOOK --env galaxy --repo iamenr0s/ansible-role-k8s
```

`.github/workflows/code-scanning-notify.yml` polls the code-scanning API every 6 hours and posts new or updated open alerts to that webhook (GitHub Actions cannot trigger on `code_scanning_alert` directly).

To release: tag a commit `vX.Y.Z` and push the tag — CI gates the Galaxy publish.

## Contributing

Contributions are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md) for the
local pipeline commands and pull request checklist. This project follows the
[Contributor Covenant Code of Conduct](CODE_OF_CONDUCT.md).

## Security

See [SECURITY.md](SECURITY.md) — GitHub private vulnerability reporting, no
public issues for security bugs.

## License

This project is licensed under the [MIT License](LICENSE).

## Author Information

Author: iamenr0s

Galaxy: `iamenr0s.ansible_role_k8s`
