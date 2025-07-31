#################################
Getting Started with HackTheBox
#################################


Common Terms
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Enumeration + Vulnerability Assessment -> Web Footprinting (includes directory enumeration) -> Initial Foothold -> Privilege Escalation -> Reporting and Documentation  

Shell
===================================

A program used to take user input and pass it on to the OS to perform a certain function/task. 
Most common one -- ``bash`` (Bourne Again Shell) -- is an enhanced version of the original Unix system's shell program ``sh``. Others -- ``zsh``, ``tcsh``, ``ksh``, ``Fish shell``, etc.

https://academy.hackthebox.com/module/77/section/725

**There are three main types of shell connections:**

.. list-table:: Shell Types
   :header-rows: 1

   * - Shell Type
     - Description
   * - ``Reverse shell``
     - Initiates a connection back to a "listener" on our attack box. **----Write properly somewhere-----** The reason a reverse shell works is because of the firewall. we may not be able to connect to a machine on the internal network becasue of the firewall blocking our IP from interacting with the port, but the internal machine can send a connection to US instead.
   * - ``Bind shell``
     - "Binds" to a specific port on the target host and waits for a connection from our attack box. https://chatgpt.com/g/g-p-67a582636598819186ad6bf1a7e52bf5-divye/c/68556e42-2e9c-8002-85a4-ef0d76f15ed4
   * - ``Web shell``
     - Runs operating system commands via the web browser, typically not interactive or semi-interactive. It can also be used to run single commands (i.e., leveraging a file upload vulnerability and uploading a ``PHP`` script to run a single command.)

``Gaining shell access`` refers to getting **interactive** shell-level access on the exploited target. 

---

TCP and UDP
===================================

*(Filler: Section reserved for detailed TCP and UDP explanations if needed.)*

---

Ports
===================================

	- https://www.youtube.com/watch?v=g2fT-g9PX9o
	- https://www.stationx.net/common-ports-cheat-sheet/
	- https://web.archive.org/web/20240315102711/https://packetlife.net/media/library/23/common-ports.pdf
	- https://nullsec.us/top-1-000-tcp-and-udp-ports-nmap-default/

Ports are virtual points where network connections begin and end. They are software-based and managed by the host operating system. Ports are associated with a specific process or service and allow computers to differentiate between different traffic types. 

---

ssh
===================================

1. what is ssh?  
Secure Shell (SSH) is an excellent tool for securely connecting to a remote machine. It is a network protocol that runs on port ``22`` by default and provides users such as system administrators a secure way to access a computer remotely.  
SSH uses a client-server model, connecting a user running an SSH client application such as OpenSSH to an SSH server.

2. how can ssh be configured?  
SSH can be configured with password authentication or passwordless using public-key authentication using an SSH public/private key pair.

If you have the credentials of a user in plaintext, use::

   ssh username@server

After which you will be prompted for the password of the user.  

If you get a private ssh key and username, use::

   ssh '/path/to/private/key/file' username@server

The ``server`` mentioned in the above commands can either be an **IP address or a hostname**. 

**Note:** It is also possible to read local private keys on a compromised system or add our public key to gain SSH access to a specific user, as we'll discuss in a later section.

---

netcat (for windows, unix-like systems)
===================================

``nc`` is a tool used to connect to shells. It can be used to connect to any listening port and interact with the service running on that port.  

**Example**::

   grilledBread@htb[/htb]$ netcat 10.10.10.10 22
   SSH-2.0-OpenSSH_8.4p1 Debian-3

Output is a banner which tells us that ssh is running on port ``22``. This method of collecting information is called ``banner grabbing``.  
``nc`` can als9o be used for file transfer. 

For Windows systems, an alternative to the ``nc`` tool is the ``PowerCat`` tool.  
For unix systems, an alternative to the ``nc`` tool is the ``socat`` tool -- **unlike ``nc``**, it has features like forwarding ports and connecting to serial devices. Socat can also be used to upgrade a shell to a fully interactive TTY.  Shell connections established using ``socat`` are more stable.

nc options
----------------------

