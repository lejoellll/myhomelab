# Progress Status
## Phase 1 – Get the basic setup running

- [x] Install OPNsense and get it running
- [x] Basic OPNsense configuration (internet works, LAN reachable, simple single-network setup)
- [x] Set up DHCP & DNS on the network – no VLANs yet
- [x] Install Tailscale 
- [ ] Install Proxmox
- [ ] Basic Proxmox configuration (network, storage, updates)
- [ ] Create first test VM or container on Proxmox
- [ ] Set up and get the first real service running

## Phase 2 – Improve & expand (iterative, partly redundant with phase 1)

- [ ] Structure the network: introduce VLANs (management, servers, IoT, guests, ...)
- [ ] Adapt/split DHCP & DNS to match the VLAN structure
- [ ] Refine firewall rules between the new networks
- [ ] Further harden access (2FA, certificates, possibly site-to-site VPN/WireGuard alongside/instead of Tailscale)
- [ ] Refine the Proxmox setup (backups, roles/permissions, possibly clustering)
- [ ] Set up additional services and clean up existing ones
- [ ] Introduce monitoring/logging