# Ansible Example

This project automates hardening and provisioning of Ubuntu servers via Ansible. It includes a Vagrant-based local development environment for safe testing before deploying to real machines.

What this setup does:

- Upgrades the system
- Installs base packages
- Enables automatic security upgrades
- Configures UFW firewall (deny all, allow SSH/HTTP/HTTPS)
- Enables fail2ban
- Installs Docker
- Reboots if required after updates

Optional roles:
- Install and configure Tailscale
- Install Cloudflared and set up a tunnel

## Requirements

- Ubuntu (or other Debian-based Linux) as the **control machine** where Ansible runs
- VirtualBox (for local Vagrant testing)
- Vagrant
- Ansible

### Install required software (Ubuntu/Debian)

On the control machine (or WSL), install Ansible, Vagrant, and VirtualBox:

```bash
# Update system and install Ansible
sudo apt update
sudo apt install -y software-properties-common
sudo apt-add-repository --yes --update ppa:ansible/ansible
sudo apt install -y ansible

# Install Vagrant
# Option A: use HashiCorp repo (recommended for latest stable)
curl -fsSL https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update
sudo apt install -y vagrant

# Option B: or use distro package (may be older)
# sudo apt install -y vagrant

# Install VirtualBox
# Option A: use official repo (recommended for latest stable)
curl -fsSL https://www.virtualbox.org/download/oracle_vbox_2016.asc | sudo gpg --dearmor -o /usr/share/keyrings/oracle-virtualbox-2016.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/oracle-virtualbox-2016.gpg] http://download.virtualbox.org/virtualbox/debian $(lsb_release -cs) contrib" | sudo tee /etc/apt/sources.list.d/virtualbox.list
sudo apt update
sudo apt install -y virtualbox-7.1  # or virtualbox-7.0 / virtualbox-6.1

# Option B: distro package (may be older)
# sudo apt install -y virtualbox
```

Verify installations:

```bash
ansible --version
vagrant --version
vboxmanage --version
```

### Windows (WSL)

If using WSL, install Ansible and Vagrant inside WSL and **VirtualBox on the Windows host** — see `docs/windows-instructions.md` for the full Windows + WSL workflow and required host/guest setup.

## Quick Start (Local Testing with Vagrant)

1. Start the VM:
   ```bash
   cd vagrant
   vagrant up
   ```

2. Get connection details for the inventory:
   ```bash
   cd vagrant
   vagrant ssh-config
   ```

3. Create the inventory file (copy from example):
   ```bash
   cd ansible
   cp inventory.ini.example inventory.ini
   ```
   Edit `inventory.ini` with the host, port, and SSH key path from `vagrant ssh-config`.

4. Run the playbook:
   ```bash
   cd ansible
   ansible-playbook -i inventory.ini playbooks/bootstrap_vagrant.yml --ask-vault-pass
   ```
   (Omit `--ask-vault-pass` if you don't use Tailscale/Cloudflared roles.)

5. Clean up:
   ```bash
   cd vagrant
   vagrant destroy -f
   ```
   If you hit SSH key issues after destroy, run:
   ```bash
   ssh-keygen -f "~/.ssh/known_hosts" -R "[172.29.112.1]:2222"
   ```

## Using on a Real Server

1. Add your server to `ansible/inventory.ini`:
   ```ini
   [a_real_server]
   yoursrv ansible_host=1.2.3.4 ansible_port=22 ansible_user=youruser ansible_ssh_private_key_file=/path/to/key
   ```

2. Run the real-server playbook:
   ```bash
   cd ansible
   ansible-playbook -i inventory.ini playbooks/bootstrap_a_real_server.yml --ask-vault-pass
   ```

The inventory group name (`a_real_server`) must match the `hosts:` value in the playbook.

## Vault Secrets

Tailscale and Cloudflared require encrypted variables:

```bash
cd ansible
ansible-vault create host_vars/{hostname}.yml
```

For a host named `default`:
```yaml
tailscale_authkey: <your_tailscale_auth_token>
cloudflared_token: <your_cloudflare_tunnel_token>
```

The vault filename is derived from `inventory_hostname`. Omit `--ask-vault-pass` if you have no optional roles enabled.

## Project Structure

- `ansible/` — All Ansible content; must run playbooks from here (`.cfg` sets `roles_path = ./roles`)
- `ansible/roles/` — Role definitions; executed in order: common → docker → tailscale → cloudflared → reboot
- `vagrant/` — Local VM definition and provisioning helper scripts

See `AGENTS.md` for detailed agent-oriented guidance.

# Troubleshooting

- **SSH issues after `vagrant destroy`** — Clear known_hosts entry for the VM IP/port:
  ```bash
  ssh-keygen -f ~/.ssh/known_hosts -R "[192.168.56.10]:22"
  ```
- **Playbook run from wrong directory** — Always `cd ansible` first; roles won't resolve from repo root.
- **Missing vault secrets** — If enabling Tailscale/Cloudflared and the playbook fails on missing vars, ensure `host_vars/{hostname}.yml` exists and is encrypted, and pass `--ask-vault-pass`.

