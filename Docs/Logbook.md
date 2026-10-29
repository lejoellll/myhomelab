# Logbook

This section documents changes, experiments, and fixes across my homelab. The goal is to keep track of what I did and why, so I can look back months or years later and understand past decisions and see how the setup has evolved over time.

Each entry follows a simple structure: **Date**, **Topic**, **Initial State**, **End State** and **Learnings** (plus optional notes on affected components or next steps).

## 2026-10-28 - Setting up Telekom's fibre connection

**Initial State:** No fibre connection, just broadband internet was in use.

**End State:**
* Bound VLAN07 to the physical WAN (igb0) interface under **Interfaces > Devices > VLAN**
* Created a Point-to-Point device under **Interfaces > Devices > Point-to-Point**
    * Selected VLAN07 as the link Interface
    * Entered the login credentials
* Changed the device to the PPPoE VLAN07 and set both the IPv4 and IPv6 configuration types to PPPoE under **Interfaces > Assignment** 

**Learning:** Telekom’s fibre connection requires VLAN07 tagged traffic to work properly.

## 2026-10-26 - Create a root CA and server certificates signed by the new root CA

**Initial State:** All web GUIs were unencrypted. To improve security, I implemented TLS encryption by creating a root CA and issuing server certificates signed by it.

**End State:**
- Created a root CA under **OPNsense > System > Trust > Authorities**
- Created and signed server certificates under **OPNsense > System > Trust > Certificates**
- **Dashboard**: Imported the root certificate into the system trust store:
  - Copied the certificate to `/usr/local/share/ca-certificates/`
  - Ran `sudo update-ca-certificates`
  - Imported the root certificate into Chromium’s own trust store:
    ```bash
    certutil -d "sql:$HOME/.local/share/pki/nssdb" \
        -A -t "C,," \
        -n "Root CA 01" \
        -i /usr/local/share/ca-certificates/root-ca01.crt
    ```
- **Iphone**: Imported the root certificate into my smartphones iOS trust store.
- **Home Assistant**: Added the server certificate and its private key via the File Editor add-on, then configured them under **Settings > System > Network** to enable HTTPS.

**Learning:**
- Chromium uses its own trust store. Importing the root CA into the system trust store is not sufficient for Chromium, it must be added there separately.
- Apple iOS imposes restrictions on root certificates:
  - Maximum validity of 825 days
  - The Common Name must not be empty
  - The certificate must have full trust permissions

## 2026-10-21 - Configure threat intelligence by Q-Feeds in order to block known malicious IP addresses.

**Initial State**: Initially I had Adguard Home to block ads and trackers based on DNS. With Q-Feeds' threat intelligence I want to expand the protection based on IP addresses.

**End State**: 

* Registered at https://qfeeds.com/ and created an API-key in order to use this service.

* Installed the plugin **os-q-feeds-connector** in OPNsense.

* A new menue appeared: 

    * Under Security > Q-Feeds Connect, I filled in the fresh API-token.

    * The plugin then fetched the threat intelligence and created firewall aliases.

* Created a firewall rule:

    **Interface**:  All VLANs\
    **Quick**:      Enabled\
    **Action**:     Block\
    **Direction**:  In\
    **Version**:    IPv4/IPv6\
    **Protocol**:   TCP/UDP\
    **Source**:     Any\
    **Source Port**: Any\
    **Destination**: __qfeeds_malware_ip\
    **Dest. Port**: Any\
    **State Type**: Keep state\
    **Log**: Enabled

**Learning**: Q-Feeds extends DNS-based filtering with IP-based threat intelligence blocking. The plugin loads known malware IP addresses into the alias, which is used as a block target via a firewall rule.

## 2026-10-19 - Redirect all outgoing NTP-Requests to the NTP server on OPNsense.

**Initial State**: There was no general rule for time synchronization (NTP; UDP 123), only rules for individual devices.

**End State**: 
* Created a destination NAT rule to redirect all NTP requests to OPNsense on UDP port 123.

    **Interface**:  Any\
    **Version**:    IPv4\
    **Protocol**:   UDP\
    **Source**:     Local_Net\
    **Source Port**: Any\
    **Destination**: !Local_Net\
    **Dest. Port**: NTP(123)\
    **Redirect Target IP**:   This Firewall\
    **Firewall rule**:  Manual

* Created a corresponding firewall rule to allow inter-VLAN traffic.

    **Interface**:  All VLANs\
    **Quick**:      Enabled\
    **Action**:     Pass\
    **Direction**:  In\
    **Version**:    IPv4\
    **Protocol**:   UDP\
    **Source**:     Local_Net\
    **Source Port**: Any\
    **Destination**: This Firewall\
    **Dest. Port**: NTP(123)\
    **State Type**: Keep state

