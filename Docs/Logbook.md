# Logbook

This section documents changes, experiments, and fixes across my homelab. The goal is to keep track of what I did and why, so I can look back months or years later and understand past decisions — and see how the setup has evolved over time.

Each entry follows a simple structure: **Date**, **Topic**, **Initial State**, **End State** and **Learnings** (plus optional notes on affected components or next steps).

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

## 2026-09-05

**Topic**: Evaluating new hardware (Lenovo M920q Tiny) intended to eventually host all my services.

**Initial State**: I bought a refurbished tiny pc, which needs to be thoroughly tested before use.

**End State**: I ran several hardware tests (CPU, RAM & storage), checked the BIOS/UEFI for a supervisor password from the previous owner, and verified the TPM version.

**Learnings**: Learned how to properly benchmark hardware and interpret the resulting values. Initially assumed one program would handle both benchmarking and monitoring, learned that these are typically separate tools.

## 2026-09-06

**Topic**: Implementing local domain name resolution

**Initial State**: It is not working properly, yet.

✅ nslookup ap.lan.internal 192.168.10.1:53053 (DNSmasq)

❌ nslookup ap.lan.internal 192.168.10.130 (Adguard Home)

AdGuard Home doesn't know how to handle local domain name resolutions, resulting in an error.

**End State**: 
* I added an entry in DNS Settings > Upstream DNS servers for a domain-specific upstream for lan.internal: 
[/lan.intternal/]192.168.10.1:53053

* Under DNSmasq > General, I enabled 'Register ISC DHCP4 leases'.

* The local domain name resolution is working now.

**Learnings**:
* *nslookup host server* helps pinpoint exactly where a DNS chain breaks.
* AdGuard Home needs an explicit upstream rule to resolve internal domains – it doesn't do this automatically.
* Local resolution required both an AdGuard Home upstream rule and Dnsmasq's lease registration setting to work.

## 2026-09-10

**Topic**: Correcting the DNS configuration

**Initial State**: In the current setup, Unbound is just the backup DNS resolver and becomes active as soon as Adguard Home stops working.

The goal is to have AdGuard Home as the DNS proxy and Unbound as the resolver, instead of relying on a public DNS resolver as before.

Client (DNS Request) → Adguard Home (DNS-Proxy) → DNSmasq (local domain name resolution)/Unbound (DNS-Resolver) → Client (DNS Response)

**End State**:
* Modifying the DNS Upstream Server table - deleting all public DNS resolver and adding Unbound to the list. 
* Finally there are two entries:
    * [/lan.internal/]192.168.10.1:53053 (DNSmasq)
    * 192.168.10.1:53 (Unbound)

**Learnings**:
* Learned the difference between a DNS resolver and a DNS forwarder, and that Adguard Home is just a DNS proxy - nothing more, nothing less. Cloudflare and smiliar services are public DNS resolvers, while Unbound keeps resolution private.