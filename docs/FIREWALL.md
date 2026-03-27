# Homelab Firewall Rules

OPNsense firewall configuration for the homelab. Follow each section in order — aliases must exist before rules can reference them.

Sections marked **[FUTURE]** should not be configured until the corresponding infrastructure is in place.

## VLAN access summary

| VLAN | Internal access | Internet access |
|------|----------------|----------------|
| 10 MGMT | All VLANs, all ports | Web only (80/443) |
| 20 SERVERS | None | Web only (80/443) |
| 30 LAN | K8S ingress only | Web only (80/443) |
| 40 IOT | Pi-hole DNS only | Web only (80/443) |
| 50 GUEST | Pi-hole DNS only | Web only (80/443) |

## Configuration conventions

These apply everywhere in this document:

- **Rules are evaluated top to bottom** on the interface where traffic enters. OPNsense stops at the first match.
- **Intra-VLAN traffic** (e.g. K8S nodes talking to each other, Pi-hole on the same subnet) never reaches OPNsense — the switch forwards it directly. No rules are needed for that traffic.
- **All interfaces end with an explicit `Reject`**, not a silent drop. Denied connections fail immediately and appear in logs. The only exception is WAN, which uses `Block` to avoid disclosing the firewall.
- **All rules have logging enabled.** View traffic in real time at `Firewall > Log Files > Live View`.
- **`!RFC1918_INTERNAL`** is a negated alias meaning "any destination that is not an internal IP". It is used on all outbound internet rules to ensure they only match actual internet destinations, not other VLANs on the same ports.

---

## Step 1 — Firewall > Aliases

`Firewall > Aliases > Add`

Create all aliases before touching any rules. The **Used in** column shows which interfaces actively reference each alias. Pre-configured aliases have no current rule reference but are created now to avoid rework later.

### Network aliases (`Type: Network`)

| Name | Value | Used in | Notes |
|------|-------|---------|-------|
| `NET_MGMT` | `192.168.10.0/24` | VLAN10_MGMT | VLAN 10 - Management |
| `NET_SERVERS` | `192.168.20.0/24` | VLAN20_SERVERS | VLAN 20 - Servers |
| `NET_LAN` | `192.168.30.0/24` | VLAN30_LAN | VLAN 30 - LAN |
| `NET_IOT` | `192.168.40.0/24` | VLAN40_IOT (future) | VLAN 40 - IoT |
| `NET_GUEST` | `192.168.50.0/24` | VLAN50_GUEST (future) | VLAN 50 - Guest |
| `NET_K8S_API_VIPS` | `192.168.20.192/29` | Pre-configured | Kubernetes API VIPs |
| `NET_K8S_INGRESS_VIPS` | `192.168.20.208/28` | VLAN30_LAN | Kubernetes ingress VIPs |
| `NET_K8S_NODES` | `192.168.20.224/27` | Pre-configured | Kubernetes nodes |
| `RFC1918_INTERNAL` | `192.168.0.0/17` | All interfaces | Entire internal address space |

### Host aliases (`Type: Host`)

| Name | Value | Used in | Notes |
|------|-------|---------|-------|
| `HOST_OPNSENSE_WAN` | `192.168.128.2` | Pre-configured | OPNsense WAN IP |
| `HOST_OPNSENSE_MGMT` | `192.168.10.1` | Pre-configured | OPNsense VLAN 10 gateway |
| `HOST_OPNSENSE_SERVERS` | `192.168.20.1` | VLAN20_SERVERS | OPNsense VLAN 20 gateway |
| `HOST_OPNSENSE_LAN` | `192.168.30.1` | VLAN30_LAN | OPNsense VLAN 30 gateway |
| `HOST_PROXMOX_PVE01` | `192.168.10.5` | Pre-configured | Proxmox node 1 |
| `HOST_PIHOLE_01` | `192.168.20.5` | VLAN20–50 | Pi-hole DNS server |
| `HOST_AP_MGMT` | `192.168.30.5` | Pre-configured | Access point management IP |

### Port aliases (`Type: Port`)

| Name | Value | Used in | Notes |
|------|-------|---------|-------|
| `PORTS_DNS` | `53` | All interfaces | TCP/UDP |
| `PORTS_NTP` | `123` | All interfaces | UDP |
| `PORTS_WEB` | `80,443` | All interfaces | HTTP/HTTPS |
| `PORTS_OPNSENSE_MGMT` | `22,443` | Pre-configured | OPNsense SSH and web UI |
| `PORTS_PROXMOX_MGMT` | `22,8006` | Pre-configured | Proxmox SSH and web UI |

---

## Step 2 — Firewall > NAT > Outbound

