# Agent Quick Reference

## Project overview
Ansible playbook collection for hardening Ubuntu servers + Vagrant dev environment. Two entrypoints: `bootstrap_vagrant.yml` (vagrant hosts) and `bootstrap_a_real_server.yml` (real servers). Roles run in fixed order: `common → docker → tailscale → cloudflared → reboot`.

## Critical executables & paths
- Run from `ansible/` directory for all playbook commands (uses relative `ansible.cfg`)
- `ansible.cfg` pins `roles_path = ./roles` — running from repo root or outside `ansible/` breaks role resolution
- Inventory: `ansible/inventory.ini` (copy from `inventory.ini.example`, never committed per `.gitignore`)
- Vault secrets: `ansible/host_vars/{hostname}.yml` encrypted with `ansible-vault`

## Running playbooks
Vagrant (default, includes all optional roles):
- `cd ansible && ansible-playbook -i inventory.ini playbooks/bootstrap_vagrant.yml --ask-vault-pass`

Real server (same roles, different host group):
- `cd ansible && ansible-playbook -i inventory.ini playbooks/bootstrap_a_real_server.yml --ask-vault-pass`

Skip vault prompt when no secrets enabled (no tailscale/cloudflared):
- Omit `--ask-vault-pass`

Alternative with password file (avoid typing):
- `--vault-password-file=/path/to/vault_password.txt`

## Inventory setup (non-obvious)
Required for both vagrant and real servers:
- `inventory.ini` must contain `[vagrant]` or `[a_real_server]` group matching the playbook target
- Group name must match the `hosts:` value in the playbook (`vagrant` vs `a_real_server`)
- `inventory.ini` is `.gitignore`-d — never commit with real keys/hosts
- Vagrant: get host/port/key via `cd vagrant && vagrant ssh-config`, then map to `ansible_host`, `ansible_port`, `ansible_ssh_private_key_file`
- Host vars use `inventory_hostname` for vault filename: `host_vars/{hostname}.yml` (e.g., `default.yml`)

## Vault secrets (required for optional roles)
Tailscale + Cloudflared roles require encrypted vars:
- Create: `cd ansible && ansible-vault create host_vars/{hostname}.yml`
- Edit: `ansible-vault edit host_vars/{hostname}.yml`
- Required keys when enabled:
  - `tailscale_authkey: <token>`
  - `cloudflared_token: <token>`
- Passphrase must be provided via `--ask-vault-pass` or `--vault-password-file`

## Role behaviors & quirks
- `common`: runs unattended-upgrades, enables UFW (deny all, allow SSH+80+443), starts fail2ban
- `docker`: adds Docker repo and GPG key with `creates=` guards (safe to rerun)
- `tailscale`: uses shell installer with `creates=/usr/sbin/tailscale`; checks `tailscale status --json` to avoid redundant `up` calls; requires `tailscale_authkey`
- `cloudflared`: installs .deb and runs `cloudflared service install {{ cloudflared_token }}` with `creates=/etc/systemd/system/cloudflared.service` guard
- `reboot`: conditional on `/var/run/reboot-required` existing after prior roles
- All roles use `become: yes` at playbook level

## Testing / local dev
Use included Vagrantfile to spin up a disposable Ubuntu VM:
- `cd vagrant && vagrant up` — provisions VM
- `cd vagrant && vagrant ssh-config` — get connection info for inventory.ini
- `cd vaglements?` — no, `cd ansible` for playbook runs
- `cd vagrant && vagrant destroy -f` — teardown
- Cleanup SSH known_hosts after destroy: `ssh-keygen -f '~/.ssh/known_hosts' -R '[172.29.112.1]:2222'` (adjust IP/port per your config)

## Tooling expectations
- Ansible installed on control machine (this repo does not manage control-machine Ansible install)
- Target machines: Ubuntu (roles assume apt/dpkg)
- Optional components: Virtualbox + Vagrant for local testing
- Windows users: WSL + Virtualbox on host (not WSL) with `VAGRANT_WSL_ENABLE_WINDOWS_ACCESS=1`

## What not to do
- Do not run playbooks from repo root — `ansible.cfg` is only effective from `ansible/`
- Do not commit `inventory.ini` or vault files
- Do not enable tailscale/cloudflared roles without providing vault secrets (playbook will fail on missing vars)
- Do not assume idempotency of `tailscale up` without the status check guard (the role includes it, but custom runs may omit it)
