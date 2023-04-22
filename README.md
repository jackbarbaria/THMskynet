<h1>TryHackMe Skynet CTF</h1>

<h2>Overview</h2>
I’ve been studying for the eLearnSecurity Junior Penetration Tester (eJPTv2) Certification and recently completed INE’s lessons on SMB Enumeration. To supplement and practice what I learned, I searched TryHackMe for rooms that are tagged with SMB or samba and found Skynet! This was a fun Terminator themed CTF challenge that took me three or four hours to complete. I am glad I took notes throughout the process, so I could continue where I left off after taking breaks.
<br />

<h2>Tools Used</h2>

- <b>Nmap</b> 
- <b>Smbclient</b>
- <b>Hydra</b>
- <b>Gobuster</b>
- <b>Burp Suite/Burp Intruder</b>
- <b>MSFvenom Payload Creator</b>
- <b>Python’s http.server module</b>
- <b>Metasploit</b>
- <b>Linpeas</b>

<h2>My journey through Skynet:</h2>

There are five tasks to complete in this module:
1.	What is Miles password for his emails?
2.	What is the hidden directory?
3.	What is the vulnerability called when you can include a remote file for malicious purposes?
4.	What is the user flag?
5.	What is the root flag?

***
<b>Port scanning and SMB Enumeration with Nmap</b>
***

I used Nmap to identify open ports on the target machine. By default, Nmap scans the 1000 most frequently used ports. I also used the -sV option so that Nmap would attempt to identify the services running on any discovered port.

```
┌──(root㉿kali)-[~]
└─# nmap -sV 10.10.71.63
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-20 15:20 UTC
Nmap scan report for ip-10-10-71-63.eu-west-1.compute.internal (10.10.71.63)
Host is up (0.011s latency).
Not shown: 994 closed tcp ports (reset)
PORT    STATE SERVICE     VERSION
22/tcp  open  ssh         OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
80/tcp  open  http        Apache httpd 2.4.18 ((Ubuntu))
110/tcp open  pop3        Dovecot pop3d
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
143/tcp open  imap        Dovecot imapd
445/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
MAC Address: 02:48:CA:C0:AC:F9 (Unknown)
Service Info: Host: SKYNET; OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 14.66 seconds </blockquote>
```

Port 80 tells me there is likely a website being hosted. But a quick look shows there is not much to see on this homepage. The source code did not reveal much, either.
<br />
<img src="https://i.imgur.com/6fdt0Sc.png" height="80%" width="80%" alt="SkyNet HomePage"/>
<br />

Port 445 shows there may be SMB shares to explore. I used Nmap again with the smb-enum-shares script to see what I could find:

```
┌──(root㉿kali)-[~]
└─# nmap 10.10.71.63 -p 139,445 --script smb-enum-shares
Starting Nmap 7.93 ( https://nmap.org ) at 2023-04-20 15:31 UTC
Nmap scan report for ip-10-10-71-63.eu-west-1.compute.internal (10.10.71.63)
Host is up (0.00024s latency).

PORT    STATE SERVICE
139/tcp open  netbios-ssn
445/tcp open  microsoft-ds
MAC Address: 02:48:CA:C0:AC:F9 (Unknown)

Host script results:
| smb-enum-shares: 
|   account_used: guest
|   \\10.10.71.63\IPC$: 
|     Type: STYPE_IPC_HIDDEN
|     Comment: IPC Service (skynet server (Samba, Ubuntu))
|     Users: 1
|     Max Users: <unlimited>
|     Path: C:\tmp
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.71.63\anonymous: 
|     Type: STYPE_DISKTREE
|     Comment: Skynet Anonymous Share
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\srv\samba
|     Anonymous access: READ/WRITE
|     Current user access: READ/WRITE
|   \\10.10.71.63\milesdyson: 
|     Type: STYPE_DISKTREE
|     Comment: Miles Dyson Personal Share
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\home\milesdyson\share
|     Anonymous access: <none>
|     Current user access: <none>
|   \\10.10.71.63\print$: 
|     Type: STYPE_DISKTREE
|     Comment: Printer Drivers
|     Users: 0
|     Max Users: <unlimited>
|     Path: C:\var\lib\samba\printers
|     Anonymous access: <none>
|_    Current user access: <none>

Nmap done: 1 IP address (1 host up) scanned in 0.89 seconds
```
<br />

***
<b>Exploring shares with SMBclient</b><br />
***
The results from the script tells me that a NULL user can connect to anonymous. From here I used the Smbclient to connect, browse, and download files from the share. I found an interesting file with the name “attention.txt.”

<br />
<img src="https://i.imgur.com/CwpAVNm.png" height="80%" width="80%" alt="Exploring with SMBClient"/>
<br />
This file contained the following clue:<br />

```
┌──(root㉿kali)-[~]
└─# cat attention.txt 
A recent system malfunction has caused various passwords to be changed. All skynet employees are 
required to change their password after seeing this.
-Miles Dyson
```
<br />
Another notable file was located in the "logs" folder, which appeared to be a list of passwords:
<br />
<img src="https://i.imgur.com/TfJhs5I.png" height="80%" width="80%" alt="Log1.txt File"/>
<br />

I attempted to use the login cracker program Hydra to see if any of those passwords matched the milesdyson share. Unfortunately, this did not work.
<br />
<img src="https://i.imgur.com/dCptaBr.png" height="80%" width="80%" alt="Hydra results"/>
<br />

***
<b>Digging into GoBuster and the website</b><br />
***
Gobuster uses wordlists to brute-force directories, files, and subdomains. I used Gobuster to enumerate the target website:
<br />
<img src="https://i.imgur.com/Wguoz1r.png" height="80%" width="80%" alt="Enumerating with Gobuster"/>
<br />
<br />
The directories admin, ai, and config led nowhere. However, Squirrelmail looked promising as it led to a login page. From here, I used Burp Suite which provides numerous tools for web application testing. The Burp Intruder tool was used to send the same HTTP request over and over again using milesdyson as the login, and the list passwords from log1.txt inserted into the password field.

<br />
<img src="https://i.imgur.com/Bu9rrjq.png" alt="Burp Suite"/>
<br />
<br />
One of the passwords generated a status of 300. All of the other passwords resulted in a status of 200. The one that was different is likely the valid password.
<br />
<img src="https://i.imgur.com/l2gqZDh.png" height="80%" width="80%" alt="Burp Intruder"/>
<br />
<br />
<b>We're in!!</> I completed the first challenge task of discovering Miles’ email password!
<br />
<br />
<img src="https://i.imgur.com/q4mTmcH.png" height="80%" width="80%" alt="Burp Intruder"/>
<br />
<br />

I opened the email with the subject “Samba Password reset.” This contained a password for the milesdyson share. I then used Smbclient to access this share and located a “notes” folder, which contained a file names “important.txt.”

```
┌──(root㉿kali)-[~]
└─# cat important.txt 

1. Add features to beta CMS /45kra24zxs28v3yd
2. Work on T-800 Model 101 blueprints
3. Spend more time with my wife
```

This pointed to the hidden folder which was Miles’ homepage and was answer to the second challenge task.


