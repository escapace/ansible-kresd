# ansible-kresd

Installs and configures Knot Resolver on RHEL-family version 10 systems with
systemd. The role enables the resolver at boot but never starts a stopped service.
Configuration changes restart it only when it is already running.

Run with root privileges and gathered facts. The role disables systemd-resolved at
boot and prevents an installed NetworkManager from managing DNS. On non-Docker
hosts, it replaces `/etc/resolv.conf` with an immutable file pointing to loopback
DNS only when the resolver is running and the managed systemd unit changes.
During image construction, provide boot-time DNS configuration separately if the
resolver is not running. Cloud-init, when installed, is configured to use loopback
DNS on subsequent boots.

## Variables

- `provider` is optional and unset by default. Only `ec2` enables EC2 environment
  file inspection and configuration; other values skip it.
- `kresd_ipv4` and `kresd_ipv6` default to `true`; both require YAML booleans.
  Enable at least one address family. These also select TLS forwarder addresses.
- `kresd_cache_size` defaults to `16`; it accepts positive integer MiB values.
- `kresd_workers` defaults to `auto`; it also accepts a positive integer.
- `kresd_log_level` defaults to `err`; choices are `crit`, `err`, `warning`,
  `notice`, `info`, and `debug`.
- `kresd_log_dnssec` defaults to `false`; it requires a YAML boolean.
- `kresd_listen` is a non-empty list of mappings with a required `interface`
  string and an optional YAML boolean `freebind` (default `false`). Its default is:

  ```yaml
  kresd_listen:
    - interface: 127.0.0.1@53
    - interface: ::@53
      freebind: true
  ```

  The role permits DNS queries only from loopback, even with wildcard listeners.

- `kresd_tls_forward` is a non-empty list of `{ address: IP, hostname: DNS-name }`
  mappings. Defaults are Cloudflare (`1.1.1.1`, `1.0.0.1`,
  `2606:4700:4700::1111`, `2606:4700:4700::1001`, hostname
  `cloudflare-dns.com`) and Google (`8.8.8.8`, `8.8.4.4`,
  `2001:4860:4860::8888`, `2001:4860:4860::8844`, hostname `dns.google`).
- `kresd_prometheus_endpoint` defaults to `false`; it requires a YAML boolean
  and enables unencrypted HTTP metrics on IPv4 and IPv6 loopback.
- `kresd_prometheus_endpoint_port` defaults to `8453`; it accepts integers from
  1 through 65535.
- `kresd_prometheus_namespace` defaults to `kresd_`. It accepts an empty string
  or a metric-name prefix starting with a letter, underscore, or colon, followed
  by letters, digits, underscores, or colons.
- `kresd_copr_rpm_key` defaults to
  `https://download.copr.fedorainfracloud.org/results/@cznic/knot-resolver/pubkey.gpg`.
- `kresd_copr_repo_url` defaults to
  `https://copr.fedorainfracloud.org/coprs/g/cznic/knot-resolver/repo/epel-10/group_cznic-knot-resolver-epel-10.repo`.
- `kresd_copr_repo_dest` defaults to
  `/etc/yum.repos.d/group_cznic-knot-resolver-epel-10.repo`; it accepts an absolute
  managed-host path.

## EC2 integration

When `provider: ec2`, the role checks `/etc/sysconfig/ec2-environment`. A regular
file enables EC2 DNS configuration. The caller owns this file and its boot-time
availability; the role neither creates it nor installs AWS tooling.

Supply systemd-compatible environment assignments for
`EC2_INSTANCE_ETH0_IPV4_VPC_DNS` and, when available,
`EC2_INSTANCE_ETH0_IPV6_VPC_DNS`, plus the corresponding
`EC2_INSTANCE_ETH0_IPV4_VPC_REVERSE_POINTERS` and
`EC2_INSTANCE_ETH0_IPV6_VPC_REVERSE_POINTERS` DNS suffixes. The role routes
`amazonaws.com.`, `aws.`, `internal.`, and those reverse zones to the VPC
resolver without caching. File presence enables the integration; the role does
not validate its contents or order its producer at boot.

With `provider: ec2`, an absent file removes only the role's EC2 policy and service
drop-in; a non-regular file at that path causes validation to fail. When `provider`
is unset or has another value, the role leaves existing EC2 configuration alone. Check mode does not
perform live service operations or replace `/etc/resolv.conf`.
