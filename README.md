# Standalone Challenge Writeups ![Repository Status](https://img.shields.io/badge/repository-active-brightgreen?style=for-the-badge)

![Banner](./IMAGES/standalone_challenges_img.png?raw=true)

![License](https://img.shields.io/badge/License-CC_BY_4.0-green)
![Writeups](https://img.shields.io/badge/Published_Writeups-4-blue)
[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-FFDD00?logo=buy-me-a-coffee&logoColor=black)](https://www.buymeacoffee.com/v4l1k4hn)
![GitHub User's stars](https://img.shields.io/github/stars/valikahn?style=flat&logo=github)
![Discord](https://img.shields.io/discord/521382216299839518?style=flat&logo=discord&color=purple)
[![TryHackMe Profile](https://img.shields.io/badge/TryHackMe-Certificates-black)](https://tryhackme.com/p/V4L1K4HN?tab=certificates)

This repository contains my personal writeups, technical notes and walkthroughs for standalone challenges completed on [TryHackMe](https://tryhackme.com/).

Unlike my learning-path repositories, the challenges documented here are not tied to a single curriculum, role or progression route. They are selected individually from the [TryHackMe challenge collection](https://tryhackme.com/challenges) according to interest, available time and the skills I want to practise.

The purpose of this repository is to record the reasoning, methodology, commands, mistakes, troubleshooting and lessons behind each completed challenge. It is intended to support revision, portfolio development and future technical reference without becoming a flag list or an answer bank.

## About This Repository

Each writeup is based on work completed within an authorised TryHackMe training environment. The machines, applications, networks, files and scenarios covered are intentionally designed for cyber security education and controlled experimentation.

Testing may be performed through the TryHackMe AttackBox or from my own Kali Linux virtual machine connected to the TryHackMe network through OpenVPN. Where relevant, writeups may refer to temporary target addresses, VPN interface addresses, hostnames and other lab-specific information.

Because this is a standalone collection, there is no fixed order, completion target or publication schedule. Challenges may cover unrelated technologies and may range from short, focused exercises to multi-stage compromises requiring extensive enumeration, exploitation and privilege escalation.

> [!IMPORTANT]
> This repository will **never** contain material taken from TryHackMe professional certification examinations, private assessments or other restricted content.
>
> Flags, answer strings, passwords, cracked credentials, private keys, tokens and other challenge-sensitive values will not be published. Where these values are required to explain the methodology, they will be replaced with a clearly marked placeholder such as `<REDACTED>`, `THM{...}`, `<TARGET_IP>` or `<TUN0_IP>`.
>
> These notes are intended to explain how a challenge was approached, not to replace the learning process. Attempt the challenge independently first. The mistakes, dead ends and small breakthroughs are usually where the real learning happens.

## What a Writeup May Include

The content and depth of each writeup will depend on the challenge, its intended learning objective and the route taken to complete it. A writeup may include:

- Challenge overview and learning objectives;
- Scope and authorised target information;
- Attacker and target IP address details;
- Environment preparation and connectivity checks;
- Passive and active reconnaissance;
- Port, service and operating system enumeration;
- Web application mapping and content discovery;
- Network service assessment;
- Request and response analysis;
- Vulnerability identification and supporting research;
- Exploit selection, modification and validation;
- Initial access methodology;
- Shell stabilisation and session management;
- Linux or Windows privilege escalation;
- Active Directory enumeration and attack techniques;
- Password, hash and authentication testing;
- Credential harvesting within the authorised lab;
- Lateral movement, pivoting and tunnelling;
- Post-exploitation enumeration;
- Custom scripts, payload logic or automation;
- Sanitised commands and selected output;
- Alternative approaches and failed attempts;
- Troubleshooting notes;
- Technical findings and potential impact;
- Detection opportunities and defensive recommendations;
- Lessons learned; and
- References and supporting documentation.

Not every writeup will contain every section. The aim is to document the useful reasoning behind the solution rather than force every challenge into the same rigid template.

## Challenge Collection

**Challenge source:** [TryHackMe Standalone Challenges](https://tryhackme.com/challenges)

This repository is an open-ended collection rather than a formal learning path. Challenges may be chosen because they:

- Reinforce a technique encountered in another room;
- Introduce an unfamiliar service, platform or vulnerability class;
- Provide practice in enumeration and technical decision-making;
- Require several weaknesses to be chained together;
- Offer a realistic troubleshooting problem;
- Support preparation for a broader learning objective; or
- Are simply interesting enough to investigate.

There is no promise that challenges will be completed in difficulty order, release order or by category. A challenge may be added when it becomes useful, relevant or irresistibly annoying enough that it must be solved.

## Published Writeups

| No. | Challenge | Difficulty | Status |
| ---: | --- | :---: | :---: |
| 1 | [SQHell](https://tryhackme.com/room/sqhell) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/sqhell_challenge.md) |
| 2 | [Attacktive Directory](https://tryhackme.com/room/attacktivedirectory) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/attacktive_directory_challenge.md) |
| 3 | [That's The Ticket](https://tryhackme.com/room/thatstheticket) | ![Medium](https://img.shields.io/badge/Medium-orange) | ![Issue_Reported](https://img.shields.io/badge/Issue-Reported-orange) |
| 4 | [TryPwnMe One](https://tryhackme.com/room/trypwnmeone) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/trypwnmeone_challenge.md) |
| 5 | [TryPwnMe Two](https://tryhackme.com/room/trypwnmetwo) | ![Hard](https://img.shields.io/badge/Hard-red) | ![Planned](https://img.shields.io/badge/Planned-lightgrey) |
| 6 | [Matryoshka](https://tryhackme.com/room/matryoshka) | ![Medium](https://img.shields.io/badge/Medium-orange) | [![Complete](https://img.shields.io/badge/Complete-brightgreen)](./WRITEUPS/matryoshka_challenge.md) |
| 7 | [Kitty](https://tryhackme.com/room/kitty) | ![Medium](https://img.shields.io/badge/Medium-orange) | ![Review](https://img.shields.io/badge/Under%20Review-blue) |

TryPwnMe One
> [!NOTE]
> New entries will be added when a challenge has been completed, reviewed, sanitised and prepared for publication. Repository activity may vary depending on study priorities and available time.

## Status Key

| Status | Meaning |
| --- | --- |
| ![Planned](https://img.shields.io/badge/Planned-lightgrey) | The room has been selected but documentation has not started. |
| ![In Progress](https://img.shields.io/badge/In%20Progress-yellow) | The room is currently being completed and documented. |
| ![Complete](https://img.shields.io/badge/Complete-brightgreen) | The writeup has been published. |
| ![Review](https://img.shields.io/badge/Under%20Review-blue) | The writeup is complete but is being checked for accuracy, clarity and sensitive content. |
| ![Revisit](https://img.shields.io/badge/Revisit-purple) | The challenge was completed, but the writeup may be expanded or approached again. |
| ![Issue_Reported](https://img.shields.io/badge/Issue-Reported-orange) | A lab or room issue has been encountered and reported. |
| ![Archived](https://img.shields.io/badge/Archived-red) | The room or writeup is no longer actively maintained. |

## Tools Commonly Used

The exact toolkit will vary between challenges. Tools may include:

| Reconnaissance and Enumeration | Web and Application Testing | Access and Exploitation | Post-Exploitation and Analysis |
| --- | --- | --- | --- |
| [Nmap](https://nmap.org/) | [Burp Suite](https://www.kali.org/tools/burpsuite/) | [Metasploit Framework](https://www.kali.org/tools/metasploit-framework/) | [LinPEAS and WinPEAS](https://www.kali.org/tools/peass-ng/) |
| [RustScan](https://github.com/RustScan/RustScan) | [OWASP ZAP](https://www.kali.org/tools/zaproxy/) | [Netcat](https://www.kali.org/tools/netcat/) | [pspy](https://www.kali.org/tools/pspy/) |
| [Gobuster](https://www.kali.org/tools/gobuster/) | [ffuf](https://www.kali.org/tools/ffuf/) | [Impacket](https://www.kali.org/tools/impacket-scripts/) | [BloodHound](https://www.kali.org/tools/bloodhound/) |
| [Feroxbuster](https://www.kali.org/tools/feroxbuster/) | [Nikto](https://www.kali.org/tools/nikto/) | [Evil-WinRM](https://www.kali.org/tools/evil-winrm/) | [NetExec](https://www.kali.org/tools/netexec/) |
| [enum4linux-ng](https://www.kali.org/tools/enum4linux-ng/) | Browser Developer Tools | [Hydra](https://www.kali.org/tools/hydra/) | [Chisel](https://www.kali.org/tools/chisel/) |
| [smbclient](https://www.kali.org/tools/samba/#smbclient-1) | [sqlmap](https://www.kali.org/tools/sqlmap/) | [John the Ripper](https://www.kali.org/tools/john/) | [Sshuttle](https://www.kali.org/tools/sshuttle/) |
| [dig](https://www.kali.org/tools/bind9/#dig) | [CeWL](https://www.kali.org/tools/cewl/) | [Hashcat](https://www.kali.org/tools/hashcat/) | [GTFOBins](https://gtfobins.github.io/) |
| [WhatWeb](https://www.kali.org/tools/whatweb/) | Custom Python or Bash scripts | SSH and OpenSSH | [LOLBAS](https://lolbas-project.github.io/) |

The inclusion of a tool does not mean it is required for every challenge or that it is the only suitable option. Tool choice should follow the evidence discovered during enumeration, not habit.

## Methodology

My general approach is:

1. Confirm that the challenge is an authorised TryHackMe environment.
2. Record the target IP address and the attacker VPN interface address where applicable.
3. Review the challenge description without assuming that it reveals the intended route.
4. Establish a clean baseline with broad reconnaissance and service enumeration.
5. Investigate each exposed service methodically and record useful observations.
6. Build and test hypotheses instead of firing tools at the target without a reason.
7. Research identified technologies, versions, behaviours and potential weaknesses.
8. Validate findings carefully and select the least disruptive route that meets the objective.
9. Gain initial access and stabilise the session where possible.
10. Enumerate the local system, users, permissions, services, scheduled tasks and trust relationships.
11. Escalate privileges or move laterally one stage at a time.
12. Capture the required evidence without publishing the final flag or answer.
13. Reconstruct the complete attack path and remove unnecessary noise from the writeup.
14. Record mistakes, alternative routes, defensive lessons and remediation opportunities.
15. Review and sanitise the writeup before publication.

The methodology is intentionally flexible. A web challenge, forensic exercise, reverse-engineering task or Active Directory environment will not follow exactly the same sequence.

## Content and Spoiler Policy

These writeups may contain spoilers, command output, vulnerability details and substantial portions of an attack or investigation path. Anyone actively completing a challenge should attempt it independently before reading the corresponding writeup.

This repository will **not** intentionally publish:

- TryHackMe flags or final answer strings;
- Passwords or cracked credentials;
- Reusable session tokens, cookies or API keys;
- Private keys or sensitive certificates;
- Professional certification examination material;
- Private or restricted assessment content;
- Copied room instructions or substantial portions of TryHackMe material;
- Material that TryHackMe or a challenge author has asked learners not to share; or
- Unnecessarily destructive steps where a safer demonstration is sufficient.

> [!CAUTION]
> A writeup may reveal the vulnerable service, exploitation route or privilege-escalation method even when the final answer is redacted. Treat every published writeup as containing spoilers.

Placeholders such as `<TARGET_IP>`, `<TUN0_IP>`, `<USERNAME>`, `<PASSWORD>`, `<HASH>` and `<REDACTED>` are used to explain the process while protecting challenge-sensitive information.

If restricted or sensitive information is included accidentally, please report it through the repository's [GitHub Discussions](https://github.com/Valikahn/TryHackMe-Standalone-Challenge/discussions) area so it can be reviewed and removed.

## Ethical Use Disclaimer

These writeups are for educational purposes only and are based on authorised TryHackMe lab environments.

All tools, commands, techniques, and methodologies referenced in these writeups were used within controlled training environments where permission was provided by the owner and/or operator of the lab platform. The systems discussed are intentionally vulnerable machines designed for cybersecurity learning, practice, and assessment.

Do not use these techniques, tools, or methods against systems, networks, applications, or services that you do not own or do not have explicit written permission to test. Unauthorised access, scanning, exploitation, or disruption of systems is illegal and unethical.

The tools and methods listed in this repository are examples of approaches used during specific rooms or learning exercises. They are not the only possible solutions, and other tools, techniques, or workflows may be used depending on the target environment, room design, and individual methodology.

## Moderation Policy for Abusive Exploit Requests

This repository is intended solely for authorised cybersecurity education, controlled laboratory exercises, Capture the Flag challenges, and legitimate defensive research.

Requests for assistance that appear to involve unauthorised access, real-world exploitation, credential theft, malware deployment, data theft, service disruption, evasion of security controls, or harm against individuals, organisations, systems, or networks will not be supported.

This includes, but is not limited to:

- Requests to attack systems without clear authorisation
- Attempts to obtain or misuse passwords, tokens, session cookies, API keys, or other credentials
- Requests involving ransomware, destructive malware, persistence, botnets, phishing, or data exfiltration
- Instructions intended to conceal malicious activity or evade detection
- Threats, harassment, extortion, doxxing, or targeted abuse
- Attempts to adapt educational material for use against real-world targets

Abusive or suspicious requests may be documented, preserved, blocked, and reported to the relevant platform, service provider, repository host, organisation, or lawful authority where appropriate.

By accessing or interacting with this repository, you agree to use its content only within environments that you own or are explicitly authorised to test.

Educational context does not excuse unlawful or harmful activity.

***No authorisation means no testing.***

## Accuracy and Maintenance

TryHackMe periodically updates, replaces or retires rooms and learning paths. Links, room sequences and path content may therefore change after a writeup is published.

Each writeup should be treated as a record of the room as it appeared on the date documented. Where a material change is identified, the relevant page may be updated or marked as archived.

If you notice a broken link, outdated instruction, formatting problem, technical error or any other noticeable issue within a writeup, please report it through the GitHub repository's **[Discussions](https://github.com/Valikahn/TryHackMe-Web-Application-Red-Teaming/discussions)** tab. When providing feedback, include the name of the affected writeup, a brief description of the problem and, where possible, the relevant section or line.

Constructive corrections are welcome and help keep the repository accurate, useful and maintainable.

## Connect With Me

Thanks for checking out my TryHackMe writeups. These notes form part of my ongoing cybersecurity learning journey, where I document rooms, techniques, tools, mistakes and lessons learned while working through different challenges.

You can view my TryHackMe profile here:

[TryHackMe Profile - V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)

I am also active within cybersecurity learning communities, including Discord, where I discuss labs, tools, methodologies and general security topics with other learners and practitioners.

Feel free to follow my progress, compare approaches or get in touch if you are working through similar rooms.

Walkthrough requests are always welcome, although publication will depend on my availability and whether sharing the content complies with the platform's rules.

Created by V4L1K4HN as part of my cybersecurity learning journey through TryHackMe.

## License

Unless otherwise stated, the original written content in this repository is licensed under the [Creative Commons Attribution 4.0 International License](https://creativecommons.org/licenses/by/4.0/legalcode.en).

Copyright © 2026 V4L1K4HN.

You may share and adapt the licensed material for any purpose, including commercially, provided that:

- appropriate credit is given to V4L1K4HN;
- a link to the license is provided; and
- any changes made to the original material are clearly indicated.

This license applies only to original material created by the repository author. TryHackMe content, branding, room materials, third-party software, trademarks, externally sourced material, and any other third-party intellectual property remain subject to their respective owners' terms and licenses.

See the [LICENSE](./LICENSE) file for the complete legal terms.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
