# All Business Software

Ansible automation for deploying self-hosted business services on Proxmox QEMU VMs running Ubuntu Server 26.04 and Docker Compose.

## Architecture

- Services are deployed by independent Ansible roles and Compose projects. A service can run on a dedicated VM, or several services can share one Docker VM.
- The inventory separates **Docker hosts** (`[docker_hosts]`) from **service groups** (`[caddy]`, `[authentik]`, `[monitoring]`, `[nextcloud_aio]`, `[dnsmasq]`). A host may belong to several service groups.
- The current Caddy fragment integration expects one active host in `[caddy]`; that host is resolved dynamically from the group.
- Prometheus, Grafana and Loki are one monitoring stack in one Compose project.
- Docker hosts are **Proxmox QEMU VMs running Ubuntu Server 26.04 LTS**. Docker Engine and Compose run inside each VM.
- Docker Compose runs the application containers. Each Compose project has its own automatically created default network.
- Containers in the same Compose project communicate using Compose service names.
- Services on different VMs communicate through private DNS names/IP addresses and published ports. No shared external Docker network is required.
- Caddy is the reverse proxy. Each application Ansible role owns its own Caddy site fragment.
- The `common` role is a baseline applied to every host in `docker_hosts`; it is not a VM or service.
- dnsmasq is installed natively on a dedicated Ubuntu VM and provides internal DNS plus optional DHCP. DHCP is disabled by default.
- Nextcloud AIO is deployed as its own Compose project. AIO manages its child containers and storage.

## Accounts and privilege escalation

Ansible connects to each target VM using a normal local Linux account, not root. The example inventory uses `ansible`, SSH key `~/.ssh/id_ed25519_ansible`, and sudo for privileged tasks.

Create the local account on each target VM, install the public SSH key in `~ansible/.ssh/authorized_keys`, and configure sudo. Protect the private key; for Forgejo Runner, provide it as a protected secret at runtime. Do not commit SSH keys or Vault passwords.

## Requirements on the Ansible control machine

Install Ansible on the machine or runner that executes the playbooks. From `ansible/`, install collections:

```bash
ansible-galaxy collection install -r requirements.yml
```

Target Ubuntu VMs need Python 3 at `/usr/bin/python3`, SSH access and sudo. No additional Python packages are normally required for the current roles.

## Configure inventory and secrets

Edit `ansible/inventory/inventory.yml` and replace the example IP addresses. Add the DNS/DHCP VM to `dnsmasq` and the Nextcloud VM to `nextcloud_aio` before deploying those roles. Add any VM that runs Docker to `docker_hosts`; dnsmasq itself is a native package and does not require Docker.

Store secrets in Ansible Vault, not plaintext inventory files. The configured Vault identities use `~/.vault_prod` and `~/.vault_qa` on the control machine.

## Running playbooks

Run commands from `ansible/`. First inspect the inventory and test SSH:

```bash
ansible-inventory --graph
ansible docker_hosts -m ansible.builtin.ping
```

Every playbook requires an explicit `--limit`:

```bash
ansible-playbook playbooks/common.yml --limit docker_hosts
ansible-playbook playbooks/caddy.yml --limit caddy
ansible-playbook playbooks/authentik.yml --limit authentik
ansible-playbook playbooks/monitoring.yml --limit monitoring
ansible-playbook playbooks/dnsmasq.yml --limit dnsmasq
ansible-playbook playbooks/nextcloud_aio.yml --limit nextcloud_aio
```

The combined `playbooks/all.yml` can also deploy roles by tag, but use both a tag and a matching host limit, for example:

```bash
ansible-playbook playbooks/all.yml --limit nextcloud_aio --tags nextcloud_aio
ansible-playbook playbooks/all.yml --limit dnsmasq --tags dnsmasq
```

## Before exposing services

- Use real DNS names and a valid Caddy ACME email.
- Ensure service DNS records point to the correct reverse proxy or private address.
- Restrict management ports, databases, Docker API and internal service ports with firewall rules.
- Do not expose the Nextcloud AIO administration port 8080 publicly.
- Enable dnsmasq DHCP only after verifying that no other DHCP server is active on the same broadcast domain.
- Back up persistent data and configuration before upgrades.

See [the Compose and Caddy fragment guide](docs/ansible-compose-and-caddy-fragments.md), [dnsmasq and Nextcloud AIO](docs/dnsmasq-and-nextcloud-aio.md), and [the Authentik Mittbolag setup](docs/authentik-mittbolag.md).
