### **Annex A: Managing Trust with a PKI Mindset**

#### **Foreword: The Foundation of Digital Trust**

In the world of information security, we cannot secure what we cannot trust. How do we know the server we are connecting to is legitimate? How do we verify the identity of a user connecting remotely? The answer is a **Public Key Infrastructure (PKI)**, and certificates are its building blocks.

Think of a certificate as a digital passport. It is issued by a trusted authority (a Certificate Authority, or CA), it has a limited validity period, it can be revoked if compromised, and it proves an identity. In OPNsense, you have the power to become your own CA, essentially your own digital passport office. This is a profound responsibility that forms the bedrock of security for many other services like VPNs and secure web access. This document explains the technical "how" of managing certificates, while the Information Security Mindset calls outs the strategic "why."

---

#### **Trust**

In OPNsense, certificates are used for ensuring trust between peers. To make using them easier, OPNsense allows creating certificates from the front-end. In addition to that, it also allows creating certificates for other purposes, avoiding the need to use the openssl command line tool. Certificates in OPNsense can be managed from **System ‣ Trust ‣ Certificates**.

Examples of OPNsense components that use certificates:
*   OpenVPN
*   IPsec
*   Captive Portal
*   Web Proxy

#### **Certificate Types**
The following types of certificate can be generated in OPNsense:
*   Client
*   Server
*   Combined Client/Server
*   **Certificate Authority**

In addition to this, OPNsense can generate a Certificate Signing Request (CSR). This can be used if you want to create a certificate signed by an external CA.

> **Governance & The Root of Trust**
>
> When you create a **Certificate Authority (CA)** in OPNsense, you are making a critical governance decision. You are establishing the **Root of Trust** for your network's security services.
>
> *   **The Responsibility:** The private key of this CA is the most sensitive secret on your firewall. If it is compromised, an attacker can mint valid certificates for any purpose, impersonating your services (like your VPN) and users. This key must be protected with the utmost care.
> *   **The Policy:** Your security policy should state that all internal services (VPN, web GUI) *must* use certificates signed by this internal CA. Using a CSR to get a certificate from an external, commercial CA is the correct policy for public-facing services.
> *   **The Principle:** This establishes a clear boundary of trust. We trust our internal CA for internal matters, and we rely on the global PKI system for external matters.

**Warning**
Make sure that you select the correct certificate type, as many clients will refuse connection (or at least show errors) if an incorrect certificate type is used. For example, you can use either a server certificate or a combined client/server certificate to secure the connection to the web interface, but not a CA or client certificate.

---

#### **Settings**
In **System ‣ Trust ‣ Settings** you can find some advanced options to manage how the system handles trust. For compliance reasons, it is possible to implement certain constraints when a default OpenSSL context is requested.

| **Options** | **Description** |
| :--- | :--- |
| Store intermediate | Allow local defined intermediate certificate authorities to be used in the local trust store. We advise to only store root certificates to prevent cross signed ones causing breakage when included but expired later in the chain. |
| Store CRL’s | Store all configured CRL’s in the default trust store. If the client or service support CRL’s, deploying to the default location eases maintenance. |
| Auto fetch CRL’s | Schedule an hourly job to download CRLs using the defined Distributionpoints in the CAs deployed in our trust store. |
| **Enable legacy** | Enable Legacy Providers. (Enables ciphers which will be removed in future versions of OpenSSL) |
| **Configuration constraints** | When enabled, you can set some default ciphers and protocols, be careful with these options as these settings will be used by quite some applications of the firewall (and might break them). |

> **Policy Enforcement & Risk Appetite**
>
> These settings are not just technical toggles; they are direct implementations of your security policy and risk appetite.
>
> *   **Risk Management:** Disabling "**Enable legacy**" is a risk reduction activity. It enforces the use of modern, strong cryptography, reducing the risk of a downgrade attack where an attacker forces a connection to use a weak, breakable cipher. Leaving it enabled may be necessary for compatibility with old devices, but this is a conscious acceptance of risk that should be documented.
> *   **Governance & Compliance:** Using "**Configuration constraints**" to set specific ciphers and protocols is a powerful governance tool. This allows you to enforce a cryptographic standard across multiple services, ensuring compliance with internal policies or external regulations (e.g., PCI-DSS, HIPAA) that mandate specific levels of encryption. This moves your configuration from "default" to "explicitly hardened."

---

#### **Revoke Certificates**

