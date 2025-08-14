## The Definitive Guide to Reverse-Proxying Internal Services with OPNsense & HAProxy

This guide provides a complete, step-by-step process for securely exposing any internal service (in a container or on bare metal) to the internet using OPNsense's HAProxy plugin, Let's Encrypt for trusted SSL, and achieving an A+ security rating.

### Core Concept: The Flow of Traffic

Understanding the path of a request is key to troubleshooting:

1.  **Public Path:** User's Browser/App -> Public DNS -> Your Public IP -> OPNsense WAN Firewall -> **HAProxy Frontend**
2.  **Internal Path:** **HAProxy Backend** -> Your LAN -> Your Service (e.g., Docker Host) -> The Application

HAProxy acts as the secure gatekeeper and traffic director.

### Prerequisites

1.  A functioning OPNsense router with an internet connection.
2.  An internal service running on a known static IP and HTTP port (e.g., `192.168.1.6:8080`).
3.  A domain name you control, preferably with a provider supported by the `os-acme-client` plugin's DNS challenge (e.g., DuckDNS, Cloudflare, Namecheap).

---

## Part 1: The Foundation - DNS and Service Preparation

1.  **Configure Your Service:**
    *   Ensure your service is running on **plain HTTP**. SSL/TLS will be handled by HAProxy.
    *   If your service has a `DOMAIN` or `SERVER_NAME` variable, set it to your full public URL (e.g., `https://myservice.yourdomain.com`).
    *   If using Docker, ensure the container's port is published to the host. **Crucially, do not bind it to `127.0.0.1` (localhost).** Bind it to `0.0.0.0` or leave the host IP blank in Portainer to make it accessible on your LAN.

2.  **Configure Dynamic DNS:**
    *   Log into your DNS provider (e.g., DuckDNS) and create the subdomain you will use (e.g., `myservice`).
    *   In OPNsense, go to **Services -> Dynamic DNS** and configure your account to keep your public IP updated for that hostname.

---

## Part 2: The Certificate - Let's Encrypt (ACME Client)

1.  **Add DNS Challenge Type:** Go to **Services -> ACME Client -> Challenge Types**. Click **Add** and configure a challenge for your DNS provider, providing your API key/token.
2.  **Create Certificate:** Go to **Services -> ACME Client -> Certificates**. Click **Add**.
    *   **Common Name:** Enter your full public hostname (e.g., `myservice.yourdomain.com`).
    *   **ACME Account:** Select your Let's Encrypt account.
    *   **Challenge Type:** Select the DNS challenge you just created.
    *   **Automations:** Add an automation to `Restart HAProxy`.
3.  Click **Save**, then find the new certificate in the list and click the **"Issue or Renew"** icon. Verify it succeeds in the log files.

---

## Part 3: The Engine - HAProxy Configuration

We build the components from the back to the front.

1.  **Create the Real Server:**
    *   Go to **Services -> HAProxy -> Real Servers**. Click **Add**.
    *   **Name:** `MyService_Server`.
    *   **Address:** The internal IP of your service (`192.168.1.6`).
    *   **Port:** The HTTP port of your service (`8080`).
    *   **SSL:** Must be **UNCHECKED**.
2.  **Create the Backend Pool:**
    *   Go to **Virtual Services -> Backend Pools**. Click **Add**.
    *   **Name:** `MyService_Backend`.
    *   **Servers:** Select `MyService_Server`.
    *   **Health Checking:** Leave disabled for now. We will revisit this in troubleshooting.
3.  **Create the Condition:**
    *   Go to **Rules & Checks -> Conditions**. Click **Add**.
    *   **Name:** `Condition_is_MyService`.
    *   **Condition type:** `Host matches`.
    *   **Value:** `myservice.yourdomain.com`.
4.  **Create the Rule:**
    *   Go to **Rules & Checks -> Rules**. Click **Add**.
    *   **Name:** `Rule_use_MyService_Backend`.
    *   **Test type:** `If` -> `Condition_is_MyService`.
    *   **Execute function:** `Use backend` -> `MyService_Backend`.
5.  **Create the Public Service (Frontend):**
    *   Go to **Virtual Services -> Public Services**. Click **Add**.
    *   **Name:** `HTTPS_MyService_Frontend`.
    *   **Listen Addresses:** `0.0.0.0:8443` (Using a high port like `8443` is recommended to avoid ISP blocks on port 443).
    *   **Type:** `http / https (SSL offloading)`.
    *   **SSL Offloading -> Certificates:** Select your `myservice.yourdomain.com` certificate.
    *   **Advanced SSL settings (for A+ Rating):**
        *   **Cipher List:** `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES256-GCM-SHA384`
        *   **Cipher Suites:** `TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256`
    *   **Rules:** Add the `Rule_use_MyService_Backend`.