`Firewall > NAT > Outbound`

Set mode to **Automatic outbound NAT**. No manual rules are needed.

OPNsense translates all outbound traffic to `192.168.128.2` (its WAN IP). This is required even though the ISP router also performs NAT — without it the ISP router receives packets from private VLAN addresses it has no route back to.

> No inbound port forwards will be created. No services are exposed outside the local network.

---

## Step 3 — Firewall > Rules > WAN

`Firewall > Rules > WAN`

**Interface settings** (`Interfaces > WAN > Edit`):

| Setting | Value | Reason |
|---------|-------|--------|
| Block private networks | **Disabled** | OPNsense WAN is itself `192.168.128.2`. Enabling this would break all connectivity. |
| Block bogon networks | **Enabled** | Safe to keep on. |

**Rules:**

| # | Action | Proto | Source | Src Port | Destination | Dst Port | Log | Description |
|---|--------|-------|--------|----------|-------------|----------|-----|-------------|
| 1 | Block | any | any | any | any | any | ✓ | Default deny — silent drop on WAN |

---

## Step 4 — Firewall > Rules > VLAN10_MGMT

`Firewall > Rules > VLAN10_MGMT`

Administrative network. Connect a laptop here via cable when performing infrastructure work. Full access to all internal networks. Internet limited to web.

| # | Action | Proto | Source | Src Port | Destination | Dst Port | Log | Description |
|---|--------|-------|--------|----------|-------------|----------|-----|-------------|
| 1 | Pass | any | `NET_MGMT` | any | `RFC1918_INTERNAL` | any | ✓ | Full access to all internal networks |
| 2 | Pass | TCP | `NET_MGMT` | any | `!RFC1918_INTERNAL` | `PORTS_WEB` | ✓ | Outbound internet — web only |
| 3 | Reject | any | any | any | any | any | ✓ | Default deny — fast reject |

> **Note:** Enable the NTP service under `Services > Network Time` so MGMT devices have a time source on the internal network.

---

## Step 5 — Firewall > Rules > VLAN20_SERVERS

`Firewall > Rules > VLAN20_SERVERS`

Server infrastructure. Outbound internet for updates and container pulls. Pi-hole can reach upstream DNS resolvers. No access to other VLANs.

| # | Action | Proto | Source | Src Port | Destination | Dst Port | Log | Description |
|---|--------|-------|--------|----------|-------------|----------|-----|-------------|
| 1 | Pass | UDP | `NET_SERVERS` | any | `HOST_OPNSENSE_SERVERS` | `PORTS_NTP` | ✓ | Time sync via OPNsense |
| 2 | Pass | TCP/UDP | `HOST_PIHOLE_01` | any | any | `PORTS_DNS` | ✓ | Pi-hole upstream DNS resolution |
| 3 | Pass | TCP | `NET_SERVERS` | any | `!RFC1918_INTERNAL` | `PORTS_WEB` | ✓ | Outbound internet — web only |
| 4 | Reject | any | any | any | any | any | ✓ | Default deny — fast reject |

---

## Step 6 — Firewall > Rules > VLAN30_LAN

`Firewall > Rules > VLAN30_LAN`

Trusted client devices. Internet access and K8S-hosted services. No direct access to MGMT or SERVERS.

| # | Action | Proto | Source | Src Port | Destination | Dst Port | Log | Description |
|---|--------|-------|--------|----------|-------------|----------|-----|-------------|
| 1 | Pass | TCP/UDP | `NET_LAN` | any | `HOST_PIHOLE_01` | `PORTS_DNS` | ✓ | DNS via Pi-hole |
| 2 | Pass | UDP | `NET_LAN` | any | `HOST_OPNSENSE_LAN` | `PORTS_NTP` | ✓ | Time sync via OPNsense |
| 3 | Pass | TCP | `NET_LAN` | any | `NET_K8S_INGRESS_VIPS` | `PORTS_WEB` | ✓ | Access self-hosted services via K8S ingress |
| 4 | Pass | TCP | `NET_LAN` | any | `!RFC1918_INTERNAL` | `PORTS_WEB` | ✓ | Outbound internet — web only |
| 5 | Reject | any | any | any | any | any | ✓ | Default deny — fast reject |

> **Note:** Rule 3 (K8S ingress) must stay above rule 4 (internet). Rule 4 would not match internal destinations anyway due to `!RFC1918_INTERNAL`, but the order makes the intent explicit.

---

## Future work

Items below are not yet configured. Each section notes what must be in place before proceeding.

---

### [FUTURE] Firewall > Rules > VLAN40_IOT

`Firewall > Rules > VLAN40_IOT`

