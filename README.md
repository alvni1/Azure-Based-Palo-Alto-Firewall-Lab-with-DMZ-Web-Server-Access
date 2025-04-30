<h1>Azure Based Palo-Alto Firewall Lab with DMZ Web Server Access</h1>

<h2>Description</h2>
 This hands-on project simulated an enterprise network security environment using Palo Alto Networks’ Next-Generation Firewall (NGFW) hosted in Microsoft Azure. The goal was to securely publish a DMZ-hosted web server to the internet using NAT, routing, and zone-based security policies.

- <b>Deployed and configured Palo Alto NGFW in Azure with multiple interfaces. </b>

- <b>Created Azure subnets: Trust, Untrust, and DMZ.</b> 

- <b>Attached network interfaces to the Palo Alto VM (eth1/1 - Untrust, eth1/2 - DMZ).</b> 

- <b>Configured static routes and NAT policies (DNAT/SNAT) to enable external HTTP access to an internal web server.</b> 

- <b>Implemented security policies to permit HTTPS management access and HTTP access to the web server from the internet.</b>

- <b>Deployed and configured an Ubuntu-based Apache web server in the DMZ.</b>

- <b>Verified end-to-end connectivity from public IP to internal DMZ resource.</b>


<h2>Things Learned/Troubleshooting</h2>

Issue 1: DNAT Rule Mismatch (Incorrect Destination IP)
- <b>Cause: The DNAT rule in the firewall initially pointed to 172.16.3.6, but the web server's private IP later changed to 172.16.1.7 after being moved to a different subnet.</b> 
- <b>Fix: The DNAT rule was updated to reflect the new IP (172.16.1.7), ensuring the destination translation pointed to the correct active address of the DMZ web server.</b> 

Issue 2:  No Traffic Log Entries (Session Not Reaching Firewall)
- <b>Cause: HTTP traffic from the internet was being sent to the management interface (eth0) instead of the untrust interface (eth1), which caused it to bypass the NAT rule entirely.</b> 
- <b>Fix: A new public IP address was created and associated with the ethernet1/1 interface, then used as the destination in the NAT policy. This routed traffic through the correct interface.</b> 

Issue 3: Empty Session Browser (NAT Rule Not Triggering)
- <b>Cause: The NAT rule’s "Original Packet" destination IP didn’t match the public IP assigned to the untrust interface, so the firewall never matched or translated incoming requests.</b> 
- <b>Fix: The NAT rule was updated with the correct public IP address associated with the firewall’s untrust NIC.</b> 

Issue 4: No Public Access Despite Correct NAT (NSG Blocking Port 80)
- <b>Cause: The Network Security Group (NSG) assigned to the firewall’s untrust NIC was missing an inbound rule for TCP port 80.</b> 
- <b>Fix: A new NSG (basicNsgUntrust) was created and assigned to the NIC, with explicit allow rules for inbound HTTP (port 80) and HTTPS (port 443).</b> 

Issue 5: Traffic Dropped Silently (IP Forwarding Disabled)
- <b>Cause: Azure NIC for the Palo Alto firewall had IP forwarding disabled by default. Azure requires IP forwarding for traffic to flow through a VM acting as a network virtual appliance.</b> 
- <b>Fix: IP forwarding was enabled on the firewall's untrust NIC (ethernet1/1), allowing the firewall to route traffic to internal VMs.</b> 

Issue 6: Initial Zone Mismatch in Security Policies
- <b>Cause: The destination zone in security rules didn’t align with the interface where the DMZ web server was reachable (untrust instead of dmz, after subnet move).</b> 
- <b>Fix: The security rule was updated to reflect the current zone associated with the web server’s subnet.</b>

Issue 7: No NSG on Critical Interfaces
- <b>Cause: The untrust interface (eth1) of the firewall had no NSG assigned, allowing Azure to apply its default implicit deny rules.</b> 
- <b>Fix: An NSG was created and explicitly attached to the firewall's untrust NIC, with allow rules for required ports.</b> 


<h2>Languages and Utilities Used</h2>

- <b>CLI troubleshooting (ping, session viewer, interface/IP checks)</b> 

<h2>Environments Used </h2>

- <b>Palo Alto NGFW (PAN-OS Web UI + CLI)</b>

- <b>Azure Virtual Network (vNET), NICs, and NSGs</b>

- <b>Apache2 Web Server on Ubuntu</b>

<h2>Program walk-through:</h2>

1. Azure-Level Setup
- <b>Deployed VMs and attached NICs to proper subnets (DMZ and Untrust).</b> 
- <b>Created and attached Public IP (paloalto-untrust-pip) to ethernet1/1</b>
- <b>Enabled IP forwarding on firewall NICs.</b>
- <b>Created/Attached Network Security Groups (NSGs) with rules to allow:
   Inbound HTTP (port 80),
   Inbound HTTPS (port 443),
   Inbound ICMP (for testing)

2. Palo Alto Firewall Configuration

- <b>Configured Layer 3 interfaces with zones (trust, dmz, untrust).</b>
- <b>Assigned DHCP Client to interfaces to pull dynamic IPs.</b>
- <b>Created Virtual Router with static default route (0.0.0.0/0) pointing to Azure’s next hop.</b>
- <b>Set up Security Policies: Untrust → DMZ (web-browsing & ssl) — Allow, Untrust → Untrust (for health probes) — Allow</b>
- <b>Configured NAT Policies: DNAT for inbound HTTP:
   Source Zone: untrust,
   Destination Address: 172.190.202.26 (Public IP),
   Translated Address: 172.16.1.7 (Private interface of FW),
   Translated Destination: 172.16.3.6 (Web server IP),

   SNAT for outbound web access (dynamic-ip-and-port on ethernet1/1)</b>

3. DMZ Web Server
   
- <b>Ubuntu Server VM in DMZ subnet.</b>
- <b>Installed and enabled Apache2 HTTP server.</b>

4. Testing Process and Validation

- <b>Verified Apache via:
-sudo systemctl status apache2
-curl http://localhost</b>

- <b>Verified that https://52.170.91.42 (Management IP) worked for Palo Alto GUI access.</b>

- <b>Attempted to reach http://52.170.91.42 and later http://172.190.202.26 for web server</b>

- <b>Performed CLI testing via:ping source 172.16.1.7 host 172.16.3.6,
test security-policy-match ...,
show session all</b>



