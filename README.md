# All Business Software

Ansible automation for deploying self-hosted business services on Proxmox QEMU VMs running Ubuntu Server 26.04 and Docker Compose.

## Architecture

- Services are deployed by independent Ansible roles and Compose projects. A service can run on a dedicated VM, or several services can share one Docker VM.
- The inventory separates **Docker hosts** (`[docker_hosts]`) from **service groups** (`[caddy]`, `[authentik]`, `[monitoring]`). A host may belong to several service groups, which is how services are consolidated onto one Docker host.
- The current Caddy fragment integration expects one active host in `[caddy]`; that host is resolved dynamically from the group rather than by a fixed inventory hostname.
- Prometheus, Grafana and Loki are one monitoring stack in one Compose project and always run together on the same host.
- The Docker hosts are **Proxmox QEMU VMs running Ubuntu Server 26.04 LTS**. Docker Engine and Compose run inside each VM.
- Docker Engine is rootful (not rootless). The SSH account configured by `ansible_user` belongs to the `docker` group and runs Docker Compose directly, without `sudo` for container operations. Membership in `docker` is effectively root-equivalent and must be treated as privileged access.\n- Docker is enabled as a system service and starts at host boot. Containers use `restart: unless-stopped` by default so Docker brings them back after daemon/host restarts.\n- All Compose projects, configuration and bind-mounted persistent data live under `/opt/<application>/` (for example `/opt/authentik`, `/opt/caddy`, `/opt/monitoring`). Do not use Docker-managed named volumes for application data.\n- Compose bind mounts use explicit paths below each application's directory. Application directories and deployment files are owned by `ansible_user`; individual data directories may require container-specific UID/GID or ACL permissions for the process inside the container.\n- Docker Compose runs the application containers. Each Compose project has its own automatically created default network.
- Containers in the same Compose project communicate using Compose service names.
- Services on different VMs communicate through private DNS names/IP addresses and published ports. No shared external Docker network is required.
- Caddy is the reverse proxy. Each application Ansible role owns its own Caddy site fragment; application roles do not rewrite a shared Caddyfile.
- The `common` role is **not a server or VM**. It is a baseline role applied to every host in `docker_hosts`. It installs the common host prerequisites, Docker Engine and the Compose plugin. Application-specific software belongs in its own role.

Each Proxmox VM provides a normal Ubuntu 26.04 environment for Docker. Run Docker Engine and the Compose plugin directly inside the VM; do not nest Docker inside a Proxmox LXC container for this design. Application containers use the official images supplied by their upstream projects and may use different base distributions.

### Proxmox VM requirements

Create a QEMU VM for each Docker host, install Ubuntu Server 26.04 LTS, enable the OpenSSH server, and ensure `/usr/bin/python3` is available. Use a normal VM disk on reliable storage and configure backups at the Proxmox level. Install Docker Engine and the Compose plugin through the `common` Ansible role.

## Accounts, Docker access and privilege escalation

Ansible connects to each Docker host (Ubuntu VM) using a **normal local Linux account**, not the root account.

The example inventory sets:

- SSH user: `ansible`
- SSH key on the Ansible control machine: `~/.ssh/id_ed25519_ansible`
- Docker access: membership in the `docker` group (effectively root-equivalent).\n- `sudo` is used only by Ansible tasks that configure the host OS, install packages, manage system services and create protected host directories; Compose deployments run as `ansible_user`.

Create the local `ansible` account in every target Ubuntu VM before running the playbooks. The account needs a home directory, the public SSH key in `~ansible/.ssh/authorized_keys`, passwordless sudo for initial host provisioning, and membership in the `docker` group. The `common` role adds it to that group automatically. Do not enable direct SSH login as root.

The example below gives this dedicated account passwordless sudo because the baseline must install packages and configure host services without interaction. The `docker` group is also effectively root-equivalent. Protect its private SSH key and restrict network access to SSH. Normal container deployment tasks do not use sudo; they run as the SSH account. The account has an interactive Bash shell for straightforward SSH administration.

### 1. Create the local account in each Ubuntu VM

