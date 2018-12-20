WireGuard
=========

A role for configuring WireGuard VPN.

Requirements
------------

The role should be self-contained, just provide vars for your hosts and run it.

Role Variables
--------------

### `wireguard_manage_keys`

If `True` ansible automatically generated public and private pair keys. Default `False`.

### `wireguard_interfaces`

Each host needs to have `wireguard_interfaces` variable set. It should be a list of WireGuard interface name the host should use, by default it is an empty list. For each `$INTERFACE` specified here the host should have:
```
wireguard_interface:
  $INTERFACE:
     key: value
```
and
```
wireguard_peers:
  $INTERFACE:
    key: value
```
vars set.

### `wireguard_interface: $INTERFACE`

This variable allows configuring the WireGuard interface on the host. It is a dict and the following keys are taken into account:

| Key | Description | Required |
| --- | ----------- | -------- |
| address | The address to be configured on the interface in CIDR format | Yes |
| private_key | The private key to use for this interface | Yes |
| listen_port | A port to listen to, a random port is used if unset | No |

Other configurable things:
- `fw_mark`
- `dns`
- `mtu`
- `table`
- `pre_up`
- `post_up`
- `pre_down`
- `post_down`
- `save_config`

These options can be configured for an interface but are unset by default, refer to `wg(8)` and `wg-quick(8)` manpages for their meaning.

### `wireguard_peers: $INTERFACE`

A hash configuring the host's peers in the form of `peer_name: { ... peer_configuration ... }`.

#### peer_configuration:
| Key | Description | Required |
| --- | ----------- | -------- |
| public_key | The public key of this peer | Yes |
| allowed_ips | The IPs to allow from this per, refer to `wg(8)` for exact format | Yes |
| endpoint | Public address to be used when connecting to this peer | No |
| preshared_key | Preshared key for additional security, refer to `wg(8)` for details | No |
| persistent_keepalive | A time interval in seconds to keep the tunnel alive | No

Example:

```yaml
wireguard_peers:
  wg0:
    fugu:
      public_key: 12345
      allowed_ips: 10.0.0.0/16
```

Dependencies
------------

None.

Examples
--------

### If wireguard_manage_keys is `False`
Star topology (multiple clients connecting to each other through one central server).

```yaml
# host_vars/someserver.yml
wireguard_interface:
  wg0:
    address: 10.0.0.1/16
    private_key: someserver_private_key
    listen_port: 12345

wireguard_peers:
  wg0:
    client1:
      public_key: client1_public_key
      allowed_ips: 10.0.0.11/32
    client2:
      public_key: client2_public_key
      allowed_ips: 10.0.0.12/32
```

```yaml
# group_vars/client.yml
wireguard_peers:
  wg0:
    someserver:
      public_key: someserver_public_key
      endpoint: someserver.example.com:12345
      allowed_ips: 10.0.0.1/16
```

```yaml
# host_vars/client1.yml
wireguard_interface:
  wg0:
    address: 10.0.0.11/16
    private_key: client1_private_key
```

```yaml
# host_vars/client2.yml
wireguard_interface:
  wg0:
    address: 10.0.0.12/16
    private_key: client2_private_key
```

```
# inventory file
someserver

[client]
client1
client2
```

```yaml
# playbook.yml
- hosts: all
  vars:
    wireguard_interfaces:
      - wg0
  roles:
     - wireguard
```

### If wireguard_manage_keys is `True`
All hosts is servers and clients (peer2peer).

```yaml
# host_vars/someserver.yml
wireguard_interface:
  wg0:
    address: 10.0.0.1/16
    private_key: someserver_private_key
    listen_port: 12345

wireguard_peers:
  wg0:
    client1:
      public_key: client1_public_key
      allowed_ips: 10.0.0.11/32
    client2:
      public_key: client2_public_key
      allowed_ips: 10.0.0.12/32
```

```yaml
# group_vars/all.yml
wireguard_listen_port: 5888
wireguard_wg0_preshared_key: secret_preshared_key
wireguard_wg0_peer_settings: >
  {%  set _peers = {} -%}
  {%- for node in groups['all'] | map('extract', hostvars) -%}
  {%-   if (node['wg0_ipv4'] is defined and inventory_hostname != node['inventory_hostname']) -%}
  {%-     set _data = {} -%}
  {%-     set _endpoint_data = [] -%}
  {{-     _endpoint_data.append(node.ip_v4) -}}
  {{-     _endpoint_data.append(wireguard_listen_port) -}}
  {%-     set _endpoint = _endpoint_data | join(':') -%}
  {%-     set x=_data.__setitem__('public_key', node.public_wg0.content | b64decode | trim) -%}
  {%-     set x=_data.__setitem__('allowed_ips', node.wg0_ipv4) -%}
  {%-     set x=_data.__setitem__('preshared_key', wireguard_wg0_preshared_key) -%}
  {%-     set x=_data.__setitem__('endpoint', _endpoint) -%}
  {%-     set x=_peers.__setitem__(node['inventory_hostname'], _data) -%}
  {%-   endif -%}
  {%- endfor -%}
  {{- _peers }}

wireguard_manage_keys: True
wireguard_manage_services: True
wireguard_interfaces:
  - wg0

wireguard_peers: 
  wg0: "{{ wireguard_wg0_peer_settings }}"

wireguard_interface:
  wg0:
    address: "{{ wg0_ipv4 }}"
    private_key: "{{ private_wg0.content | b64decode | trim }}"
    listen_port: "{{ wireguard_listen_port }}"
```

```
# inventory file
[wireguard-servers]
server1 wg0_ipv4=10.0.0.1/32
server2 wg0_ipv4=10.0.0.2/32
server3 wg0_ipv4=10.0.0.3/32
```

```yaml
# playbook.yml
- hosts: all
  vars:
    wireguard_interfaces:
      - wg0
  roles:
     - wireguard
```


Supported platforms
-------------------

- Arch Linux
- Debian
- EL7 and derivatives
- Fedora
- Ubuntu

License
-------

MIT
