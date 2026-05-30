# Network Monitoring — Zeek + Suricata

Zeek and Suricata run on VLAN 310 and receive mirrored traffic from both DMZ VLANs (110 and 210). Because both firewall VMs have a leg on VLAN 310, traffic can be mirrored at the OPNsense level using the built-in packet capture or a dedicated span.

## Traffic mirroring approach (OPNsense)

OPNsense supports traffic capture per interface. For a persistent mirror to the Zeek/Suricata VM:

1. On `fw-wan-a`: mirror `vtnet1.110` (DMZ VLAN) to VLAN 310 via `tcpreplay` or OPNsense's traffic shaper mirror rule
2. On `fw-wan-b`: same for `vtnet1.210`
3. Zeek/Suricata VM listens on its VLAN 310 interface in promiscuous mode

## Dual-stack monitoring

Both IPv4 and IPv6 flows need to be captured. Suricata rules apply to both stacks by default when rules reference `any`. Zeek logs both transparently.

Key logs:
- `conn.log` — all connections (both stacks)
- `dns.log` — DNS queries (watch for tunneling, unusual resolvers)
- `http.log` — HTTP metadata
- `ssl.log` — TLS metadata (SNI, cert details)
- `files.log` — file transfers across protocols

## Feeds

Suricata ET Open rules cover known C2, scanners, and exploit attempts. Update daily via `suricata-update`.
