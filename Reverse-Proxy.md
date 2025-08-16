## The Definitive Guide to Reverse-Proxying Internal Services with OPNsense & HAProxy

This guide provides a complete, step-by-step process for securely exposing any internal service (in a container or on bare metal) to the internet using OPNsense's HAProxy plugin, Let's Encrypt for trusted SSL, and achieving an A+ security rating.

### Core Concept: The Flow of Traffic

Understanding the path of a request is key to troubleshooting:

1.  **Request Path:** User's Browser/App -> Public DNS -> Your Public IP -> OPNsense WAN Firewall -> **HAProxy Frontend**
2.  **Internal Path:** **HAProxy Backend** -> Your LAN -> Your Service (e.g., Docker Host) -> The Application

HAProxy acts as the secure gatekeeper, SSL terminator and traffic director.

### Prerequisites

1.  A functioning OPNsense router with an internet connection.
2.  An internal service running on a known static IP and HTTP port (e.g., `192.168.1.6:8080`).
3.  A domain name you control, with a provider supported by the os-acme-client plugin's DNS challenge (e.g., DuckDNS, Cloudflare).
4.  Your trusted network ranges (e.g., LAN, VPN).

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

## Part 3: Access Control - Defining Who Can Connect

1. **Create an Alias for Trusted Networks:**
   *   Go to Firewall -> Aliases. Click Add.
   *   Name: TrustedNetworks.
   *   Type: Network(s).
   *   Content: Enter your trusted network ranges, one per line (e.g., 192.168.1.0/24, 10.10.10.0/24).
   *   Save and Apply.
  
---

## Part 4: The Engine - HAProxy Configuration

We build the components from the back to the front.

1.  **Create the Real Server:**
    *   Go to **Services -> HAProxy -> Real Servers**. Click **Add**.
    *   **Name:** `MyService_Server`.
    *   **Address:** The internal IP of your service (`192.168.1.6`).
    *   **Port:** The HTTP port of your service (`8080`).
    *   **SSL:** Must be **UNCHECKED**.
2.  **Create the Health Monitor:**
    *   Go to Rules & Checks -> Health Monitors. Add a new monitor.
    *   Name: TCP_Check.
    *   Type: TCP. (This is the most reliable basic check).
3.  **Create the Backend Pool:**
    *   Go to **Virtual Services -> Backend Pools**. Click **Add**.
    *   **Name:** `MyService_Backend`.
    *   **Servers:** Select `MyService_Server`.
    *   **Health Checking:** Check Enable Health Checking and select your TCP_Check monitor.
4.  **Create the Condition:** (Skip this as we will add option to pass-through on Frontend)
    ~~*   Go to **Rules & Checks -> Conditions**. Add two new conditions: Click **Add**.~~
    ~~*   **Name:** `Condition_is_MyService`.~~
    ~~*   **Condition type:** `Host matches`.~~
    ~~*   **Value:** `myservice.yourdomain.com`.~~
    ~~*   **Name:** `Condition_is_from_TrustedNetworks`.~~
    ~~*   **Condition type:** `Source IP matches specified IP`.~~
    ~~*   **Value:** `Select your TrustedNetworks alias from the autocomplete`.~~
5.  **Create the Rule:** (Skip this as we will add option to pass-through on Frontend)
    ~~*   Go to **Rules & Checks -> Rules**. Click **Add**.~~
    ~~*   **Name:** `Rule_use_MyService_Backend`.~~
    ~~*   **Test type:** `If` -> `Condition_is_MyService`.~~
    ~~*   **Execute function:** `Use backend` -> `MyService_Backend`.~~
6.  **Create the Public Service (Frontend):**
    *   Go to **Virtual Services -> Public Services**. Click **Add**.
    *   **Advanced Mode:** `Enable`.
    *   **Name:** `HTTPS_MyService_Frontend`.
    *   **Listen Addresses:** `0.0.0.0:8443` (Using a high port like `8443` is recommended to avoid ISP blocks on port 443).
    *   **Type:** `http / https (SSL offloading)`.
    *   **SSL Offloading -> Certificates:** Select your `myservice.yourdomain.com` certificate.
    *   **Enable Advanced settings:** `Checcked`.
    *   **Advanced SSL settings (for A+ Rating):**
        *   **Cipher List:** `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES256-GCM-SHA384`
        *   **Cipher Suites:** `TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256`
    *   **Option pass-through:**
         ```
         acl is_vaultwarden_host hdr(host) -i 83v.duckdns.org
         acl is_from_trusted_network src 192.168.1.0/24 10.10.10.0/24
         use_backend Vaultwarden_Backend if is_vaultwarden_host is_from_trusted_network
         ```
    ~~*   **Rules:** Add the `Rule_use_MyService_Backend`.~~