6.  **Save and Apply** all HAProxy changes.

---

## Part 4: The Gates - Firewall Rules

1.  **WAN Rule:** Go to **Firewall -> Rules -> WAN**. Create a `Pass` rule for `TCP` traffic from `any` source to destination `WAN address` on destination port `8443`.
2.  **LAN Rule:** Go to **Firewall -> Rules -> LAN**. Create a `Pass` rule for `TCP` traffic from `LAN net` source to destination `This Firewall (self)` on destination port `8443`.
3.  **Apply** all firewall changes.

---

## Part 5: The Internal Map - Split DNS

This ensures seamless access from inside your network.

1.  Go to **Services -> Unbound DNS -> Overrides**.
2.  Create a new **Host Override**:
    *   **Host:** `myservice`
    *   **Domain:** `yourdomain.com`
    *   **IP Address:** Your OPNsense LAN IP (e.g., `192.168.1.1`).
3.  **Save** and **Apply**.
4.  Go to **Services -> DHCPv4 -> [LAN]** and ensure the **DNS servers** field contains your OPNsense LAN IP (`192.168.1.1`).

---

## Part 6: Systematic Verification

1.  **Check Public DNS:** Use `dnschecker.org` to confirm your domain points to your public IP.
2.  **Check Public Port:** Use `canyouseeme.org` to confirm port `8443` is open.
3.  **Check Internal DNS:** On a local computer, open a terminal and run `nslookup myservice.yourdomain.com`. The result **must** be your OPNsense LAN IP (`192.168.1.1`). If not, flush your DNS (`ipconfig /flushdns`) and check your computer's network settings.
4.  **Final Test:** Navigate to `https://myservice.yourdomain.com:8443`. The service should load.

---

## Lessons Learned & Troubleshooting Guide

This is the most critical section, compiled from real-world troubleshooting.

| Symptom / Error | Probable Cause | Solution & Key Lesson |
| :--- | :--- | :--- |
| **Connection Timed Out (External)** | ISP is blocking the port, or the WAN firewall rule is wrong. | **Lesson:** Don't assume standard ports are open. **Solution:** Change to a high port like `8443` in both the HAProxy Frontend and the WAN Firewall Rule. |
| **Connection Timed Out (Internal)** | The LAN firewall is blocking access to the OPNsense router itself. | **Lesson:** Default "allow" rules may not exist in hardened setups. **Solution:** Create a specific LAN rule allowing `LAN net` to connect to `This Firewall (self)` on the HAProxy port (`8443`). |
| **Wrong SSL Certificate is Served** | The OPNsense Web GUI service is "stealing" the port from HAProxy. | **Lesson:** Services can conflict even on different interfaces. **Solution:** Go to **System -> Settings -> Administration** and set **Listen Interfaces** to `LAN` only. This forces the GUI off the WAN interface. |
| **`503 Service Unavailable`** | This error is from HAProxy. It means it received the request but believes the backend is down. | **Lesson:** This isolates the problem to the HAProxy -> Backend connection. There are two main causes:<br>1. **Health Check Failure:** HAProxy's check fails even if the server is up. **Solution:** Create a simple **Health Monitor** of type `TCP` and apply it to the Backend Pool. If that fails, disable health checks entirely for that backend.<br>2. **Connectivity Failure:** HAProxy actually can't connect. **Solution:** Use the OPNsense **Port Probe** diagnostic to test the connection to your service's IP:Port. This bypasses HAProxy and tests the raw network path. |
| **Port Probe: `Connection Refused`** | The Docker host's OS received the request but no application was listening on that port. | **Lesson:** The "localhost trap" is a common Docker mistake. **Solution:** The container's port mapping is bound to `127.0.0.1`. Edit the container and change the host port mapping from `127.0.0.1:8080` to just `8080` to bind it to all interfaces. |
| **Port Probe: `Timed Out`** | The Docker host's OS did not respond at all. | **Lesson:** The operating system's own firewall is blocking the connection. **Solution:** Log into the Docker host machine and add a firewall rule to allow incoming traffic on the required port (e.g., `sudo ufw allow 8080/tcp`). |
| **Split DNS Fails** (HAProxy logs show connections from your public IP) | The client computer is not using OPNsense for DNS. | **Lesson:** A DNS override only works if clients use it. **Solution:** Go to **Services -> DHCPv4 -> [LAN]** and ensure the **DNS servers** field is set to your OPNsense LAN IP. Then, on the client, renew the DHCP lease and flush the DNS cache. |
| **HAProxy Fails to Start (`critical errors`)** | A syntax error exists in the HAProxy configuration. | **Lesson:** HAProxy is extremely strict. **Solution:** This is almost always a typo in the **Cipher List** or **Cipher Suites** in the Frontend settings. Re-copy and paste them carefully. |
