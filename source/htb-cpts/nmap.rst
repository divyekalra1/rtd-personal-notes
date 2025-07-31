#######
Nmap
#######

Starting out
^^^^^^^^^^^^^^

options
===================================

.. code-block:: console

   grilledBread@htb[/htb]$ nmap --help

   <SNIP>
   SCAN TECHNIQUES:
     -sS/sT/sA/sW/sM: TCP SYN/Connect()/ACK/Window/Maimon scans
     -sU: UDP Scan
     -sN/sF/sX: TCP Null, FIN, and Xmas scans
     --scanflags <flags>: Customize TCP scan flags
     -sI <zombie host[:probeport]>: Idle scan
     -sY/sZ: SCTP INIT/COOKIE-ECHO scans
     -sO: IP protocol scan
     -b <FTP relay host>: FTP bounce scan
   <SNIP>

For the TCP-SYN (``-sS``) type of scan, which is the default:

- If our target sends a **SYN-ACK** flagged packet back to us, Nmap detects that the port is **open**.
- If the target responds with an **RST** flagged packet, it indicates the port is **closed**.
- If Nmap does not receive a packet back, it will display it as **filtered**.  
  Depending on the firewall configuration, certain packets may be dropped or ignored.

**--------Explain other options here--------**

.. list-table:: Common Nmap Options
   :header-rows: 1
   :widths: 20 40 30 20

   * - Option
     - Description
     - Use Case
     - Requires Root?
   * - ``-sC``
     - Run **default NSE scripts** (same as ``--script=default``)
     - Quick service/script-based enumeration
     - Yes (if combined with ``-sS``)
   * - ``-sn``
     - **Ping scan only** (no port scan)
     - Find live hosts only
     - Yes (for ARP/ICMP ping)
   * - ``-sS``
     - **SYN scan** (stealthy, default if root)
     - Fast, stealthy TCP scan
     - Yes – Requires raw sockets
   * - ``-sT``
     - **TCP connect scan** (full 3-way handshake)
     - Non-root scans or filtered networks
     - No
   * - ``-sU``
     - **UDP scan**
     - Discover UDP services like DNS/SNMP
     - Yes – Uses raw UDP packets
   * - ``--script``
     - Run **specific NSE scripts** or categories
     - Custom recon or vuln checks
     - Yes (if combined with raw packet scans)
   * - ``-sV``
     - **Service version detection**
     - Identify running services & versions
     - Yes (full accuracy)
   * - ``-p-``
     - Scan **all 65535 TCP ports**
     - Full port coverage
     - No
   * - ``-p <range>``
     - Scan **specific ports**
     - Target known/suspected ports
     - No
   * - ``-F``
     - **Fast scan** (top 100 ports)
     - Quick recon, low noise
     - No
   * - ``-Pn``
     - Skip ping; assume hosts are **up**
     - Bypass ICMP blocks
     - No – Treats all hosts as online



Scanning Performance
===================================

Reference: `HackTheBox Academy <https://academy.hackthebox.com/module/19/section/105>`_

Difference between ``--initial-rtt-timeout`` and ``--max-rtt-timeout``:

``--initial-rtt-timeout``
--------------------------

- **What it does**: Sets the **initial timeout** value Nmap uses when waiting for a probe response.
- **Default**: 100 ms
- **Purpose**: Determines how long Nmap initially waits before assuming a packet is lost.

**Example**::

   nmap --initial-rtt-timeout 500ms <target>

**Initially** means:

- Nmap’s starting timeout before it measures real latency.
- Dynamic timing will adjust as the scan progresses.
- Initial RTT is the **first guess**; after a few probes, Nmap adapts.

``--max-rtt-timeout``
--------------------------

- **What it does**: Sets the **maximum timeout** Nmap can ever reach for waiting on a probe.
- **Default**: 1000 ms (1s)
- **Purpose**: Caps timeout growth to prevent very slow scans.

**Example**::

   nmap --max-rtt-timeout 2s <target>

Even if latency is high, Nmap won’t wait more than 2s for responses.



Packet Rates
===================================

If we know the **network bandwidth**, we can adjust Nmap’s packet rate to speed up scans.

- ``--min-rate <number>``: Send at least `<number>` packets per second.  
- Helps achieve faster scans but increases detection risk.

**Default Scan Example**::

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.default

   <SNIP>
   Nmap done: 256 IP addresses (10 hosts up) scanned in 29.83 seconds