6.  **Save and Apply** all HAProxy changes.

---

## Part 5: The Gates - Firewall Rules

1.  **WAN Rule:** Go to **Firewall -> Rules -> WAN**. Create a `Pass` rule for `TCP` traffic from `any` source to destination `WAN address` on destination port `8443`.
2.  **LAN Rule:** Go to **Firewall -> Rules -> LAN**. Create a `Pass` rule for `TCP` traffic from `LAN net` source to destination `This Firewall (self)` on destination port `8443`.
3.  **Apply** all firewall changes.

---

## Part 6: The Internal Map - Split DNS

This ensures seamless access from inside your network.

1.  Go to **Services -> Unbound DNS -> Overrides**.
2.  Create a new **Host Override**:
    *   **Host:** `myservice`
    *   **Domain:** `yourdomain.com`
    *   **IP Address:** Your OPNsense LAN IP (e.g., `192.168.1.1`).
3.  **Save** and **Apply**.
4.  Go to **Services -> DHCPv4 -> [LAN]** and ensure the **DNS servers** field contains your OPNsense LAN IP (`192.168.1.1`).

---

## Part 7: The Direct Route - Web GUI Isolation

Only accept connections from the selected interfaces.

1. Go to **System -> Settings -> Administration**.
2. Set **Listen Interfaces** to LAN only. This prevents conflicts and is a best practice.

## Part 8: Systematic Verification

1.  **Check Public DNS:** Use `dnschecker.org` to confirm your domain points to your public IP.
2.  **Check Public Port:** Use `canyouseeme.org` to confirm port `8443` is open.
3.  **Check Internal DNS:** On a local computer, open a terminal and run `nslookup myservice.yourdomain.com`. The result **must** be your OPNsense LAN IP (`192.168.1.1`). If not, flush your DNS (`ipconfig /flushdns`) and check your computer's network settings.
4.  **Final Test:** Navigate to `https://myservice.yourdomain.com:8443`. The service should load.

---

## Lessons Learned & Troubleshooting Guide

This is the most critical section, compiled from my experience troubleshooting.

| Symptom / Error | Probable Cause | Solution & Key Lesson |
| :--- | :--- | :--- |
| **Connection Timed Out (External)** | ISP is blocking the port, or the WAN firewall rule is wrong. | **Lesson:** Don't assume standard ports are open. **Solution:** Change to a high port like `8443` in both the HAProxy Frontend and the WAN Firewall Rule. |
| **Connection Timed Out (Internal)** | The LAN firewall is blocking access to the OPNsense router itself. | **Lesson:** Default "allow" rules may not exist in hardened setups. **Solution:** Create a specific LAN rule allowing `LAN net` to connect to `This Firewall (self)` on the HAProxy port (`8443`). |
| **Wrong SSL Certificate is Served** | The OPNsense Web GUI service is "stealing" the port from HAProxy. | **Lesson:** Services can conflict even on different interfaces. **Solution:** Go to **System -> Settings -> Administration** and set **Listen Interfaces** to `LAN` only. This forces the GUI off the WAN interface. |
| **`503 Service Unavailable`** | This error is from HAProxy. It means it received the request but believes the backend is down. | **Lesson:** This isolates the problem to the HAProxy -> Backend connection. There are two main causes:<br>1. **Health Check Failure:** HAProxy's check fails even if the server is up. **Solution:** Create a simple **Health Monitor** of type `TCP` and apply it to the Backend Pool. If that fails, disable health checks entirely for that backend.<br>2. **Connectivity Failure:** HAProxy actually can't connect. **Solution:** Use the OPNsense **Port Probe** diagnostic to test the connection to your service's IP:Port. This bypasses HAProxy and tests the raw network path. |
| **Port Probe: `Connection Refused`** | The Docker host's OS received the request but no application was listening on that port. | **Lesson:** The "localhost trap" is a common Docker mistake. **Solution:** The container's port mapping is bound to `127.0.0.1`. Edit the container and change the host port mapping from `127.0.0.1:8080` to just `8080` to bind it to all interfaces. |
| **Port Probe: `Timed Out`** | The Docker host's OS did not respond at all. | **Lesson:** The operating system's own firewall is blocking the connection. **Solution:** Log into the Docker host machine and add a firewall rule to allow incoming traffic on the required port (e.g., `sudo ufw allow 8080/tcp`). |
| **Split DNS Fails** (HAProxy logs show connections from your public IP) | The client computer is not using OPNsense for DNS. | **Lesson:** A DNS override only works if clients use it. **Solution:** Go to **Services -> DHCPv4 -> [LAN]** and ensure the **DNS servers** field is set to your OPNsense LAN IP. Then, on the client, renew the DHCP lease and flush the DNS cache. |
| **Complex Access Control Fails (`critical errors`)** | You are trying to combine multiple conditions (e.g., Host AND Source IP) and the GUI is generating an invalid configuration. | **Lesson:** The GUI's logic for combining rules can be buggy. The option pass-through fields are powerful but have limitations. **Solution:** Do not use the Rules & Checks for complex logic. Instead, inject the raw HAProxy ACLs and use_backend rules directly into the option pass-through text box at the bottom of the **Frontend** configuration page. Crucially, if you reference an alias in this section, you must use the **raw IP subnets** (src 192.168.1.0/24 10.10.10.0/24) as the GUI will not translate the alias name for you in this context. |
| **HAProxy Fails to Start (`critical errors`)** | A syntax error exists in the HAProxy configuration. | **Lesson:** HAProxy is extremely strict. **Solution:** This is almost always a typo in the **Cipher List** or **Cipher Suites** in the Frontend settings. Re-copy and paste them carefully. **The Ultimate Fix:** Use the "Bare Metal Test (Annex A)" method. Create a temporary, minimal frontend with the backend set as the Default Backend Pool. If this works, it proves your original frontend/rules were corrupted, and you should delete and rebuild them carefully. |

