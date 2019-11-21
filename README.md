# Raspberry Pi: Pi-Hole Ad-Blocking + Unbound DNS + Wireguard VPN

This setup was done on a Raspberry Pi 3 Model B Plus Rev 1.3 with a 16GB MicroSD card. A MacBook Pro running macOS Mojave was used in prepping, backing up, and restoring the SD card. That said, this process should work on any Raspberry Pi 2 v1.2 and above, and there are Windows/Linux tools to handle the SD card management.

## Install Raspbian
N00BS on MicroSD card
Boot into N00BS installer on Raspberry Pi
Install Raspbian Lite (console only)

## Boot into Raspbian
Login with user pi, password raspberry
Change default user’s password with: `passwd`
Run `sudo raspi-config` to get Configuration tool
Enable SSH under Interfacing Options
Restart Raspberry Pi (you should be prompted to do this, otherwise use `sudo reboot`)

## SSH into Raspberry Pi
From your Mac’s Terminal: `ssh pi@IP_ADDRESS` where IP_ADDRESS is the IP or hostname of your Pi
Type the new password you set for the pi user
Note: If you get an error that says the host identification has changed, you’ll need to remove an old entry from your `Users/USERNAME/.ssh/known_hosts` file first, then try again

Optional: Change pi username
Create a new user to login with, or try to replace pi user with new username
https://askubuntu.com/a/34075
You’ll need to `sudo su` into root to do this, most likely as another user, since you’ll need to kill all of pi’s processes

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

## References

There are several write-ups out there on how to do this, as well as install scripts to do it for you. Since the Raspberry Pi was meant to be a learning tool, I used this opportunity to figure things out on my own with the help of documentation from both software creators and the community. If it weren't for the latter, I doubt I would've been able to do this on my own.

https://www.sethenoka.com/build-your-own-wireguard-vpn-server-with-pi-hole-for-dns-level-ad-blocking/
https://github.com/harrypnyce/raspbian10-buster
https://github.com/crozuk/pi-hole-wireguard-privoxy
https://github.com/anudeepND/pihole-unbound
https://www.nlnetlabs.nl/documentation/unbound/howto-anchor/
https://github.com/ShaneCaler/EasyAsPiInstaller
