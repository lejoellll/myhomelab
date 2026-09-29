# Logbook

This section documents changes, experiments, and fixes across my homelab. The goal is to keep track of what I did and why, so I can look back months or years later and understand past decisions — and see how the setup has evolved over time.

Each entry follows a simple structure: **Date**, **Topic**, **Initial State**, **End State** and **Learnings** (plus optional notes on affected components or next steps).


## 2026-09-20

**Topic**: Stepwise VLAN segmentation and migration of Home Assistant OS to a Proxmox VM.

**Initial State**: Home Assistant OS was running on my Raspberry Pi 5. I wanted to move it to a Proxmox VM without setting up its settings, apps, and integrations again. VLAN segmentation was still in progress.

**End State**:
* Continued assigning devices to their intended VLANs.

* Downloaded the HAOS disk image (.qcow2 file, pre-built disk image for QEMU)

* Created a Home Assistant OS VM on Proxmox.

    * Machine: q35
    * BIOS: OVMF (UEFI)
    * Checked "Add EFI DISK"
    * Deleted the default disk that was created
    * Sockets: 1
    * Cores: 4
    * Type: host -> the VM is able to use the CPUs full instruction set
    * Memory: 4096 MB
    * Bridge: vmbr0
    * Vlan Tag: 20
    * Model VirtIO

    * SSH into Promox and imported the downloaded image into the VM.
    *qm importdisk 101 /tmp/haos_ova-18.3.qcow2.xz local-lvm*

    * Attached it as scsi0, and set it as the first boot device. 

* Created a backup of my existing HAOS setup on the Raspberry Pi 5 and attempted to restore it on the VM.

* The migration was not completed. The VM's web GUI was not reachable after the restore attempt, and other issues remained unresolved.

* Shut down the HAOS VM. HAOS continues to run on the Raspberry Pi 5.

**Learnings**:
* Dnsmasq was not enabled on the VLAN10 interface. After enabling it, DHCP clients in VLAN10 could obtain an IP address.

* A device such as a TV sends untagged traffic. Its switch port needs to be assigned to the intended VLAN as untagged, with the PVID set to the same VLAN ID.

* The HAOS QCOW2 image is a pre-built **bootable disk, not an installer ISO**; the imported disk replaces the empty default disk

**Next Steps**:
* Implement the IoT VLAN and assign the IoT devices to it.

* Create and test the required firewall rules.

* Create a fresh backup of the HAOS instance on the Raspberry Pi 5 and retry the migration.

## 2026-09-16

**Topic**: Creating my first Linux container, installing AdGuard Home, and assigning it to VLAN20.

**Initial State**: I'd like to move my services (Adguard Home & Home Assistant OS), which are currently running on my Raspberry Pi devices, to the server. That way, I can use my Raspberry Pi devices for other purposes, such as smart display or as jump server.

**End State**:
* Created a LXC with the following attributes:

    * cores:    1
    * memory:   512
    * swap:     512
    * hostname: adguard
    * os:       debian-12-standard_12.12-1_amd64.tar.zst
    * net0:     name=eth0, bridge=vmbr0, tag=20, firewall=1, ip=10.10.20.130/24, gw=10.10.20.1, ip6=dhcp
    * unprivileged: 1

* Ran AdGuard Home's official install script:

    * curl -sSL https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

* After running the script the web GUI is reachable for finishing the installation.

* Created five firewall rules for Adguard Home:

    * Adguard Home → OPNsense (port 53 & 53053)
    * Adguard Home → Blocklist Updates (port 443)
    * Adguard Home X Internet (port 80)
    * Trusted → AH101 WebGUI (port 80)
    * Trusted → DNS Request (AH101)

**Learnings**: Without any firewall rules the whole traffic is blocked by the implicit deny rule. 

## 2026-09-14

**Topic**: Installing proxmox ve on the server machine

**Initial State**: After testing the server hardware and confirming that everything is fine, the next step was to install pve on the machine and make some network configuration. I'd like to have the proxmox machine assigned to VLAN30 and all the running services in VLAN20.

**End State**:
* Created a boot stick for installing proxmox ve.

* Enabled CPU virtualization (Intel VT-x/AMD-V) and IOMMU (Intel VT-d) in the BIOS/UEFI to prepare the host for virtual machines and potential PCIe passthrough.

* Adjusted the boot order - booting from usb-stick first.

* Selected ext4 as format for the SSD.

* Configured network settings such as MGMT interface, hostname, static IP-address, gateway and DNS-server.

* After the reboot, the web GUI is reachable over https://IP-address:8006.

* First step was to deactivate the enterprise-repository and using instead the community-repos.

**Learnings**: CPU virtualization (VT-x/AMD-V) enables hardware-assisted virtual machines, while IOMMU (VT-d/AMD-Vi) is needed for passing physical PCIe devices through to a VM.

## 2026-09-13

**Topic**: Preparing OPNsense for VLAN segmentation 

**Initial State**: The basic setup is running properly. Now it is time to structure the local area network to make it more secure. Therefore I started to create VLAN10 to VLAN60 in OPNsense, assigned them to a physical interface and did some more configuration in terms of DHCP and the IP range.

