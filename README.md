# Ansible Keepalived role

Installs and configures Keepalived on Ubuntu 26.04, 24.04, and 22.04 LTS;
Debian 13 and 12; and Rocky Linux 10, 9, and 8. All non-secret configuration
comes from normal inventory variables. Group variables hold shared settings
and host variables supply node-specific state, priority, interfaces, and
addresses. Secrets must be supplied as extra vars. Ansible Core 2.20 or newer
is required so the current target operating systems' Python versions are
supported.

## Host selection

Run the role against `all` and set `keepalived_state` in inventory:

- `present` installs, configures, enables, and starts Keepalived.
- `absent` stops and uninstalls Keepalived and removes its managed config file.
- `unmanaged` makes no changes (the safe default).

This three-state marker is intentional. A boolean cannot distinguish "remove
Keepalived" from "do not manage this host." Use an existing functional child
group, such as `load_balancers`, rather than adding a group solely because the
hosts run Keepalived.

```yaml
# site.yml
---
- name: Manage Keepalived where requested by inventory
  hosts: all
  become: true
  roles:
    - role: keepalived
```

## Inventory configuration

Put configuration shared by the cluster in the variables for an existing
application or environment child group. The group can opt all its members in,
while host variables provide only the values that differ per node.

```yaml
# group_vars/load_balancers/keepalived.yml
---
keepalived_state: present

keepalived_global_defs:
  router_id: "{{ inventory_hostname }}"
  enable_script_security: true

keepalived_vrrp_scripts:
  - name: check_web
    script: /usr/bin/curl -fsS http://127.0.0.1/health
    interval: 2
    timeout: 1
    fall: 2
    rise: 2
    weight: -20

keepalived_vrrp_instances:
  - name: VI_WEB
    state: "{{ keepalived_node_state }}"
    interface: "{{ keepalived_node_interface }}"
    virtual_router_id: 51
    priority: "{{ keepalived_node_priority }}"
    advert_int: 1
    auth_type: PASS
    unicast_src_ip: "{{ keepalived_node_address }}"
    unicast_peers: "{{ keepalived_peer_addresses }}"
    virtual_ipaddresses:
      - "192.0.2.100/24 dev {{ keepalived_node_interface }}"
    track_scripts:
      - check_web
```

```yaml
# host_vars/lb01.yml
---
keepalived_node_state: MASTER
keepalived_node_interface: ens18
keepalived_node_priority: 150
keepalived_node_address: 192.0.2.11
keepalived_peer_addresses:
  - 192.0.2.12
```

```yaml
# host_vars/lb02.yml
---
keepalived_node_state: BACKUP
keepalived_node_interface: ens18
keepalived_node_priority: 100
keepalived_node_address: 192.0.2.12
keepalived_peer_addresses:
  - 192.0.2.11
```

For a host where Keepalived must be removed, set only:

```yaml
keepalived_state: absent
```

## Secrets as extra vars

When an instance has `auth_type`, the role requires its password in
`keepalived_vrrp_auth_passes`, keyed by VRRP instance name. Pass that mapping
as extra vars and do not define it in inventory:

```bash
ansible-playbook -i inventory site.yml \
  --extra-vars '{"keepalived_vrrp_auth_passes":{"VI_WEB":"change-me"}}'
```

For automation, inject the same extra-var mapping from the CI/CD secret store
instead of putting the value on a command line that may be saved in shell
history. The role rejects the legacy `auth_pass` instance key to prevent an
inventory secret from being used accidentally. Keepalived's `PASS`
authentication uses only the first eight characters and is not strong
authentication.

## Supported configuration keys

`keepalived_global_defs` accepts direct Keepalived global directives as a
mapping. Boolean `true` emits a standalone directive; lists repeat a directive.

Each `keepalived_vrrp_scripts` item supports `name`, `script`, `interval`,
`timeout`, `weight`, `rise`, `fall`, and `user`.

Each `keepalived_vrrp_instances` item requires `name`, `interface`,
`virtual_router_id`, `priority`, and `virtual_ipaddresses`. It also supports
`state`, `advert_int`, `nopreempt`, `preempt_delay`, `auth_type`,
`unicast_src_ip`, `unicast_peers`, `track_interfaces`, `track_scripts`, and the
`notify_master`, `notify_backup`, and `notify_fault` commands.