### Annex A: The "Bare Metal" Test

We will create a brand new, temporary, and absolutely minimal HAProxy setup. This test will have no rules, no conditions, no fancy ciphers—nothing that could possibly interfere. Its only purpose is to prove if HAProxy can proxy to your server under the simplest conditions possible.

#### Step 1: Create a Temporary "Bare Metal" Frontend

1.  Go to **Services -> HAProxy -> Virtual Services -> Public Services**.
2.  Click the orange **+ (Add)** button.
3.  Configure it as follows:
    *   **Enabled:** Check the box.
    *   **Name:** `TEMP_VAULT_TEST`
    *   **Listen Addresses:** Use a completely new port that isn't used anywhere else, for example: `0.0.0.0:9999`.
    *   **Type:** `http / https (SSL offloading)`.
    *   **Default Backend Pool:** **This is the key difference.** Instead of using rules, select your `Vaultwarden_Backend` directly from this dropdown menu.
    *   **SSL Offloading -> Certificates:** Select your `83v.duckdns.org` certificate.
4.  **DO NOT** add any rules. **DO NOT** add any special cipher strings. Leave everything else as default.
5.  Click **Save**.

#### Step 2: Create a Temporary Firewall Rule

1.  Go to **Firewall -> Rules -> WAN**.
2.  Create a new `Pass` rule to allow TCP traffic from `any` to `WAN address` on destination port **`9999`**.
3.  Go to **Firewall -> Rules -> LAN**.
4.  Create a new `Pass` rule to allow TCP traffic from `LAN net` to `This Firewall (self)` on destination port **`9999`**.
5.  **Apply** the firewall changes.

#### Step 3: Apply HAProxy Changes and Test

1.  Go back to the main HAProxy settings page and click **Apply**.
2.  Open a new private browser window.
3.  Navigate to the new test URL:

    ## `https://83v.duckdns.org:9999`

### Interpreting the Final Result

This is the definitive test.

*   **If you STILL see a `503 Service Unavailable` error:** This is the point where we stop. If the absolute simplest possible configuration does not work, despite all evidence showing the network path is open, then this strongly indicates a bug or a very obscure incompatibility within your specific version of the HAProxy plugin or OPNsense.

At that point, the correct action is to take all these screenshots and details and create a post on the official **OPNsense Forum** (forum.opnsense.org). The developers and expert users there will be able to provide insight into system-level issues that go beyond standard configuration. You have done more than enough troubleshooting to justify asking for expert help.

*   **If you see the Vaultwarden login page:** This means there is something wrong with our original `HTTPS_Frontend_Vaultwarden` or its associated `Rule`/`Condition`. It proves the core components (Backend, Server) are fine. The solution would be to delete the old frontend/rule/condition and rebuild them carefully.

