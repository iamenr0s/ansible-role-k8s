# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

### Linting
```bash
# YAML lint (matches CI)
yamllint .

# Ansible lint
ansible-lint
```

### Molecule testing (requires Docker/Podman)
```bash
# Install test dependencies (ansible<10 + Python 3.12 keeps ansible-core 2.16.x
# on EL8 legs, which ship managed-node Python 3.6.8 — newer ansible-core
# requires Python 3.8+ on managed nodes)
pip3 install ansible molecule molecule-plugins[docker] docker cryptography

# Run tests against default distro (almalinux10)
molecule test

# Run tests against a specific distro
MOLECULE_DISTRO=ubuntu2404 molecule test
MOLECULE_DISTRO=debian12 molecule test
MOLECULE_DISTRO=fedora42 molecule test

# Run individual molecule phases
molecule create
molecule converge
molecule verify
molecule destroy
```

### Install Ansible collection dependencies
```bash
ansible-galaxy collection install -r requirements.yml
```

## Architecture

This is an Ansible role (`iamenr0s.ansible_role_k8s`) that installs kubelet/kubeadm/kubectl and bootstraps a Kubernetes cluster across Debian/Ubuntu and RHEL/Fedora hosts.

### Execution flow

`tasks/main.yml` runs a single sequence (no per-package-manager task-file split):

1. **Repo + package install** — dispatches on `ansible_facts['os_family']`/`ansible_facts['distribution']`: Debian/Ubuntu via `apt_repository` + `apt`, Fedora via `yum_repository` + `dnf`, other RHEL-family via `iamenr0s.ansible_role_pkg_management`. Controlled by `k8s_manage_repos`, `k8s_version_channel`, `k8s_packages`.
2. **Enable kubelet** — `systemd` enable+start, gated on `k8s_enable_kubelet`.
3. **Memory cgroup check/fix** — detects whether cgroup v2 `memory` controller or cgroup v1 memory mount is available; if not, edits GRUB/grubby kernel args and reboots (Debian/Ubuntu and RHEL/Fedora paths differ).
4. **Firewalld + SELinux** — via `iamenr0s.ansible_role_firewalld`, different port sets for `k8s_control_plane_group` ("masters") vs `k8s_workers_group` ("workers") hosts.
5. **Kernel networking** (optional, `k8s_configure_kernel_networking`) — `br_netfilter`/`overlay` modules + bridge-nf sysctl via `iamenr0s.ansible_role_kernel_configuration`.
6. **Control plane: `kubeadm init` + Flannel** — only runs on hosts in `groups[k8s_control_plane_group]`. Disables swap, verifies memory cgroup (fails or bypasses preflight per `k8s_allow_memcg_bypass`), renders `templates/kubeadm-init.yaml.j2`, runs `kubeadm init`, sets up root's kubeconfig, waits for the API server, generates the join command as a fact, installs/waits for Flannel.
7. **Workers: join** — only runs on hosts in `groups[k8s_workers_group]`. Verifies memory cgroup, runs the join command from the control-plane host's fact (via `hostvars`), fetches kubeconfig via `delegate_to`.
8. **Rolling upgrade (opt-in)** — gated by `k8s_upgrade_enabled` (default `false`), tagged `k8s-upgrade`. `tasks/upgrade.yml` builds an ordered host list (the control-plane host first, then every worker) and drives `tasks/upgrade_node.yml` via `include_tasks` + `loop` + `run_once: true`, so the rollout is one-node-at-a-time regardless of the calling playbook's `serial:` setting. Each node: skip if already at `k8s_upgrade_version`, `kubectl drain`, upgrade the `kubeadm` package, run `kubeadm upgrade apply`/`node`, upgrade `kubelet`/`kubectl` packages, restart kubelet, `kubectl uncordon`, wait for `Ready`, assert the reported version. No automatic rollback on failure; no CNI upgrade.

### Key variables

- `k8s_control_plane_group` / `k8s_workers_group` — inventory group names gating steps 6/7. If a host isn't in either group, only steps 1-5 run against it.
- `k8s_allow_memcg_bypass` — not in `defaults/main.yml` (opt-in only); when true, bypasses the memory-cgroup preflight failure by setting `--ignore-preflight-errors=SystemVerification` instead of failing.
- `k8s_ignore_preflight_errors` — passed straight through to `kubeadm init`/the memcg-bypass logic.

### Testing

Molecule uses Podman locally (`driver: podman`) but Docker in CI. `MOLECULE_DISTRO` selects the container image from `iamenr0s/docker-<distro>-ansible:latest`. `converge.yml` bootstraps Python 3 via `raw` (some base images lack it), resets the connection, then gathers facts explicitly — required because the role branches on `ansible_facts['os_family']`/`['distribution']`.

**Important CI limitation**: the default converge is a single ungrouped instance, so `k8s_control_plane_group`/`k8s_workers_group` never match any host — steps 6/7 above (`kubeadm init`/join, Flannel) never execute in Molecule. This is intentional, not a gap to "fix" by faking group membership: the role installs no container runtime (containerd/cri-o), so `kubeadm init` would fail in any container regardless. `verify.yml` only asserts what actually runs: repo/keyring files present, kubelet/kubeadm/kubectl installed, kubelet unit enabled, swap disabled. The rolling-upgrade path (step 8) is inert in Molecule for the same reason — it's gated by `k8s_upgrade_enabled: false` by default, and there's no live cluster in a container to upgrade against even if forced on.

### Lint rules

`.yamllint` extends `default` with `line-length: disable`, `trailing-spaces: disable`, `indentation: disable`, and `truthy: disable`. `.ansible-lint` skips rules `106` and `503`.

### CI triggers

- **molecule.yml**: single workflow — `lint` job, then `molecule` job (full distro matrix, `fail-fast: false`) on pushes to `main`, PRs, and `v*.*.*` tags, then a `release` job (`needs: [lint, molecule]`, only on `v*` tag pushes, `environment: galaxy`) that publishes to Ansible Galaxy.
- **code-scanning-notify.yml**: polls the code-scanning API on a 6-hour schedule (`code_scanning_alert` is webhook-only, not a valid Actions trigger) and posts new/updated open alerts to the webhook in the `SECURITY_ALERT_WEBHOOK` secret.
