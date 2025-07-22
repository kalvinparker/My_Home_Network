# OPNsense Home Network Guide

  ![Network Diagram](https://raw.githubusercontent.com/kalvinparker/My_Home_Network/main/Network%20Diagram.png)

A comprehensive, step-by-step guide to building a secure, segmented, and powerful home network using OPNsense as the core firewall and router.

## Table of Contents

- [Project Goal](#project-goal)
- [Key Features](#key-features)
- [Network Architecture Overview](#network-architecture-overview)
- [Hardware Bill of Materials (BOM)](#hardware-bill-of-materials-bom)
- [Guide Structure](#guide-structure)
- [Getting Started](#getting-started)
- [Contributing](#contributing)
- [License](#license)

## Project Goal

The objective of this project is to provide a detailed, end-to-end guide for moving beyond a standard consumer-grade router to a professional-grade network setup. This guide enables users to build a highly secure and flexible home network that can protect personal data, isolate insecure IoT devices, and provide robust remote access capabilities.

This project is for home lab enthusiasts, tech professionals, and anyone serious about taking control of their home network security.

## Key Features

This guide will walk you through the setup and configuration of a network with the following features:

- **Enterprise-Grade Firewall:** Using the powerful and open-source OPNsense firewall.
- **Network Segmentation:** Creation of multiple VLANs to isolate network traffic, including:
  - A **Trusted LAN** for personal computers and sensitive devices (e.g., NAS).
  - An **Untrusted VLAN** for IoT devices and guests, preventing them from accessing the trusted network.
- **Secure Remote Access:** Full setup and configuration of a **WireGuard VPN server** for secure access to your home network from anywhere.
- **Enhanced DNS:** Configuration of Unbound DNS for more control over your network's DNS resolution.
- **PKI and Trusted Certificates:** A detailed annex on creating your own Certificate Authority (CA) and replacing the default self-signed certificate for the OPNsense web UI.

## Network Architecture Overview

The network is designed around a central OPNsense firewall connected to a managed switch. This allows for the creation of separate VLANs, which are then broadcast by a VLAN-aware wireless access point.

- **Internet -> Modem -> OPNsense Firewall -> Managed Switch**
  - **VLAN 1 (Trusted LAN):** Connects to trusted wired and wireless devices.
  - **VLAN 10 (Untrusted LAN):** Connects to IoT and guest wireless devices.

This segmented design is a security best practice that significantly reduces the risk of a compromised IoT device affecting your primary computers and data.

## Hardware Bill of Materials (BOM)

The guide is based on a specific set of tested, cost-effective hardware, but the principles can be applied to similar components.

| Component   | Specification                               |
|-------------|---------------------------------------------|
| **CPU**     | Intel Alder Lake N100 (4 cores, 3.40 GHz)   |
| **RAM**     | Minimum 8GB DDR5                            |
| **Storage** | 500GB NVMe or SATA III SSD                  |
| **Switch**  | 5+ Port Managed Gigabit Switch (e.g., Netgear GS305E) |
| **AP**      | VLAN-aware Wireless Access Point (e.g., Grandstream)  |

For full details and purchase links, please see the **"Components"** section in the full PDF guide.

## Guide Structure

The included PDF, **"Build your Home Network with OPNsense.pdf"**, is a comprehensive 36-page guide that covers:
1.  **Planning Your Network:** Understanding the logical and physical layout.
2.  **Initial Setup:** Installing OPNsense from a bootable USB.
3.  **Basic Configuration:** Setting up system parameters and web UI access.
4.  **Interface & VLAN Configuration:** The core of network segmentation.
5.  **Firewall Rules:** Creating rules to control traffic between VLANs and the internet.
6.  **Services:** Configuring DHCP and DNS.
7.  **Annexes:** Detailed how-to guides for advanced topics like replacing SSL certificates and setting up a WireGuard VPN.

## Getting Started

1.  **Read the Plan:** Start by reading **Section 2: Planning Your Network** in the PDF to understand the architecture.
2.  **Procure Hardware:** Acquire the necessary components listed in the **Bill of Materials**.
3.  **Follow the Guide:** Begin with **Section 3: Initial Setup** and follow the step-by-step instructions.

It is highly recommended to read through the entire guide once before beginning the installation and configuration process.

## Contributing

Contributions, suggestions, and corrections are welcome! Please feel free to open an issue or submit a pull request if you find any errors or have ideas for improvements.

## License

This project is licensed under the CC BY-NC-SA 4.0 License - see the [LICENSE.md](LICENSE.md) file for details.
```

---
