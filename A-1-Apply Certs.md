### **Process Guide: Applying a Trusted Certificate to the OPNsense Web UI**

#### **1. Strategic Objective: Securing the Management Plane**

Your current action directly addresses a key principle of security governance: **securing administrative interfaces**. By default, OPNsense uses a *self-signed certificate*. While encrypted, it provides no proof of identity, forcing you to click through browser security warnings. This is a low-maturity control.

Our goal is to implement a high-maturity control by replacing that certificate with one issued by our trusted **Intermediate CA**.

*   **The Risk:** An attacker on your network could perform a Man-in-the-Middle (MITM) attack, impersonating your router's login page to steal your credentials. Because you are used to clicking through security warnings, you might not notice.
*   **The Control:** By using a certificate from a CA that your client computer explicitly trusts, any attempt to impersonate the router will result in a clear, undeniable browser error. This provides **identity assurance**.
*   **The Process:** We will not apply the Intermediate CA directly. Instead, we will use the Intermediate CA to **issue a new server certificate** (a leaf certificate) specifically for the OPNsense router. Then, we will configure OPNsense to use this new certificate for its web interface.

---

#### **Phase 1: Issue a Leaf Certificate for the OPNsense Router**

This procedure follows the logic from pages 23-24 of your guide, "Issuing a Leaf Certificate."

##### **Step 1: Navigate to the Certificate Manager**
1.  From the OPNsense main menu, go to **System ‣ Trust ‣ Certificates**.

##### **Step 2: Initiate the Creation of a New Certificate**
1.  Click the **Add** (+) button in the top right corner of the page.

##### **Step 3: Configure the Router's Certificate Details**
Fill in the fields on the "Edit Certificate" page. This is the most critical step.

| Field | Value / Action | Strategic Purpose (The "Why") |
| :--- | :--- | :--- |
| **Method** | Select `Create an Internal Certificate`. | We are creating an end-entity certificate, not another CA. |
| **Description** | `OPNsense Web UI Certificate` | A clear, human-readable name for identification. |
| **Type** | `Server Certificate` | **This is critical.** It defines the certificate's purpose. Using the wrong type will cause connection errors. |
| **Issuer** | **Select your `Intermediate CA` from the dropdown.** | **This is the most important setting.** You are commanding your Intermediate CA to sign and vouch for this new server certificate, creating the proper chain of trust. |
| **Lifetime (days)** | `365` (1 year) | Server certificates should have a much shorter lifetime than CAs. This follows the principle of regular renewal, reducing the window of opportunity if a key is compromised. |
| **Common Name** | `router.homenetwork.local` (or whatever FQDN you use) | This must match the exact hostname you type into your browser's address bar to access OPNsense. |
| **Alternative Names** | Add the following: <br> • **IP Address:** `192.168.1.1` <br> • **DNS domain name:** `opnsense.local` | This is vital for preventing browser errors. It lists all valid names (IP and DNS) for the router, ensuring the certificate is valid no matter how you access it. |

> **CISM Mindset Toolkit Applied: Control Maturity & Identity Assurance**
>
> You have just moved from a low-maturity control (an untrusted, self-signed certificate) to a high-maturity one (a certificate chained to a trusted root).
> *   **Identity Assurance:** The `Common Name` and `Alternative Names` fields are not just text; they are a binding assertion of identity. You are declaring, "This certificate is valid *only* for these specific names." This is the core of preventing impersonation.
> *   **Chain of Trust:** By selecting your `Intermediate CA` as the `Issuer`, you have created a verifiable link: Your client trusts the Root CA -> the Root CA trusts the Intermediate CA -> the Intermediate CA trusts this new server certificate.

##### **Step 4: Save the New Certificate**
1.  Scroll to the bottom and click **Save**. You will now have a new certificate listed under **System ‣ Trust ‣ Certificates**.

---

#### **Phase 2: Apply the New Certificate to the OPNsense Web Service**

Now we tell OPNsense to stop using the default certificate and start using the one we just created.

1.  Navigate to **System ‣ Settings ‣ Administration**.
2.  Find the **SSL Certificate** setting.
3.  Click the dropdown menu. You should see the certificate you just created (`OPNsense Web UI Certificate`). **Select it.**
4.  Scroll to the bottom and click **Save**.

OPNsense will save the setting and restart its web server. Your browser may be disconnected.

---

#### **Phase 3: Verification and Final Trust**

1.  Close your browser tab and open a new one. Navigate to your OPNsense router (e.g., `https://192.168.1.1`).
2.  **Expected Behavior:** Your browser will likely still show a security warning. **This is normal.** While the certificate is now correctly chained (`Server Cert <- Intermediate CA <- Root CA`), your browser does not yet trust your **Root CA**.
3.  **Final Step - Establish Trust:**
    *   Go back to **System ‣ Trust ‣ Authorities**.
    *   Find your **Root-CA** in the list.
    *   In the "Commands" column, click the **export certificate (cloud with down-arrow)** icon.
    *   Install this downloaded `Root-CA.crt` file into your computer's (or browser's) "Trusted Root Certification Authorities" store. The method for this varies by operating system.
4.  Once the Root CA is trusted by your system, close and reopen your browser. When you navigate to the OPNsense UI, you should see a **secure padlock icon with no warnings**.

You have now successfully secured your router's management interface with your own internal PKI, completing a critical step in hardening your network.