* Tested the new settings with my laptop by sending an NTP request to a Cloudflare NTP server while capturing packets on the WAN and VLAN10 interfaces: `w32tm /stripchart /computer:162.159.200.1 /dataonly /samples:5`

    * Entries appeared only on the VLAN10 interface; there were no entries on the WAN interface.

    * Additionally, the response arrived in less than 100 µs, which indicates that OPNsense was acting as the NTP server.

**Learnings**: Destination NAT can redirect traffic to a single address—in this case, OPNsense acting as the NTP server. For this to work correctly, a corresponding firewall rule is required to allow traffic from the local subnets or devices to the firewall on UDP port 123. The packet capture jobs are very useful for testing settings like this.

## 2026-10-15 - Finish VLAN Segmentation and create corresponding firewall rules.

**Initial State**: PS5 and Proxmox weren't assigned to a dedicated VLAN.

**End State**: 
* Created a separate GAMING VLAN for the PS5. 

* Created a Linux VLAN interface (vmbr0.30) and configured its IPv4 and gateway.

* Removed IPv4 address and gateway from the Linux Bridge settings. 

* Proxmox is now part of the MGMT VLAN.

**Learnings**: 
* I learned that a Linux Bridge (vmbr0) in Proxmox acts like a virtual switch for  virtual machines and connect them to the physical interface. 

* I created a Linux VLAN interface (vmbr0.30) to assign the Proxmox host itself to the MGMT VLAN.

## 2026-10-04 - Adapt the Pi5's window settings, migrate into the network and run the first test.

**Initial State**: Vertical screen orientation, login screen and the Raspberry Pi doesn't start directly to the dashboard.

**End State**:
* Adapted the screen orientation from vertical to horizontal via gui by clicking through the menue like below:
**Preferences > Control Centre > Screens > DSI-2 > Orientation > Right**

* Activated the auto login:
**Preferences > Control Centre > System > Desktop Auto Login > Enable**

* Extended the autostart file to put the Pi5 into the kiosk mode and start directly with the dashboard page.
```bash
        nano .config/labwc/autostart
        /usr/bin/chromium \
        --password-store=basic \
        --kiosk \
        --noerrdialogs \
        --disable-infobars \
        --no-first-run \
        --enable-features=OverlayScrollbar \
        --start-maximized \
        'https://homeassistant.local:8123/dashboard-panel/ubersicht' &
```
**Learnings**: 
- Chromium’s `--kiosk` mode starts the browser fullscreen without UI elements, making it suitable for a dashboard display.
- Auto-login is required so the dashboard starts automatically without manual login.

## 2026-10-03 - Install RaspberryPi OS on the RBPi5, connect the Pi to the touch display and assemble the case.

**Initial State**: Raspberry Pi 5 was unconfigured; no operating system, display integration, or enclosure.

**End State**: Raspberry Pi OS was installed on the Pi 5, the official touch display was connected and working, and the case was assembled.

**Learnings**: -

## 2026-09-27/28 - Implement the IoT VLAN and assign the IoT devices to it. Create and test the required firewall rules.

**Initial State**: All IoT devices were connected to the trusted VLAN. This was a security risk because compromised or misbehaving smart devices could directly access trusted services and clients. Therefore, I wanted to move all smart devices into a separate, highly restricted IoT VLAN.

**End State**:
* Created a separate, VLAN-tagged Wi-Fi SSID for IoT devices.
* Added the IoT VLAN tag to the trunk connection between the access point and the switch.
* Added the IoT VLAN tag to the trunk connection between the switch and OPNsense.
* The VLAN interface and DHCP range for the IoT VLAN had already been created, assigned, and enabled on 2026-09-13.
* Enabled the DHCP server for the IoT VLAN.
* Created entries for IP reservations for the IoT devices.
* Created a firewall rule allowing Home Assistant in the Services VLAN to connect to the IoT devices on TCP port 6053 for ESPHome.
* Connected all ESP32 microcontrollers to the new IoT Wi-Fi.
* Verified that Home Assistant can reach the ESPHome devices across the VLAN boundary.

**Learnings**:
* For ESPHome, Home Assistant initiates the connection to the ESP32 device on TCP port 6053. Therefore, the firewall rule is required on the Home Assistant / Services VLAN interface, from Home Assistant to the IoT device.


