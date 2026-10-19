Writeup for TryHackMe CTF Pickle Rick

The first step I scanned ports with nmap. As a result, two open ports were found.
```
nmap -sS -sV 10.114.186.45
Starting Nmap 7.94SVN ( https://nmap.org ) at 2026-10-19 15:01 UTC
Nmap scan report for ip-10-114-186-45.eu-central-1.compute.internal (10.114.186.45)
Host is up (0.00037s latency).
Not shown: 998 closed tcp ports (reset)
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 6.37 seconds
```

Then I navigate to the target machine's IP address in a browser and I got to the main page:
![](./screenshots/1.png)
