# High Availability DNS & DHCP Deployment

This document explains the project structure, design decisions, how to use Ansible roles (full run and DNS-only updates), and step-by-step **failure tests** for both DNS and DHCP from a client perspective.

---

## 1. Project Structure

```
jb_ta/
├── ansible.cfg                # Ansible configuration (inventory path, vault_password_file)
├── inventory/                 # Hosts inventory and group_vars
│   ├── hosts.yml              # Defines groups
│   └── group_vars/
│       ├── all.yml            # Global variables
│       ├── all/vault.yml      # Encrypted credentials
│       ├── dns.yml            # DNS role vars
│       ├── dhcp.yml           # DHCP role vars
│       └── keepalived.yml     # Keepalived global vars
├── playbooks/
│   └── all.yml                # Main orchestrator
└── roles/
    ├── common/                # Baseline system configuration
    │   ├── defaults/main.yml  
    │   ├── tasks/main.yml     
    │   └── handlers/main.yml  
    ├── dns/                   # BIND9 DNS service
    │   ├── defaults/main.yml  
    │   ├── tasks/main.yml     
    │   ├── tasks/records.yml  
    │   ├── handlers/main.yml  
    │   └── templates/         # Jinja2 templates for DNS
    │       ├── named.conf.options.j2  # Global DNS options 
    │       ├── named.conf.local.j2    # Zone declarations 
    │       └── db.jb-ta.local.j2      # Zone file
    ├── dhcp/                  # ISC DHCP service
    │   ├── defaults/main.yml  
    │   ├── tasks/main.yml     
    │   ├── handlers/main.yml  
    │   └── templates/         # Jinja2 templates for DHCP
    │       └── dhcpd.conf.j2   # DHCP config 
    └── keepalived/            # VRRP failover using Keepalived
        ├── defaults/main.yml  
        ├── tasks/main.yml     
        ├── handlers/main.yml  
        └── templates/
            └── keepalived.conf.j2  # VRRP instance template (state via bind_role, health-check script)
```

---

## 2. Design Decisions

- **Keepalived (VRRP)**: Provides automatic failover of the virtual IP (`virtual_ip`) between primary and secondary nodes in 3–5 seconds.
- **BIND9 primary/secondary**: primary and secondary zones with Jinja2 templating.
- **ISC-DHCP-server**: Dynamic pool and optional static reservations managed via Jinja2 templates.
- **Ansible**: Fully automated provisioning, idempotent roles, and vault for sensitive data.

### 2.1 High Availability Architecture

The overall HA scheme consists of three layers:

1. **VRRP with Keepalived**
   - Both nodes run Keepalived configured with the same `virtual_router_id` and `virtual_ip`.
   - The primary node holds the `virtual_ip` and sends advertisements every `advert_int` seconds.
   - If the primary fails health checks (VRRP script `chk_dns` fails or service is down), the backup node takes over the `virtual_ip` within ~3–5 seconds.
2. **BIND9 Primary/Secondary**
   - The zone file is deployed only on the primary node; once the template changes, the Ansible handler restarts BIND (`systemctl restart bind9`) so the new zone is loaded.
   - The secondary node receives zone updates through BIND’s NOTIFY + IXFR/AXFR mechanism whenever the SOA serial on the primary changes.
   - Zone transfers ensure DNS continuity even if the primary is offline.
3. **DHCP via Shared `virtual_ip`**
   - The DHCP service binds to the `virtual_ip`, so both primary and backup serve requests on the same IP.
   - Clients always send DHCP DISCOVER to the `virtual_ip`; failover of Keepalived moves the `virtual_ip`, and the backup responds seamlessly.

---

### 2.2 DNS Zone SOA Timing Parameters

In the DNS zone template (`db.jb-ta.local.j2`), these parameters control synchronization and caching:

```jinja
$TTL    604800
@   IN  SOA  ns1.{{ dns_zone_name }}. admin.{{ dns_zone_name }}. (
        {{ ansible_date_time.year }}{{ "%02d"|format(ansible_date_time.month|int) }}{{ "%02d"|format(ansible_date_time.day|int) }}01 ; Serial
        60          ; **Refresh** — how often the secondary checks for a new serial (1 minute)
        120         ; **Retry** — how often to retry if the refresh fails (2 minutes)
        604800      ; **Expire** — how long the secondary will keep serving the zone without a successful refresh (7 days)
        86400       ; **Negative TTL** — time-to-live for NXDOMAIN responses (1 day)
)
```

- **`$TTL 604800`**: Default TTL for all records (7 days) unless overridden.

---

