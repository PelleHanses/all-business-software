> **Historiskt designunderlag – inte en beskrivning av den aktuella implementationen.** Katalogstruktur, inventarie, playbooks och roller i denna fil har avvikit från repots nuvarande kod. Följ i stället [README.md](../README.md) och [Compose/Caddy-guiden](ansible-compose-and-caddy-fragments.md) för aktuella instruktioner. Den här filen sparas endast som tidigare arkitekturanteckningar.

# Ansible-struktur för serverhallen

Ett repo, en roll per tjänst, komponerbara playbooks. Allt följer Ansible-standard: rollens egna `templates/`, `defaults/`, `handlers/`, `tasks/`. Caddy-sites genereras av varje tjänsteroll som fragment som Caddy-rollen samlar in.

## Katalogträd

```text
ansible/
├── ansible.cfg
├── inventory/
│   ├── hosts.yml
│   └── group_vars/
│       ├── all.yml
│       ├── all.vault.yml
│       ├── docker_hosts.yml
│       └── observability.yml
├── playbooks/
│   ├── site.yml
│   ├── observability.yml
│   ├── mail.yml
│   ├── dev.yml
│   └── service.yml
└── roles/
    ├── common/
    ├── caddy/
    ├── authentik/
    ├── vaultwarden/
    ├── postgresql/
    ├── prometheus/
    ├── grafana/
    ├── loki/
    ├── alloy/
    └── ... (en per tjänst)
```

## ansible.cfg

```ini
[defaults]
inventory = inventory/hosts.yml
roles_path = roles
stdout_callback = yaml
interpreter_python = auto_silent
vault_identity_list = prod@~/.vault_prod, qa@~/.vault_qa

[privilege_escalation]
become = True
```

Vault-lösenorden hämtas vid behov från Vaultwarden av den som kör (kopiera till `~/.vault_prod` eller använd `--vault-id prod@prompt`).

## inventory/hosts.yml

```yaml
all:
  children:
    docker_hosts:
      hosts:
        docker1.example.se:
        docker2.example.se:
    opnsense:
      hosts:
        fw.example.se:
    pbs:
      hosts:
        pbs.example.se:
    observability:
      hosts:
        docker1.example.se:
    mail:
      hosts:
        docker1.example.se:
    dev:
      hosts:
        docker2.example.se:
```

## inventory/group_vars/all.yml

```yaml
# Grund
domain: example.se
admin_mail: drift@example.se
timezone: Europe/Stockholm

# Docker
docker_compose_dir: /opt/compose

# Caddy: varje tjänsteroll lägger fragment här
caddy_sites_dir: /etc/caddy/sites
caddy_acme_dns_provider: cloudflare   # för DNS-01 mot intern-zon

# Vaultwarden-lookup av vault-lösenord sker manuellt; hemligheterna
# själva ligger i group_vars/<grupp>.vault.yml (ansible-vault).
```

## inventory/group_vars/all.vault.yml

Krypterad med `ansible-vault create`, per miljö via vault-id:

```bash
ansible-vault create inventory/group_vars/all.vault.yml --vault-id prod@prompt
```

```yaml
vault_postgres_password: "..."
vault_authentik_secret: "..."
vault_stalwart_admin: "..."
```

## playbooks/site.yml

```yaml
# site.yml - allt, kör alltid med -l (t.ex. -l prod eller -l docker1)
- name: Serverhallen - grund
  hosts: all
  pre_tasks:
    - name: Logga vem som startade playbooken
      debug:
        msg: "Playbook triggered by {{ lookup('env','USER') }}"

    - name: Verifiera att -l-argumentet är satt
      fail:
        msg: "❌ Denna playbook får inte köras utan -l-argument (t.ex. -l docker1 eller -l prod)"
      run_once: true
      when: ansible_limit is not defined

  roles:
    - common
    - alloy
```

## playbooks/service.yml

```yaml
# service.yml - kör en enskild tjänst: -e service=nextcloud -l docker1
- name: Serverhallen - enskild tjänst
  hosts: all
  pre_tasks:
    - name: Verifiera att -l-argumentet är satt
      fail:
        msg: "❌ Denna playbook får inte köras utan -l-argument"
      run_once: true
      when: ansible_limit is not defined

    - name: Verifiera att service är satt
      fail:
        msg: "❌ Kör med -e service=<rollnamn>"
      when: service is not defined

  roles:
    - role: "{{ service }}"
```

## playbooks/observability.yml

```yaml
# observability.yml - hela driftstacken
- name: Serverhallen - driftstack
  hosts: observability
  pre_tasks:
    - name: Logga vem som startade playbooken
      debug:
        msg: "Playbook triggered by {{ lookup('env','USER') }}"

    - name: Verifiera att -l-argumentet är satt
      fail:
        msg: "❌ Denna playbook får inte köras utan -l-argument"
      run_once: true
      when: ansible_limit is not defined

  roles:
    - prometheus
    - loki
    - grafana
```