**Prerequisites:** VLAN 40 created on switch and OPNsense. Add alias `HOST_OPNSENSE_IOT` (`192.168.40.1`, Type: Host) before creating these rules.

IoT devices. Internet and DNS only. Isolated from all internal networks.

| # | Action | Proto | Source | Src Port | Destination | Dst Port | Log | Description |
|---|--------|-------|--------|----------|-------------|----------|-----|-------------|
| 1 | Pass | TCP/UDP | `NET_IOT` | any | `HOST_PIHOLE_01` | `PORTS_DNS` | ✓ | DNS via Pi-hole |
| 2 | Pass | UDP | `NET_IOT` | any | `HOST_OPNSENSE_IOT` | `PORTS_NTP` | ✓ | Time sync via OPNsense |
| 3 | Pass | TCP | `NET_IOT` | any | `!RFC1918_INTERNAL` | `PORTS_WEB` | ✓ | Outbound internet — web only |
| 4 | Reject | any | any | any | any | any | ✓ | Default deny — fast reject |

---

### [FUTURE] Firewall > Rules > VLAN50_GUEST

`Firewall > Rules > VLAN50_GUEST`

**Prerequisites:** VLAN 50 created on switch and OPNsense. Add alias `HOST_OPNSENSE_GUEST` (`192.168.50.1`, Type: Host) before creating these rules.

Guest devices. Internet and DNS only. Isolated from all internal networks.

| # | Action | Proto | Source | Src Port | Destination | Dst Port | Log | Description |
|---|--------|-------|--------|----------|-------------|----------|-----|-------------|
| 1 | Pass | TCP/UDP | `NET_GUEST` | any | `HOST_PIHOLE_01` | `PORTS_DNS` | ✓ | DNS via Pi-hole |
| 2 | Pass | UDP | `NET_GUEST` | any | `HOST_OPNSENSE_GUEST` | `PORTS_NTP` | ✓ | Time sync via OPNsense |
| 3 | Pass | TCP | `NET_GUEST` | any | `!RFC1918_INTERNAL` | `PORTS_WEB` | ✓ | Outbound internet — web only |
| 4 | Reject | any | any | any | any | any | ✓ | Default deny — fast reject |

---

### [FUTURE] DNS redundancy

**Problem:** Pi-hole (`192.168.20.5`) is the sole DNS server for all VLANs. If it goes down, DNS resolution fails across the entire network.

**Solution:** Deploy a second Pi-hole (`pihole-02` at `192.168.20.6`). Use [Gravity Sync](https://github.com/vmstan/gravity-sync) to replicate blocklists between them. Configure both IPs as DNS servers in OPNsense DHCP settings for each VLAN.

**Firewall impact:** Add `HOST_PIHOLE_02` alias. Update VLAN20_SERVERS rule 2 to include `HOST_PIHOLE_02` as an additional DNS source alias.

---

### [FUTURE] Remote access

**Option A — Tailscale** (recommended for personal admin access): Installs as a VM or container in VLAN 20. Acts as a private mesh VPN — enrolled devices connect as if on MGMT. No ports opened on the ISP router.

**Option B — Cloudflare Tunnel**: Exposes specific services via Cloudflare's CDN using an outbound-only connection. Better suited for sharing a service (e.g. a dashboard) than for full admin access.

**Firewall impact for Tailscale:** Tailscale node in VLAN 20 needs outbound access on UDP `41641` (or falls back to TCP `443`). Add to VLAN20_SERVERS rules. Inbound from Tailscale peers is handled by the Tailscale daemon, not OPNsense rules.

---

### [FUTURE] Access point VLAN support

Current AP (`192.168.30.5`) does not support VLANs and must remain on VLAN 30. When the AP is replaced with a VLAN-capable model:

- Move the management IP to VLAN 10 and update `HOST_AP_MGMT` alias.
- Add a MGMT SSID (VLAN 10) for administrative wireless access.
- Keep a LAN SSID (VLAN 30) for daily use.
- Keep IoT/Guest SSIDs mapped to VLANs 40/50 when those are activated.

---

### [FUTURE] OPNsense HA

Single OPNsense instance is a network single point of failure. When a second unit is available, configure High Availability using CARP virtual IPs under `System > High Availability`. All gateway IPs documented in this file would become CARP VIPs shared between the two nodes.

---

### [FUTURE] Backup strategy

- **OPNsense config:** Export after every change at `System > Configuration > Backups`. Store off-box.
- **Pi-hole settings:** Use Teleporter (`Settings > Teleporter`) to export. Also covered by Gravity Sync if a second Pi-hole is deployed.
