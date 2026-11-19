---
name: extending-shroud
description: Use when adding new roles, modifying firewall rules, adding hosts, or integrating new services into the Shroud Ansible project
---

# Extending Shroud

## Overview

This project uses a role-per-concern structure. Each role is independently runnable via `--tags <rolename>`. Role execution order is load-bearing for security — violating it locks you out of the server.

## Checklist: Adding a New Role

When adding a role, touch these files in order:

| Step | File | What to do |
|---|---|---|
| 1 | `roles/<name>/defaults/main.yml` | Every role variable with its default and inline comments |
| 2 | `roles/<name>/tasks/main.yml` | Install, configure, and enable the service; FQCN module names |
| 3 | `roles/<name>/handlers/main.yml` | ONLY if tasks include a `notify` directive — do not create an empty handlers file |
| 4 | `roles/<name>/templates/*.j2` | ONLY if role deploys managed config files — include `# {{ ansible_managed }}` as first line |
| 5 | `inventory/group_vars/all.yml` | ONLY if `verify.yml` or another standalone playbook also needs the variable |
| 6 | `playbooks/harden.yml` | Add the role entry in the correct slot |
| 7 | `playbooks/verify.yml` | Add the service to the `service_facts` loop and any `assert` tasks |
| 8 | `requirements.yml` | Only if you need a Galaxy collection not already listed |
| 9 | `README.md` | Add variables to the Variables table and the role to the Roles table |

## Execution Order (Do Not Violate)

```
baseline → tailscale → os_hardening → ssh_hardening → firewall → [hardening roles] → [service roles]
```

**The critical constraint:** `tailscale` MUST complete before `firewall`. The firewall role blocks public SSH and allows SSH only on `tailscale0`. If firewall runs before Tailscale joins the tailnet, you are locked out.

| Slot | Roles that belong here |
|---|---|
| Before tailscale | Nothing — baseline must be first |
| Between tailscale and firewall | Nothing — this gap is the lock-out window |
| After firewall | fail2ban, auto_updates, auditd, service_cleanup |
| Last (service/observability) | Any new service role (monitoring, backup, etc.) |

New roles go after `service_cleanup` unless they are hardening (go before it).

## Role Conventions

A role owns the packages, config, and service for its concern. `baseline` installs only generic packages and refreshes the apt cache; do not add a role's own package to `baseline_packages`. Packages only one host needs go in `baseline_extra_packages` in that host's `local.yml`.

```yaml
# tasks/main.yml — always use FQCN module names
- name: Install <package>
  ansible.builtin.apt:
    name: <package>
    state: present
    update_cache: false   # baseline handles cache; omitting is ambiguous, be explicit
  register: <role>_package

# Gate everything that needs the package so --check works on a fresh host
- name: Configure and start <service>
  when: not ansible_check_mode or not <role>_package.changed
  block:
    - name: Deploy config
      ansible.builtin.template:
        src: config.j2
        dest: /etc/service/config
        mode: '0644'
        owner: root
        group: root
      notify: Restart <service>

    - name: Enable <service>
      ansible.builtin.systemd:
        name: <service>
        enabled: true
        state: started
```

Check mode cannot install packages, so tasks that need the package (templates into its config dir, `systemd`, `command`) must be skipped when the install is only predicted. Put the shared condition on one `block`, not on every task. Never use `ignore_errors: "{{ ansible_check_mode }}"` or `set_fact` "runtime safe" flags. Read-only `command` probes whose result later tasks depend on get `check_mode: false`.

```yaml
# handlers/main.yml
- name: Restart <service>
  ansible.builtin.systemd:   # NOT ansible.builtin.service — except auditd (systemd refuses manual restart)
    name: <service>
    state: restarted
```

## Variable Placement

Role variables live in one place: `roles/<name>/defaults/main.yml`. Operators override them per host in `inventory/host_vars/<host>/local.yml` (gitignored).

```yaml
# roles/<name>/defaults/main.yml
my_service_port: 9100
my_service_tailscale_only: true   # inline comment explains the tradeoff
```

Name booleans positively, as `<thing>_enabled` where `true` means on (`samba_enabled: false`, not `disable_samba: true`).

`inventory/group_vars/all.yml` holds only:
- `devsec.hardening` variables (`ssh_*`, `sysctl_overwrite`, `os_*`)
- `ssh_allowed_users`, the fallback list; each host overrides it in `host_vars/<host>/local.yml`
- shared policy that standalone playbooks need, which do not load role defaults (currently `dokploy_install` and `dokploy_docker_dns_servers`, used by `verify.yml`)

