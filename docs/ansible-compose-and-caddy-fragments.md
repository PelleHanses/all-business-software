# Ansible: Docker Compose and Caddy fragments

## Architecture

- Each Docker VM can run one or more independent Compose projects under `/opt/docker/compose/<project>`. Service roles can share a host; the inventory service groups determine placement.
- Prometheus, Grafana and Loki are a single monitoring stack and run together on the host in `[monitoring]`.
- Docker Compose creates one default network per project. Services in the same project use Compose service names.
- Services on different VMs communicate through inventory DNS/IP addresses and published host ports. There are no shared or externally-created Docker networks.
- Published application ports bind to each VM's `ansible_host` address by default (`service_bind_address`). Restrict access with the host/network firewall.

## Caddy configuration ownership

The Caddy role owns only the Caddy Compose project and the small root `Caddyfile`. It imports:

```caddyfile
import /etc/caddy/sites/*.caddy
```

Each application role owns one fragment file, for example:

- Authentik role: `authentik.caddy`
- Monitoring role: `grafana.caddy`

The application role writes its fragment to `caddy` inventory group (the first host in that group) using `delegate_to`, then notifies a handler that reloads Caddy. It does not rewrite or append to the central Caddyfile. Disabling the corresponding `*_caddy_enabled` setting removes only that role's fragment.

To enable the examples, set real DNS names in `inventory/group_vars/all.yml` and set `authentik_caddy_enabled: true` and/or `monitoring_caddy_enabled: true`. The default hostnames are placeholders and the fragments are disabled by default.

When adding a new application role, follow the same pattern: render one `<service>.caddy` file on `caddy` inventory group (the first host in that group), point `reverse_proxy` at the application's private VM address and published port, and notify the Caddy reload handler. Do not make application containers join the Caddy Compose network.

Caddy exposes ports 80 and 443 only. Its admin API (2019) is not published.

## Secrets

1. Copy `examples/all.vault.yml.example` to `inventory/group_vars/all.vault.yml`.
2. Replace every placeholder with a unique random secret. For example, generate values with `openssl rand -base64 36` (for the Authentik secret key, use `openssl rand -base64 60`). Keep the PostgreSQL password below 99 characters.
3. Encrypt it before committing:

   ```bash
   ansible-vault encrypt inventory/group_vars/all.vault.yml --vault-id prod@prompt
   ```

   Use the matching configured vault identity when running playbooks. Never commit the plaintext example copy.

Authentik and Grafana Compose files contain credentials and are written with mode `0600`; their template tasks suppress Ansible output and diffs.

## Install collections and deploy

From the `ansible/` directory:

```bash
ansible-galaxy collection install -r requirements.yml
ansible-playbook playbooks/all.yml -l docker_hosts
```

The `-l` limit is intentional. Use a narrower host limit for a single host, for example `-l authentik`. The example inventory uses service groups (`[caddy]`, `[authentik]`, `[monitoring]`) as well as `[docker_hosts]`. A single Docker host can belong to multiple service groups when services are consolidated. Prometheus, Grafana and Loki are deployed together on one monitoring host.

## Monitoring notes

- Prometheus scrapes itself locally inside its own container. Remote node exporters must be explicitly listed in `prometheus_node_exporter_targets`; the role does not invent `localhost:9100` targets on the Prometheus container.
- Loki listens on the monitoring VM's private address so Alloy agents on other Docker VMs can push logs.
- Alloy uses a separate Compose project on every Docker host. A Docker Socket Proxy is used to restrict the Docker API to read-oriented endpoints; the Docker socket remains a highly privileged interface and should be treated accordingly.
- The Loki schema keeps the existing BoltDB/v11 period readable and writes new data using TSDB/v13. Back up existing Loki data before changing versions or storage configuration.
- Firewall rules should permit Authentik's 9000/tcp only from Caddy, Loki's 3100/tcp from Docker hosts running Alloy, and monitoring UI/API ports only from trusted admin networks.