.. list-table:: Netcat Options
   :header-rows: 1

   * - Option
     - Meaning
   * - ``-l``
     - **Listen mode**: Netcat waits for incoming connections instead of initiating one.
   * - ``-v``
     - **Verbose**: Shows more output/details about what Netcat is doing. Useful for debugging or observing connections.
   * - ``-n``
     - **No DNS resolution**: Prevents Netcat from doing reverse DNS lookups. Faster and avoids leaking info. **If you don't use -n, your machine may try to resolve IPs or hostnames via DNS. This means a DNS query is sent — often to the target's internal DNS server, ISP DNS, or even public DNS (like Google or Cloudflare).**
   * - ``-p``
     - **Local port number**: Specifies the port number to listen on. Must be followed by a port (e.g., ``4444``).

socat syntax
----------------------

::

   socat [options] <address-type-1>:<address-1> <address-type-2>:<address-2>

Has a few features that netcat does not support, like forwarding ports and connecting to serial devices. Socat can also be used to upgrade a shell to a fully interactive TTY.  
A standalone binary of Socat can be transferred to a system after obtaining remote code execution to get a more stable reverse shell connection.

---

vim
===================================

Text editor that can be used for writing code or editing text files on Linux systems.

**Example**::

   grilledBread@htb[/htb]$ vim /etc/hosts

- If we want to create a new file, input the new file name, and Vim will open a new window with that file.  
- Once we open a file, we are in **read-only normal mode**. Press ``i`` to enter **insert mode** (shows ``-- INSERT --`` at the bottom).  
- After editing, press **Esc** to return to normal mode.

.. list-table:: Vim Commands
   :header-rows: 1

   * - Command
     - Description
   * - ``x``
     - Cut character
   * - ``dw``
     - Cut word
   * - ``dd``
     - Cut full line
   * - ``yw``
     - Copy word
   * - ``yy``
     - Copy full line
   * - ``p``
     - Paste

**Multiply commands** by prefixing with a number, e.g., ``4yw`` copies 4 words.

**Save/Quit Commands**

.. list-table::
   :header-rows: 1

   * - Command
     - Description
   * - ``:1``
     - Go to line number 1
   * - ``:w``
     - Write the file, save
   * - ``:q``
     - Quit
   * - ``:q!``
     - Quit without saving
   * - ``:wq``
     - Write and quit

---

tmux (Terminal Multiplexer)
===================================

### Difference between terminal emulators like terminator and tmux?

Imagine a web browser (Terminator):

- It provides the window on your desktop.
- You can open multiple tabs in the browser to view different websites.
- You can sometimes split the browser window to see two web pages side-by-side.

Now imagine a web application like Google Docs running inside one of those browser tabs (tmux):

- Google Docs itself allows you to have multiple documents open (like tmux windows) and perhaps even view different sections of a document side-by-side (like tmux panes).
- If you close the browser tab (lose connection), the document on Google Docs (your tmux session) is still there on the server, and you can open a new tab and log back in to continue working.

This basically means that ``tmux`` allows you to disconnect from the server while the processes in the terminal(s) continue running, and reconnect later.

.. image:: b59ebe4fde16e3f6f4c1b9b4b8dd03c7.png
.. image:: ba6ffa85877fab3508aee822220ee03b.png

Use ``Ctrl+B`` and then ``C`` to open a new tmux window.  
Windows are indexed at the bottom of the terminal pane.  

- Switch to each window: prefix + window number (e.g., 0 or 1)  
- Split a window vertically: prefix + ``Shift+%``

---

Special Port Numbers
===================================

Port numbers range from 1 to 65,535, with the range of well-known ports 1 to 1,023 being reserved for privileged services.  
**Port 0** is reserved and treated as a wildcard.  
If anything attempts to bind to port 0, it will bind to the next available port above 1,024.

---

Nmap
===================================

Default scan is quick because Nmap will only scan the 1,000 most common ports by default.  
By default, **Nmap will conduct a TCP scan unless specifically requested to perform a UDP scan.**