**Optimized Scan Example**::

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.0/24 -F -oN tnet.minrate300 --min-rate 300

   <SNIP>
   Nmap done: 256 IP addresses (10 hosts up) scanned in 8.67 seconds



Default Timing Templates
===================================

Nmap offers **six timing templates** (``-T 0`` to ``-T 5``):

- ``-T 0`` — **paranoid**
- ``-T 1`` — **sneaky**
- ``-T 2`` — **polite**
- ``-T 3`` — **normal** *(default)*
- ``-T 4`` — **aggressive**
- ``-T 5`` — **insane**

Reference: `Nmap Timing Templates <https://nmap.org/book/performance-timing-templates.html>`_




Host Discovery
^^^^^^^^^^^^^^^

.. note::
   **Learning progression in this section**:
   ``-sn`` option → host discovery using ARP packets (default, noisy) →  
   host discovery using **ICMP packets only** (disabling ARP ping) →  
   advantages of ICMP vs ARP for discovery on local and remote networks.

Host discovery refers to **finding live systems on a network**.  
The most effective method is the **ICMP Echo Request**, which helps:

- Determine if the target is alive.
- Identify the system by analyzing its response behavior.



Example Scan
===================================

.. code-block:: console

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping 

   Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 00:12 CEST
   SENT (0.0107s) ICMP [10.10.14.2 > 10.129.2.18 Echo request (type=8/code=0) id=13607 seq=0] IP [ttl=255 id=23541 iplen=28 ]
   RCVD (0.0152s) ICMP [10.129.2.18 > 10.10.14.2 Echo reply (type=0/code=0) id=13607 seq=0] IP [ttl=128 id=40622 iplen=28 ]
   Nmap scan report for 10.129.2.18
   Host is up (0.086s latency).
   MAC Address: DE:AD:00:00:BE:EF
   Nmap done: 1 IP address (1 host up) scanned in 0.11 seconds

**Notes about the result above:**

- ``-PE`` ensures ICMP Echo Requests are sent.
- ``--packet-trace`` shows the actual packets sent and received.
- The **TTL value** of the reply (128) suggests the host may be running **Windows**.

Common default TTL values:

- **Windows**: 128
- **Linux**: 64
- **FreeBSD/macOS**: 64
- **Cisco devices**: 255

The ``--ttl`` option allows specifying the TTL for Nmap’s packets.  
- **TTL 1** → restricts scans to the local subnet  
- **Higher TTL** → reaches further networks  
- TTL analysis can also help detect **firewalls and IDS** based on response patterns.



Interpreting and Understanding Results
===================================

**Important:** Using ``-PE`` does **not guarantee** only ICMP packets are sent.

``What -PE Actually Does``
--------------------------

- ``-PE`` tells Nmap to use **ICMP Echo Request** packets for host discovery.
- Actual behavior depends on **context**, **privileges**, and **options used**.

``Cases Where Other Packets May Be Sent``
----------------------------------------

.. list-table:: Scenarios Affecting ICMP vs ARP Behavior
   :header-rows: 1
   :widths: 40 60

   * - Scenario
     - Explanation
   * - Scanning local subnet without disabling ARP
     - Nmap still sends **ARP requests** because it's faster and more reliable on LANs.
   * - ``--disable-arp-ping`` not used
     - ICMP is sent, but ARP may still be used on local networks.
   * - Other probes (``-PP``, ``-PM``) enabled
     - Multiple ICMP types (timestamp, netmask) may be sent together.
   * - ``-Pn`` used
     - Disables all host discovery, overriding ICMP ping.

**Key Insight:**  
On **local subnets**, ARP is faster and always works, so Nmap defaults to ARP.  
To **force ICMP instead of ARP**, use:

.. code-block:: console

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.18 -sn -oA host -PE --packet-trace --disable-arp-ping



Behavior for Remote Targets
===================================

If scanning **outside the local subnet**:

- ARP cannot cross routers.
- Nmap defaults to **ICMP Echo Requests** and may add timestamp (``-PP``) and netmask (``-PM``) probes.
- ICMP is used automatically for remote discovery.



Why Force ICMP Instead of ARP?
===================================

1. **Scanning Beyond the Local Subnet (Remote Hosts)**  
   - ARP works only on **Layer 2** (local subnet).  
   - ICMP (Layer 3) reaches across routers and multiple hops.

2. **Avoid Layer 2 Noise**  
   - ARP requests are **broadcasts**, seen by all devices.  
   - ICMP is **unicast**, reducing unnecessary traffic and alerts.

