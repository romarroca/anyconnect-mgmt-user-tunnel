# anyconnect-mgmt-user-tunnel
Configuration and documentation for Cisco Secure Client (AnyConnect) with management tunnel and user tunnel deployment. This repository covers setup, testing, and configuration for Always-On VPN, certificate-based authentication, and the transition from machine-level to user-level connectivity.

## Components used
- Cisco Secure Client (AnyConnect) 5.x
- Cisco Firepower Threat Defense (FTD) with FMC 7.4.X
- PKI (machine and user certificates) (Windows Server 2025)
- DUO for 2FA, SAML, and passwordless (optional)

## What You Should Already Know
- Basic Cisco Firepower Threat Defense (FTD) and Firepower Management Center (FMC) concepts
- How Cisco Secure Client (AnyConnect) establishes VPN tunnels
- Public Key Infrastructure (PKI): issuing, deploying, and managing machine/user certificates
- Fundamentals of remote access VPN (RAVPN) policies
- Familiarity with SAML/DUO MFA authentication flows

# PKI Setup

<img width="781" height="484" alt="image" src="https://github.com/user-attachments/assets/e60c0c2e-61c9-4e58-9f6d-5e64d7c37049" />

This deployment relies only on a **machine certificate** to establish device trust.  
Instead of issuing separate user certificates, the configuration chains authentication so that:

1. The **machine certificate** proves the endpoint is a corporate asset.
2. The **user login** (AAA/SAML/etc.) proves the identity of the user.

- **Certificate Authority (CA):** Microsoft Active Directory Certificate Services (AD CS)
- **Machine Certificate:**  
  - Template: Computer (auto-enrolled via GPO)  
  - Purpose: Client Authentication (EKU: 1.3.6.1.5.5.7.3.2)  
  - Used for both management tunnel (pre-logon) and user tunnel (chained with user login)


## Group Policy: AnyConnect_Management

The **AnyConnect_Management** group policy defines the behavior of the management tunnel.  
This ensures that endpoints can connect automatically pre-logon, with only limited network access.

### General Settings

<img width="704" height="799" alt="image" src="https://github.com/user-attachments/assets/a88b72ff-8552-4426-96fe-a065e1427b95" />

- **VPN Protocols:** SSL and IPsec-IKEv2 both enabled  
  (ensures compatibility and fallback support)

### Secure Client → Management Profile

<img width="698" height="798" alt="image" src="https://github.com/user-attachments/assets/5a73f8b1-4082-48c8-b7b2-9a0bbc2cb303" />

- **Profile:** `mgmtprofile.xml`  
  - This XML defines the Management VPN Tunnel behavior.  
  - It is deployed to the endpoint so the tunnel comes up automatically before user login.  

### Secure Client → Connection Settings

<img width="694" height="802" alt="image" src="https://github.com/user-attachments/assets/c8eafad3-500b-46e7-a2b5-fade8c5cc95b" />

- **Client Bypass Protocol:** Enabled  
  - Ensures that the management tunnel does not block local network access when needed.  

### Secure Client → Custom Attributes

<img width="702" height="796" alt="image" src="https://github.com/user-attachments/assets/7ba0a355-6939-41ec-bbb6-2ee7ff861aea" />

- **Attribute:** `ManagementTunnelAllAllowed = true`  
  - Allows the management tunnel for all endpoints with valid machine certificates.

📌 **Notes:**
- This group policy is tied only to the management tunnel profile (`mgmtVPN`).  
- Access should be tightly restricted (typically DCs, SCCM, AV servers).  
- The combination of `mgmtprofile.xml` and the certificate requirement ensures that only corporate machines can establish this tunnel.

## Group Policy: SSO-GP (User Tunnel)

The **SSO-GP** group policy defines the **user tunnel** behavior.  
Unlike the management tunnel, this policy provides the user with full corporate access after login.

### General Settings

<img width="696" height="801" alt="image" src="https://github.com/user-attachments/assets/cf601036-a18a-471b-a894-a134c89f915a" />

- **VPN Protocols:** SSL and IPsec-IKEv2 both enabled  
  (provides flexibility depending on client configuration)

### Secure Client → Client Profile

<img width="701" height="672" alt="image" src="https://github.com/user-attachments/assets/8887c69c-b72d-4912-85a9-dd29711a777e" />

- **Profile:** `sso.xml`  
  - Defines the user tunnel configuration delivered to endpoints.  
  - Includes Always-On behavior and SSO (SAML/AAA) settings.  

### Secure Client → Management Profile

<img width="694" height="678" alt="image" src="https://github.com/user-attachments/assets/bddfc725-95cc-4054-91e8-62d8cef5d0d1" />

- **Profile:** `mgmtprofile.xml`  
  - Ensures that the **management tunnel** is available pre-logon.  
  - The user tunnel takes over after login, replacing the management tunnel.  

