---
name: extending-secure-server-setup
description: Use when adding new roles, modifying firewall rules, adding hosts, or integrating new services into the secure-server-setup Ansible project
---

# Extending secure-server-setup

## Overview

This project uses a role-per-concern structure. Each role is independently runnable via `--tags <rolename>`. Role execution order is load-bearing for security — violating it locks you out of the server.

## Checklist: Adding a New Role

When adding a role, touch these files in order:

| Step | File | What to do |
|---|---|---|
| 1 | `roles/<name>/defaults/main.yml` | Role defaults with inline comments |
| 2 | `roles/<name>/tasks/main.yml` | Tasks using FQCN module names |
| 3 | `roles/<name>/handlers/main.yml` | ONLY if tasks include a `notify` directive — do not create an empty handlers file |
| 4 | `roles/<name>/templates/*.j2` | ONLY if role deploys managed config files — include `# {{ ansible_managed }}` as first line |
| 5 | `inventory/group_vars/all.yml` | Duplicate operator-tunable variables with a section header comment |
| 6 | `playbooks/harden.yml` | Add role entry + update comment block numbering |
| 7 | `playbooks/verify.yml` | Add `systemctl is-active` check and any `assert` tasks |
| 8 | `requirements.yml` | Only if you need a Galaxy collection not already listed |

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

```yaml
# tasks/main.yml — always use FQCN module names
- name: Install <package>
  ansible.builtin.apt:
    name: <package>
    state: present
    update_cache: false   # baseline handles cache; omitting is ambiguous, be explicit

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

```yaml
# handlers/main.yml
- name: Restart <service>
  ansible.builtin.systemd:   # NOT ansible.builtin.service — except auditd (needs signal)
    name: <service>
    state: restarted
```

## Variable Placement

**Two-level convention:** Variables exist in two places simultaneously.

```yaml
# roles/<name>/defaults/main.yml — role fallback, always present
my_service_port: 9100
my_service_tailscale_only: true
```

```yaml
# inventory/group_vars/all.yml — operator-visible overrides, add a section header
# =============================================================================
# My Service role (custom)
# =============================================================================
my_service_port: 9100
my_service_tailscale_only: true   # inline comment explains the tradeoff
```

Put variables in `group_vars/all.yml` when operators would reasonably need to tune them per deployment. Internal implementation details stay in `defaults/` only.

## UFW Rules in Service Roles

Service roles MAY add their own UFW rules — this is the established pattern (see `dokploy_tailscale_only` in the firewall role, and the Tailscale-only conditional used throughout).

**Critical:** service roles that call `community.general.ufw` implicitly depend on:
1. `ufw` package installed (by `baseline` role)
2. UFW enabled (by `firewall` role)

Document this in the task comment:

```yaml
# Requires: ufw installed (baseline), UFW enabled (firewall role).
# Safe to run standalone only after both have completed at least once.
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

# Update the comment block at the top of harden.yml:
#  10. my-new-role   — brief description
```

Tag must be a single-element list `[rolename]`. Bare string tags are not used in this project.

## Adding to verify.yml

Every new service role should add at minimum:

```yaml
- name: Check <service> is running
  ansible.builtin.command: systemctl is-active <service>
  register: <service>_active
  changed_when: false
  failed_when: false

- name: Assert <service> is active
  ansible.builtin.assert:
    that: "<service>_active.stdout == 'active'"
    fail_msg: "FAIL — <service> is not running"
    success_msg: "PASS — <service> is running"
```

Add these tasks before the final summary `debug` task at the bottom of verify.yml.

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

## Common Mistakes

| Mistake | Correct approach |
|---|---|
| Running service role before firewall role | Place new roles after `service_cleanup` in harden.yml |
| Adding UFW rules without dependency comment | Add `# Requires: ufw installed (baseline), UFW enabled (firewall role)` |
| Skipping `update_cache: false` on apt installs | Always be explicit — baseline handles the cache update |
| Forgetting to update verify.yml | New services without verification checks are unmonitored after hardening |
| Using bare tag string instead of list | Use `tags: [rolename]`, not `tags: rolename` |
| Adding variables only to defaults/, not group_vars/ | Operator-tunable vars go in both — group_vars is the operator's interface |
| Creating handlers/main.yml with no `notify` in tasks | Empty handler file — delete handlers/ entirely if no tasks use `notify` |
