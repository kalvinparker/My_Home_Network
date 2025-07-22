**Project Charter: Home Network Security Enhancement**

1.  **Project Title:** Home Network Security Enhancement Initiative.
2.  **Project Manager:** kalvinparker
3.  **Sponsor / Key Stakeholder:** My wife
4.  **Business Objective:** *(As defined above)* To enhance the security, control, and resilience of the home network.
5.  **Scope:**
    *   **In Scope:**
        *   Procurement and assembly of a dedicated physical appliance for OPNsense.
        *   Installation and configuration of OPNsense firewall software.
        *   Network segmentation into at least three VLANs: Management, Trusted LAN, and Untrusted IoT/Guest.
        *   Configuration of firewall rules, DHCP, and DNS services.
        *   Deployment and configuration of a managed switch and a wireless access point.
        *   Implementation of a secure remote access solution (WireGuard VPN).
        *   Replacement of the self-signed web UI certificate with a trusted certificate.
    *   **Out of Scope:**
        *   Virtualization of OPNsense. To keep intial costs down and test the firewall's capabilities.
        *   Advanced enterprise features not detailed in the guide (e.g., advanced intrusion prevention systems, high-availability clustering).
6.  **High-Level Requirements:**
    *   The solution must support gigabit speeds.
    *   The solution must allow for secure remote access from mobile and laptop clients.
    *   The solution must isolate insecure IoT devices from the primary trusted network.
    *   The solution must be manageable via a web interface from a dedicated management computer.
7.  **Key Deliverables:**
    *   A fully assembled and tested OPNsense firewall appliance.
    *   A configured managed switch and wireless access point.
    *   A fully functional, segmented home network.
    *   A secure and operational WireGuard VPN service.
8.  **Budget / Bill of Materials (BOM):**
    *   CPU: Intel Alder Lake N100
    *   RAM: 8GB DDR5
    *   OS Storage: 500GB NVMe SSD
    *   Switch: Netgear GS305E
    *   AP: Grandstream Wireless AP
    *   **Total Estimated Cost: £200**
9.  **High-Level Risks:**
    *   **Risk:** Incorrect configuration could lead to a total loss of internet connectivity.
        *   **Mitigation:** Perform initial setup during a low-usage period; have the old router ready to be plugged back in as a fallback.
    *   **Risk:** Supply chain issues could delay the arrival of hardware components.
        *   **Mitigation:** Order from reputable vendors with clear shipping timelines.
    *   **Risk:** A misconfigured firewall rule could inadvertently expose internal services to the internet.
        *   **Mitigation:** Follow the guide's "default deny" approach and test rules carefully.