## 2026-09-20 - Stepwise VLAN segmentation and migration of Home Assistant OS to a Proxmox VM.

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

    * SSH into Promox and imported the downloaded image into the VM.\
    `qm importdisk 101 /tmp/haos_ova-18.3.qcow2.xz local-lvm`

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

## 2026-09-16 - Creating my first Linux container, installing AdGuard Home, and assigning it to VLAN20.

**Initial State**: I'd like to move my services (Adguard Home & Home Assistant OS), which are currently running on my Raspberry Pi devices, to the server. That way, I can use my Raspberry Pi devices for other purposes, such as smart display or as jump server.

**End State**:
* Created a LXC with the following attributes:

    * cores:    1
    * memory:   512
    * swap:     512
    * hostname: adguard
    * os:       debian-12-standard_12.12-1_amd64.tar.zst
    * net0:     name=eth0, bridge=vmbr0, tag=<VLAN_ID>, firewall=1, ip=<ADGUARD_IP>/24, gw=<GATEWAY_IP>, ip6=dhcp
    * unprivileged: 1

* Ran AdGuard Home's official install script:

    * curl -sSL https://raw.githubusercontent.com/AdguardTeam/AdGuardHome/master/scripts/install.sh | sh -s -- -v

* After running the script the web GUI is reachable for finishing the installation.

* Created five firewall rules for Adguard Home:

    * AdGuard Home → OPNsense DNS (TCP/UDP 53 and 53053)
    * AdGuard Home → Internet for blocklist updates (TCP 443)
    * AdGuard Home → Internet (TCP 80)
    * Trusted → AdGuard Home web GUI (TCP 80)
    * Trusted → AdGuard Home DNS (TCP/UDP 53)

**Learnings**: Without any firewall rules the whole traffic is blocked by the implicit deny rule. 

## 2026-09-14 - Installing proxmox ve on the server machine.

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

## 2026-09-13 - Preparing OPNsense for VLAN segmentation.

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

## 2026-09-10 - Correcting the DNS configuration.

**Initial State**: In the current setup, Unbound is just the backup DNS resolver and becomes active as soon as Adguard Home stops working.

The goal is to have AdGuard Home as the DNS proxy and Unbound as the resolver, instead of relying on a public DNS resolver as before.

Client (DNS Request) → Adguard Home (DNS-Proxy) → DNSmasq (local domain name resolution)/Unbound (DNS-Resolver) → Client (DNS Response)

**End State**:
* Modified the DNS Upstream Server table - deleted all public DNS resolver and added Unbound to the list. 

* Finally there are two entries:

    * [/lan.internal/]<DNSmasq_IP>:<DNSmasq_PORT>
    * <UNBOUND_IP>:53

**Learnings**: Learned the difference between a DNS resolver and a DNS forwarder, and that Adguard Home is just a DNS proxy - nothing more, nothing less. Cloudflare and smiliar services are public DNS resolvers, while Unbound keeps resolution private.

## 2026-09-06 - Implementing local domain name resolution.

**Initial State**: It is not working properly, yet.

✅ nslookup ap.lan.internal <DNSmasq_IP>:<DNSmasq_PORT>

❌ nslookup ap.lan.internal <ADGUARDHOME_IP>

AdGuard Home doesn't know how to handle local domain name resolutions, resulting in an error.

**End State**: 
* Added an entry in DNS Settings > Upstream DNS servers for a domain-specific upstream for lan.internal:  
[/lan.internal/]<DNSmasq_IP>:<DNSmasq_PORT>

* Under DNSmasq > General, I enabled 'Register ISC DHCP4 leases'.

* The local domain name resolution is working now.

**Learnings**:
* *nslookup host server* helps pinpoint exactly where a DNS chain breaks.

* AdGuard Home needs an explicit upstream rule to resolve internal domains – it doesn't do this automatically.

* Local resolution required both an AdGuard Home upstream rule and Dnsmasq's lease registration setting to work.

## 2026-09-05 - Evaluating new hardware (Lenovo M920q Tiny) intended to eventually host all my services.

**Initial State**: I bought a refurbished tiny pc, which needs to be thoroughly tested before use.

**End State**: I ran several hardware tests (CPU, RAM & storage), checked the BIOS/UEFI for a supervisor password from the previous owner, and verified the TPM version.

**Learnings**: Learned how to properly benchmark hardware and interpret the resulting values. Initially assumed one program would handle both benchmarking and monitoring, learned that these are typically separate tools.

## 2026-08-28 - Preparing OPNsense for double NAT (Fritzbox as upstream router, OPNsense behind it).

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