3. **Firewall and Network Testing**  
   - ICMP can be rate-limited, filtered, or allowed selectively.  
   - Using ICMP discovery helps identify **firewall rules and behavior**.

4. **Stealth and Evasion**  
   - ARP is noisy and local-only.  
   - ICMP can be **customized, spoofed, or fragmented** for stealth scans.

Verify Why Host Is Marked Alive
===================================

The ``--reason`` option explains why Nmap considered a host "up":

.. code-block:: console

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.18 -sn -oA host -PE --reason 

   Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 00:10 CEST
   SENT (0.0074s) ARP who-has 10.129.2.18 tell 10.10.14.2
   RCVD (0.0309s) ARP reply 10.129.2.18 is-at DE:AD:00:00:BE:EF
   Nmap scan report for 10.129.2.18
   Host is up, received arp-response (0.028s latency).
   MAC Address: DE:AD:00:00:BE:EF
   Nmap done: 1 IP address (1 host up) scanned in 0.03 seconds

To **see exactly what’s sent**, use:

.. code-block:: console

   sudo nmap -sn <target> --packet-trace

More info: `Nmap Host Discovery <https://nmap.org/book/host-discovery-strategies.html>`_


---

Host and Port Scanning
^^^^^^^^^^^^^^^^^^^^^^^^

Common Scan Options
===================================

.. list-table:: Nmap Scan Options
   :header-rows: 1
   :widths: 15 50 25 20

   * - Option
     - Description
     - Use Case
     - Requires Root?
   * - ``-sS``
     - **SYN scan** (stealthy, default if root)
     - Fast, stealthy TCP scan
     - Yes – Requires raw socket privileges
   * - ``-sT``
     - **TCP connect scan** (3-way handshake, default if user)
     - Non-root or filtered environments
     - No – Uses standard system calls

---

Port States and Their Meaning
===================================

.. list-table:: Nmap Port States
   :header-rows: 1
   :widths: 15 85

   * - State
     - Description
   * - ``open``
     - Connection to the scanned port is established (TCP, UDP datagrams, or SCTP associations).
   * - ``closed``
     - Port responds with **TCP RST**. Indicates port is reachable but closed.
   * - ``filtered``
     - No response or error received. Cannot confirm open or closed.
   * - ``unfiltered``
     - Seen in TCP-ACK scans. Port accessible but state cannot be determined.
   * - ``open|filtered``
     - No response received. Port may be open but filtered by a firewall.
   * - ``closed|filtered``
     - Occurs in IP ID idle scans. State cannot be determined.

---

Understanding Scan Behavior
===================================

**TCP SYN Scan (``-sS``)**

- Half-open scan (3-way handshake not fully established).
- Stealthier and faster.
- Default when run as **root**.

**TCP Connect Scan (``-sT``)**

- Full 3-way handshake.
- Noisy but reliable; triggers IDS/IPS.
- Default for **non-root** users.

**Key ICMP Insights**

- During TCP scans:
  - ICMP **Type 3 Code 3** = Port unreachable.
  - If host is confirmed alive, firewall likely blocking the port.
- During UDP scans:
  - ICMP **Type 3 Code 3** = Port **closed**.
  - Open ports may **not respond** unless the service is configured to reply.

---

TCP Scan Example
-----------------------------------

.. code-block:: console

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.28 -p 445 --packet-trace -n --disable-arp-ping -Pn

   Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 15:55 CEST
   SENT (0.0388s) TCP 10.129.2.28:52472 > 10.129.2.28:445 S ttl=49 id=21763 iplen=44 seq=1418633433 win=1024 <mss 1460>
   RCVD (0.0487s) ICMP [10.129.2.28 > 10.129.2.28 Port 445 unreachable (type=3/code=3) ] IP [ttl=64 id=20998 iplen=72 ]
   Nmap scan report for 10.129.2.28
   Host is up (0.0099s latency).

   PORT    STATE    SERVICE
   445/tcp filtered microsoft-ds
   MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

   Nmap done: 1 IP address (1 host up) scanned in 0.05 seconds

---

UDP Scan Example - Closed Port
-----------------------------------

