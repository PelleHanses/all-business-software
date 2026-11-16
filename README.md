# All Business Software

Ansible automation for deploying self-hosted business services on separate Linux VMs with Docker Compose.

## Architecture

- Each service (for example Caddy, Authentik and monitoring) runs on its own VM unless the inventory is deliberately configured otherwise.
- The VMs run **Ubuntu Server 26.04 LTS**.
- Docker Compose runs the application containers. Each Compose project has its own automatically created default network.
- Containers in the same Compose project communicate using Compose service names.
- Services on different VMs communicate through private DNS names/IP addresses and published ports. No shared external Docker network is required.
- Caddy is the reverse proxy. Each application Ansible role owns its own Caddy site fragment; application roles do not rewrite a shared Caddyfile.
- The `common` role is **not a server or VM**. It is a baseline role applied to every host in `docker_hosts`. It installs the common host prerequisites, Docker Engine and the Compose plugin. Application-specific software belongs in its own role.

Container images are supplied by each upstream project and may use different Linux distributions internally. The host operating system being Ubuntu 26.04 does not mean every container must be based on Ubuntu.

## Accounts and privilege escalation

Ansible connects to each VM using a **normal local Linux account**, not the root account.

The example inventory sets:

- SSH user: `ansible`
- SSH key on the Ansible control machine: `~/.ssh/id_ed25519_ansible`
- Privilege escalation: `sudo` to `root` for tasks that need administrative privileges.

Create the local `ansible` account on every target VM before running the playbooks. The account needs a home directory, the public SSH key in `~ansible/.ssh/authorized_keys`, and permission to use sudo. Do not enable direct SSH login as root.

For unattended deployments (for example from a Forgejo runner), the example below gives this dedicated account passwordless sudo. This account is consequently highly privileged: protect its private SSH key, restrict network access to SSH, and do not reuse it for interactive administration. If you prefer sudo to require a password, omit the NOPASSWD sudoers rule and run Ansible with `--ask-become-pass`.

### 1. Create the local account on each Ubuntu VM

Log in using an existing administrator account and run:

```bash
sudo adduser --disabled-password --gecos "" ansible
sudo install -d -m 0700 -o ansible -g ansible /home/ansible/.ssh
```

Allow the account to use sudo without a password for automation:

```bash
echo 'ansible ALL=(root) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
sudo visudo -cf /etc/sudoers.d/ansible
```

Do this on **each** target VM. If the account already exists, check its home directory and sudo configuration rather than creating it again.

### 2. Create an SSH key on the Ansible control machine

Run this as the same local user that will execute Ansible (or the Forgejo runner service account):

```bash
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519_ansible -C "ansible-deployment"
chmod 0600 ~/.ssh/id_ed25519_ansible
```

Use a passphrase for an interactive control machine. For unattended Forgejo jobs, store the private key as a protected Forgejo secret and install it at runtime with permissions `0600`; do not commit the private key to Git.

### 3. Copy the public key to every VM

For each VM, run `ssh-copy-id` using its IP address or DNS name:

```bash
ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub ansible@192.168.1.10
ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub ansible@192.168.1.20
ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub ansible@192.168.1.30
ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub ansible@192.168.1.40
```

Replace the example addresses with the actual VM addresses. The command asks for the password of the existing administrator account used for the initial setup. It copies only the **public** key; never copy the private key to a server.

Test a connection:

```bash
ssh -i ~/.ssh/id_ed25519_ansible ansible@192.168.1.20
```

The login should open a shell as `ansible`. Verify that sudo works:

```bash
sudo -n whoami
```

It should print `root`. This does not mean Ansible logs in as root: SSH still uses the local `ansible` account, and sudo is used only for privileged tasks.

### 4. Configure the inventory

Edit `ansible/inventory/hosts.ini`:

- Replace the example IP addresses with the private IPs or resolvable DNS names of your VMs.
- Change `ansible_user` if you chose a different local account name.
- Change `ansible_ssh_private_key_file` if your key is stored elsewhere.
- Keep every Docker VM in the `docker_hosts` group.

The sample host named `docker-host-01` is just a generic Docker VM. It is not a special host for the `common` role. Rename it to match your own host if you prefer.

## Requirements on the Ansible control machine

Install Ansible on the machine or runner that will execute the playbooks. It does not have to be one of the Docker VMs.

From the repository's `ansible/` directory, install the required collections:

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

The target Ubuntu VMs need Python 3 available at `/usr/bin/python3` (the inventory sets this interpreter), SSH access, and sudo configured as described above.

## Configure secrets

Secrets such as the Authentik secret key, PostgreSQL password and Grafana administrator password must be stored in an Ansible Vault file. Do not put real secrets in ordinary YAML files or commit plaintext credentials.

Create the file:

```bash
cp examples/all.vault.yml.example inventory/group_vars/all.vault.yml
```

Replace the placeholders with generated secrets. Then encrypt it using the `prod` Vault identity:

```bash
ansible-vault encrypt inventory/group_vars/all.vault.yml --vault-id prod@prompt
```

The checked-in `ansible.cfg` is configured to look for Vault password files named `~/.vault_prod` and `~/.vault_qa` on the control machine. For normal non-interactive runs, create the appropriate password file with restrictive permissions (for example `chmod 0600 ~/.vault_prod`) and put the Vault password in it. Do not commit these files. A Forgejo runner should receive the Vault password through a protected secret and write it to a temporary file with mode `0600` before running Ansible.

## Running the playbooks

Run commands from the `ansible/` directory. The examples below assume the default inventory configured in `ansible.cfg`.

First check that Ansible can read the inventory:

```bash
ansible-inventory --graph
```

Test SSH connectivity to all Docker hosts:

```bash
ansible docker_hosts -m ansible.builtin.ping
```

Check playbook syntax before deploying:

```bash
ansible-playbook playbooks/all.yml --syntax-check
```

**Every playbook requires an explicit `--limit` / `-l` argument.** This is intentional protection against accidentally running a deployment against the entire inventory.

### Prepare all Docker VMs

```bash
ansible-playbook playbooks/common.yml --limit docker_hosts
```

This applies the `common` baseline and Alloy agent to all hosts in `docker_hosts`.

### Deploy all services

```bash
ansible-playbook playbooks/all.yml --limit docker_hosts
```

This prepares all Docker hosts and deploys Caddy, Authentik and monitoring on the hosts assigned to those groups.

### Deploy or update one service

```bash
ansible-playbook playbooks/caddy.yml --limit caddy-server
ansible-playbook playbooks/authentik.yml --limit authentik-server
ansible-playbook playbooks/monitoring.yml --limit monitoring-server
```

To run the common baseline on one VM only:

```bash
ansible-playbook playbooks/common.yml --limit caddy-server
```

If sudo requires a password, add `--ask-become-pass` to the command. With the recommended NOPASSWD configuration, this is not necessary.

## Before exposing services

- Set real DNS names and the Caddy ACME email in `inventory/group_vars/all.yml`.
- Enable `authentik_caddy_enabled` and/or `monitoring_caddy_enabled` only when their DNS names resolve to the Caddy VM.
- The default `service_bind_address` binds published service ports to each VM's `ansible_host` address. Ensure that this is the private interface address and restrict access with firewall rules.
- Permit only the required traffic between VMs and trusted admin networks. Do not expose database, Redis, Docker API or Caddy's admin API ports to the public Internet.
- Back up persistent Compose volumes and configuration before upgrades.

See [the Compose and Caddy fragment guide](docs/ansible-compose-and-caddy-fragments.md) for details about service networking, Caddy fragment ownership and monitoring.