📌 **Notes:**
- The user tunnel chains authentication by requiring both:  
  - A valid **machine certificate** (to confirm corporate asset)  
  - User login via **AAA/SAML** (to confirm user identity)  
- This allows you to enforce *device trust + user trust* without issuing separate user certificates.  
- Unlike the management group policy, this policy provides broader access (based on corporate ACLs).

## Connection Profiles

Connection profiles link the **VPN entry point** (what the client sees) with the correct **Group Policy**.  
This is where management and user tunnels are separated.

---

### Management Tunnel Connection Profile

<img width="708" height="670" alt="image" src="https://github.com/user-attachments/assets/d14930c1-f0b8-4b18-b33a-0d68ba406426" />

- **Name:** `mgmtVPN`  
- **Group Policy:** `AnyConnect_Management`

<img width="703" height="661" alt="image" src="https://github.com/user-attachments/assets/29208861-5ccd-418f-b161-bc9e541a1bcb" />

- **Authentication:** `Client Certificate Only`  
  - Ensures only endpoints with a valid machine certificate can establish the management tunnel.  
- **Alias:** `mgmtVPN`

  <img width="699" height="646" alt="image" src="https://github.com/user-attachments/assets/cfdd359b-1258-447c-8f46-6b558dd8aef9" />

  - URL Alias configured (`https://<fqdn>/mgmtVPN`)  
  - Automatically connects endpoints using the management tunnel pre-logon.  

📌 **Notes:**  
This tunnel provides restricted access (AD, SCCM, patch servers) and is not user-interactive.  

---

### User Tunnel Connection Profile

<img width="697" height="665" alt="image" src="https://github.com/user-attachments/assets/026b75d7-9075-4795-abbf-84579a705b60" />

- **Name:** `ssoVPN` (example based on screenshots)  
- **Group Policy:** `SSO-GP`  
- **Authentication:** Chained (certificate validation + AAA/SAML)

  <img width="707" height="493" alt="image" src="https://github.com/user-attachments/assets/de0b4e6c-2b35-410f-a2b3-d9341b00bee5" />

  - Machine cert ensures device trust.  
  - AAA/SAML ensures user trust.  
- **Alias:** `ssoVPN`
  
  <img width="695" height="497" alt="image" src="https://github.com/user-attachments/assets/c9e20f55-acb7-4c70-9fc0-9cba882a5abf" />

  - URL Alias used for user login portals.  

📌 **Notes:**  
The user tunnel replaces the management tunnel after login. It provides the full access policy defined under `SSO-GP`.  


## Management Tunnel Profile (mgmttunnel.xml)

<img width="320" height="66" alt="image" src="https://github.com/user-attachments/assets/acf05f62-4b84-46f2-8a87-82b88179da36" />

The management tunnel profile (`mgmttunnel.xml`) was created using the **VPN Management Tunnel Standalone Profile Editor**.  
This XML controls how the Always-On management tunnel behaves on endpoints.

### Preferences

<img width="917" height="788" alt="image" src="https://github.com/user-attachments/assets/a59c44de-27ea-45a1-9872-cc022b9a80fc" />

<img width="923" height="1168" alt="image" src="https://github.com/user-attachments/assets/4cf7520c-ba65-43b4-8c53-db5c831f3944" />

<img width="919" height="817" alt="image" src="https://github.com/user-attachments/assets/96d24b3c-4da5-40d5-8129-fea42312ed2f" />

<img width="909" height="911" alt="image" src="https://github.com/user-attachments/assets/02415e3a-d0c2-4d57-a0b0-995289a5f2b0" />

- **Client Certificate Store:** Machine (Windows & Linux) / System (macOS) 
- **Auto Reconnect Behavior:** ReconnectAfterResume  
- **IP Protocol Supported:** IPv4, IPv6  
- **Other Settings:**  
  - Local LAN Access: Disabled  
  - Captive Portal Detection: Disabled  
  - Clear SmartCard PIN: Enabled  

📌 **Purpose:** Ensures the tunnel uses a **machine certificate** to establish trust, auto-recovers after sleep/hibernation, and supports both IPv4/IPv6.

---

### Certificate Matching
- **Key Usage:** Digital Signature  
- **Extended Key Usage:** Client Authentication  
- **Distinguished Name Match:**  
  - `ISSUER-CN = DC01-CA`  
  - `ISSUER-DC = rnetworks`  
  - `ISSUER-DC = local`  

📌 **Purpose:** Restricts which certificates can be used by matching the issuing CA and ensuring the EKU includes Client Authentication.

---

### Server List
- **Primary Server:**  
  - FQDN/IP: `<redacted>`  
  - User Group: `mgmtVPN`  
  - Protocol: SSL  
  - Auth Method: EAP-AnyConnect  
- **Backup Servers:** None configured  

📌 **Purpose:** The management tunnel connects directly to the FTD using the `mgmtVPN` connection profile. Backup servers can be added for redundancy if needed.