.. code-block:: console

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 100 --reason

   Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:25 CEST
   SENT (0.0445s) UDP 10.10.14.2:63825 > 10.129.2.28:100 ttl=57 id=29925 iplen=28
   RCVD (0.1498s) ICMP [10.129.2.28 > 10.10.14.2 Port unreachable (type=3/code=3) ] IP [ttl=64 id=11903 iplen=56 ]
   Nmap scan report for 10.129.2.28
   Host is up, received user-set (0.11s latency).

   PORT    STATE  SERVICE REASON
   100/udp closed unknown port-unreach ttl 64
   MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

   Nmap done: 1 IP address (1 host up) scanned in  0.15 seconds

---

UDP Scan Example - Open|Filtered Port
-----------------------------------

.. code-block:: console

   grilledBread@htb[/htb]$ sudo nmap 10.129.2.28 -sU -Pn -n --disable-arp-ping --packet-trace -p 138 --reason

   Starting Nmap 7.80 ( https://nmap.org ) at 2020-06-15 16:32 CEST
   SENT (0.0380s) UDP 10.10.14.2:52341 > 10.129.2.28:138 ttl=50 id=65159 iplen=28
   SENT (1.0392s) UDP 10.10.14.2:52342 > 10.129.2.28:138 ttl=40 id=24444 iplen=28
   Nmap scan report for 10.129.2.28
   Host is up, received user-set.

   PORT    STATE         SERVICE     REASON
   138/udp open|filtered netbios-dgm no-response
   MAC Address: DE:AD:00:00:BE:EF (Intel Corporate)

   Nmap done: 1 IP address (1 host up) scanned in 2.06 seconds

---

Firewall and IDS/IPS Evasion
===================================

- **ACK scans (``-sA``)** are used for **firewall detection**, not port state detection.
- Helps identify filtering devices and firewall rules.

Reference: `HackTheBox Academy - Module 19 <https://academy.hackthebox.com/module/19/section/106>`_

---

Saving Nmap Scan Results
===================================

- ``-oA <filename>``: Saves **all output formats** (Nmap, GNMAP, XML).  
  Produces:

  - ``filename.nmap``
  - ``filename.gnmap``
  - ``filename.xml``

- XML results can be converted to HTML:

.. code-block:: console

   xsltproc filename.xml -o filename.html


---

Service Scanning
^^^^^^^^^^^^^^^^^^

Service Detection
===================================

- Use ``-sV`` to list services running on open ports.

**Example Usage**::

   sudo nmap <target> -sV

---

Banner Grabbing
===================================

- Banner grabbing involves collecting initial response banners from services to identify software and versions.
- Can be enhanced using **PSH flag** manipulations in crafted TCP packets.

Reference: `HTB Academy - Banner Grabbing <https://academy.hackthebox.com/module/19/section/103>`_

---

Aggressive Scan (``-A``)
===================================

- Combines multiple detection features:
  1. **Service detection (``-sV``)**
  2. **OS detection (``-O``)**
  3. **Traceroute (``--traceroute``)**
  4. **Default NSE scripts (``-sC``)**

Example::

   sudo nmap <target> -A

---

Nmap Scripting Engine (NSE)
===================================

- NSE allows **custom interaction with services** to extract information or exploit vulnerabilities.
- Nmap includes **14 categories of scripts**.

.. list-table:: NSE Script Categories
   :header-rows: 1
   :widths: 15 85

   * - Category
     - Description
   * - ``auth``
     - Determination of authentication credentials.
   * - ``broadcast``
     - Host discovery by broadcasting; discovered hosts can be added to scans.
   * - ``brute``
     - Performs brute-force login attempts on services.
   * - ``default``
     - Default scripts executed with ``-sC``.
   * - ``discovery``
     - Evaluates accessible services.
   * - ``dos``
     - Checks for DoS vulnerabilities; rarely used as it may harm services.
   * - ``exploit``
     - Attempts to exploit known vulnerabilities for the scanned port.
   * - ``external``
     - Uses external services for additional processing.
   * - ``fuzzer``
     - Sends unexpected packets to find vulnerabilities or unusual behavior.
   * - ``intrusive``
     - Scripts that could negatively affect the target system.
   * - ``malware``
     - Checks if the target system is infected with malware.
   * - ``safe``
     - Defensive scripts; non-intrusive and non-destructive.
   * - ``version``
     - Enhances service detection.
   * - ``vuln``
     - Identifies specific vulnerabilities.

---

Syntax Examples
===================================

**Run all scripts in a category**::

   sudo nmap <target> --script <category>

**Run multiple specific scripts**::

   sudo nmap <target> --script <script1>,<script2>,<script3>

References:

- `Nmap NSE Documentation <https://nmap.org/nsedoc/scripts/>`_