### 2.3 DHCP Failover Peer Setup

To ensure both DHCP servers share the same lease database, configure the built-in ISC DHCP failover protocol in `dhcpd.conf.j2`:
With this configuration, both servers replicate leases over TCP, guaranteeing consistent assignments across failover.

---

## 3. Usage

### 3.1 Full Deployment

**Environment Preparation:**

### IP Address Configuration Locations

You can change any of the following IP address settings by editing the specified files:

- **Inventory hosts:**
  - `inventory/hosts.yml` — host names to IP mappings (`ansible_host`).
- **Per-host overrides:**
  - `inventory/host_vars/dns-dhcp-1.yml` — set `ansible_host` and any host-specific IPs.
  - `inventory/host_vars/dns-dhcp-2.yml` — set `ansible_host` and any host-specific IPs.
- **Virtual IP (VIP):**
  - `inventory/group_vars/all/all.yml` — `virtual_ip` for DNS & DHCP failover.
- **DHCP network and pool:**
  - `inventory/group_vars/dhcp.yml` — variables: `dhcp_network`, `dhcp_netmask`, `dhcp_range_start`, `dhcp_range_end`, `dhcp_peer_ip`.
- **DNS primary/secondary IPs and records:**
  - `inventory/group_vars/dns.yml` — `dns_primary_ip`, `dns_secondary_ip`, plus any `dns_records` record values.
- **Keepalived global settings:**
  - `inventory/group_vars/keepalived.yml` — verify `virtual_router_id` and health-check script settings.


**Preparation:** Before running the full deployment, populate the Vault file with your deployment credentials:

```bash
ansible-vault edit inventory/group_vars/all/vault.yml
```
```yaml
# Example contents:
ansible_user: deploy_user
ansible_password: deploy_pass
keepalived_auth_pass: keepalived_pass
```

This playbook and roles were tested on **Ubuntu 24.04**.

Because the `vault_password_file` is configured in `ansible.cfg`, Ansible automatically decrypts the Vault file at runtime. You can run the playbook without additional flags:

```bash
ansible-playbook -i playbooks/all.yml
```

This executes all roles in order: `common`, `dhcp`, `dns`, `keepalived`.

### 3.2 DNS-Only Zone Update

To update only the DNS zone file (tasks tagged `dns_records`), simply run:

```bash
ansible-playbook -i playbooks/all.yml -t dns_records
```

This will render the zone file from the `dns_records` variable and trigger the *Restart bind9* handler on the primary node.

### 3.3 Vault Usage

Credentials such as `ansible_user`, `ansible_password`, and `keepalived_auth_pass` are stored encrypted in:

```text
inventory/group_vars/all/vault.yml
```

Because the `vault_password_file` is configured in `ansible.cfg`, Ansible automatically decrypts this file at runtime. No additional flags are required for normal runs.

To **edit** the vault file:
```bash
ansible-vault edit inventory/group_vars/all/vault.yml
```

To **view** vaulted contents without editing:
```bash
ansible-vault view inventory/group_vars/all/vault.yml
```

---

## 4. Client-Side Failover Tests

### 4.1 DNS Failover Test
1. **Resolve the test record** using the virtual IP as the DNS server:
   ```bash
   dig @172.16.28.254 check.jb-ta.local +short
   ```
   Expect: `172.16.28.228`.
2. **Stop the primary DNS node** (on the primary server):
   ```bash
   sudo systemctl stop keepalived
   ```
   This causes VRRP failover in ~3–5 seconds.
3. **Clear DNS cache** on the client:
   - Linux: `sudo resolvectl flush-caches`
   - macOS: `sudo killall -HUP mDNSResponder`
4. **Resolve the test record again**:
   ```bash
   dig @172.16.28.254 check.jb-ta.local +short
   ```
   Expect: still `172.16.28.228`, with no interruption.

Total downtime should be under 5 seconds (VRRP failover time).

### 4.2 DHCP Failover Test
1. **Release and renew the DHCP lease** on the client:
   ```bash
   sudo dhclient -r && sudo dhclient
   ```
2. **Verify the assigned IP** and DNS server (should be the virtual IP).
3. **Stop the primary DHCP node** (on the primary server):
   ```bash
   sudo systemctl stop keepalived
   ```
4. **Release and renew the DHCP lease again** within a few seconds:
   ```bash
   sudo dhclient -r && sudo dhclient
   ```
5. **Verify** that the client continues to obtain a lease via the virtual IP without noticeable downtime.

Both DNS and DHCP services should seamlessly fail over via the shared `virtual_ip`, ensuring continuous client connectivity.

---
