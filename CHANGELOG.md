# Changelog

All notable changes to this project are documented in this file.
This project follows [Semantic Versioning](https://semver.org/).

## [Unreleased]

### Changed
- Standardized repo tooling to match the `ansible-role-upgrade` golden baseline: consolidated CI into a single `lint → molecule → release` workflow (previously two workflows), replaced the `.ansible-lint` scaffold-default with a real config, added `.pre-commit-config.yaml`, `renovate.json`, community health files, `meta/argument_specs.yml`, and project-local `.claude/` tooling (skills/agent/CLAUDE.md).
- Reduced the supported-distro scope in `meta/main.yml`/README/CI matrix to a rolling two-version window (Debian 12/13, EL 8/9/10, Fedora 42/43/44, Ubuntu 22.04/24.04), dropping EOL/non-LTS entries (Debian 11 "bullseye", Ubuntu 20.04 "focal" and 22.10 "kinetic", Fedora 39-41). Nothing in `tasks/main.yml` special-cased the dropped versions.
- Reworked the Molecule harness: `molecule.yml` now uses `extra_opts: ["--cgroupns=host"]` instead of `cgroupns_mode: host` (schema compatibility with current molecule-plugins), `converge.yml` bootstraps Python 3 via `raw` and gathers facts explicitly, and a new `verify.yml` asserts package/service state (the removed `prepare.yml` was dead weight — never wired into the provisioner).

## [1.0.4] - 2026-04-06

### Added
- `templates/kubeadm-init.yaml.j2` — kubeadm ClusterConfiguration template replacing
  inline flags; supports `k8s_control_plane_endpoint`, pod CIDR, and
  `allocate-node-cidrs` cleanly via Jinja2 conditionals
- `k8s_control_plane_endpoint` default (empty string) for HA load-balancer setups
- `k8s_configure_kernel_networking` default (`false`) — when enabled, loads
  `br_netfilter`/`overlay` modules and sets bridge-nf sysctl via
  `iamenr0s.ansible_role_kernel_configuration`
- Wait for Flannel DaemonSet rollout after CNI install (`kubectl rollout status`)
- Worker kubeconfig setup: fetches `admin.conf` from control plane via delegation
  and writes to `/root/.kube/config` on each worker
- `iamenr0s.ansible_role_kernel_configuration` added to `requirements.yml`

### Changed
- `kubeadm init` now uses `--config /tmp/kubeadm-init.yaml` (generated from
  template) instead of inline `--pod-network-cidr` flag
- Removed `set_fact` hack that overrode `k8s_use_pod_cidr` for Flannel; pod CIDR
  logic is now handled entirely inside the template

## [1.0.3] - 2026-04-06

### Changed
- Full revert to v1.0.0 state after regressions introduced in v1.0.1 and v1.0.2
  caused CI failures in Molecule tests

## [1.0.2] - 2026-04-06

### Fixed
- Reverted conditional role execution from `meta/main.yml` dependencies; roles
  listed in `meta` run unconditionally before all tasks, bypassing `when` guards

## [1.0.1] - 2026-04-06

### Fixed
- Idempotency gap on Flannel CNI install (check before apply)
- Duplicate task names causing ansible-lint warnings
- Decoupled kernel networking sysctl from CNI choice
- Galaxy dependency configuration in `meta/main.yml`

## [1.0.0] - 2025-11-14

### Added
- Initial release: Kubernetes installation and cluster bootstrap role
- Debian/Ubuntu and RHEL/Fedora/AlmaLinux package repository setup
- Kubelet, kubeadm, kubectl installation
- Memory cgroup detection and GRUB configuration for Debian and EL families
- Swap disable with fstab cleanup
- Firewalld and SELinux configuration for control plane and worker nodes
- `kubeadm init` with pod CIDR and preflight error bypass support
- Flannel CNI install (`k8s_install_flannel`, pinned to v0.25.5 manifest)
- Worker node join via `kubeadm join` with retry and `hostvars` token sharing
- Wait for Kubernetes API server before join command retrieval
- Memory cgroup verification on workers with bypass option
- `k8s_kubeconfig_setup` to copy `admin.conf` to `/root/.kube/config`
- Molecule testing with Podman driver on AlmaLinux 10