> **Incident Response & Lifecycle Management**
>
> Creating a certificate is easy. The true test of a mature PKI program is having a robust and tested **revocation process**. This is a critical incident response capability.
>
> *   **The Scenario:** A user's laptop containing a client VPN certificate is stolen. If you cannot revoke that certificate, the thief has a valid key to your network. The time between compromise and revocation is your window of exposure.
> *   **The Control:** Certificate Revocation Lists (CRLs) and the Online Certificate Status Protocol (OCSP) are the technical controls for invalidating a "digital passport" before its expiry date.
> *   **The Program View:** Your security program must include a procedure for revocation. Who has the authority to request a revocation? How is it performed? How quickly can it be done? This is about managing the entire certificate lifecycle, not just creation.

#### **Certificate Revocation Lists**


#### **Online Certificate Status Protocol**


---

#### **Internal Organisation**


---

### **Process Guide: Creating a Root Certificate Authority (CA) in OPNsense**

#### **1. Strategic Objective: Establishing the Root of Trust**

This process follows the best practice of creating a Root CA that will be used *only* to sign an Intermediate CA. This protects the highly sensitive root key from being used in day-to-day operations.

---

#### **2. Prerequisites**

*   You have a functional OPNsense installation.
*   You are logged into the OPNsense Web UI with administrative privileges.

---

#### **3. Step-by-Step Procedure to Create the Root CA**

This procedure replaces detailed instructions on page 22 from the guide, "Build your Home Network with OPNsense."

##### **Step 1: Navigate to the Certificate Authority Manager**

1.  From the OPNsense main menu, go to **System ‣ Trust ‣ Authorities**.

##### **Step 2: Initiate the Creation of a New Authority**

1.  Click the **Add** (+) button in the top right corner of the page to create a new authority.

##### **Step 3: Configure the Root CA Details**

You will now see the "Edit Certificate" page. Fill in the fields as follows to define your Root CA.

| Field | Value / Action | Strategic Purpose (The "Why") |
| :--- | :--- | :--- |
| **Method** | Select `Create an internal Certificate Authority` | You are explicitly telling OPNsense to generate a new, self-signed CA that will serve as the foundation of your PKI. |
| **Description** | `Home Network Root CA` (or similar) | A human-readable name for easy identification. Good documentation is a core part of governance. |
| **Key type** | `RSA-2048` (Default) | Defines the cryptographic algorithm and strength. 2048-bit RSA is a strong, widely compatible choice for a root. |
| **Digest Algorithm** | `SHA256` (Default) | Selects the hashing algorithm for the certificate's signature. SHA256 is the modern security standard. |
| **Lifetime (days)** | `3650` (10 years) | **This is a critical risk management decision.** A long lifetime is standard for a Root CA because it is difficult to replace. This is acceptable *only* because we will protect its key and use an Intermediate CA for all signing operations. |
| **Country Code** | Enter your two-letter country code (e.g., US, GB). | Adds identifying metadata to the certificate's subject name. |
| **Organization** | `Home Network` (or your name) | Adds identifying metadata. |
| **Common Name** | `My Home Root CA` (or similar) | This is the official "name" of the CA. It must be unique within your PKI. |

> **Asset Protection & Lifecycle Management**
>
> The configuration you just entered defines a critical security asset: the **Root CA's private key**.
>
> *   **Asset Identification:** You have just defined the "crown jewel" of your network's trust system.
> *   **Protection:** In a corporate environment, this Root CA's private key would be generated and stored on a hardware security module (HSM) or an offline machine that is never connected to the network. For our home network, simply recognizing its importance is key. Its job is to sign an Intermediate CA *once*, and then be left alone.
> *   **Lifecycle:** The 10-year lifetime you set defines the maximum validity of your entire PKI. This is a deliberate policy choice balancing security (shorter lifetimes are better) with operational overhead (longer lifetimes are easier).

##### **Step 4: Save and Create the Root CA**

1.  Scroll to the bottom of the page and click the **Save** button.

---

#### **4. Verification and Next Steps**

You have successfully created your Root CA. You will be returned to the **System ‣ Trust ‣ Authorities** page, where you should now see your "Home Network Root CA" listed.

**Your work is not done.** A Root CA should never be used to sign end-entity (leaf) certificates directly.

1.  **Immediate Next Step:** Follow the procedure on **page 22-23** from the guide, "Build your Home Network with OPNsense.", "Issuing the Intermediate CA." You will create a new Intermediate CA and, for the `Issuer` field, you will select the Root CA you just created.
2.  **Operational Plan:** The Intermediate CA will then be used to sign all your server and user certificates (as shown on page 23-24). This creates a proper chain of trust (`Root CA -> Intermediate CA -> Leaf Certificate`) and protects your root key from exposure.
3.  **Deployment:** To make this trust system functional, you must export the *public key* of the Root CA and install it in the trust store of all your client devices (laptops, phones). This tells those devices to trust anything signed by your PKI.
