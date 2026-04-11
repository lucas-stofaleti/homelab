---
applyTo: "docs/FIREWALL.md,docs/NETWORK.md"
---

# Instructions: Editing FIREWALL.md and NETWORK.md

These two files are tightly coupled. Any change to one almost always requires a corresponding change to the other. Always check both files before and after making edits.

## File roles

- **`docs/NETWORK.md`** — source of truth for VLANs, subnets, and IPAM (all IPs and ranges live here)
- **`docs/FIREWALL.md`** — OPNsense configuration runbook; references IPs/ranges defined in NETWORK.md

## DHCP ranges

- DHCP ranges in `FIREWALL.md` (Step 7) must exactly match the IPAM table in `NETWORK.md`.
- When a DHCP range changes, update both files in the same commit.
- No MAC-based static mappings. Devices needing fixed IPs have them configured directly on the device, outside the DHCP range. Never add static mapping tables to DHCP sections.

## VLAN access policy

- **VLAN 10 MGMT** — out-of-band recovery network. DNS (Pi-hole) + NTP + web outbound only. Proxmox and OPNsense are reachable intra-VLAN via the switch without firewall rules. Day-to-day infra management is done from VLAN 30 LAN.
- **VLAN 20 Servers** — DNS (Pi-hole intra-VLAN) + NTP + web outbound + Pi-hole upstream DNS (port 53 outbound). No access to other VLANs.
- **VLAN 30 LAN** — trusted primary workstation network. Full unrestricted access to all VLANs and internet (any port/proto). Single pass-all rule.
- **VLAN 40 IoT / VLAN 50 Guest** — DNS (Pi-hole) + NTP + full internet (`!RFC1918_INTERNAL`, any port/proto). No access to internal VLANs.

- **Reject** (sends TCP RST) on all VLAN interfaces — never silent Block on internal interfaces.
- **Block** (silent drop) on WAN only.
- Every rule must have logging enabled (`Log: ✓`).
- Internet-bound rules always use `!RFC1918_INTERNAL` as the destination, not `any`. This prevents accidentally matching internal IPs on ports 80/443.
- `RFC1918_INTERNAL` is defined as `192.168.0.0/17` (covers all internal VLANs).
- NTP rules point to the OPNsense gateway of the respective VLAN, not `any`.
- Intra-VLAN traffic never reaches OPNsense (the switch handles it). Do not add rules for traffic between hosts on the same VLAN.

## OPNsense WAN specifics

- WAN IP (`192.168.128.2`) is behind the ISP router on a private subnet (`192.168.128.0/29`).
- **`Block private networks` must be DISABLED on WAN.** Enabling it drops all traffic because the WAN IP itself is private.
- Outbound NAT must be enabled. Without it the ISP router receives packets from VLAN IPs it has no route back to.

## Adding a new VLAN

Follow this order — each step depends on the previous:

1. Add the subnet and IPAM entries to `NETWORK.md`.
2. Add aliases to the Aliases table in `FIREWALL.md` (network alias, gateway host alias).
3. Add firewall rules following the pattern of existing VLANs (DNS → NTP → permitted traffic → Reject all).
4. Add DHCP configuration to Step 7 in `FIREWALL.md` and update IPAM in `NETWORK.md` simultaneously.

## VLAN 20 DHCP note

VLAN 20 DHCP range (`192.168.20.16/28`, i.e. `.16–.31`) is for **initial VM provisioning only**. Once a VM is set up, a static IP is configured on the VM itself and it no longer uses DHCP.

## Future work sections

The `## Future work` section in `FIREWALL.md` contains `[FUTURE]` blocks for VLANs not yet active (currently VLAN 40 IoT and VLAN 50 Guest). Do not activate these sections until the prerequisites listed in each block are met. Do not add new future-work sections without explicit user instruction.

## Commit hygiene

- Changes to FIREWALL.md and NETWORK.md that are related (e.g., a DHCP range update) must be committed together, not separately.
- Do not reference AI tools in commit messages.
