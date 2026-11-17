# dnsmasq and Nextcloud AIO

## dnsmasq: internal DNS and DHCP

The `dnsmasq` role installs dnsmasq directly on an Ubuntu VM (not in Docker). It serves configured internal DNS records and forwards other DNS queries to the configured upstream resolvers.

DHCP is deliberately disabled by default. Set `dnsmasq_dhcp_enabled: true` only after confirming that this VM is intended to be the active DHCP server for the selected network. Running two DHCP servers on one broadcast domain can hand clients conflicting network settings.

Example inventory variables for a dedicated DNS/DHCP VM:

```yaml
dnsmasq_domain: "kullgarden.se"
dnsmasq_listen_addresses:
  - "192.168.1.10"
dnsmasq_upstream_servers:
  - "1.1.1.1"
  - "9.9.9.9"
dnsmasq_dns_records:
  - name: "proxmox.kullgarden.se"
    ip: "192.168.1.2"
  - name: "files.kullgarden.se"
    ip: "192.168.1.15"
dnsmasq_cname_records:
  - alias: "nextcloud.kullgarden.se"
    target: "files.kullgarden.se"

# Enable only when dnsmasq is meant to provide DHCP on this interface/subnet.
dnsmasq_dhcp_enabled: true
dnsmasq_dhcp_interface: "ens18"
dnsmasq_dhcp_range_start: "192.168.1.100"
dnsmasq_dhcp_range_end: "192.168.1.199"
dnsmasq_dhcp_netmask: "255.255.255.0"
dnsmasq_dhcp_router: "192.168.1.1"
dnsmasq_dhcp_dns_servers:
  - "192.168.1.10"
dnsmasq_dhcp_lease_time: "12h"
dnsmasq_dhcp_reservations:
  - mac: "52:54:00:12:34:56"
    ip: "192.168.1.20"
    name: "caddy-01"
```

Use an internal DNS domain you control. If using a subdomain of a public domain (for example `lan.kullgarden.se`), make sure the public and private DNS views are intentionally designed. Do not configure a private DNS server as authoritative for a public domain unless it contains all records clients need.

The DNS role listens only on the addresses listed in `dnsmasq_listen_addresses`. Ensure the VM has a stable address outside the dynamic DHCP pool. Configure firewall rules to permit UDP/TCP 53 from trusted LAN/VPN networks and UDP 67 from the intended DHCP client network. DHCP broadcasts generally do not cross routers; configure a DHCP relay if clients are on routed VLANs.

The role does not change the Ubuntu host's systemd-resolved or `/etc/resolv.conf` configuration. This avoids disrupting the DNS configuration of the server itself.

Deploy with:

```bash
ansible-playbook playbooks/dnsmasq.yml --limit dnsmasq
```

## Nextcloud All-in-One

The `nextcloud_aio` role deploys the official Nextcloud AIO mastercontainer using a dedicated Compose project. AIO manages its own child containers and storage. The Docker socket is mounted read-only as required by AIO; this is still a highly privileged integration and the Docker host must be trusted.

The role defaults to an external reverse proxy:

- AIO administration UI: `127.0.0.1:8080` on the Nextcloud VM. Use an SSH tunnel to reach it, for example `ssh -L 8080:127.0.0.1:8080 ansible@NEXTCLOUD_VM`.
- Nextcloud Apache endpoint: TCP `11000`, bound to the Nextcloud VM's inventory address.
- Caddy: when `nextcloud_aio_caddy_enabled: true`, the role writes its own `nextcloud.caddy` fragment to the Caddy host. Caddy connects to the Nextcloud VM on the configured Apache endpoint.

Set a real domain and ensure public DNS, firewall/NAT and Caddy are configured before completing AIO's domain validation. For a private-only installation, configure private DNS and an appropriate internal TLS approach; domain validation may need to be skipped explicitly. Do not expose the AIO administration port 8080 to the Internet.

Example variables:

```yaml
nextcloud_aio_domain: "cloud.kullgarden.se"
nextcloud_aio_apache_bind_address: "192.168.1.50"
nextcloud_aio_admin_bind_address: "127.0.0.1"
nextcloud_aio_caddy_enabled: true
nextcloud_aio_caddy_hostname: "cloud.kullgarden.se"
nextcloud_aio_caddy_upstream: "192.168.1.50:11000"
```

Deploy with:

```bash
ansible-playbook playbooks/nextcloud_aio.yml --limit nextcloud_aio
```

Open the AIO setup UI through the SSH tunnel, enter the configured Nextcloud domain, and follow the AIO setup flow. AIO updates its managed child containers itself; update the mastercontainer image through the Compose role and follow the upstream AIO upgrade guidance. Back up the AIO data and configuration before upgrades.

Official upstream guide: https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md

## Caddy with PHP-FPM

The existing Caddy role can optionally run a PHP-FPM container in the same Compose project. It shares the directory configured by `caddy_php_root_dir` with Caddy at `/srv`. The default PHP-FPM image is `php:8.4-fpm-alpine`; add PHP extensions or use a custom image when an application requires them.

A PHP site fragment can look like this:

```caddyfile
example.kullgarden.se {
    root * /srv/example
    encode zstd gzip
    php_fastcgi php-fpm:9000
    file_server
}
```

Place the site's files under `{{ caddy_php_root_dir }}/example`. The PHP-FPM service is not published on a host port; it is reachable only over the private Compose network shared with Caddy. Do not use this generic PHP-FPM instance for Nextcloud AIO, which includes its own web/PHP stack.

Set `caddy_php_fpm_enabled: false` to omit PHP-FPM when it is not needed.