Do not copy role defaults into `group_vars`. Two copies drift.

## UFW Rules in Service Roles

Service roles MAY add their own UFW rules — this is the established pattern (see `dokploy_tailscale_only` in the firewall role, and the Tailscale-only conditional used throughout).

**Critical:** service roles that call `community.general.ufw` implicitly depend on:
1. `ufw` package installed (by the `firewall` role)
2. UFW enabled (by the `firewall` role)

Document this in the task comment:

```yaml
# Requires: UFW installed and enabled (firewall role).
# Safe to run standalone only after the firewall role has completed at least once.
- name: Allow <service> via Tailscale only
  community.general.ufw:
    rule: allow
    port: "{{ my_service_port | string }}"
    proto: tcp
    interface: tailscale0
    direction: in
    comment: "<service> via Tailscale only"
  when: my_service_tailscale_only | bool
```

## Wiring into harden.yml

```yaml
# In the roles list — match tag to role directory name exactly
    - role: my-new-role
      tags: [my-new-role]
```

Tag must be a single-element list `[rolename]`. Bare string tags are not used in this project. The roles run in a second play after a `raw` bootstrap play that installs `python3` and `python3-apt`; add roles to the second play only. There is no numbered order comment to update; the list order is the order.

## Adding to verify.yml

`verify.yml` is read-only and runs with `check_mode: false` at play level, so it reports real state even under `--check`. Service state comes from one `service_facts` call. Add a new service to the loop:

```yaml
- name: Assert required security services are active
  ansible.builtin.assert:
    that:
      - "(item + '.service') in ansible_facts.services"
      - "ansible_facts.services[item + '.service'].state == 'running'"
  loop:
    - fail2ban
    - auditd
    - unattended-upgrades
    - <service>    # add here
```

Also assert config values the role manages (see the sysctl, audit rules, and Dokploy DNS checks). Gate optional-service checks with `when: <feature>_install | bool`, and keep that variable in `group_vars/all.yml` because `verify.yml` does not load role defaults. Diagnostic `debug` output uses `verbosity: 1`.

## External Collections

Only update `requirements.yml` when you need a collection NOT already present:

```bash
# Check what's already pinned
cat requirements.yml
# Already available: devsec.hardening, community.general, ansible.posix
```

If you add a collection:
```yaml
  - name: prometheus.prometheus
    version: ">=0.14.0"
```

Run `ansible-galaxy collection install -r requirements.yml` after adding.

## Adding a Host

Add the host to `inventory/hosts.yml` (`ansible_host`, `ansible_user`, `ansible_become`, `ansible_port`). Connection details and per-host overrides go in `inventory/host_vars/<host>/local.yml`, which is gitignored; copy `local.yml.example` as a starting point. Set `ssh_allowed_users` there to include the host's `ansible_user`, or SSH hardening will lock that user out. Do not store the Tailscale auth key; pass a one-time key on the host's first run with `-e tailscale_authkey=tskey-auth-...`. `ansible.cfg` sets the inventory path, so commands need no `-i`.

## Common Mistakes

| Mistake | Correct approach |
|---|---|
| Running service role before firewall role | Place new roles after `service_cleanup` in harden.yml |
| Adding UFW rules without dependency comment | Add `# Requires: UFW installed and enabled (firewall role)` |
| Skipping `update_cache: false` on apt installs | Always be explicit — baseline handles the cache update |
| Adding a role's package to `baseline_packages` | The role installs its own package |
| Gating each task with a check-mode flag or `ignore_errors` | One `when: not ansible_check_mode or not <pkg>.changed` block |
| Forgetting to update verify.yml | Add the service to the `service_facts` loop |
| Using bare tag string instead of list | Use `tags: [rolename]`, not `tags: rolename` |
| Copying role defaults into group_vars | Defaults live only in `roles/<name>/defaults/main.yml` |
| Needing a variable in verify.yml but leaving it only in role defaults | Put it in `group_vars/all.yml`; standalone playbooks do not load role defaults |
| Creating handlers/main.yml with no `notify` in tasks | Empty handler file — delete handlers/ entirely if no tasks use `notify` |
| Adding a host without setting `ssh_allowed_users` in its `local.yml` | The new `ansible_user` is locked out of SSH |
