# Raspberry Pi: Pi-Hole Ad-Blocking + Unbound DNS + Wireguard VPN
This project is centered around getting a Raspberry Pi setup on a simple home network in order to block ads and naughty DNS requests, secure the DNS requests of all devices on the network, and provide a VPN solution for when any of these devices are outside of the network and would like to take advantage of the security (and speed) benefits of the network remotely.

There are several guides written about this or similar setups, but in praactice there was always something missing or assumptions were made about certain steps in the process. This guide is meant to shed some light on those steps, simplify the process of getting setup, and explain my findings as figured things out on my own.

This is what worked for me, your miles may vary.

## Prerequisites
While I won't have time to troubleshoot other setups, I'm sharing what I had to work with here. You'll need:

+ Raspberry Pi 3 Model B Plus Rev 1.3
+ Raspbian 10 Buster Lite
+ 16GB MicroSD card (4GB might be enough, but I'd stick with at least 8GB)
+ Mac running macOS Mojave (for prepping, backing up, and restoring the SD card)
+ USB SD card reader (I use this simple [Anker 2-in-1 card reader](https://amzn.to/2OaNBd1))
+ Mouse and keyboard for intial Raspberry Pi setup (I used a Bluetooth mouse that has its own USB transmitter, and my wife's [Apple Magic Keyboard connected via a USB to Lightning cable](https://www.reddit.com/r/mac/comments/6m3rpm/question_possible_to_use_magic_keyboard_2_without/))
+ HDMI cable and montior for initial Raspberry Pi setup (I hooked mine to a TV)
+ Apple Time Capsule router (connected directly to a cable modem, acting as a DHCP server)

That said, this process should work on any Raspberry Pi 2 v1.2 and above, and there are Windows/Linux tools to handle the SD card management.

## Installing Raspbian on the Raspberry Pi
There are many operating systems available to run on the Raspberry Pi, but we'll be using the latest version of Raspbian for this tutorial. There are also several different ways to install Rasbian on your Raspberry Pi, including N00Bs (New Out Of the Box Software). It's an easy operating system installer that allows you to erase your Raspberry Pi's SD card and start from scratch quickly, which is great when you're experimenting with your new device.

You can [download N00BS](https://www.raspberrypi.org/downloads/noobs/) for free in either the normal version (includes Raspbian and LibreELEC installation files) or the lite version (nothing is pre-loaded, you'll download installation files from the internet). Once downloaded, follow [the simple instructions](https://projects.raspberrypi.org/en/projects/raspberry-pi-setting-up/3) to get an SD card ready for your Raspberry Pi.

### Booting N00BS and Installing Raspbian
Once the SD card is ready, insert it into your Raspberry Pi and boot it up. You'll be taken to the N00BS installer screen where you can choose to install any one of several operating systems, including Raspbian. You can choose from Lite (console only), Desktop (includes a GUI), or Full (Desktop and recommended applications).

For the purposes of this setup, we'll install Raspbian Lite (console only). If you ever want to add the desktop GUI, you can [always add that later](https://gist.github.com/kmpm/8e535a12a45a32f6d36cf26c7c6cef51).

Installation generally takes about 10 minutes, but depends on whether you chose Lite or the normal N00BS files and how fast your internet connection is. Once installed, your Raspberry Pi will reboot into the console of a fresh install of Raspbian.

## Initial Setup of Raspbian
To login to your freshly baked Raspberry Pi, use the default username/password pair of `pi` and `raspberry` (yes, very original). The first thing you should do is change your password:

```
passwd
```

Enter your current password (`raspberry`) and then type and retype a new password.

### Optional: Change pi Username
If you'd rather not stick with the default username of `pi`, changing your Raspbian username is unfortunately not as easy as you'd think. Because you're currently logged in as `pi` and you can't change the username of the current user, you have to get creative with how you go about this. You could create a new user, logout of the `pi` user, login as the new user, and then as root you can change the username of `pi` and its home directory. But now you have a second user.

You can also follow along with [these instructions on adding a temporary user and using it to change pi's username](https://askubuntu.com/a/34075), but I can't vouch for this method as I skipped it altogether.

### Prepping Raspbian for SSH
We should enable SSH on the Raspberry Pi so that we can SSH into it from any device on the network, thus no longer needing the mouse, keyboard, HDMI cable, and screen. You'll need to know the IP address of your Raspberry Pi to continue.

### Setting Up a Static IP Address
First we need the IP address and MAC address of the Raspberry Pi. It's highly recommeneded that you setup a static IP address on your network for your Raspberry Pi so that you can easily SSH into it, and later, point other services directly to it.

To get the current IP address of your Raspberry Pi, run:
```
ifconfig
```
And you'll see some output that should include both `eth0` details and `wlan0` details. The former is your ethernet (wired) network interface, while the latter is your wireless (wifi) network interface. Depending on how you'll be connecting your Raspberry Pi to your network, you'll need to focus on one or the other.

If your Raspberry Pi is wired into your network via an ethernet cable, take a look at the `eth0` interface and find the following line:
```
inet 192.168.x.x  netmask 255.255.255.0  broadcast 192.168.x.255
```
Where `192.168.x.x` is the IP address of the Raspberry Pi (I've obfuscated this for privacy reasons). The Raspberry Pi's MAC address is just below that on the line:
```
ether ab:cd:ef:12:34:gh  txqueuelen 1000  (Ethernet)
```
Where `ab:cd:ef:12:34:gh` would be the unique MAC address of the `eth0` interface on this Raspberry Pi. In order to force your network's router to give the Raspberry Pi a static IP address (the same IP every time it connects), you'll need to edit the settings of your router and add a DHCP Reservation:
```
Description: Raspberry Pi
Reserve Address By: MAC Address
MAC Address: ab:cd:ef:12:34:gh
IPv4 Address: 192.168.x.x
```
Once you've created a static IP address of your choosing (matching your router's existing subnet schema), save your changes and restart your router.

### Enabling SSH on the Raspberry Pi
Now we'll open the Raspberry Pi's Configuration Tool:
```
sudo raspi-config
```
Under the **Interfacing Options** group, go to **SSH** and enable it. You should be prompted to restart your Raspberry Pi. If you aren't, exit the Configuration Tool and type:
```
sudo reboot
```
## SSH into the Raspberry Pi
Now that you've enabled SSH on the Raspberry Pi and given it a static IP address, you should be able to do all of the following setup from another device by connecting via SSH. I'll be using a Mac in the following steps.

Open the macOS Terminal application and type:
```
ssh raspberrypi@192.168.x.x
```
Where `192.168.x.x` is the static IP address you just setup for your Raspberry Pi. Then type the new password you set for the `pi` user to login.

**Note**: If you get an error that says the host identification has changed, you’ll need to remove the old entry (the one with the same static IP address) from your `~/.ssh/known_hosts` file first, then try again.

## Setup SSH Keys
On your computer (Mac in this case), open Terminal
Generate SSH key pairs with `ssh-keygen -t rsa -b 4096` and enter a password for SSH key
If you already have SSH key(s), give this pair a different name (id_rsa_pi)
Copy public key to your Raspberry Pi with `ssh-copy-id username@IP_ADDRESS` where username is the username you use on the Pi and the IP_ADDRESS is the IP or hostname of your Pi
Enter your Pi username’s password to confirm the copy
Store the SSH key in your macOS Keychain with `ssh-add -K ~/.ssh/id_rsa_pi` or whatever you named your SSH key pair
Enter your macOS passphrase
Configure SSH to always use your Mac’s Keychain by adding to your `~/.ssh/config` file
https://apple.stackexchange.com/a/250572
If you already have one, add a new `IdentityFile` line with the name of this new key
Now SSH into your Pi in your Mac’s Terminal by typing `ssh username@IP_ADDRESS` where IP_ADDRESS is the IP or hostname of your Pi, and you shouldn’t have to enter a password again!

## Optional: Additional Raspbian Configuration
Run `sudo raspi-config` to get Configuration tool
Change Hostname to something other than `raspberrypi` under Network Options
Change Locale under Localisation Options (add en-us UTF8 to en-gb UTF8, then select US as default, then change timezone)

## Update Raspbian and Packages
Run `sudo apt update` and then `sudo apt-get upgrade -y`
This may take a while, so grab a coffee/beer/tea…

## Optional: Create a Backup of Your Pi SD Card
Once you get to this point, you may want to have a backup of your Raspberry Pi setup so you don’t have to start from scratch if you make a mistake like I have (several times). You may also want to do this again after you have a working Pi-Hole + Unbound + Wireguard setup.
Shutdown your Raspberry Pi with `sudo halt` which is short for `sudo shutdown -h now`
Remove the SD card from your Raspberry Pi
Using a supported SD card reader, insert the SD card from your Raspberry Pi into your Mac
On a Mac, open Terminal and type `diskutil list` to see a list of drives attached to your Mac
Create a Disk Image of the SD card: `sudo dd if=/dev/disk2 of=~/RaspberryPiBackup.dmg` where `/dev/disk2` is the path to your SD card’s disk and `~/RaspberryPiBackup.dmg` is the path and filename on your Mac to save the Disk Image
Enter your Mac user’s password when prompted
This may take a while depending on the size of your SD card and what you have installed on it

Note: If you get input/output errors try using Disk Utility instead.

Open the macOS Disk Utility
Choose View > Show All Devices
Right click on the device name of the SD card and not the partition(s)
Choose the “Image from ‘DEVICE NAME’” option
Give the file a name and choose a location to save it
Choose the DVD/CD master option for Format, and no Encryption
Disk Utility will create a .CDR file with your SD card’s files

Or better yet, just use the free ApplePi-Baker for macOS to do backups and restores. It can even shrink/expand some Linux partitions so that the resulting .IMG file is much smaller than the actual SD card’s capacity.
https://www.tweaking4all.com/hardware/raspberry-pi/applepi-baker-v2/

## Optional: Restore Your Pi from Backup
If you’ve made a backup of your Raspberry Pi’s SD card previously, you can restore it any time. If you used a Mac to create the backup as a Disk Image, use the following instructions
Shutdown your Raspberry Pi with `sudo halt` which is short for `sudo shutdown -h now`
Remove the SD card from your Raspberry Pi
Using a supported SD card reader, insert the SD card from your Raspberry Pi into your Mac
On a Mac, open Terminal and type `diskutil list` to see a list of drives attached to your Mac
Unmount the SD card: `diskutil unmountDisk /dev/disk2` where `/dev/disk2` is the path to your SD card’s disk
Restore the SD card from backup: `sudo dd if=~/RaspberryPiBackup.dmg of=/dev/disk2` where `/dev/disk2` is the path to your SD card’s disk and `~/RaspberryPiBackup.dmg` is the path and filename on your Mac to save the Disk Image
This may take a while depending on the size of your SD card backup

Note: If you backed up the SD card as a .CDR, you can rename it to .ISO and restore it to any SD card with the free Etcher (https://www.balena.io/etcher/) software for macOS

Give Your Raspberry Pi a Static IP
Using your network router’s admin software, setup a static IP address for the Raspberry Pi so it doesn’t change during a restart or after a network outage. When setting up Pi-Hole and other services, you’ll use this static IP address in configuration files.
Make sure to reboot the Raspberry Pi after you set this up: `sudo reboot`

## Installing Pi-Hole
Run the Pi-Hole installer: `sudo curl -sSL https://install.pi-hole.net | bash`
Note: If you see an error “Script called with non-root privileges” you can download the install script and run it as root instead
Get the Pi-Hole install script: `wget -O basic-install.sh https://install.pi-hole.net`
Run the installer as root: `sudo bash basic-install.sh`
Select OK for the first few screens, then select `eth0` for ethernet or `wlan0` for wireless on the Choose An Interface screen
Select any Upstream DNS Provider since we’ll be using our own Unbound server later (I choose Cloudflare here)
Choose any Block Lists you want to use
Choose both IPv4 and IPv6 on the Select Protocols screen
Use the current network settings on the next screen, assuming you gave your Raspberry Pi a static IP address
Decide if you want to install the web admin interface (and then install lighthttpd to serve it)
Log queries? What privacy mode do you want for FTL?
Setup will finish, DNS service will be running

## Using Pi-Hole’s Web Interface
After Pi-Hole setup is complete, change the default Web Interface password with `pihole -a -p`
Access the Pi-Hole Web Interface in your browser by going to http://IP_ADDRESS/admin where IP_ADDRESS is the static IP of your Raspberry Pi (you can also use htt://pi.hole/admin once you point your router to use Pi-Hole as your DNS service)
Go to Login, then enter the new password you set for the Web Interface and check the “Remember me for 7 days” checkbox before logging in
You won’t see much on the Dashboard yet since nothing on your network is using Pi-Hole

## Point Your Router to Pi-Hole
To make sure Pi-Hole is working, you can set a single device to use it as your DNS service or use your network’s router instead to force (almost) every device on your network to use Pi-Hole
Most routers have a setting for Primary and Secondary DNS, point the primary to the static IP address of your Raspberry Pi and the secondary to something like Google (8.8.8.8) or Cloudflare (1.1.1.1) in case your Pi-Hole server goes down for some reason and you don’t want to lose all connectivity to the outside world
Restart your router, and watch as the Pi-Hole Dashboard (now available at http://pi.hole/admin) fills up with blocked queries!

## Install Unbound DNS
Install Unbound: `sudo apt install unbound`
Get the root.hints file: `wget -O root.hints https://www.internic.net/domain/named.root`
Move the root.hints file to the Unbound directory: `sudo mv root.hints /var/lib/unbound/`

## Configure Unbound DNS
Edit the Pi-Hole configuration file: `sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf`
See documentation for all options: https://www.nlnetlabs.nl/documentation/unbound/unbound.conf/

```
server:
# If no logfile is specified, syslog is used
# logfile: "/var/log/unbound/unbound.log"

# Level 2 gives detailed operational information
verbosity: 2

port: 5353
do-ip4: yes
do-udp: yes
do-tcp: yes

# May be set to yes if you have IPv6 connectivity
do-ip6: no

# Use this only when you downloaded the list of primary root servers!
root-hints: "/var/lib/unbound/root.hints"

# Respond to DNS requests on all interfaces
interface: 0.0.0.0
# Maximum  UDP response size, default is 4096
max-udp-size: 3072

# IPs authorized to access the DNS Server
access-control: 0.0.0.0/0 refuse
access-control: 127.0.0.1 allow
access-control: 192.168.1.0/24 allow

# Hide DNS Server info
hide-identity: yes
hide-version: yes

# Trust glue only if it is within the servers authority
harden-glue: yes

# Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
harden-dnssec-stripped: yes

# Burdens the authority servers, not RFC standard, and could lead to performance problems
harden-referral-path: no

# Add an unwanted reply threshold to clean the cache and avoid, when possible, DNS poisoning
unwanted-reply-threshold: 10000000

# Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
# see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
use-caps-for-id: no

# Reduce EDNS reassembly buffer size.
# Suggested by the unbound man page to reduce fragmentation reassembly problems
edns-buffer-size: 1472

# Perform prefetching of close to expired message cache entries
# This only applies to domains that have been frequently queried
prefetch: yes
# Fetch the DNSKEYs earlier in the validation process, which lowers the latency of requests
# but also uses a little more CPU
prefetch-key: yes

# Time To Live (in seconds) for DNS cache. Set cache-min-ttl to 0 remove caching (default).
# Max cache default is 86400 (1 day).
cache-min-ttl: 3600
cache-max-ttl: 86400

# If enabled, attempt to serve old responses from cache without waiting for the actual
# resolution to finish.
# serve-expired: yes
# serve-expired-ttl: 3600

# Use about 2x more for rrset cache, total memory use is about 2-2.5x
# total cache size. Current setting is way overkill for a small network.
# Judging from my used cache size you can get away with 8/16 and still
# have lots of room, but I've got the ram and I'm not using it on anything else.
# Default is 4m/4m
msg-cache-size: 128m
rrset-cache-size: 256m

# One thread should be sufficient, can be increased on beefy machines.
# In reality for most users running on small networks or on a single machine it should
# be unnecessary to seek performance enhancement by increasing num-threads above 1.
num-threads: 1

# Ensure kernel buffer is large enough to not lose messages in traffic spikes
so-rcvbuf: 1m

# Ensure privacy of local IP ranges
private-address: 192.168.0.0/16
private-address: 169.254.0.0/16
private-address: 172.16.0.0/12
private-address: 10.0.0.0/8
private-address: fd00::/8
private-address: fe80::/10
```

Start the Unbound DNS server: `sudo service unbound start`
Test that Unbound DNS is running: `dig pi-hole.net @127.0.0.1 -p 5353`
This command should return a status of SERVFAIL: `dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5353`
This command should return a status of NOERROR: `dig sigok.verteiltesysteme.net @127.0.0.1 -p 5353`

## Allow Pi-Hole to Use Unbound DNS
Open the Pi-Hole Web Interface using a web browser: http://pi.hole/admin
Go to Settings > DNS
Uncheck any third party Upstream DNS Servers you had selected during setup
Add `127.0.0.1#5353` to the Custom 1 (IPv4) Upstream DNS Servers input
Uncheck “Never forward non-FQDNs” and “Never forward reverse lookups for private IP ranges”
Check “Use Conditional Forwarding” and enter your router’s IP address, along with the Domain Name (this can be set on your router, usually under the DHCP settings, to something like “home”)
Don’t check “Use DNSSEC” as Pi-Hole uses Unbound DNS, which already enables DNSSEC
Save your settings

Check to make sure DNSSEC (DNS Security Extensions) is working by visiting: http://dnssec.vs.uni-due.de

## Setting Up Your VPN with WireGuard
Install necessary packages for everything we’ll be doing:
`sudo apt install raspberrypi-kernel-headers libelf-dev libmnl-dev build-essential git`

Switch to root with`sudo su` and enter the next 2 commands per the Debian installation commands from: https://www.wireguard.com/install/
Run `echo "deb http://deb.debian.org/debian/ unstable main" > /etc/apt/sources.list.d/unstable.list`
Then `printf 'Package: *\nPin: release a=unstable\nPin-Priority: 90\n' > /etc/apt/preferences.d/limit-unstable`
Then exit root with: `exit`

Update everything with `sudo apt update` and ignore the error
Install dirmngr for handling certificates: `sudo apt install dirmngr`

Add pubic keys
`sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7638D0442B90D010`
`sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC`

### Update and Install WireGuard
Run `sudo apt update` and then `sudo apt install wireguard`
Enable IP forwarding with: `sudo perl -pi -e 's/#{1,}?net.ipv4.ip_forward ?= ?(0|1)/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf`
Reboot your Raspberry Pi with `sudo reboot`
After reboot, verify that IP forwarding is enabled by running: `sysctl net.ipv4.ip_forward`
You should see `net.ipv4.ip_forward = 1` as a result, otherwise add the above command to your `/etc/sysctl.conf` file

### Generate Private & Public Keys for Wireguard
Become root with `sudo su`
Change to Wireguard directory: `cd /etc/wireguard`
Set permissions on directory: `umask 077`
Generate the server’s private & public keys: `wg genkey | tee server_privatekey | wg pubkey > server_publickey`
Generate a client’s private & public keys: `wg genkey | tee peer1_privatekey | wg pubkey > peer1_publickey`
Confirm the keys were generated and permissioned: `ls -la`
View and save keys for Wireguard configuration:
`cat server_privatekey`
`cat peer1_publickey`
Exit root: `exit`

### Configure Wireguard Server
Create and edit the configuration file: `sudo nano /etc/wireguard/wg0.conf`
Be sure to replace `<server_privatekey>` and `<peer1_publickey>` with the keys you generated earlier
And replace `192.168.x.x` with the IP of your Pi-Hole (in this case it’s the Raspberry Pi you’re already on)
```
[Interface]
Address = 10.9.0.1/24
# Default Wireguard port, change to anything that doesn’t conflict
ListenPort = 51820
DNS = 192.168.x.x
PrivateKey = <server_privatekey>

# Replace eth0 with the interface open to the internet (e.g might be wlan0 if wifi)
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

[Peer]
# Peer 1
PublicKey = <peer1_publickey>
AllowedIPs = 10.9.0.2/32
# If this device is behind a NAT, uncomment to keep connection alive
#PersistentkeepAlive = 60 
```

### Configure a Wireguard Client
Create and edit a client configuration file: `sudo nano /etc/wireguard/peer1.conf`
Be sure to replace `<peer1_privatekey>` and `<server_publickey>` with the keys you generated earlier
And replace `192.168.x.x` with the IP of your Pi-Hole (in this case it’s the Raspberry Pi you’re already on)
```
[Interface]
Address = 10.9.0.2/32
DNS = 192.168.x.x 
PrivateKey = <peer1_privatekey>

[Peer]
PublicKey = <server_publickey>
Endpoint = YOUR-PUBLIC-IP/DDNS:ListenPort
# For full tunnel use 0.0.0.0/0, ::/0 and for split tunnel use 192.168.1.0/24
# or whatever your router’s subnet is
AllowedIPs = 0.0.0.0/0, ::/0
# If this device is behind a NAT, uncomment to keep connection alive
#PersistentkeepAlive = 60 
```

### Optional: Setup Dynamic DNS for Your Public IP address
If your ISP does not provide you with a static IP address (most don’t), and your IP changes for some reason (cable modem reboot, connectivity issues, etc.), your home network may be unreachable from the outside until you update it in your configuration files.  The solution is to use a DDNS (Dynamic DNS) service where you choose a readable domain name and can automatically notify when your public IP address changes.

So instead of worry about whether your public IP address is 98.10.200.11 or 98.10.200.42, you can instead point a domain name like `username.us.to` at your public IP address and have the DDNS service update the domain record when your public IP address changes.

There are plenty of DDNS services out there, but I’m using Afraid.org’s Free DNS service because it doesn’t nag you to login every 30 days, even on the completely free plan.

#### Get a Free DNS Subdomain
Create a free account at: http://freedns.afraid.org
Verify your account using the link you’ll receive in your email
Go to Add Subdomain
Enter a Subdomain (like a username, your name, etc.) and choose a Domain (there are many to choose from besides what’s listed)
Leave Type set to an A record, TTL is set to 3600 for free accounts, and Wildcard functionality is only for paid accounts
Your public IP address should be automatically filled in, but you can visit http://www.ipleak.com in a browser to get your IP address
Enter the CAPTCHA text and hit Save
You should now have a shiny new subdomain pointing to your network’s public IP address

#### Automatically Update Your Public IP Address
While logged in at Afraid.org, go to the Dynamic DNS page and look for your new subdomain record
Right click the Direct URL link associated with your subdomain record and Copy Link
On your Raspberry Pi, create a cronjob that runs every 5 minutes (replacing the XXXXX with the unique identifier in your Direct URL):
`crontab -l | { cat; echo "*/5 * * * * curl http://freedns.afraid.org/dynamic/update.php?XXXXX”; } | crontab -`
You’ll see `no crontab for …` on the console, you can safely ignore that
Verify that you’ve added the command correctly with: `crontab -l`
Restart the cron service with: `sudo service cron restart`

#### Use Your Dynamic Subdomain in Place of Your Public IP address
Go back to your Wireguard configuration file and use your new subdomain with the `ListenPort` you set earlier and never worry about your public IP address changing!
In the `/etc/wireguard/peer1.conf` file, edit this line:
`Endpoint = YOUR-PUBLIC-IP/DDNS:ListenPort`
to read:
`Endpoint = username.us.to:51820`
using the subdomain you chose at Afraid.org and the `ListenPort` you set in your `/etc/wireguard/wg0.conf` file

### Setting Up Your Phone to Use the VPN
Unlike IPSec or IKEv2, WireGuard isn’t built into the Android or iOS operating system, so you’ll have to download the WireGuard app to connect to your new VPN.

Find and install the iOS or Android native app for WireGuard
Android: https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en_US
iOS: https://apps.apple.com/us/app/wireguard/id1441195209

#### Export Your Client Configuration with a QR Code
Install the QR encoder with: `sudo apt install qrencode`
Become the root user in order to read the WireGuard client config: `sudo su`
Create a QR code from your client configuration: `qrencode -t ansiutf8 < /etc/wireguard/peer1.conf`
Note: You may have to adjust the size of your Terminal window to properly show the QR code generated
Open the WireGuard app on your phone, tap “Add a tunnel” and select “Create from QR code”
Scan the QR code with your phone’s camera, give the tunnel a name, and allow WireGuard to add VPN configurations to the OS

### Finish WireGuard Installation
On your Raspberry Pi, complete setup of WireGuard
Allow WireGuard to start on boot: `sudo systemctl enable wg-quick@wg0`
Set the correct permissions on the Wireguard configuration file with:
`sudo chown -R root:root /etc/wireguard/`
`sudo chmod -R og-rwx /etc/wireguard/`

In your Pi-Hole Web Interface, go to Settings > DNS and choose the “Listen on all interfaces, permit all origins” option under “Interface listening behavior”
Save your settings

Run `sudo wg-quick up wg0` to Start Wireguard
Check to see if interface was created successfully: `ifconfig wg0`

Restart your Raspberry Pi with: `sudo reboot`
Use `sudo wg` to check if WireGuard working (You should see interface wg0 and a peer)

### Open WireGuard VPN Port on Your Router
If you have an Apple Time Capsule or AirPort device, open the AirPort Utility in macOS
Edit the settings for your device, and go to Network
Under Port Settings, click the + icon to add a new port forward Give the port forward a name (example: “WireGuard VPN”)
Enter 51820 (or whatever you set your `ListenPort` to) in the Public/Private UDP Ports and Public/Private TCP Ports fields
Enter the static IP address of your Raspberry Pi in the Private IP Address field

### Setup VPN Client for Mac
https://itunes.apple.com/us/app/wireguard/id1451685025

## References

There are several write-ups out there on how to do this, as well as install scripts to do it for you. Since the Raspberry Pi was meant to be a learning tool, I used this opportunity to figure things out on my own with the help of documentation from both software creators and the community. If it weren't for the latter, I doubt I would've been able to do this on my own.

https://www.sethenoka.com/build-your-own-wireguard-vpn-server-with-pi-hole-for-dns-level-ad-blocking/
https://github.com/harrypnyce/raspbian10-buster
https://github.com/crozuk/pi-hole-wireguard-privoxy
https://github.com/anudeepND/pihole-unbound
https://www.nlnetlabs.nl/documentation/unbound/howto-anchor/
https://github.com/ShaneCaler/EasyAsPiInstaller
