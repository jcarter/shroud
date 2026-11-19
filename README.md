# Shroud - Secure Server Setup

Ansible playbook for hardening new Debian/Ubuntu VPS instances. Run from macOS; targets servers where a non-root sudo user with SSH key access already exists.

## Design

- **No public SSH**: SSH is locked to the Tailscale WireGuard interface only.
- **UFW + ufw-docker**: Docker bypasses UFW by default. The `firewall` role deploys the [ufw-docker](https://github.com/chaifeng/ufw-docker) `DOCKER-USER` chain rules so UFW controls container routing.
- **devsec.hardening**: OS and SSH hardening use the battle-tested [devsec.hardening](https://github.com/dev-sec/ansible-collection-hardening) collection, configured via variables.
- **Idempotent**: Safe to re-run. Each role checks current state before making changes.

## Architecture

```
baseline → tailscale → os_hardening → ssh_hardening → firewall → fail2ban → auto_updates → auditd → service_cleanup → dokploy
```

Role order is important. Tailscale must be up **before** the firewall role blocks public SSH.

## Prerequisites

- ansible

```bash
cd shroud
ansible-galaxy collection install -r requirements.yml
```

**Tailscale pre-auth key:**
Generate a non-reusable pre-auth key at [https://login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys) and pass it on the host's first run with `-e tailscale_authkey=tskey-auth-...`. It is not stored anywhere; later runs skip `tailscale up` because the node is already joined. Start the command with a space so fish keeps it out of history.

**Target server:**

- Debian 12+ or Ubuntu 22.04+
- Non-root user with `sudo` and SSH key access already configured
- Python is installed automatically by the playbook's first play if missing

## Inventory

`inventory/hosts.yml` defines the host. The provided file uses `debian` as the host name — rename the key and the host_vars directory if your server has a different name.

Copy the host_vars template and fill in your connection details:

```bash
cp inventory/host_vars/debian/local.yml.example inventory/host_vars/debian/local.yml
```

Edit `local.yml` with your server's IP and user. This file is gitignored and loaded automatically — no extra flags needed. After the first run, switch `ansible_host` to the Tailscale IP (e.g. `100.x.y.z`) or MagicDNS name.

## Usage

```bash
# If the remote user requires a password for sudo, append -K (prompts once):
# ansible-playbook ... -K

# Preview changes (dry run — no changes made)
ansible-playbook playbooks/harden.yml --check --diff -K

# First run (leading space keeps the one-time key out of fish history)
 ansible-playbook playbooks/harden.yml -K -e tailscale_authkey=tskey-auth-...

# Subsequent runs (via Tailscale — sudo password still required unless NOPASSWD)
ansible-playbook playbooks/harden.yml -K

# Target a specific role
ansible-playbook playbooks/harden.yml --tags firewall -K

# Verify hardening
ansible-playbook playbooks/verify.yml -K
```

> **Tip:** To avoid typing the password on every run, add `deploy ALL=(ALL) NOPASSWD:ALL`
> to `/etc/sudoers.d/deploy` on the target server (substitute your username). This is safe when SSH key auth is the
> only login mechanism (no password SSH, no root SSH).

## Variables

Role defaults live in each role's `defaults/main.yml`. Shared deployment policy used by roles and standalone playbooks lives in `inventory/group_vars/all.yml`. Host-specific overrides (including secrets) go in `inventory/host_vars/[[ORCA_RICH_MD:626624042c47bceb881600e2bb5248f5:inline-html:%3Chostname%3E]]/local.yml` (gitignored — see Inventory above).


| Variable                       | Default                                 | Description                                                                                                  |
| ------------------------------ | --------------------------------------- | ------------------------------------------------------------------------------------------------------------ |
| `tailscale_authkey`            | *(none)*                                | One-time Tailscale pre-auth key. Required only on a host's first run; pass with `-e`.                        |
| `tailscale_ssh_enabled`        | `false`                                 | Enable Tailscale SSH daemon (separate from sshd).                                                            |
| `tailscale_up_flags`           | `""`                                    | Extra `tailscale up` flags, first join only; joined hosts use `sudo tailscale set --accept-routes=false`.    |
| `dokploy_install`              | `false`                                 | Install Dokploy on this host.                                                                                |
| `dokploy_docker_dns_servers`   | `["100.100.100.100"]`                   | Stable upstream DNS for Docker containers; defaults to Tailscale Quad100.                                    |
| `dokploy_port`                 | `3000`                                  | Dokploy panel port.                                                                                          |
| `dokploy_tailscale_only`       | `true`                                  | Restrict Dokploy to Tailscale interface.                                                                     |
| `dokploy_version`              | `""`                                    | Pin Dokploy version (empty = latest).                                                                        |
| `dokploy_advertise_addr`       | `""`                                    | Docker Swarm advertise address (empty = auto-detect).                                                        |
| `firewall_public_tcp_ports`    | `[80, 443]` if `dokploy_install`, else `[]` | TCP ports open to the public internet.                                                                   |
| `docker_public_tcp_ports`      | `[80, 443]` if `dokploy_install`, else `[]` | Container TCP ports routed from the public internet.                                                     |
| `ufw_forward_policy`           | `"DROP"`                                | UFW default forward policy.                                                                                  |
| `fail2ban_bantime`             | `1h`                                    | Ban duration.                                                                                                |
| `fail2ban_findtime`            | `10m`                                   | Window in which failures are counted.                                                                        |
| `fail2ban_maxretry`            | `5`                                     | Failures before ban.                                                                                         |
| `fail2ban_backend`             | `"systemd"`                             | Log backend for fail2ban.                                                                                    |
| `auto_reboot`                  | `true`                                  | Reboot when an update requires it; skipped while users are logged in.                                        |
| `auto_reboot_time`             | `"09:00"`                               | Reboot time in host local time (UTC by default).                                                             |
| `auto_update_restart_services` | `true`                                  | needrestart restarts services using updated libraries; docker and containerd excluded.                       |
| `auto_update_remove_unused`    | `true`                                  | Remove unused dependencies after upgrades.                                                                   |
| `audit_log_size`               | `100`                                   | MB per audit log file (`max_log_file`).                                                                      |
| `audit_num_logs`               | `5`                                     | Number of audit log files kept.                                                                              |
| `audit_backlog_limit`          | `8192`                                  | Kernel audit backlog buffer size.                                                                            |
| `baseline_timezone`            | `"UTC"`                                 | Host timezone.                                                                                               |
| `baseline_packages`            | `[libpam-pwquality, curl, gnupg, lsb-release, apt-transport-https]` | Base packages installed on every host.                                       |
| `baseline_extra_packages`      | `[]`                                    | Host-specific packages; set `[linux-headers-amd64]` on hosts with dkms modules (e.g. NVIDIA).                |
| `disable_avahi`                | `true`                                  | Disable avahi-daemon (mDNS).                                                                                 |
| `disable_bluetooth`            | `true`                                  | Disable bluetooth service.                                                                                   |
| `disable_samba`                | `false`                                 | Disable Samba services.                                                                                      |
| `ssh_allowed_users`            | `[deploy]`                              | SSH login allowlist; set per host in local.yml and must include `ansible_user`.                              |
| `sysctl_overwrite`             | `net.ipv4.ip_forward: 1`                | Preserve Docker IP forwarding while applying OS hardening.                                                   |


## Roles


| Role                             | Tag               | Description                                            |
| -------------------------------- | ----------------- | ------------------------------------------------------ |
| `baseline`                       | `baseline`        | apt upgrade, package install, timezone                 |
| `tailscale`                      | `tailscale`       | Install Tailscale, join tailnet                        |
| `devsec.hardening.os_hardening`  | `os_hardening`    | Kernel/sysctl hardening                                |
| `devsec.hardening.ssh_hardening` | `ssh_hardening`   | sshd configuration                                     |
| `firewall`                       | `firewall`        | UFW + ufw-docker                                       |
| `fail2ban`                       | `fail2ban`        | SSH jail                                               |
| `auto_updates`                   | `auto_updates`    | unattended-upgrades                                    |
| `auditd`                         | `auditd`          | Audit logging rules                                    |
| `service_cleanup`                | `service_cleanup` | Disable unnecessary services                           |
| `dokploy`                        | `dokploy`         | Install Dokploy (optional, gated by `dokploy_install`) |


## Risks


| Risk                                     | Mitigation                                                                                     |
| ---------------------------------------- | ---------------------------------------------------------------------------------------------- |
| UFW locks out SSH                        | Tailscale role runs before firewall; SSH allowed on `tailscale0` before UFW enables            |
| Docker loses network                     | ufw-docker DOCKER-USER chain rules + `ufw route allow` preserve container traffic              |
| Containers capture an empty DNS resolver | Docker uses the configured Tailscale Quad100 resolver instead of transient host resolver state |
| os\_hardening disables IP forwarding     | `sysctl_overwrite` in group\_vars preserves Docker forwarding                                  |
| Tailscale auth key already consumed      | Playbook checks backend state; skips `tailscale up` if already `Running`                       |
| Tailscale goes down                      | sshd still listens on port 22; keep VPS provider console access as emergency backdoor          |
| Unattended reboot leaves host down or GPU missing (dkms module fails on new kernel; recovery needs physical console since SSH is Tailscale-only) | Reboots only at `auto_reboot_time` when required; set `auto_reboot: false` in a host's local.yml to opt out |


## Migrating Dokploy to New Host

In order to move Dokploy from one server to another, you'll need to first set everything up in shroud.
After doing that, you'll want to 

- Make a backup of the Web Server and restore it on the new instance
- Move Docker volumes from the old instance to the new instance

Previously this included,

- Stop the Docker service - `systemctl stop docker.socket docker.service`
- Tar the Docker volumes on the old server at `/var/lib/docker/volumes/`
- Move tars to new server
- Untar volumes
- Start the Docker service