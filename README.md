# Ansible Example

This project is the result of my efforts in automating a linux machine.

> Don't expect this to be fully functional or 100% correct and production worthy. Suggestions are welcomed!

I included a vagrant example with ubuntu so you can test it locally before you think about using it on real machines.

This setup will do some common tasks for you:

- Upgrade the system
- Install base packages
- Enable automatic security upgrades
- Enable Firewall: 
    - Close everything
    - Open SSH, HTTP and HTTPS ports
- Enables fail2ban
- Installs docker
- Reboots machine if needed

And some optional roles can be tested too:
- Install and setup tailscale
- Install cloudflared and setup tunnel

### Requirements

- Virtualbox or other virtualization solution of your preference.
- Vagrant
- Ansible

#### Windows

If you're using Windows you need to enable WSL first and install Virtualbox on your host (not on WSL).

Then, while on WSL, you need to enable access from vagrant to virtualbox.

This will enable permanently:
```
echo "export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS=1 >> ~/.bash_profile
source ~/.bash_profile
```
If you prefer to do it only for the current session just do:

```
export VAGRANT_WSL_ENABLE_WINDOWS_ACCESS="1"
```

# Ansible

## Set up

### Inventory
To check the data needed for ansible to connect, run the following from your WSL terminal:
```
cd vagrant
vagrant ssh-config
```

Then rename `ansible/inventory.ini.example` to `inventory.ini` and change accordingly to use host, port and the ssh key from vagrant ssh-config.

It should look similar to this:
```
[vagrant]
default ansible_host={vagrant_ip} ansible_port={vagrant_port} ansible_user=vagrant ansible_ssh_private_key_file={IdentityFile path}
```

### Optional roles
You can enable tailscale and cloudflare setup and configuration. Just uncomment the roles from `ansible/playbooks/bootstrap_vagrant.yml` if you wish to test them locally or just use the `bootstrap_a_real_server.yml` when you're ready to provision a real machine.

### Using this in a real VPS/Server

You will have to add an inventory group the machine connection entry.
Add to `ansible/inventory.ini` like so:

```
[a_real_server]
default ansible_host={IP of your real machine} ansible_port=2222 ansible_user={the username} ansible_ssh_private_key_file={path_to_private_key}
```

The **a_real_server** group matches the `bootstrap_a_real_server` playbook. You can rename as you please, as long as ansible knows how to connect to the machine.

You can also change the **hostname** from default, or add others. Just bare in mind that you need to name the vault file accordingly unders **ansible/host_vars** 

### Vault secrets
This example includes both tailscale and cloudflared tunnel setup.
If you wish to use either, you will have to setup two keys inside a vault

```
cd ansible
ansible-vault create host_vars/{hostname}.yml
```
> For single host setup, you can use `default` hostname.

You will be asked to provide a password for encrypting/decrypting. Note it down.
On the editor, create the entries:

```
tailscale_authkey: [your tailscape auth token]
cloudflared_token: [your Cloudflare ZeroTrust Connection token]
```

If you need to change these values later, you can run:
`ansible-vault edit host_vars/default.yml`


## Run the playbook!
Finally, you're ready to test it out.

If you enabled `tailscale` or `cloudflare` roles, run:

```
cd ansible
ansible-playbook -i inventory.ini playbooks/bootstrap.yml --ask-vault-pass
```
> You can use `--vault-password-file=/path/to/vault_password.txt` and put your vault password in a file somewhere in your home folder so you don't have to type it everytime.

If no vault secrets are involved

```
cd ansible
ansible-playbook -i inventory.ini playbooks/bootstrap.yml
```

Then just sit back and wait. The process may take a while to finish.

You should see something like the following at the end of the run:

```
PLAY RECAP *****************************************************************************************************
default                    : ok=21   changed=13   unreachable=0    failed=0    skipped=0    rescued=0    ignored=0
```

And that's it. Your machine should be upgraded and "hardened" with some base packages installed + docker. If you added tailscale and/or cloudflare correcty they should now appear on their respective dashboards.

Run the playbook as many times as you wish. That's the whole point - to keep your system up to date.

# Troubleshooting:

### Vagrant

#### SSH issues
If you destroy the vagrant machine and run again you will most likely run into an issue with ssh key. You should clear your `known_hosts` file first:

> replace IP and port according to your configuration

`ssh-keygen -f '~/.ssh/known_hosts' -R '[172.29.112.1]:2222'`

# Cleaning up

When you're done experimenting with vagrant you can destroy your machine:

```
cd vagrant
vagrant destroy -f
```

If you used tailscale and/or cloudflared cleanup via the service dashboard.