Log in using an existing administrator account and run:

```bash
sudo adduser --disabled-password --gecos "" --shell /bin/bash ansible
sudo install -d -m 0700 -o ansible -g ansible /home/ansible/.ssh
```

Allow the account to use sudo without a password for automation:

```bash
echo 'ansible ALL=(root) NOPASSWD:ALL' | sudo tee /etc/sudoers.d/ansible
sudo chmod 0440 /etc/sudoers.d/ansible
sudo visudo -cf /etc/sudoers.d/ansible
```

Do this in **each** target Ubuntu VM. The `--shell /bin/bash` option ensures that SSH logins open a normal interactive Bash shell. If the account already exists, check its home directory and sudo configuration, and set its shell explicitly with `sudo usermod --shell /bin/bash ansible`.

The `common` Ansible role also enforces `/bin/bash` for the inventory SSH account on every Docker host, so an existing account is corrected automatically when the baseline runs.

### 2. Create an SSH key on the Ansible control machine

Run this as the same local user that will execute Ansible (or the Forgejo runner service account):

```bash
ssh-keygen -t ed25519 -a 64 -f ~/.ssh/id_ed25519_ansible -C "ansible-deployment"
chmod 0600 ~/.ssh/id_ed25519_ansible
```

Use a passphrase for an interactive control machine. For unattended Forgejo jobs, store the private key as a protected Forgejo secret and install it at runtime with permissions `0600`; do not commit the private key to Git.

### 3. Install the public key in every Ubuntu VM

Because the `ansible` account was created with password login disabled, you cannot use `ssh-copy-id ansible@host` for the initial key installation. Use an existing administrator account that already has SSH access and sudo rights.

One way is to display the public key on the control machine:

```bash
cat ~/.ssh/id_ed25519_ansible.pub
```

In each Ubuntu VM, logged in as the existing administrator, add that single public-key line to the Ansible account:

```bash
sudo install -d -m 0700 -o ansible -g ansible /home/ansible/.ssh
sudoedit /home/ansible/.ssh/authorized_keys
sudo chown ansible:ansible /home/ansible/.ssh/authorized_keys
sudo chmod 0600 /home/ansible/.ssh/authorized_keys
```

Paste the public key line into the editor and save. Repeat in every Ubuntu VM. Only the `.pub` public key is installed on servers; never copy the private key there.

If you prefer `ssh-copy-id`, you can temporarily enable a password for the `ansible` account, run `ssh-copy-id -i ~/.ssh/id_ed25519_ansible.pub ansible@HOST`, and then lock its password again with `sudo passwd -l ansible`. The manual method above avoids enabling password login for the automation account.

Test a connection:

```bash
ssh -i ~/.ssh/id_ed25519_ansible ansible@192.168.1.20
```

The login should open a shell as `ansible`. Verify that sudo works:

```bash
sudo -n whoami
```

It should print `root`. Then verify Docker access without sudo:\n\n```bash\ndocker info\n```\n\nThe `common` role adds the account to the Docker group and resets the SSH connection so the new group membership applies. SSH still logs in as the local `ansible` account.

### 4. Configure the inventory

Edit `ansible/inventory/inventory.yml`:

- Replace the example IP addresses with the private IPs or resolvable DNS names of your Ubuntu VMs.
- Change `ansible_user` if you chose a different local account name.
- Change `ansible_ssh_private_key_file` if your key is stored elsewhere.
- Keep every Docker VM in the `docker_hosts` group.

The example has three separate VMs, but this is not a requirement. To run all services on one Docker VM, define it once under `docker_hosts.hosts` and reference that same hostname in each service group's `hosts` mapping:

```yaml
docker_hosts:
  hosts:
    apps-01:
      ansible_host: 192.168.1.50

caddy:
  hosts:
    apps-01:

authentik:
  hosts:
    apps-01:

monitoring:
  hosts:
    apps-01:
```

To place only Caddy and Authentik together, list the same host in `[caddy]` and `[authentik]` while keeping monitoring on another host. Prometheus, Grafana and Loki are intentionally deployed together by the single `monitoring` role and Compose project.

