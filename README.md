# SMB Relay – Initial Attack Vector (Active Directory Home Lab)

> **Project:** Active Directory Home Lab   
> **Chapter:** Attacking Active Directory → Initial Attack Vectors → SMB Relay

## Executive Summary
SMB Relay is an attack that relays NTLM authentication to another host instead of cracking it. When SMB signing is not enforced, an attacker can authenticate to another system using the victim's credentials in real time.

## Sections
- Lab Overview
- What is SMB Relay?
- Prerequisites
- Attack Workflow
- Tools Used
- Walkthrough
- Pentester Observations
- Detection
- Mitigations
- MITRE ATT&CK
- References

## What is SMB Relay?
SMB Relay abuses NTLM authentication by forwarding a victim's authentication request to another machine that accepts NTLM and does not require SMB signing. If the victim has sufficient privileges, the attacker may gain administrative access without knowing the password.

## Attack Workflow
```text
Victim -> Attacker -> Target
   NTLM Authentication
        |
   Relayed in real time
        |
Target authenticates attacker as victim
```

## Walkthrough
1. Identify systems without mandatory SMB signing.
2. Configure Responder appropriately.
3. Launch ntlmrelayx against eligible targets.
4. Trigger NTLM authentication from a victim.
5. Relay authentication to the target.
6. Verify successful execution or access.

## Pentester Observations
- No password cracking is required.
- SMB signing is the primary defense.
- Administrative users significantly increase impact.

## Detection
Monitor NTLM authentication, SMB sessions, unusual lateral movement, and authentication from unexpected systems.

## Mitigations
- Enable SMB signing.
- Disable LLMNR/NBT-NS.
- Prefer Kerberos over NTLM.
- Apply least privilege.
- Segment networks.
- Monitor authentication events.

## MITRE ATT&CK
- T1557 Adversary-in-the-Middle
- T1021.002 SMB/Windows Admin Shares
- T1078 Valid Accounts

## References
- Microsoft SMB documentation
- Impacket
- MITRE ATT&CK

## Next Chapter
Use the obtained access for subsequent Active Directory attack paths in the Home Lab.
