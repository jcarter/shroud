# Shroud - Secure Server Setup

Ansible playbook for hardening new Debian/Ubuntu VPS instances. Run from macOS; targets servers where a non-root sudo user with SSH key access already exists.

## Design

- **No public SSH**: SSH is locked to the Tailscale WireGuard interface only. Zero public attack surface.
- **UFW + ufw-docker**: Docker bypasses UFW by default. The `firewall` role deploys the [ufw-docker](https://github.com/chaifeng/ufw-docker) `DOCKER-USER` chain rules so UFW controls container routing.
- **devsec.hardening**: OS and SSH hardening use the battle-tested [devsec.hardening](https://github.com/dev-sec/ansible-collection-hardening) collection, configured via variables.
- **Idempotent**: Safe to re-run. Each role checks current state before making changes.

## Architecture

```
baseline → tailscale → os_hardening → ssh_hardening → firewall → fail2ban → auto_updates → auditd → service_cleanup → dokploy
```

Role order is load-bearing: Tailscale must be up **before** the firewall role blocks public SSH.

## Prerequisites

**On the control machine (macOS):**
```bash
brew install ansible 1password-cli
cd ~/Source/shroud
ansible-galaxy collection install -r requirements.yml
```

**Tailscale pre-auth key (1Password):**
Sign in with `op`, then enable 1Password desktop CLI integration. Generate a new non-reusable pre-auth key at https://login.tailscale.com/admin/settings/keys and store it in the `password` field of an item named `Tailscale`. The local vars template reads it with `lookup('community.general.onepassword', 'Tailscale', field='password')`; run the playbook normally. The auth keys are one time uses pretty much.

**Target server:**
- Debian 12+ or Ubuntu 22.04+
- Non-root user with `sudo` and SSH key access already configured

## Inventory

`inventory/hosts.yml` defines the host. The provided file uses `debian` as the host name — rename the key and the host_vars directory if your server has a different name.

Copy the host_vars template and fill in your connection details:

```bash
cp inventory/host_vars/debian/local.yml.example inventory/host_vars/debian/local.yml
```

Edit `local.yml` with your server's IP and user. The Tailscale auth key is read from the 1Password item `Tailscale` by default. This file is gitignored and loaded automatically — no extra flags needed. After the first run, switch `ansible_host` to the Tailscale IP (e.g. `100.x.y.z`) or MagicDNS name.

## Usage

```bash
# If the remote user requires a password for sudo, append -K (prompts once):
# ansible-playbook ... -K

# Preview changes (dry run — no changes made)
ansible-playbook -i inventory/hosts.yml playbooks/harden.yml --check --diff -K

# First run (tailscale_authkey is read from host_vars/debian/local.yml)
ansible-playbook -i inventory/hosts.yml playbooks/harden.yml -K

# Subsequent runs (via Tailscale — sudo password still required unless NOPASSWD)
ansible-playbook -i inventory/hosts.yml playbooks/harden.yml -K

# Target a specific role
ansible-playbook -i inventory/hosts.yml playbooks/harden.yml --tags firewall -K

# Verify hardening
ansible-playbook -i inventory/hosts.yml playbooks/verify.yml -K
```

> **Tip:** To avoid typing the password on every run, add `deploy ALL=(ALL) NOPASSWD:ALL`
> to `/etc/sudoers.d/deploy` on the target server (substitute your username). This is safe when SSH key auth is the
> only login mechanism (no password SSH, no root SSH).

## Variables

Defaults are in `inventory/group_vars/all.yml`. Host-specific overrides (including secrets) go in `inventory/host_vars/<hostname>/local.yml` (gitignored — see Inventory above).

| Variable | Default | Description |
|---|---|---|
| `tailscale_authkey` | _(none — required)_ | Tailscale pre-auth key. Set in `host_vars/*/local.yml`; the template reads the 1Password item `Tailscale` password field. |
| `tailscale_ssh_enabled` | `false` | Enable Tailscale SSH daemon (separate from sshd). |
| `firewall_public_tcp_ports` | `[80, 443]` | TCP ports open to the public internet. |
| `dokploy_port` | `3000` | Dokploy panel port. |
| `dokploy_tailscale_only` | `true` | Restrict Dokploy to Tailscale interface. |
| `dokploy_install` | `false` | Install Dokploy on this host. |
| `dokploy_version` | `""` | Pin Dokploy version (empty = latest). |
| `dokploy_advertise_addr` | `""` | Docker Swarm advertise address (empty = auto-detect). |
| `fail2ban_bantime` | `1h` | Ban duration. |
| `fail2ban_maxretry` | `5` | Failures before ban. |
| `auto_reboot` | `false` | Auto-reboot after unattended upgrades. |
| `auto_update_mail` | `""` | Email for upgrade notifications. |
| `disable_avahi` | `true` | Disable avahi-daemon (mDNS). |
| `disable_bluetooth` | `true` | Disable bluetooth service. |
| `disable_samba` | `false` | Disable Samba services. |
| `os_network_forwarding` | `true` | IP forwarding (required for Docker). |

## Roles

| Role | Tag | Description |
|---|---|---|
| `baseline` | `baseline` | apt upgrade, package install, timezone |
| `tailscale` | `tailscale` | Install Tailscale, join tailnet |
| `devsec.hardening.os_hardening` | `os_hardening` | Kernel/sysctl hardening |
| `devsec.hardening.ssh_hardening` | `ssh_hardening` | sshd configuration |
| `firewall` | `firewall` | UFW + ufw-docker |
| `fail2ban` | `fail2ban` | SSH jail |
| `auto_updates` | `auto_updates` | unattended-upgrades |
| `auditd` | `auditd` | Audit logging rules |
| `service_cleanup` | `service_cleanup` | Disable unnecessary services |
| `dokploy` | `dokploy` | Install Dokploy (optional, gated by `dokploy_install`) |

## Risks

| Risk | Mitigation |
|---|---|
| UFW locks out SSH | Tailscale role runs before firewall; SSH allowed on `tailscale0` before UFW enables |
| Docker loses network | ufw-docker DOCKER-USER chain rules + `ufw route allow` preserve container traffic |
| os_hardening disables IP forwarding | `os_network_forwarding: true` in group_vars |
| Tailscale auth key already consumed | Playbook checks backend state; skips `tailscale up` if already `Running` |
| Tailscale goes down | sshd still listens on port 22; keep VPS provider console access as emergency backdoor |