## Roll: common

### roles/common/defaults/main.yml

```yaml
docker_packages:
  - docker-ce
  - docker-ce-cli
  - containerd.io
  - docker-compose-plugin
```

### roles/common/tasks/main.yml

```yaml
- name: Baspaket
  ansible.builtin.package:
    name:
      - vim
      - curl
      - rsync
    state: present

- name: Docker-repo (Debian/Ubuntu)
  ansible.builtin.get_url:
    url: https://download.docker.com/linux/ubuntu/gpg
    dest: /etc/apt/keyrings/docker.asc
    mode: "0644"
  when: ansible_os_family == "Debian"

- name: Installera Docker
  ansible.builtin.package:
    name: "{{ docker_packages }}"
    state: present
  when: ansible_os_family == "Debian"

- name: Aktivera Docker
  ansible.builtin.systemd:
    name: docker
    enabled: true
    state: started
```

## Roll: caddy (mönster för fragmentinsamling)

### roles/caddy/defaults/main.yml

```yaml
caddy_image: caddy:2
caddy_sites_dir: /etc/caddy/sites
```

### roles/caddy/tasks/main.yml

```yaml
- name: Skapa katalog för site-fragment
  ansible.builtin.file:
    path: "{{ caddy_sites_dir }}"
    state: directory
    mode: "0750"

- name: Compose-fil
  ansible.builtin.template:
    src: compose.yml.j2
    dest: "{{ docker_compose_dir }}/caddy/compose.yml"
    mode: "0600"
  notify: Uppdatera caddy

- name: Caddyfile (inkluderar alla fragment)
  ansible.builtin.template:
    src: Caddyfile.j2
    dest: "{{ docker_compose_dir }}/caddy/Caddyfile"
  notify: Uppdatera caddy
```

### roles/caddy/handlers/main.yml

```yaml
- name: Uppdatera caddy
  ansible.builtin.command:
    cmd: docker compose up -d --force-recreate
    chdir: "{{ docker_compose_dir }}/caddy"
  changed_when: true
```

### roles/caddy/templates/Caddyfile.j2

```jinja
{
    email {{ admin_mail }}
}

import {{ caddy_sites_dir }}/*.caddy
```

## Roll: vaultwarden (exempel på tjänsteroll)

Varje tjänsteroll följer samma mönster: compose-template, env med hemligheter från vault, Caddy-fragment.

### roles/vaultwarden/defaults/main.yml

```yaml
vaultwarden_image: vaultwarden/server:latest
vaultwarden_port: "8222"
```

### roles/vaultwarden/tasks/main.yml

```yaml
- name: Katalogstruktur
  ansible.builtin.file:
    path: "{{ docker_compose_dir }}/vaultwarden/data"
    state: directory
    mode: "0700"

- name: Compose-fil
  ansible.builtin.template:
    src: compose.yml.j2
    dest: "{{ docker_compose_dir }}/vaultwarden/compose.yml"
    mode: "0600"
  notify: Starta om vaultwarden

- name: Caddy-site-fragment
  ansible.builtin.template:
    src: caddy-site.j2
    dest: "{{ caddy_sites_dir }}/vaultwarden.caddy"
  notify: Uppdatera caddy
```

### roles/vaultwarden/handlers/main.yml

```yaml
- name: Starta om vaultwarden
  ansible.builtin.command:
    cmd: docker compose up -d
    chdir: "{{ docker_compose_dir }}/vaultwarden"
  changed_when: true

- name: Uppdatera caddy
  ansible.builtin.command:
    cmd: docker compose exec caddy caddy reload --config /etc/caddy/Caddyfile
    chdir: "{{ docker_compose_dir }}/caddy"
  changed_when: true
```

### roles/vaultwarden/templates/compose.yml.j2

```jinja
services:
  vaultwarden:
    image: {{ vaultwarden_image }}
    restart: unless-stopped
    environment:
      DOMAIN: https://vault.{{ domain }}
      ADMIN_TOKEN: {{ vault_vaultwarden_admin }}
    volumes:
      - ./data:/data
    ports:
      - "127.0.0.1:{{ vaultwarden_port }}:80"
```

### roles/vaultwarden/templates/caddy-site.j2

```jinja
vault.{{ domain }} {
    reverse_proxy 127.0.0.1:{{ vaultwarden_port }}
}
```

## Arbetsgång

1. `common` + `caddy` + `vaultwarden` först – verifiera fragment-/vault-mönstret.
2. `authentik`, sedan resterande roller enligt installationsordningen i planen.
3. Renovate pekas mot repot för image-bumpar i compose-filerna.

## Konventioner

- Hemligheter refereras alltid som `vault_<namn>` och ligger endast i `.vault.yml`-filer.
- Varje tjänst lyssnar bara på `127.0.0.1:<port>`; Caddy är enda ingången.
- Hanterare heter på svenska: `Uppdatera caddy`, `Starta om <tjänst>`.
- Playbooks körs aldrig utan `-l`.