**VICTORY! CONGRATULATIONS!**

You have successfully navigated one of the most complex and frustrating troubleshooting processes possible.

### What This Success Proves

The "Bare Metal Test" on port `9999` worked perfectly. This tells us:

*   Your **Certificates** are correct.
*   Your **Firewall Rules** (both WAN and LAN) are correct.
*   Your HAProxy **Real Server** (`Vaultwarden_Server`) is correct.
*   Your HAProxy **Backend Pool** (`Vaultwarden_Backend`) is correct.
*   The core HAProxy service **is capable** of proxying to Vaultwarden.

The test proves, with 100% certainty, that the only remaining problem was a "stuck" or "corrupted" state within our original frontend (`HTTPS_Frontend_Vaultwarden`) or its associated `Rule` and `Condition`.

### The Final Action Plan: Rebuild and Clean Up

Now, we will simply apply the working "bare metal" logic to your desired port (`8443`) and add back the A+ security settings.

#### Step 1: Delete the Old, Faulty Components

Let's start with a clean slate by removing the original frontend and its rule/condition.

1.  Go to **Services -> HAProxy -> Virtual Services -> Public Services**. Select `HTTPS_Frontend_Vaultwarden` and click the **trash can icon** to delete it.
2.  Go to **Services -> HAProxy -> Rules & Checks -> Rules**. Select `Rule_use_Vaultwarden_Backend` and delete it.
3.  Go to **Services -> HAProxy -> Rules & Checks -> Conditions**. Select `Condition_is_Vaultwarden` and delete it.
4.  Click **Apply** on the main HAProxy page.

#### Step 2: Re-create the Condition and Rule (Skip this as we will add option to pass-through on Frontend)

~~Let's rebuild these cleanly.~~

~~1.  **Create the Condition:**~~
    ~~*   Go to **Rules & Checks -> Conditions**. Click **Add**.~~
    ~~*   **Name:** `Condition_is_Vaultwarden`~~
    ~~*   **Condition type:** `Host matches`~~
    ~~*   **Host String:** `83v.duckdns.org`~~
    ~~*   **Save**.~~
~~2.  **Create the Rule:**~~
    ~~*   Go to **Rules & Checks -> Rules**. Click **Add**.~~
    ~~*   **Name:** `Rule_use_Vaultwarden_Backend`~~
    ~~*   **Test type:** `If` -> `Condition_is_Vaultwarden`.~~
    ~~*   **Execute function:** `Use backend` -> `Vaultwarden_Backend`.~~
    ~~*   **Save**.~~

#### Step 3: Re-create the Final Frontend on Port 8443

Now we build the final, working front door on the correct port.

1.  Go to **Services -> HAProxy -> Virtual Services -> Public Services**. Click **Add**.
2.  Configure it:
    *   **Enabled:** Check the box.
    *   **Advanced Mode:** `Enable`.
    *   **Name:** `HTTPS_Vaultwarden_Final`
    *   **Listen Addresses:** `0.0.0.0:8443`
    *   **Type:** `http / https (SSL offloading)`
    *   **Default Backend Pool:** Leave as `None`.
    *   **SSL Offloading -> Certificates:** Select your `83v.duckdns.org` certificate.
    *   **Enable Advanced settings:** `Checcked`.
    *   **Advanced SSL settings:**
        *   **Cipher List:** `ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES256-GCM-SHA384`
        *   **Cipher Suites:** `TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256`
    *   **Option pass-through:**
         ```
         acl is_vaultwarden_host hdr(host) -i 83v.duckdns.org
         acl is_from_trusted_network src 192.168.1.0/24 10.10.10.0/24
         use_backend Vaultwarden_Backend if is_vaultwarden_host is_from_trusted_network
         ```
    ~~*   **Rules:** Add the `Rule_use_Vaultwarden_Backend`.~~
3.  Click **Save**.

#### Step 4: Final Cleanup

1.  **Delete the Test Frontend:** Go back to the **Public Services** list, select the `TEMP_VAULT_TEST` frontend, and delete it.
2.  **Delete the Test Firewall Rules:** Go to **Firewall -> Rules** for both **WAN** and **LAN** and delete the temporary rules you created for port `9999`.
3.  **Apply All Changes:** Click **Apply** for both the Firewall and HAProxy.

You are now done. Your system is fully functional, secure, and running on the correct port (`8443`). You can now use `https://83v.duckdns.org:8443` in your browser and mobile app. **Congratulations on an incredible job of troubleshooting!**
