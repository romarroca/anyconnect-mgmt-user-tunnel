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

📌 **Note:** No separate user certificate is required in this design. The FTD validates that the machine is trusted before allowing user-level authentication.

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
