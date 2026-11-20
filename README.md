# blog-bind-dynamic-dns

Companion code for the blog post
[Setting Up Dynamic DNS Updates with BIND and TSIG Authentication](https://blog.bsdserver.nl/setting-up-dynamic-dns-updates-with-bind-and-tsig-authentication/).

It builds the post's setup on two machines:

- `ns1`: BIND 9 with the post's TSIG keys (`ddns-key`, `dhcp-key`, `k8s-update-key`,
  `admin-update-key`), one file each under `/etc/bind/keys/` (RHEL: `/etc/named/keys/`), mode
  0440 root:bind, and the dynamic zones with their `allow-update` / `update-policy` rules:

  | Zone | Who may update |
  |------|----------------|
  | `dynamic.example.com` | `ddns-key` (allow-update, with journal) |
  | `internal.example.com` | `dhcp-key`: A, AAAA, DHCID |
  | `1.168.192.in-addr.arpa` | `dhcp-key`: PTR, DHCID |
  | `k8s.example.com` | `k8s-update-key`: anything below `k8s.example.com` |
  | `example.com` | `admin-update-key`: anything in the zone |

  Update logging goes to `/var/log/named/update.log`.
- `dhcp1`: Kea DHCPv4 (`kea-dhcp4`) and Kea DHCP-DDNS (`kea-dhcp-ddns`, D2) with the
  configuration from the post. When a client gets a lease, D2 adds its A, DHCID and PTR
  records, signed with `dhcp-key`; when the lease is released or expires, it removes them.
  Clients that send no hostname get a generated name such as `dhcp-192-168-1-100`.

BIND and Kea can share one host: put it in both inventory groups.

## Differences from the post

- On Debian/Ubuntu the role keeps keys in `/etc/bind/keys` and dynamic zones in
  `/var/lib/bind/dynamic` instead of the post's `/etc/named/keys` and `/var/named/dynamic`.
  Ubuntu's AppArmor profile for `named` covers `/etc/bind` and `/var/lib/bind`, not
  `/etc/named` or `/var/named`. On RHEL the role uses the post's paths and labels the zone
  directory `named_zone_t`, as the post does.
- The zone files the role creates hold only SOA and NS. They are written once; after that
  `named` owns them (and the journals), so a later playbook run does not overwrite records
  added by updates.

## What is in here

| Path | Purpose |
|------|---------|
| `terraform/` | Two Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider, cloud-init), outputs an Ansible inventory |
| `ansible/site.yml` | One playbook for the whole setup |
| `ansible/roles/bind` | Installs BIND and writes `named.conf` with includes |
| `ansible/roles/bind_ddns` | Key files, dynamic zones, update policies, update logging |
| `ansible/roles/kea_ddns` | `kea-dhcp4.conf` and `kea-dhcp-ddns.conf` (0640 root:_kea), services |

## Requirements

- Terraform >= 1.6 and a Proxmox VE API token (only if you want Terraform to create the VMs).
  Put the VMs on a bridge where no other DHCP server answers.
- ansible-core >= 2.15 (tested with 2.20.1) and `community.general` (see `requirements.yml`)
- BIND host: Ubuntu 24.04 (tested, BIND 9.18) or RHEL/Rocky 9 (handled, not lab-tested)
- Kea host: Debian/Ubuntu (the post installs the Ubuntu packages; Kea 2.4.1 on 24.04)

## Quick start

```bash
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init && terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

cd ../ansible                                   # or: cp inventory.example.yml inventory.yml
cp group_vars/all/vault.yml.example group_vars/all/vault.yml
# four secrets from `tsig-keygen -a hmac-sha256 <name>`, then:
ansible-vault encrypt group_vars/all/vault.yml
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml --ask-vault-pass
```

An update with a key (on `ns1`; copy a key file elsewhere to run `nsupdate` remotely):

```bash
sudo nsupdate -k /etc/bind/keys/ddns.key <<EOF
server 192.168.1.10
zone dynamic.example.com
update add web.dynamic.example.com 300 A 192.0.2.10
send
EOF
dig @192.168.1.10 +short web.dynamic.example.com
```

A record and its PTR by hand (the post's "Reverse DNS Updates"): use `dhcp-key`, the only key
granted on both `internal.example.com` (A) and the reverse zone (PTR). `ddns-key` gets `REFUSED` there.

```bash
sudo nsupdate -k /etc/bind/keys/dhcp.key <<EOF
server 192.168.1.10
zone 1.168.192.in-addr.arpa
update add 100.1.168.192.in-addr.arpa 300 PTR hostname.example.com.
send
EOF
dig @192.168.1.10 +short -x 192.168.1.100
```

After a client on the DHCP subnet gets a lease:

```bash
dig @192.168.1.10 +short <hostname>.internal.example.com A
dig @192.168.1.10 +short <hostname>.internal.example.com DHCID
dig @192.168.1.10 +short -x 192.168.1.100
```

## Variables

Set in `ansible/group_vars/all/vars.yml` (defaults in the roles' `defaults/main.yml`):

| Variable | Default | Meaning |
|----------|---------|---------|
| `bind_ddns_keys` | the post's four keys | name, file, algorithm, secret (from vault) |
| `vault_ddns_*_secret` | none | The secrets, in `group_vars/all/vault.yml` |
| `bind_ddns_zones` | the five zones above | name, file, and `allow_update` or `update_policy` rules |
| `bind_ddns_server_address` / `bind_ddns_ns_name` | `192.168.1.10` / `ns1.example.com.` | NS record in the new zones |
| `bind_ddns_update_logging` | `true` | `update` and `update-security` categories to `update.log` |
| `kea_ddns_interfaces` | `["eth0"]` | Interfaces Kea listens on |
| `kea_ddns_subnet` / `kea_ddns_pool` | `192.168.1.0/24` / `.100 - .200` | DHCP subnet and pool |
| `kea_ddns_routers`, `kea_ddns_domain_name`, `kea_ddns_domain_name_servers` | from the post | DHCP options |
| `kea_ddns_qualifying_suffix` / `kea_ddns_generated_prefix` | `internal.example.com.` / `dhcp` | Names Kea builds |
| `kea_ddns_forward_zone` / `kea_ddns_reverse_zone` / `kea_ddns_dns_server` | from the post | Where D2 sends updates |
| `kea_ddns_tsig_key_name` / `kea_ddns_tsig_secret` | `dhcp-key` / vault | D2's key, same as BIND's |

If you change the DHCP subnet, change the reverse zone in `bind_ddns_zones` and
`kea_ddns_reverse_zone` to match.

## Tested

`site.yml` was run against fresh Ubuntu 24.04 systemd containers (BIND 9.18.39, Kea 2.4.1);
the second run, and a third one after updates had been made, report `changed=0`. Checks
afterwards: a DHCP client with hostname `host42` got a lease and D2 created the A, DHCID and
PTR (plus reverse DHCID) records; after the release they were gone (Kea reclaims released
leases on its timer, so removal takes a few seconds). A client without a hostname got
`dhcp-<address>` records. `nsupdate` with each key worked where its policy allows it and got
`REFUSED` elsewhere (`ddns-key` on `internal.example.com`, `k8s-update-key` on `example.com`,
`dhcp-key` adding TXT, no key at all). ansible-lint (production profile), yamllint,
`terraform validate` and tflint are clean. The Terraform code has not been applied against a
real Proxmox cluster.

## License

MIT
