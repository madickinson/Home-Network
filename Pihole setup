### OPNsense 25.7 + Pi-hole + Unbound (with Dnsmasq DHCP)
Use Pi-hole for filtering while keeping Unbound on OPNsense as your recursive resolver. DHCP is provided by Dnsmasq (default in 25.7) and will hand out the Pi-hole IP to clients.

- Pi-hole IP: 192.168.1.49
- OPNsense LAN IP: 192.168.1.1 (adjust if yours differs)
- Goal path: Clients -> Pi-hole (192.168.1.49) -> Unbound on OPNsense (192.168.1.1) -> Internet

References:
- [Dnsmasq DNS & DHCP — OPNsense documentation](https://docs.opnsense.org/manual/dnsmasq.html)
- [DHCP — OPNsense documentation](https://docs.opnsense.org/manual/dhcp.html)

---

### 1) Prepare Unbound on OPNsense
#### Enable and allow Pi-hole to query Unbound
- Services -> Unbound DNS -> General
  - Enable Unbound DNS: checked
  - Network Interfaces: include LAN (and the interface Pi-hole can reach)
  - Outgoing Network Interfaces: WAN
  - DNSSEC: enabled
- Services -> Unbound DNS -> Access Lists
  - Add entry: 192.168.1.49/32, Action = Allow, Description = Pi-hole

#### Optional: Firewall allow for DNS to “This Firewall”
- See Step 4 for full rule details and recommended order.

Notes:
- Keep DNSSEC in Unbound and disable DNSSEC in Pi-hole to avoid double validation.

---

### 2) Configure Pi-hole to use OPNsense Unbound
Pi-hole Admin -> Settings -> DNS
- Upstream DNS Servers:
  - Custom 1 (IPv4): 192.168.1.1#53
- DNSSEC: disabled (Unbound validates)
- Conditional Forwarding (Pi-hole v6 format):
  - In the “Conditional forwarding” section, add one line:
    - true,192.168.1.0/24,192.168.1.1,midnightdojo.org
  - Field meanings: enabled, local-CIDR, router-IP, local-domain
  - If you prefer to omit the domain: true,192.168.1.0/24,192.168.1.1

Where to confirm your local domain on OPNsense:
- System -> Settings -> General -> Domain
- Or Services -> Dnsmasq DNS & DHCP -> DHCP default domain / DHCP local domain

Best practice for new setups is using home.arpa or a subdomain like lan.example.org, but midnightdojo.org is OK if that’s your active LAN domain.

---

### 3) Make Dnsmasq DHCP hand out Pi-hole as the DNS server
OPNsense: Services -> Dnsmasq DNS & DHCP -> DHCP options
- Click “+ Add”, then fill:
  - Type: Set
  - Option: dns-server [6]
  - Option6: None
  - Interface: LAN (repeat for each VLAN)
  - Tag: leave empty
  - Value: 192.168.1.49
  - Force: unchecked
  - Description: DNS -> Pi-hole 192.168.1.49
- Save, then Apply.

Prerequisites:
- Ensure you have a DHCPv4 range defined under Services -> Dnsmasq DNS & DHCP -> DHCP ranges for each interface you serve.

IPv6 (optional):
- If you run DHCPv6 via Dnsmasq, add another entry:
  - Type: Set, Option6: dns-server [23], Interface: LAN, Value: <Pi-hole IPv6 address>

---

### 4) Firewall rules on LAN for DNS to Unbound
OPNsense: Firewall -> Rules -> LAN -> Add
- Action: Pass
- Interface: LAN
- Direction: in
- TCP/IP Version: IPv4 (or IPv4+IPv6 if applicable)
- Protocol: TCP/UDP
- Source: Single host or alias (192.168.1.49)
- Destination: This Firewall
- Destination port: DNS (53)
- Description: Allow Pi-hole (192.168.1.49) -> Unbound (DNS 53)
- Log: optional during testing
- Save and Apply.

Rule order guidance:
- If you just want it to work: Your existing “Default allow LAN to any” already permits DNS; the specific rule can sit below and will rarely match.
- If you want to enforce that only Pi-hole can query Unbound:
  1) Place the “Allow Pi-hole -> This Firewall DNS 53” rule at the top.
  2) Directly under it, add a Block rule:
     - Action: Block
     - Interface: LAN
     - Protocol: TCP/UDP
     - Source: LAN net
     - Destination: This Firewall
     - Destination port: DNS (53)
     - Description: Block LAN -> This Firewall DNS 53 except Pi-hole
     - Log: enabled
  3) Keep your default allow rules below these two.

Tip: Create an alias PIHOLE_HOST with 192.168.1.49 and use it in the rules for readability.

---

### 5) Optional: Force all client DNS to Pi-hole (NAT redirect)
If you want to prevent clients from bypassing Pi-hole:

Firewall -> NAT -> Port Forward (LAN)
- Interface: LAN
- Protocol: TCP/UDP
- Source: LAN net
- Destination: any
- Destination port: 53
- Redirect target IP: 192.168.1.49
- Redirect target port: 53
- Description: Force LAN DNS to Pi-hole
- Place above other LAN NAT rules, Save and Apply.

Add rule exemptions (before this redirect) for:
- Source: 192.168.1.49 (Pi-hole)
- Source: This Firewall
So their own DNS flows are not redirected.

---

### 6) Order of operations and verification
- Apply Unbound, Dnsmasq, DHCP options, Firewall rules.
- Restart Pi-hole DNS or reboot the Pi-hole host if needed.
- Renew a client lease (disable/enable NIC, ipconfig /renew, or dhclient -r && dhclient).

Client verification:
- DNS server should be 192.168.1.49.
- Queries should appear in Pi-hole Query Log.

Pi-hole/Unbound path tests (from Pi-hole host):
- dig @192.168.1.1 example.com +tcp
- dig @192.168.1.1 example.com +notcp
- dig -x 192.168.1.50 @192.168.1.49
- dig host-in-lan.midnightdojo.org @192.168.1.49

---

### Common pitfalls and fixes
- Double DNSSEC: Disable DNSSEC in Pi-hole; keep it enabled in Unbound.
- Unbound “denied” for Pi-hole: Add 192.168.1.49 to Unbound Access Lists and ensure the LAN Pass rule to port 53 exists.
- Local names not shown in Pi-hole: Ensure Conditional Forwarding is set (Pi-hole v6 line format) and OPNsense’s domain matches.
- Dnsmasq DNS feature: Keep Unbound as the resolver on OPNsense; don’t also enable Dnsmasq’s DNS forwarder role when using Unbound.
- Multiple VLANs: Add DHCP Option 6 per interface and one Conditional Forwarding line per network in Pi-hole v6.

---

### Topology options
- Recommended: Clients -> Pi-hole (192.168.1.49) -> OPNsense Unbound (192.168.1.1)
- Alternate: Run Unbound on the Pi-hole host and have Pi-hole use 127.0.0.1; then Dnsmasq only distributes the Pi-hole address. This bypasses OPNsense Unbound.

---

### Appendix: Quick panels and paths
- Unbound Access List: Services -> Unbound DNS -> Access Lists -> Add 192.168.1.49/32 Allow
- Pi-hole v6 Conditional Forwarding line:
  - true,192.168.1.0/24,192.168.1.1,midnightdojo.org
- Dnsmasq DHCP Option 6 (IPv4):
  - Services -> Dnsmasq DNS & DHCP -> DHCP options -> Add
  - Type Set, Option dns-server [6], Interface LAN, Value 192.168.1.49

Save this file to your repo and adjust only the IPs and domain if your environment changes.