**Example**::

   grilledBread@htb[/htb]$ nmap 10.129.42.253

   Starting Nmap 7.80 ( https://nmap.org ) at 2021-02-25 16:07 EST
   Nmap scan report for 10.129.42.253
   Host is up (0.11s latency).
   Not shown: 995 closed ports
   PORT    STATE SERVICE
   21/tcp  open  ftp
   22/tcp  open  ssh
   80/tcp  open  http
   139/tcp open  netbios-ssn
   445/tcp open  microsoft-ds

**STATE = filtered** means:

- Firewall or IDS is preventing our IP from determining port state.
- Attacker has no way to know if the packet was dropped or lost.

---

### Nmap options

.. list-table:: Common Nmap Options
   :header-rows: 1

   * - Option
     - Description
     - Use Case
     - Requires Root?
   * - ``-sC``
     - Run **default NSE scripts** (same as ``--script=default``)
     - Quick service/script-based enumeration
     - Yes (if combined with ``-sS``)
   * - ``--script``
     - Run **specific NSE scripts** or categories
     - Custom recon or vuln checks
     - Yes (if used with ``-sS`` or raw packet scans)
   * - ``-sV``
     - **Service version detection**
     - Identify running services & versions
     - Yes (for full accuracy)
   * - ``-p-``
     - Scan **all 65535 TCP ports**
     - Full port coverage (deep recon)
     - No
   * - ``-p <range>``
     - Scan **specific port(s)**
     - Target known/suspected ports
     - No
   * - ``-F``
     - **Fast scan** (top 100 ports)
     - Quick recon, low noise
     - No
   * - ``-Pn``
     - Skip ping; assume hosts are **up**
     - Bypass ICMP blocks (firewalled hosts)
     - No
   * - ``-sn``
     - **Ping scan only** (no port scan)
     - Find live hosts only
     - Yes (for ARP or ICMP ping)
   * - ``-sS``
     - **SYN scan** (stealthy, default)
     - Fast, stealthy TCP scan
     - Yes
   * - ``-sT``
     - **TCP connect scan** (3-way handshake)
     - For non-root users or filtered networks
     - No
   * - ``-sU``
     - **UDP scan**
     - Discover UDP services like DNS, SNMP
     - Yes

Nmap also gives us the ability to scan our target with the aggressive option (``-A``).  
``-A`` includes:

- service detection (``-sV``)  
- OS detection (``-O``)  
- traceroute (``--traceroute``)  
- default NSE scripts (``-sC``)

**What -sC Scripts Do**

- Service discovery  
- Version detection enhancement  
- Common vulnerability checks  
- Metadata extraction (SSL certs, HTTP headers)

---

Web Enumeration
===================================

*(Filler: Placeholder for detailed web enumeration notes.)*

---

Public Exploits
===================================

*(Filler: Placeholder for public exploit references.)*

---

Privilege Escalation
===================================

https://academy.hackthebox.com/module/77/section/844

**PrivEsc Checklists**

Once we gain initial access to a box, we want to thoroughly enumerate the box to find any potential vulnerabilities we can exploit to achieve a higher privilege level. We can find many checklists and cheat sheets online that have a collection of checks we can run and the commands to run these checks. One excellent resource is `HackTricks <https://book.hacktricks.xyz/>`_, which has an excellent checklist for both `Linux <https://book.hacktricks.wiki/en/linux-hardening/linux-privilege-escalation-checklist.html>`_ and `Windows <https://book.hacktricks.wiki/en/windows-hardening/checklist-windows-privilege-escalation.html>`_ local privilege escalation. Another excellent repository is `PayloadsAllTheThings <https://github.com/swisskyrepo/PayloadsAllTheThings>`_, which also has checklists for both `Linux <https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Linux%20-%20Privilege%20Escalation.md>`_ and `Windows <https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Windows%20-%20Privilege%20Escalation.md>`_. We must start experimenting with various commands and techniques and get familiar with them to understand multiple weaknesses that can lead to escalating our privileges.


---

Enumeration Scripts
===================================

Many of the above commands may be automatically run with a script to go through the report and look for any weaknesses. We can run many scripts to automatically enumerate the server by running common commands that return any interesting findings. Some of the common Linux enumeration scripts include LinEnum and linuxprivchecker, and for Windows include Seatbelt and JAWS.

Another useful tool we may use for server enumeration is the Privilege Escalation Awesome Scripts SUITE (PEASS), as it is well maintained to remain up to date and includes scripts for enumerating both Linux and Windows
---

Kernel Exploits
===================================

*(Filler: Placeholder for kernel exploit notes.)*

---

Vulnerable Software
===================================

*(Filler: Placeholder for vulnerable software notes.)*

---

User Privileges
===================================

*(Filler: Placeholder for user privilege enumeration notes.)*

---

Scheduled Tasks
===================================

*(Filler: Placeholder for scheduled task notes.)*

---

Exposed Credentials
===================================

*(Filler: Placeholder for exposed credentials notes.)*

---

SSH Keys
===================================

*(Filler: Placeholder for SSH key persistence notes.)*

---

Transferring Files
===================================

*(Filler: Placeholder for file transfer notes.)*