**End State**:
* Created all six VLANs under Interfaces > Devices > VLAN and selected the LAN interface as parent interface which will carry the VLAN tagged traffic. I left the VLAN priority untouched (Default: Best Effort). 

* After setting up the basic VLAN config, I assigned the VLANs that were just created to the logical interfaces. Did this under Interfaces > Assignments. 

* I enabled the newly created interfaces and gave them a static IPv4 adress such as 10.10.10.1/24 - this is the gateway address for the subnet.

* Under Services > DNSmasq DNS & DHCP > DHCP ranges I created for each VLAN one entry with a IP range such as 10.10.10.99 - 10.10.10.199. Finished this setup by adding the logical interfaces under Services > DNSmasq DNS & DHCP > General > Interface otherwise the devices in the certain VLANs don't get an IP address and so on.

* The next step would be creating firewall rules for each VLAN, configuring the managed switch and the access point - I skipped this for the moment.

**Learnings**: 
* A VLAN in OPNsense is created on a parent interface first, then assigned and enabled as a logical interface.

* Configuring VLANs in OPNsense does not complete the network setup. The switch and access point must carry or assign the matching VLAN tags, and firewall rules will define which traffic may pass between networks. This part has not been configured yet.

## 2026-09-10

**Topic**: Correcting the DNS configuration

**Initial State**: In the current setup, Unbound is just the backup DNS resolver and becomes active as soon as Adguard Home stops working.

The goal is to have AdGuard Home as the DNS proxy and Unbound as the resolver, instead of relying on a public DNS resolver as before.

Client (DNS Request) → Adguard Home (DNS-Proxy) → DNSmasq (local domain name resolution)/Unbound (DNS-Resolver) → Client (DNS Response)

**End State**:
* Modified the DNS Upstream Server table - deleted all public DNS resolver and added Unbound to the list. 

* Finally there are two entries:

    * [/lan.internal/]192.168.10.1:53053 (DNSmasq)
    * 192.168.10.1:53 (Unbound)

**Learnings**: Learned the difference between a DNS resolver and a DNS forwarder, and that Adguard Home is just a DNS proxy - nothing more, nothing less. Cloudflare and smiliar services are public DNS resolvers, while Unbound keeps resolution private.

## 2026-09-06

**Topic**: Implementing local domain name resolution

**Initial State**: It is not working properly, yet.

✅ nslookup ap.lan.internal 192.168.10.1:53053 (DNSmasq)

❌ nslookup ap.lan.internal 192.168.10.130 (Adguard Home)

AdGuard Home doesn't know how to handle local domain name resolutions, resulting in an error.

**End State**: 
* Added an entry in DNS Settings > Upstream DNS servers for a domain-specific upstream for lan.internal:  
[/lan.intternal/]192.168.10.1:53053

* Under DNSmasq > General, I enabled 'Register ISC DHCP4 leases'.

* The local domain name resolution is working now.

**Learnings**:
* *nslookup host server* helps pinpoint exactly where a DNS chain breaks.

* AdGuard Home needs an explicit upstream rule to resolve internal domains – it doesn't do this automatically.

* Local resolution required both an AdGuard Home upstream rule and Dnsmasq's lease registration setting to work.

## 2026-09-05

**Topic**: Evaluating new hardware (Lenovo M920q Tiny) intended to eventually host all my services.

**Initial State**: I bought a refurbished tiny pc, which needs to be thoroughly tested before use.

**End State**: I ran several hardware tests (CPU, RAM & storage), checked the BIOS/UEFI for a supervisor password from the previous owner, and verified the TPM version.

**Learnings**: Learned how to properly benchmark hardware and interpret the resulting values. Initially assumed one program would handle both benchmarking and monitoring, learned that these are typically separate tools.

## 2026-08-28
**Topic**: Preparing OPNsense for double NAT (Fritzbox as upstream router, OPNsense behind it).

**Initial State**: OPNsense was not prepared yet for double NAT behind the Fritzbox; no static IP assignments, no exposed host, and no UPnP set up for devices like the PS5.

**End State**:
* Noted the MAC addresses of the most important devices to later assign static IPs.

* Created static IP entries in Dnsmasq.

* Set the dynamic DHCP range so that the static IPs also fall within it (otherwise domain name resolution issues can occur, OPNsense bug).

* Configured the interaction between Dnsmasq and Unbound with AI assistance – don't fully understand yet how the two work together, needs further study.

* Set OPNsense as the **exposed host** in the Fritzbox settings.

* Enabled UPnP in OPNsense via the community plugin os-upnp.

* Added an ACL entry for the PS5 set to "Allow" (not tested yet).

* Afterward, all Ethernet cables on the switch had to be unplugged and replugged for the new connections to renegotiate.

**Learnings**: 
* With DNSmasq specifically, static reservations need to fall inside the dynamic DHCP range for DNS registration to work correctly – this is the opposite of I learned.

* The interaction between Dnsmasq and Unbound is still unclear, need further study.

* Replugging cables can be necessary for switch ports to renegotiate the connection.