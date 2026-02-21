# Windows + WSL Setup (Legacy)

This document captures the original Windows-focused workflow for running the Ansible playbooks via WSL. Most users on Linux can skip this — see `README.md` for the primary Linux-based instructions.

## Prerequisites

- Windows 10/11 with WSL enabled
- VirtualBox installed on the **Windows host** (not inside WSL)
- Vagrant installed inside WSL
- Ansible installed inside WSL

## WSL ⇄ VirtualBox Access

Vagrant inside WSL needs access to the Windows-hosted VirtualBox. Enable permanent access:

```bash
echo 'export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS=1' >> ~/.bash_profile
source ~/.bash_profile
```

For a single-session (temporary) enable:

```bash
export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"
```

## Workflow

1. Start the VM from WSL:

   ```bash
   cd vagrant
   vagrant up
   ```

2. Get the VM connection details:

   ```bash
   cd vagrant
   vagrant ssh-config
   ```

3. Build the inventory (from WSL):

   ```bash
   cd ansible
   cp inventory.ini.example inventory.ini
   ```

   Edit `inventory.ini` and populate `ansible_host`, `ansible_port`, and `ansible_ssh_private_key_file` from `vagrant ssh-config`.

4. Run the playbook:

   ```bash
   cd ansible
   ansible-playbook -i inventory.ini playbooks/bootstrap_vagrant.yml --ask-vault-pass
   ```

   Omit `--ask-vault-pass` if you are not using Tailscale or Cloudflared.

5. (Optional) Tear down:

   ```bash
   cd vagrant
   vagrant destroy -f
   ```

## Known Issues

- **SSH key collisions after `vagrant destroy`** — Clear the known_hosts entry for the VM:

  ```bash
  ssh-keygen -f "~/.ssh/known_hosts" -R "[172.29.112.1]:2222"
  ```

  Adjust IP/port to match your Vagrant VM.

- **VirtualBox permissions / device access** — If VMs fail to start, ensure your WSL user has permission to access VirtualBox from Windows and that virtualization is enabled in your host BIOS.

## Notes

- This project targets Ubuntu guests. The control machine (WSL) only needs Ansible and network access to the guest.
- For production use, prefer running Ansible from a Linux control host (or WSL) against real servers; Vagrant is intended for local testing only.
- The Ansible playbooks and roles are identical regardless of control-machine OS. Only the environment setup (VirtualBox access) differs on Windows.