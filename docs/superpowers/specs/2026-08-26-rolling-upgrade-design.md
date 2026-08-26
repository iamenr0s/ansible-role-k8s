# Rolling kubeadm/kubelet/kubectl upgrade — design

Date: 2026-08-26

## Problem

The role only bootstraps a cluster (install packages, `kubeadm init`/`join`). It has
no path to upgrade an already-running cluster. A real kubeadm minor/patch upgrade
must run in a specific order per node (upgrade `kubeadm` → `kubeadm upgrade
apply`/`node` → upgrade `kubelet`/`kubectl` → restart kubelet) and must not touch
more than one node at a time, or the cluster loses capacity/availability during
the upgrade window.

## Scope

- **In scope:** rolling upgrade of `kubeadm`, `kubelet`, `kubectl` across the
  single control-plane node, then each worker one at a time, using the real
  `kubeadm upgrade apply`/`kubeadm upgrade node` workflow (not a bare package
  bump).
- **Out of scope:**
  - CNI (Flannel) version upgrades.
  - Automatic rollback on failure — a failed drain/upgrade stops the rolling
    loop and requires manual operator intervention. kubeadm upgrades are not
    safely auto-revertible; retrying after a fix is the supported path.
  - True multi-control-plane (HA) orchestration. The role already only
    supports a single control-plane host (`tasks/main.yml` reads the join
    command from `groups[k8s_control_plane_group][0]` only); this feature
    keeps that assumption rather than introducing HA handling as a side
    effect.
  - Exercising this end-to-end in Molecule. CI already can't run
    `kubeadm init`/join (no container runtime, no real cluster — see the
    existing "Important CI limitation" note in `CLAUDE.md`), so it can't run
    a real upgrade either. `k8s_upgrade_enabled` defaults to `false`, so the
    new code path is inert in CI; it's covered by an idempotency assertion
    (the new tasks produce no changes when disabled) rather than a live
    upgrade run.

## Trigger and gating

New task file `tasks/upgrade.yml`, included from `tasks/main.yml`:

```yaml
- name: "Kubernetes | Rolling upgrade"
  when: k8s_upgrade_enabled
  tags: [k8s, k8s-upgrade]
  ansible.builtin.include_tasks: upgrade.yml
```

Gated by `k8s_upgrade_enabled: false` (default) so normal bootstrap runs never
attempt an upgrade. When enabled, `k8s_upgrade_version` is required (a bare
version like `"1.32.5"`, no leading `v`) — validated with an `assert` at the
top of `upgrade.yml`.

## Node ordering and serialization

The role must be rolling regardless of the calling playbook's `serial:`
setting, so `upgrade.yml` runs as a single `run_once: true` block on the play,
looping over an ordered host list built once:

```yaml
- name: "Build ordered upgrade host list"
  run_once: true
  ansible.builtin.set_fact:
    k8s_upgrade_host_order: >-
      {{ (groups[k8s_control_plane_group] | default([]))[:1]
         + (groups[k8s_workers_group] | default([])) }}
```

Then a `loop: "{{ k8s_upgrade_host_order }}"` drives the per-node block below,
with `delegate_to: "{{ item }}"` for node-local work (package upgrade, kubeadm
upgrade command) and `delegate_to: "{{ k8s_upgrade_host_order[0] }}"` for
cluster-facing `kubectl` calls (drain/wait/uncordon), since that's the only
host guaranteed to have a working kubeconfig.

Because `k8s_upgrade_host_order` always starts with the control-plane host,
`item == k8s_upgrade_host_order[0]` distinguishes the control-plane step
(`kubeadm upgrade apply`) from worker steps (`kubeadm upgrade node`) inside
the same loop body — no separate control-plane/worker task blocks needed.

## Per-node steps (loop body)

1. **Idempotency check** — read the node's installed kubelet version
   (`dpkg-query`/`rpm -q` depending on `ansible_facts['os_family']`,
   delegated to `item`). If it already matches `k8s_upgrade_version`, skip
   the remaining steps for this node (`when` on a registered fact).
2. **Drain** (delegated to the control-plane host):
   `kubectl drain <node_name> --ignore-daemonsets --delete-emptydir-data
   --force --timeout={{ k8s_upgrade_drain_timeout }}`. `kubectl drain`
   cordons automatically, so there's no separate cordon step.
3. **Upgrade kubeadm package** (delegated to `item`), pinned to the target
   version using the same wildcard syntax the upstream kubeadm docs use:
   - Debian/Ubuntu (apt): `kubeadm={{ k8s_upgrade_version }}-*`
   - Fedora/RHEL (dnf, mirrors the existing install task's distro branching):
     `kubeadm-{{ k8s_upgrade_version }}*`
4. **Run the kubeadm upgrade command** (delegated to `item`):
   - Control-plane node: `kubeadm upgrade apply v{{ k8s_upgrade_version }} -y`
   - Worker node: `kubeadm upgrade node`
5. **Upgrade kubelet + kubectl packages** (delegated to `item`), same pinning
   pattern as step 3.
6. **Restart kubelet** (delegated to `item`): `ansible.builtin.systemd`,
   `state: restarted`.
7. **Uncordon + wait Ready** (delegated to the control-plane host):
   `kubectl uncordon <node_name>`, then
   `kubectl wait --for=condition=Ready node/<node_name> --timeout=300s`.
8. **Verify** (delegated to the control-plane host): assert
   `kubectl get node <node_name> -o jsonpath='{.status.nodeInfo.kubeletVersion}'`
   contains `k8s_upgrade_version`.

`<node_name>` is `hostvars[item]['ansible_nodename']` (the actual kubelet
registration name), not `inventory_hostname`, since kubeadm registers nodes
by kernel hostname and the two can differ.

## New variables

Added to `defaults/main.yml`, `meta/argument_specs.yml`, and the README
variables table (all three, per project convention — see
`readme-vars-sync`):

- `k8s_upgrade_enabled` (bool, default: `false`): Enable the rolling
  kubeadm/kubelet/kubectl upgrade path.
- `k8s_upgrade_version` (string, default: `""`): Target Kubernetes version
  (e.g. `"1.32.5"`, no leading `v`). Required when `k8s_upgrade_enabled` is
  `true`.
- `k8s_upgrade_drain_timeout` (string, default: `"300s"`): Timeout passed to
  `kubectl drain`.

## Documentation updates required

- `README.md`: add the three new variables to the Role Variables table, and
  a new "Rolling Upgrade" usage section under Example Playbook showing
  `k8s_upgrade_enabled`/`k8s_upgrade_version` with the `k8s-upgrade` tag.
- `meta/argument_specs.yml`: add the three new options.
- `CLAUDE.md`: extend the "Execution flow" numbered list with a new step 8
  describing the opt-in rolling upgrade path and its ordering guarantee, and
  extend the "Important CI limitation" paragraph to note the upgrade path
  is also inert in Molecule for the same reason (no real cluster).

## Testing

- `ansible-lint`/`yamllint` must pass on the new task file.
- Molecule: `k8s_upgrade_enabled` defaults to `false`, so existing
  converge/verify runs are unaffected. No new Molecule scenario is added
  (matches the existing documented limitation that steps needing a live
  cluster can't run in CI containers) — this is a documented gap, not a
  silently skipped one.
- Manual verification: since there's no real cluster in CI, correctness of
  the kubeadm command sequence is verified by construction (matching
  upstream kubeadm upgrade documentation) and by a syntax check
  (`ansible-playbook --syntax-check`) plus `ansible-lint` on `upgrade.yml`.
