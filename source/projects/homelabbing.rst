RPI Home Cloud Project: How I Accidentally Became a Budget Sysadmin
===================================================================

So here’s the deal: I got tired of paying subscriptions for Google One, iCloud, and
all the other “we own your data, pay us rent” services. My goal? Build my own 
private cloud at home. Self-host, baby. Stick it to Big Tech. Save some money. 
Also, flex when friends come over: *“Yeah, that’s my server. No big deal.”*

The Dream Setup
---------------

In my head, this was simple. Grab a Raspberry Pi 5 (8GB RAM), hook up a couple 
of drives, sync my phone, PC, Mac, toaster—whatever—and boom: personal cloud.  

Final requirements looked like this:

1. Sync phone media across devices (iPhone, Android, PC, Mac, fridge if possible).
2. Have backups off-site (RAID 1 or JBOD mode).
3. Access data from anywhere in the world via VPN (because “hacker vibes”).

The Broke Student Reality Check
-------------------------------

Then reality hits. Raspberry Pis cost like $60–90. For context, that’s basically my 
weekly food budget. Hard drives aren’t exactly raining from the sky either.  

I even considered an RPi Zero 2W, but Reddit folks were like: *“lol bro, good luck 
running that with 512MB RAM. Might as well try hosting your cloud on a potato.”*  

Enter: **Hitesh**. My undergrad buddy who always tinkered with gadgets. He told 
me to forget Pis and start looking at used mini-PCs like Intel NUCs or Dell Optiplexes.  
Basically: “Stop being cute, get real hardware.”

The Hunt for Cheap Hardware
---------------------------

So I go hunting on eBay like a raccoon digging through tech trash. Intel NUCs? 
Too expensive. Dell Optiplexes? Decent, but still kinda pricey.  

After days of scrolling through “refurbished” listings from sellers with usernames 
like *TechLord420*, I was about to give up. Then it happened.  

A friend from Hopkins casually drops: *“Oh yeah, I’ve got a couple old PCs lying 
around, wanna buy one?”*  

Excuse me? I talk to this guy almost every day and he waits until *now* to mention 
he’s hoarding old hardware like a dragon?  

Anyway, long story short: he sold me a Dell Optiplex 5060 (8GB RAM, 256GB SSD) 
for $25. Absolute jackpot. I basically robbed him (thanks Shaiv, you’re the GOAT).

The Plan Evolves
----------------

At first, I thought: *“Easy, just throw TrueNAS Scale on it, done.”* But then I 
discovered **Pi-hole** (network-wide ad-blocker) and **Proxmox** (hypervisor 
wizardry).  

Suddenly the plan wasn’t just “install TrueNAS.” Nope. Now it was:  

- Run Proxmox as the hypervisor  
- Spin up TrueNAS Scale in a VM  
- Run Pi-hole in an LXC container  
- Pretend I know what I’m doing  

Networking Hell
---------------

Everything was peachy… until I realized: my apartment WiFi is run by **Dojo Networks**.  
Translation: the ISP set up one router for the entire building. Multiple apartments. 
No router access for me. Managed network prison.  

Wired connection? Not in my unit. My Optiplex didn’t even come with WiFi. Bruh.  

I finally scavenged an ethernet port in the apartment and hooked it to a friend’s 
laptop (because *of course* mine didn’t have an ethernet port). Got these details:

- LAN IPv4: ``10.254.171.53/23``  
- WLAN IPv4: ``172.161.77.88/24``  

Cool, numbers. Now what? Yeah… that’s where I’m at right now.  

Conclusion (aka: What Have I Done?)
-----------------------------------

So in the end, I went from *“gonna buy a Raspberry Pi”* to *“running Proxmox 
on a dusty Dell Optiplex I got for the price of a Chipotle burrito.”*  

Is my home cloud running perfectly yet? No. Is it at least alive? Kind of.  
But hey, that’s homelabbing for you: equal parts tech, chaos, and existential dread.  

Pro tip: before you spend hours searching eBay, maybe ask your friends if they 
have old hardware collecting dust. You might save money *and* a friendship.
