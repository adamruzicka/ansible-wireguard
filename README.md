WireGuard
=========

A role for configuring WireGuard VPN.

Requirements
------------

The role should be self-contained, just provide vars for your hosts and run it.

Role Variables
--------------

### `wireguard_networks`

Each host needs to have `wireguard_networks` variable set. It should be a list of WireGuard interface name the host should use, by default it is an empty list. For each `$INTERFACE` specified here the host should have `wireguard_$INTERFACE_interface` and `wireguard_$INTERFACE_peers` vars set.

### `wireguard_$INTERFACE_interface`

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

### `wireguard_$INTERFACE_peers`

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
wireguard_wg0_peers:
  - fugu:
      public_key: 12345
      allowed_ips: 10.0.0.0/16
```

Dependencies
------------

None.

Example
-------

Star topology (multiple clients connecting to each other through one central server).

```yaml
# host_vars/someserver.yml
wireguard_wg0_interface:
  address: 10.0.0.1/16
  private_key: someserver_private_key
  listen_port: 12345

wireguard_wg0_peers:
  client1:
    public_key: client1_public_key
    allowed_ips: 10.0.0.11/32
  client2:
    public_key: client2_public_key
    allowed_ips: 10.0.0.12/32
```

```yaml
# group_vars/client.yml
wireguard_wg0_peers:
  someserver:
    public_key: someserver_public_key
    endpoint: someserver.example.com:12345
    allowed_ips: 10.0.0.1/16
```

```yaml
# host_vars/client1.yml
wireguard_wg0_interface:
  address: 10.0.0.11/16
  private_key: client1_private_key
```

```yaml
# host_vars/client2.yml
wireguard_wg0_interface:
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
