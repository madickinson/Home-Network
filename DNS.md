There are a couple of common ways to integrate Pi-hole with OPNsense using Dnsmasq for DHCP and optionally Unbound for recursive lookups.
The simplest and most recommended method to ensure clients use Pi-hole is to configure Dnsmasq on OPNsense to advertise the Pi-hole's IP address as the DNS server for clients.
Method 1: OPNsense Dnsmasq for DHCP, Pi-hole for DNS
This setup ensures that all devices receiving a DHCP lease from OPNsense will use the Pi-hole as their primary DNS server.
 * Set a Static IP for Pi-hole:
   * Ensure your Pi-hole device has a static IP address, preferably set via a DHCP Static Mapping in OPNsense's Dnsmasq service.
   * Services \rightarrow Dnsmasq DNS & DHCP \rightarrow DHCP Leases tab, or DHCP Static Mappings tab.
 * Configure Dnsmasq to Advertise Pi-hole DNS:
   * In OPNsense, navigate to Services \rightarrow Dnsmasq DNS & DHCP \rightarrow DHCP Options.
   * Click the + button to add a new option.
     * Option: Select dns-server [6].
     * Value: Enter the static IP address of your Pi-hole. You can optionally add a secondary DNS server (like a public one) separated by a comma, but this can lead to inconsistent ad-blocking as clients may sometimes bypass the Pi-hole.
     * Interface: Select the LAN interface(s) where clients should use Pi-hole.
   * Click Save and Apply Changes.
 * Configure Pi-hole Upstream DNS:
   * On the Pi-hole's web interface, go to Settings \rightarrow DNS.
   * For the Upstream DNS servers, you have two primary choices:
     * Use OPNsense Unbound (Recommended for Recursive DNS): Select Custom and enter the OPNsense firewall's IP address (e.g., 192.168.1.1#53) if you want Unbound on OPNsense to handle recursive resolution.
     * Use External DNS: Select public DNS servers (like Cloudflare or Quad9).
 * Optional: Force All DNS Traffic (Prevent Bypassing):
   * To prevent devices from bypassing Pi-hole by using hardcoded DNS servers (like 8.8.8.8), you can create a Firewall Rule on your LAN interface to redirect all outbound traffic on port 53 (TCP/UDP) destined for any IP (except the Pi-hole itself) to the Pi-hole's IP address.
Method 2: Pi-hole as Upstream for OPNsense Dnsmasq/Unbound (Less Client Visibility)
This method simplifies the client configuration but results in all client requests being logged in Pi-hole as coming from the OPNsense IP address, losing per-client visibility.
 * Configure Pi-hole as OPNsense's Primary DNS:
   * In OPNsense, go to System \rightarrow Settings \rightarrow General.
   * Under Networking, set the DNS Servers to the static IP address of your Pi-hole.
   * Uncheck Allow DNS server list to be overridden by DHCP/PPP on WAN.
   * Click Save.
 * Configure Client DNS Delivery:
   * Clients should be set (via Dnsmasq) to use the OPNsense firewall's IP as their DNS server (this is often the default behavior when no custom DNS option is set in Dnsmasq).
 * Configure Pi-hole Upstream DNS:
   * On Pi-hole, set the Upstream DNS to the OPNsense Unbound IP/Port (e.g., 192.168.1.1#53) if you want Unbound to perform the recursive lookups, or use external DNS servers.
This video demonstrates a setup on OPNsense that installs Pi-hole on Proxmox and configures OPNsense Unbound as the upstream DNS provider for the Pi-hole. Installing Pi-hole on Proxmox and using OPNsense Unbound DNS Upstream

