# 🔥 HTB Noxious — Network Forensics & LLMNR Poisoning Analysis

## 🧨 The Problem  
An IDS alert was triggered by suspicious LLMNR traffic within Forela’s internal Active Directory VLAN. The alert pointed to possible `LLMNR Poisoning`, a technique used by attackers to capture NTLM hashes by impersonating the domain controller. The packet capture suggested rogue behavior targeting a victim machine. The objective was to validate the attack, identify the rogue device, extract the captured hash, and determine if the attack was successful in stealing credentials.

## 🛠 Tools Used

| Tool | Purpose |
|------|---------|
| **Wireshark** | Packet analysis from `.pcap` file |
| **Hashcat** | Cracking NTLMv2 hash from captured NTLMSSP traffic |
| **LLMNR / SMB / DHCP Filters** | Isolating traffic types within Wireshark |
| **Responder (attacker-side)** | Referenced as tool likely used in the attack |

## 🔍 Method (Investigation Steps)

1. **Initial Setup and Packet Review**  
   - Extracted the `.pcap` from the `noxious.zip` archive.
   - Opened the PCAP in `Wireshark` and reviewed top IPs using `Statistics → Endpoints`.

2. **LLMNR Traffic Analysis**  
   - Applied filter `udp.port == 5355` to isolate **LLMNR traffic**.
   - Determined the actual **Domain Controller IP** by filtering for `DNS` traffic and resolving `DC01` FQDN.
   - Found a rogue IP responding to an LLMNR query that should have gone to the domain controller, indicating spoofing.

3. **DHCP Inspection to Identify Hostname**  
   - Used filter `ip.addr == <rogue_ip> && dhcp` to discover the hostname assigned during DHCP negotiation.

4. **NTLM Hash Capture Verification**  
   - Applied the `ntlmssp` filter to inspect NTLM negotiation packets.
   - Identified the **user account** involved in the NTLM authentication initiated by the victim machine.

5. **Timeline Reconstruction**  
   - Switched time format to UTC in Wireshark and documented the timestamp of the first NTLM authentication sequence, aligning with the LLMNR spoof response.

6. **Captured Hash Extraction**  
   - Located the **NTLM Server Challenge** and **NTProofStr** from NTLMSSP packets.
   - Constructed the hash cracking input format using NTLMv2 details from the packet capture.

7. **Password Cracking with Hashcat**  
   - Used Hashcat with mode `-m 5600` to crack the NTLMv2 hash.
   - Confirmed that the attacker could have recovered the user’s password using wordlist-based cracking.

8. **Victim Behavior & Typo Detection**  
   - Analyzed the LLMNR and SMB traffic to understand the user’s mistake.
   - Discovered the victim typed a mistyped server name (`DCC01`) instead of `DC01`, triggering a fallback to LLMNR resolution.

9. **File Share Contextual Review**  
   - Reviewed `SMB2` traffic for tree connect events to uncover the actual file share the user attempted to access.

## ✅ The Solution  
The investigation confirmed a successful `LLMNR Poisoning attack` using a rogue device masquerading as the domain controller. The attacker exploited a user’s typo when navigating to a file share (`DCC01`), triggering a fallback to LLMNR. The rogue machine intercepted the query, responded, and successfully captured the user's `NTLMv2 hash`, which was later proven crackable with `Hashcat`. This scenario highlights how small user errors in legacy protocol environments can lead to credential theft.

> 🧠 *This lab demonstrates how to detect and analyze LLMNR-based attacks and showcases the value of endpoint protocol filtering, NTLM inspection, and timeline correlation.*

## 🧭 Framework Mapping

🔶 **MITRE ATT&CK**

| Tactic               | Technique                          | ID         | Description                                         |
|----------------------|-------------------------------------|------------|-----------------------------------------------------|
| **Credential Access**| LLMNR/NBT-NS Poisoning             | T1557.001  | Spoofing name resolution responses to steal hashes  |
| **Credential Access**| NTLM Credential Capture            | T1111      | Capturing NTLMv2 hashes via spoofed servers         |
| **Execution**        | User Execution (Typo-Induced)      | T1204      | Indirect user error leading to credential exposure  |
| **Defense Evasion**  | Masquerading                       | T1036      | Rogue host pretended to be the domain controller    |

🟦 **NIST Cybersecurity Framework (CSF)**

| Function   | Category             | Subcategory                                     |
|------------|----------------------|-------------------------------------------------|
| **Identify** | Asset Management    | Detect rogue devices within enterprise network  |
| **Detect** | Anomalies and Events | Identify spoofed responses in LLMNR traffic     |
| **Respond**| Mitigation           | Remove malicious endpoints, block legacy protocols |

🌐 **ISO/IEC 27001:2013**

| Control Area            | Control ID | Description                                  |
|--------------------------|------------|----------------------------------------------|
| **Access Control**       | A.9.2      | Authentication & protocol configuration       |
| **Operations Security**  | A.12.4     | Logging and monitoring of network events      |
| **Communications Security** | A.13.1 | Protecting against unauthorized network access|

## ⚠ Disclaimer
```
All activities, tools, and techniques presented in this portfolio were performed exclusively within legally authorized,
controlled environments for the purpose of cybersecurity education and ethical research. This work adheres to established
ethical hacking standards, relevant legal regulations, and responsible disclosure protocols. Any application of these
methods outside of approved environments without explicit authorization is strictly prohibited and may be unlawful.

This repository aligns with HackTheBox’s content usage policies. This write-up is based on a retired Hack The Box
room called CampFire-1. It does not contain walkthroughs, solutions, or flag disclosures. All content is intended to
demonstrate professional development and applied learning through high-level summaries and overviews.
```