The `common` role is not a VM: it is applied to every host in `[docker_hosts]`. Add each physical/virtual Docker host there, regardless of which services it runs.

## Requirements on the Ansible control machine

Install Ansible on the machine or runner that will execute the playbooks. It does not have to be one of the Docker VMs.

From the repository's `ansible/` directory, install the required collections:

```bash
cd ansible
ansible-galaxy collection install -r requirements.yml
```

The target Ubuntu VMs need Python 3 available at `/usr/bin/python3` (the inventory sets this interpreter), SSH access, passwordless sudo for host provisioning, and Docker group access configured by the `common` role.

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

This prepares all Docker hosts and deploys every service whose group matches those hosts.

### Deploy or update one service only

Use the dedicated service playbook. It targets only that service group, even if the same inventory host also belongs to other service groups:

```bash
ansible-playbook playbooks/caddy.yml --limit caddy
ansible-playbook playbooks/authentik.yml --limit authentik
ansible-playbook playbooks/monitoring.yml --limit monitoring
```

The combined `all.yml` playbook also has service tags. If you prefer to use it for a single service, specify **both** the service tag and the matching limit:

```bash
ansible-playbook playbooks/all.yml --limit authentik --tags authentik
ansible-playbook playbooks/all.yml --limit caddy --tags caddy
ansible-playbook playbooks/all.yml --limit monitoring --tags monitoring
```

Do not use `playbooks/all.yml --limit authentik` by itself when hosts are shared: `--limit` selects hosts, not service plays. The `--tags` option is what restricts the combined playbook to the requested service. The dedicated service playbooks above are the recommended way to deploy one service; they also run the common baseline on the selected host.

To run only the common baseline on one Docker host:

```bash
ansible-playbook playbooks/common.yml --limit apps-01
```

The Compose deployment runs as `ansible_user`; privileged host preparation tasks use sudo through Ansible's `become` mechanism.

## Before exposing services

- Set real DNS names and the Caddy ACME email in `inventory/group_vars/all.yml`.
- Enable `authentik_caddy_enabled` and/or `monitoring_caddy_enabled` only when their DNS names resolve to the Caddy VM.
- The default `service_bind_address` binds published service ports to each VM's `ansible_host` address. Ensure that this is the private interface address and restrict access with firewall rules.
- Permit only the required traffic between VMs and trusted admin networks. Do not expose database, Redis, Docker API or Caddy's admin API ports to the public Internet.
- Back up `/opt/<application>/` directories. Database backups must be application-consistent (for example, use PostgreSQL logical/physical backup procedures rather than copying a live data directory blindly).

See [the Compose and Caddy fragment guide](docs/ansible-compose-and-caddy-fragments.md) for details about service networking, Caddy fragment ownership and monitoring.
See [the Authentik Mittbolag setup](docs/authentik-mittbolag.md) for the initial users and groups, domain/DNS requirements, and per-application OIDC configuration.

## Additional service roles

The repository also includes:

- `dnsmasq`: native Ubuntu service for internal DNS and optional DHCP. DHCP is disabled by default; configure and verify the correct network interface and DHCP range before enabling it.
- `nextcloud_aio`: official Nextcloud All-in-One mastercontainer in a separate Docker Compose project, with an optional Caddy reverse-proxy fragment.
- Caddy can optionally run a generic PHP-FPM container in its own Compose project. This is intended for other PHP applications, not Nextcloud AIO.

Add the dedicated DNS/DHCP VM to the `dnsmasq` inventory group and the Nextcloud VM to `nextcloud_aio`. The example inventory leaves these groups empty until you assign real hosts.

Deploy the new services from `ansible/`:

```bash
ansible-playbook playbooks/dnsmasq.yml --limit dnsmasq
ansible-playbook playbooks/nextcloud_aio.yml --limit nextcloud_aio
```

See [dnsmasq and Nextcloud AIO](docs/dnsmasq-and-nextcloud-aio.md) for configuration examples, DHCP safety notes, Caddy/PHP-FPM usage and Nextcloud AIO setup.
