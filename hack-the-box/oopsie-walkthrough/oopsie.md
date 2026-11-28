---
Title: "Oopsie HTB Walkthrough"
Date: 24th November 2026
Tags:
  - Hack The Box
  - web
  - IDOR
  - file upload
  - privilege escalation
Platform: "Hack The Box, Starting Point Tier 2"
Difficulty: "Easy (Starting Point Tier 2)"
---

<img width="308" height="149" alt="image" src="https://github.com/user-attachments/assets/f1d086a8-29fa-4110-b7fb-f0c372349638" />


## Summary

Oopsie is a Hack The Box machine in the Starting Point Tier 1 track. I found weak authorization controls in the web application and unsafe command handling in a privileged helper. I obtained a shell through the upload workflow, used a credential found in `db.php` to access a local account, and reached root.

## Recon

### Connectivity and services

**What I tried:** I checked connectivity with `ping` and used Nmap to identify open services and attempt OS fingerprinting.

**What the output showed:** The scan reported SSH on port 22 and HTTP on port 80.

```text
# Nmap 7.92 scan
Nmap scan report for <TARGET_IP>
22/tcp open  ssh   OpenSSH 7.6p1 Ubuntu 4ubuntu0.3
80/tcp open  http  Apache httpd 2.4.29 (Ubuntu)
```

<img width="1447" height="851" alt="0" src="https://github.com/user-attachments/assets/3b0de19e-5384-4fbc-8626-ea22c1a14267" />


**Why it mattered:** The results identified the web service to inspect and SSH as another exposed service.

### Web source inspection

**What I tried:** I browsed the site and inspected its page source.

**What the output showed:** A script tag disclosed `/cdn-cgi/login/`, which was not linked from the visible page.

<img width="1920" height="1080" alt="1" src="https://github.com/user-attachments/assets/642d274a-b8a9-478a-ad7a-25e52e0576cc" />


**Why it worked:** Reviewing the source exposed a login path that normal page navigation did not reveal.

## Foothold

### Guest access and upload authorization

**What I tried:** I attempted default credentials, then used the “Login as Guest” link and opened the uploads page.

**What the output showed:** The default credentials failed. As a guest, I saw “This action require super admin rights” when I tried to access uploads.

<img width="1920" height="1080" alt="2" src="https://github.com/user-attachments/assets/d930f8ec-6def-4316-a4e7-98cd6bc704b2" />



**Why it mattered:** The denial showed that the upload function was protected by a role check.

### Account and authorization behavior

**What I tried:** While logged in as guest, I inspected account data and browser request state.

**What the output showed:** The account view exposed another account’s Access ID, name, and email. The browser showed user and role cookie fields, and the application later granted super-admin access. The cookie contained user and role fields, visible below.
<img width="1847" height="403" alt="3" src="https://github.com/user-attachments/assets/7cca0450-10bb-46bd-be87-18f7c23f47d0" />

<img width="1920" height="1068" alt="4" src="https://github.com/user-attachments/assets/2336d107-96bb-407a-ac1d-53acb8a6baa8" />


**Why it mattered:** The behavior showed that account data and authorization state were not adequately protected server-side.

### Upload path and shell

**What I tried:** After reaching the upload page, I submitted a PHP payload. The first upload did not produce a session. I checked an image path and searched for hidden directories.

**What the output showed:** The uploads page accepted a file. Gobuster found `/uploads/` as a valid directory.

<img width="1920" height="670" alt="5" src="https://github.com/user-attachments/assets/971ef763-acd2-4856-9b42-06800610d33b" />


```text
gobuster dir -u http://<TARGET_IP>/ -w /usr/share/seclists/Discovery/Web-Content/common.txt
uploads  (Status: 301) --> <TARGET_IP>/uploads/
```

<img width="1916" height="658" alt="6" src="https://github.com/user-attachments/assets/f41f2998-4de8-4696-972c-ee1ddb90152b" />


```text
nc -nvlp 8888
Ncat: Connection from <TARGET_IP>.
Linux oopsie 4.15.0-76-generic ... x86_64 GNU/Linux
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ python3 -c 'import pty;pty.spawn("/bin/bash")'
```

**Why it worked:** Finding and accessing the stored file explained why the first upload alone did not produce a session.

### Application credential and SSH

I found a hardcoded credential in `db.php` and used it to connect over SSH as a local account. The password and account name are not reproduced. I found the user flag.

## Privilege Escalation

I found a root-owned SUID helper associated with the `bugtracker` group. The program accepted bug IDs 1–4; an error for other input indicated it tried to read a report.

``` text
robert@oopsie:/tmp$ export PATH=/tmp:$PATH
robert@oopsie:/tmp$ bugtracker
Provide Bug ID: 2
# python3 -c 'import pty;pty.spawn("/bin/bash")'
root@oopsie:/tmp# cd /root
root@oopsie:/root# cat root.txt
```

## Where I got stuck

Default credentials did not work, and my Intruder attempt was slow. My first upload did not produce a session; I found the stored-file directory later. My first setup for the SUID step also failed before I corrected it.

## Lessons Learned

- Inspect page source and intercepted requests when a page or feature is not linked in the visible site.
- Treat client-provided account and role data as untrusted; authorization must be enforced by the application.
- Confirm where an upload is stored and how it is accessed before assuming it ran.
- Review hardcoded application credentials and privileged helpers during authorized assessment.
