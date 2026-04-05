# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is an Ansible role (`iamenr0s.ansible_role_k8s`) that installs Kubernetes (kubelet, kubeadm, kubectl) and bootstraps a cluster across Debian/Ubuntu and RHEL/Fedora/EL family systems. It handles repo setup, swap disabling, firewalld/SELinux configuration, kubeadm init on control planes, Flannel CNI deployment, and worker node joining.

## Commands

```bash
# Install Python dependencies (Molecule, ansible-lint)
pip install -r requirements.txt

# Install Ansible role/collection dependencies
ansible-galaxy install -r requirements.yml

# Lint YAML
yamllint .

# Lint Ansible
ansible-lint

# Run Molecule tests (uses Podman locally, Docker in CI)
molecule test

# Run against a specific distro
MOLECULE_DISTRO=almalinux10 molecule test
MOLECULE_DISTRO=debian12 molecule test
MOLECULE_DISTRO=ubuntu2204 molecule test

# Molecule lifecycle steps individually
molecule create
molecule converge
molecule verify
molecule destroy
```

## Architecture

**Role structure (standard Ansible layout):**
- `tasks/main.yml` — single task file; all logic is inline (no task includes/imports)
- `defaults/main.yml` — all variables with defaults; the primary reference for role behaviour
- `handlers/main.yml` — service handlers
- `meta/main.yml` — Galaxy metadata and role dependencies
- `molecule/default/` — Molecule test scenario using Podman (local) / Docker (CI)

**Execution flow in `tasks/main.yml`:**
1. Repo + package install (branched by OS family: Debian → `apt`, Fedora → `dnf` direct, RHEL/EL → via `iamenr0s.ansible_role_pkg_management`)
2. Swap disable (via `iamenr0s.ansible_role_swap` with a direct fallback)
3. Kernel modules + sysctl (`br_netfilter`, bridge-nf iptables) — general k8s prerequisite, not CNI-specific; controlled by `k8s_configure_kernel_networking` (default: `true`), via `iamenr0s.ansible_role_kernel_configuration`
4. kubelet enable/start
5. Firewalld + SELinux (permissive) via `iamenr0s.ansible_role_firewalld` — different ports for control plane vs workers
6. Control plane: `kubeadm init` with a generated config file, kubeconfig setup, API server wait, join command extraction, Flannel CNI install
7. Workers: receive kubeconfig and join command via `hostvars`, run `kubeadm join`

**Inventory groups:** `k8s_control_plane_group` (default: `masters`) and `k8s_workers_group` (default: `workers`)

**Key variables** (all in `defaults/main.yml`):
- `k8s_version_channel` — pkgs.k8s.io channel (e.g. `v1.32`)
- `k8s_install_flannel` — controls CNI installation (pod CIDR is automatically included in kubeadm config when enabled)
- `k8s_configure_kernel_networking` — controls `br_netfilter` + bridge-nf sysctl setup (kubeadm prerequisite, independent of CNI)
- `k8s_use_pod_cidr` / `k8s_pod_network_cidr` — pod subnet (auto-enabled for Flannel)
- `k8s_control_plane_endpoint` — optional load balancer endpoint
- `k8s_manage_repos` — set false to skip repo configuration

## CI/CD

- **Molecule workflow** (`.github/workflows/molecule.yml`): triggers on version tags (`v*.*.*`) and PRs; runs lint then molecule test across 13+ distros
- **Release workflow** (`.github/workflows/release.yml`): publishes to Ansible Galaxy after Molecule succeeds; requires `GALAXY_API_KEY` secret
- **Release process:** push a semver tag (e.g. `git tag v1.0.8 && git push origin v1.0.8`) to trigger both workflows

## Conventions

- Task names must follow `{stem} | Description` format (enforced by `ansible-lint` `name[prefix]` rule)
- All variables use `snake_case` with the `k8s_` prefix (enforced by `var_naming_pattern` in `.ansible-lint`)
- YAML: `yamllint` with relaxed line-length and indentation rules (see `.yamllint`); unix line endings required
- `become: true` is applied at block level, not per-task
- Idempotency gates: use `ansible.builtin.stat` checks before `kubeadm init`/join; `failed_when: false` + `changed_when: false` on read-only commands
