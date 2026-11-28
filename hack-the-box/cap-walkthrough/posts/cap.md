---
Title: "Cap HTB Walkthrough"
Date: 25th November 2026
Tags:
  - Hack The Box
  - IDOR
  - PCAP analysis
  - Linux capabilities
Platform: "Hack The Box"
Difficulty: "Easy"

---
<img width="298" height="137" alt="Screenshot_20261128_100714" src="https://github.com/user-attachments/assets/5f5f9d3d-62e4-48dd-b368-add50b84620f" />


## Summary

Cap is a Hack The Box machine where I assessed exposed network services and a web-based security-audit application. The main weaknesses in my notes were an IDOR that exposed packet captures and a later privilege-escalation opportunity involving file capabilities. I recovered a credential from plaintext FTP traffic, reused it for SSH, found the user flag, and ultimately obtained root.

## Recon

### Connectivity check

**What I tried:** I checked whether the target was reachable with `ping`.

```text
ping <TARGET_IP>
PING <TARGET_IP> (<TARGET_IP>) 56(84) bytes of data.
64 bytes from <TARGET_IP>: icmp_seq=1 ttl=63 time=328 ms
64 bytes from <TARGET_IP>: icmp_seq=2 ttl=63 time=249 ms
64 bytes from <TARGET_IP>: icmp_seq=3 ttl=63 time=374 ms
^C
--- <TARGET_IP> ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 3002ms
rtt min/avg/max/mdev = 249.107/337.493/398.333/56.912 ms
```

### Service scan

**What I tried:** I used Nmap to identify exposed services.

```text
nmap -sS -sV -sC -O -oN nmap.txt -v <TARGET_IP>
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.2 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey:
|   3072 fa:80:a9:b2:ca:3b:88:69:a4:28:9e:39:0d:27:d5:75 (RSA)
|   256 96:d8:f8:e3:e8:f7:71:36:c5:49:d5:9d:b6:a4:c9:0c (ECDSA)
|_  256 3f:d0:ff:91:eb:3b:f6:e1:9f:2e:8d:de:b3:de:b2:18 (ED25519)
80/tcp open  http    gunicorn
| fingerprint-strings:
|   FourOhFourRequest:
|     HTTP/1.0 404 NOT FOUND
|     Server: gunicorn
```

**What the output showed:** The scan identified FTP, SSH, and a web server on port 80.

**Why it worked:** The scan gave me a short list of services to investigate. 

## Foothold

### Web application and IDOR

**What I tried:** I browsed the web application, which appeared to be a network-security audit tool, and inspected its data area. The literal route is withheld here.


<img width="1920" height="1080" alt="4" src="https://github.com/user-attachments/assets/dc1ae46e-8f9c-4bef-b446-2aafec2cc148" />



**What the output showed:** The site displayed network status and IP information. I found an IDOR that exposed a security snapshot and links to packet captures.


<img width="1920" height="527" alt="5" src="https://github.com/user-attachments/assets/cae1d191-e7e2-4202-bc55-ffb361ce9258" />


<img width="1920" height="527" alt="6" src="https://github.com/user-attachments/assets/5a575fc7-5bba-4611-9a5e-30feebb598ae" />



**Why it worked:** The application did not adequately restrict access to the referenced data. The Security Snapshot feature loaded packet data by ID in the URL (/data/<id>). The default, /data/5, came back empty — zero packets. Checking other IDs lower down, /data/1 had 695 packets sitting in it, belonging to another session entirely. The app never checked whether the ID I asked for was actually mine before handing the data back.

### FTP traffic and SSH access

**What I tried:** I analyzed the downloaded packet captures in Wireshark. I also tried anonymous FTP access, but it failed.


<img width="1920" height="1072" alt="8" src="https://github.com/user-attachments/assets/d8cc60ce-7db1-49ca-8097-e863b718b5c6" />


**What the output showed:** One packet contained FTP credentials in plaintext. I reused those credentials for SSH and reached the machine. I found the user flag; its value is not reproduced:


<img width="1920" height="823" alt="9" src="https://github.com/user-attachments/assets/42b9d0ab-6450-4f5a-9dbb-5552defa1691" />



**Why it worked:** The packet capture exposed a credential, and the same credential was accepted by SSH.

## Privilege Escalation

**What I tried:** I performed local enumeration which didn't give any value information, then i used LinPEAS for more detailed findings. I researched a file's capabilities I noticed and followed a capability-based escalation approach.


<img width="1908" height="171" alt="pasted file" src="https://github.com/user-attachments/assets/1aa083a7-eda8-405d-bc36-59634dbf4d3d" />


```text
cd /usr/bin
ls -al python3.8
-rwxr-xr-x 1 root root 5486384 Jan 27  2021 python3.8
./python3.8 -c 'import os; os.setuid(0); os.system("/bin/bash")'
id
uid=0(root) gid=1001([nathan]) groups=1001([nathan])
cd /root
ls
root.txt  snap
cat root.txt
```


**What the output showed:** LinPEAS drew my attention to files with interesting capabilities. I then obtained root privileges.

**Why it worked:** LinPEAS listed python3.8 as carrying the cap_setuid capability, with the effective and permitted flags set. That capability lets the binary call setuid() to any UID — including root — without needing the SUID bit at all. Spawning a shell from Python and calling os.setuid(0) before os.system("/bin/bash") used exactly that capability to drop into a root shell.

## Where I got stuck

- Anonymous FTP login did not work. 
- My initial local enumeration did not reveal a useful escalation path.

## Lessons Learned

- Check whether network services expose useful entry points before focusing on one service.
- Enforce object-level authorization for downloaded data; a data reference should not grant access to another user's capture.
- Avoid sending credentials in plaintext protocols, and do not reuse credentials across services.
- Review file capabilities during authorized local security assessments, but record the exact file, capability, and validation evidence so the escalation path can be reviewed.
