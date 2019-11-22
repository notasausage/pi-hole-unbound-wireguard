# Raspberry Pi: Pi-Hole Ad-Blocking + Unbound DNS + Wireguard VPN
This project is centered around getting a Raspberry Pi setup on a simple home network in order to block ads and naughty DNS requests, secure the DNS requests of all devices on the network, and provide a VPN solution for when any of these devices are outside of the network and would like to take advantage of the security (and speed) benefits of the network remotely.

There are several guides written about this or similar setups, but in praactice there was always something missing or assumptions were made about certain steps in the process. This guide is meant to shed some light on those steps, simplify the process of getting setup, and explain my findings in order to help anyone else trying to do the same.

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

#### Setting Up a Static IP Address
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

#### Enabling SSH on the Raspberry Pi
Now we'll open the Raspberry Pi's Configuration Tool:
```
sudo raspi-config
```
Under the **Interfacing Options** group, go to **SSH** and enable it. You should be prompted to restart your Raspberry Pi. If you aren't, exit the Configuration Tool and type:
```
sudo reboot
```
##### Verify You Can SSH into the Raspberry Pi
Now that you've enabled SSH on the Raspberry Pi and given it a static IP address, you should be able to do all of the following setup from another device by connecting via SSH. I'll be using a Mac in the following steps.

Open the macOS Terminal application and type:
```
ssh pi@192.168.x.x
```
Where `pi` is the username on your Raspberry Pi and `192.168.x.x` is the static IP address you just setup. Then type the new password you set for the `pi` user to login. You should now be greeted with the Raspian console.

**Note**: If you get an error that says the host identification has changed, you’ll need to remove the old entry (the one with the same static IP address) from your `~/.ssh/known_hosts` file first, then try again.

##### Passwordless SSH Access
It's possible to [configure the Raspberry Pi to allow a computer to access it without providing a password each time](https://www.raspberrypi.org/documentation/remote-access/ssh/passwordless.md) you try to connect. In your Mac's Terminal app, generate SSH key pairs with:
```
ssh-keygen -t rsa -b 4096
```
and enter a password for the SSH key (store this somewhere secure, you may need it at a later time). If you already have SSH key(s), give this pair a different name (such as `id_rsa_pi`). Now that the public and private keys have been generated on your Mac, you'll need to copy the public key to your Raspberry Pi using:
```
ssh-copy-id pi@192.168.x.x
```
where `pi` is the username you use on the Raspberry Pi and the `192.168.x.x` is the static IP address of the Raspberry Pi. You'll need to enter the password for the `pi` user to confirm the copy.

##### Keep Client SSH Keys in macOS Keychain
You can store your SSH key(s) in your Mac's Keychain for easy (and secure) access. Add your newly created SSH key pair to the macOS Keychain with:
```
ssh-add -K ~/.ssh/id_rsa_pi
```
where `id_rsa_pi` is the name you used to create the SSH key pair. Enter your macOS password to confirm. In order to prevent macOS from forgetting your SSH keys on restart, you can add it to your `~/.ssh/config` file based on [these instructions](https://apple.stackexchange.com/a/250572). Your `~/.ssh/config` file should look something like this:
```
Host *
    UseKeychain yes
    AddKeysToAgent yes
    IdentityFile ~/.ssh/id_rsa
    IdentityFile ~/.ssh/id_rsa_pi
```
Now once again, SSH into your Raspberry Pi using the Terminal application by typing `ssh pi@192.168.x.x` where `192.168.x.x` is the static IP address of your Raspberry Pi, and you shouldn’t have to enter a password again from this device! Repeat these steps on other devices to generate and share their SSH key(s) with the Raspberry Pi.

### Optional: Additional Raspbian Configuration
Here are some additional steps I prefer to take upon setting up my Raspberry Pi. First, open the Configuration Tool via your SSH session:
```
sudo raspi-config
```
+ Under **Network Options**, change the **Hostname** to something other than `raspberrypi` (most of the devices on my network are [named after Transformers](https://tfwiki.net/wiki/The_Transformers_(toyline))).
+ Under **Localisation Options**, add to the **Locale** `en-us UTF8` and select it as the default locale (*do not* remove the default `en-gb UTF8` as it seems to cause issues)
+ Under **Localisation Options**, change the timezone to match yours

### Update Raspbian and Packages
Once you've got everything up and running, you'll want to update all the Raspbian packages and upgrade any that need it using [APT (Advanced Packaging Tool)](https://wiki.debian.org/Apt) by running:
```
sudo apt update
```
to update packages and then:
```
sudo apt-get upgrade -y
```
to upgrade any packages you have installed to their latest versions. This may take a while, so you may want to go grab a coffee, beer, tea...

## Backup and Restore a Raspberry Pi
Throughout the process of setting things up, you may want to have a backup of your Raspberry Pi so you don’t have to start from scratch if you make a mistake like I have (several times). You may even want to clone your Raspberry Pi after each successful step so you can return to it if the next step goes wrong. For instance, I was sure to create a backup of my Raspberry Pi after initial setup, after I had Pi-Hole working, after I added Unbound, and after I got WireGuard setup. That way if I made any changes that broke my setup, I could always revert the previous working installation.

### Backup the Raspberry Pi SD Card
There are many ways to do this, but if you're on a Mac there is a very simple (and free) solution. You'll want to download the [ApplePi-Baker](https://www.tweaking4all.com/hardware/raspberry-pi/applepi-baker-v2/) app for macOS, which handles backing up your Raspberry Pi's SD card and can restore from several different formats as well.

At any point you want to backup your Raspberry Pi's configuration, first shutdown the device with:
```
sudo halt
```
or the more verbose:
```
sudo shutdown -h now
```
and then unplug your Rasbperry Pi (or get a [Pi Switch](https://amzn.to/2s7axRP) to make life easier). Remove the SD card from the Raspberry Pi and insert into a supported SD card reader (like the [Anker 2-in-1 card reader](https://amzn.to/2OaNBd1) I mentioned earlier), and connect to your Mac.

When you open the ApplePi-Baker app, you should be able to select the SD card from the **Select a Disk** options (make sure you select the right disk), then choose the **Backup** option. In the file choose window, select **IMG** from the Format selections at the bottom of the window, and then give the backup file a name. This should take some time, be prepared to wait.

**Note**: The IMG format allows ApplePi-Baker to shrink the Linux partition on backup and expand it on restore, saving you from having to backup the entire size of the SD card rather than just the actual contents.

#### Using the macOS Terminal
Another way to backup your SD card on a Mac is to open Terminal and type:
```
diskutil list
```
to see a list of drives attached to your Mac. To create a Disk Image of the SD card:
```
sudo dd if=/dev/rdisk2 of=~/RaspberryPiBackup.dmg
```
where `/dev/rdisk2` is the path to your SD card’s disk (with an added `r`) and `~/RaspberryPiBackup.dmg` is the path and filename on your Mac to save the Disk Image. Enter your Mac user’s password when prompted. This may take a while depending on the size of your SD card and what you have installed on it.

**Note**: If you see any input/output errors, try using Disk Utility instead.

#### Using the macOS Disk Utility
Open the macOS Disk Utility application, and choose View > Show All Devices. Right click on the device name of the SD card (not the individual partition(s)!) and choose the **Image from ‘DEVICE NAME’** option. Give the file a name and choose a location to save it. Choose the DVD/CD master option for Format, and no Encryption. Disk Utility will create a .CDR file with your SD card’s files.

**Note**: Both the Disk Utility and Terminal methods failed for me, both creating input/output errors. Maybe it was my SD card, or my card reader, but then using ApplePi-Baker gave me no issues, so I didn't bother to investigate further.

### Restoring a Raspberry Pi from Backup
If you’ve made a backup of your Raspberry Pi’s SD card previously, you can restore it any time and overrite your current configuration. Using the ApplePi-Baker app on macOS, this is a simple task.

First, shutdown your Raspberry Pi:
```
sudo halt
```
and remove the SD card from your Raspberry Pi. Using a supported SD card reader, insert the SD card from your Raspberry Pi into your Mac. Launch the ApplePi-Baker app and select the correct SD card disk, then choose the **Restore** option. ApplePi-Baker will expand the backup file and overrite the complete contents of your SD card with the backup file's contents, so make sure you really want to do this.

#### Using the macOS Terminal to Restore
You can also use the macOS Terminal application to this by typing:
```
diskutil list
```
to see a list of drives attached to your Mac. You'll need to unmount the SD card to proceed (your Mac can still see the files, but you won't see the mounted disk(s) anymore):
```
diskutil unmountDisk /dev/disk2
```
where `/dev/disk2` is the path to your SD card’s disk. Restore the SD card from backup using:
```
sudo dd if=~/RaspberryPiBackup.dmg of=/dev/disk2
```
where `/dev/disk2` is the path to your SD card’s disk and `~/RaspberryPiBackup.dmg` is the path and filename on your Mac to save the Disk Image. This may take a while depending on the size of your SD card backup

**Note**: If you backed up the SD card as a .CDR, you can rename it to .ISO and restore it to any SD card with the free [Etcher application](https://www.balena.io/etcher/) for macOS.

## Setting Up Pi-Hole
[Pi-Hole](https://pi-hole.net) provides ad-blocking at the network level, meaning you not only stop ads from making it to any of the devices on your network, but you also block the unnecessary network requests for those ads and thus reduce bandwidth usage. Pi-Hole pairs nicely with a VPN (Virtual Private Network) so that you can connect remotely and still take advantage of ad-blocking from anywhere outside your network.

First you'll need to run the Pi-Hole installer:
```
sudo curl -sSL https://install.pi-hole.net | bash
```
**Note**: If you see the error “Script called with non-root privileges” during setup, you can download the install script and run it as root instead. Do this by downloading the script:
```
wget -O basic-install.sh https://install.pi-hole.net
```
and then run the installer as root:
```
sudo bash basic-install.sh
```
During setup, select **OK** for the first few screens, then select either `eth0` for ethernet (wired) or `wlan0` for wireless (wifi) on the **Choose An Interface** screen. Next, select any **Upstream DNS Provider** since we’ll be using our own Unbound server later. Choose any **Block Lists** you want to use, or leave them all checked by default. Choose both IPv4 and IPv6 on the **Select Protocols** screen. Use the current network settings on the next screen, assuming you gave your Raspberry Pi a static IP address earlier. Then decide if you want to install the Web Interface for Pi-Hole (and `lighthttpd` to serve it), which you'll typically want to keep an eye on your traffic and blocked queries (and to make additional configuration changes) in a web browser.

Lastly, decide how you want to log queries and what privacy mode do you want for FTL (Faster Than Light). Setup will finish, and the Pi-Hole DNS service will be running.

### Using Pi-Hole’s Web Interface
After Pi-Hole setup is complete, you should see the default Web Interface password on the console. You can change the  password using:
```
pihole -a -p
```
Now you can access the Pi-Hole Web Interface in your browser by going to `http://192.168.x.x/admin` where `192.168.x.x` is the static IP of your Raspberry Pi (you can also use http://pi.hole/admin once you point your router to use Pi-Hole as your DNS service in the next step). Go to Login, then enter the new password you set for the Web Interface and check the “Remember me for 7 days” checkbox before logging in. You won’t see much on the Dashboard yet since nothing on your network is using Pi-Hole, but that should change momentarily.

### Use Pi-Hole as Your DNS Server
To make sure Pi-Hole is working, you can set a single device to use it as its DNS service or you can point your network’s router to it instead to force (almost) every device on your network to use Pi-Hole as its DNS service. Most routers have a setting for Primary and Secondary DNS, and you'll want to point the Primary DNS Server to the static IP address of your Raspberry Pi (`192.168.x.x`) and the Secondary DNS Server to a 3rd party DNS service like Google (8.8.8.8) or Cloudflare (1.1.1.1) in case your Pi-Hole server goes down for some reason and you don’t want to lose all connectivity to the outside world (like when you're fiddling around with your Raspberry Pi in the upcoming steps).

Restart your router and watch as the Pi-Hole Dashboard (now available on your internal network at http://pi.hole/admin) fills up with blocked queries!

## Setup Unbound as Your DNS Service
> [Unbound](https://www.nlnetlabs.nl/projects/unbound/about/) is a validating, recursive, caching DNS resolver. It is designed to be fast and lean and incorporates modern features based on open standards.

To install Unbound on the Raspberry Pi:
```
sudo apt install unbound
```
Afterwards we'll need to download a `root.hints` file to replace the built-in hints:
```
wget -O root.hints https://www.internic.net/domain/named.root
```
Move the `root.hints` file to the Unbound configuration directory:
```
sudo mv root.hints /var/lib/unbound/
```

### Configure Unbound DNS
Unbound includes a lot of different configuration options that you can adjust and try out. Feel free to scan the [Unbound configuration file documentation](https://www.nlnetlabs.nl/documentation/unbound/unbound.conf/) for details about each option.

To get started, edit the Pi-Hole configuration file:
```
sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
```
and remove anything already in the file before copying and pasting the contents of the [sample pi-hole.conf](pi-hole.conf) configuration file in this repository. When you're done, exit and save the file.

Some things to note:
```
port: 5353
```
The default port for Unbound is `53` but we're changing it to `5353` here. Feel free to change it to whatever you like, but you'll need to remember it later when we tell Pi-Hole where to send upstream DNS requests.
```
# Use this only when you downloaded the list of primary root servers!
root-hints: "/var/lib/unbound/root.hints"
```
This points to the `root.hints` file you just downloaded.
```
# IPs authorized to access the DNS Server
access-control: 0.0.0.0/0 refuse
access-control: 127.0.0.1 allow
access-control: 192.168.1.0/24 allow
```
Here we're refusing connections to all interfaces and then we're allowing anything from this device (your Raspberry Pi) and anything from our local subnet (if your subnet is not `192.168.1.x` then you'll need to change this).
```
# Time To Live (in seconds) for DNS cache. Set cache-min-ttl to 0 remove caching (default).
# Max cache default is 86400 (1 day).
cache-min-ttl: 3600
cache-max-ttl: 86400
```
You can adjust the cache settings if you like. Instead of the default of not caching, here we set the minimum TTL (Time To Live) to 1 hour, afterwards the DNS will do another lookup of the cached data.

Once the configuration file is saved, start the Unbound DNS server:
```
sudo service unbound start
```
And test to make sure the Unbound DNS is running:
```
dig pi-hole.net @127.0.0.1 -p 5353
```
If you run this command, it should return a status of SERVFAIL:
```
dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5353
```
And this command should return a status of NOERROR:
```
dig sigok.verteiltesysteme.net @127.0.0.1 -p 5353
```

### Allow Pi-Hole to Use Unbound DNS
Now that your Unbound recursive DNS resolver is running locally, we'll force Pi-Hole to use it for DNS rather than an outside source like Google (8.8.8.8) or Cloudflare (1.1.1.1) and keep all of our network traffic contained. Any traffic on your network will be sent to Pi-Hole, which in turn will use Unbound for DNS resolution before blocking the appropriate domains and returning data.

First, open the Pi-Hole Web Interface in a web browser on your local network: http://pi.hole/admin

Then go to **Settings > DNS** and uncheck any third party Upstream DNS Servers you had selected during setup. To the Custom 1 (IPv4) Upstream DNS Servers input, add:
```
127.0.0.1#5353
```
Where `127.0.0.1` points the Pi-Hole server (the Raspberry Pi) to itself on port `5353`. If you changed the port in your Unbound configuration file, use that port here instead.

Next, uncheck **Never forward non-FQDNs** and **Never forward reverse lookups for private IP ranges** and check **Use Conditional Forwarding** and enter your router’s IP address (typically something like `x.x.x.1` on your subnet), along with the Domain Name (this can be set on your router, usually under the DHCP settings, to something like `home`).

Lastly, *do not* check **Use DNSSEC** as Pi-Hole is going to be using your Unbound DNS, which already enables DNSSEC (Domain Name System Security Extensions).

When you're done, don't forget to save your settings. To check and make sure DNSSEC is working, visit this URL in your browser within your network: http://dnssec.vs.uni-due.de

## Setting Up a VPN with WireGuard
> [WireGuard](https://www.wireguard.com) is an extremely simple yet fast and modern VPN that utilizes state-of-the-art cryptography. It aims to be faster, simpler, leaner, and more useful than IPsec, while avoiding the massive headache. It intends to be considerably more performant than OpenVPN.

First, install the necessary packages before WireGuard setup begins:
```
sudo apt install raspberrypi-kernel-headers libelf-dev libmnl-dev build-essential git
```
Switch to root with `sudo su` and enter the next 2 commands per the [Debian installation commands](https://www.wireguard.com/install/) (check to see if these have changed, and use the official Debian instructions if they have):
```
echo "deb http://deb.debian.org/debian/ unstable main" > /etc/apt/sources.list.d/unstable.list
printf 'Package: *\nPin: release a=unstable\nPin-Priority: 90\n' > /etc/apt/preferences.d/limit-unstable
```
Then `exit` root.

Run an APT update and ignore the error:
```
sudo apt update
```
Then install dirmngr for handling certificates:
```
sudo apt install dirmngr
```

You'll need to connect with the keyserver with both of these keys, though I'm not exactly sure why...
```
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 7638D0442B90D010
sudo apt-key adv --keyserver keyserver.ubuntu.com --recv-keys 04EE7237B7D453EC
```

### Update and Install WireGuard
Run another APT update:
```
sudo apt update
```
and then install WireGuard:
```
sudo apt install wireguard
```
Next we'll enable IP forwarding:
```
sudo perl -pi -e 's/#{1,}?net.ipv4.ip_forward ?= ?(0|1)/net.ipv4.ip_forward = 1/g' /etc/sysctl.conf
```
Finally, reboot your Raspberry Pi:
```
sudo reboot
```
After reboot, verify that IP forwarding is enabled by running:
```
sysctl net.ipv4.ip_forward
```
You should see `net.ipv4.ip_forward = 1` as a result, otherwise add the above command to your `/etc/sysctl.conf` file.

### Generate Private & Public Keys for WireGuard
In the next steps, we'll need to create private and public keys for both the WireGuard server as well as a VPN client. Once everything is setup, we can create additional keys for other clients to use the VPN as well.

I've found it easiset to first become root before running the commands below:
```
sudo su
```
Then switch to the directory where we'll store the WireGuard keys:
```
cd /etc/wireguard
```
If this directory doesn't exist, just run `mkdir /etc/wireguard` and then `cd /etc/wireguard`. Set permissions on the entire directory so that only the `root` user can read or write data here:
```
umask 077
```
Next, generate the server’s private & public keys in a single command:
```
wg genkey | tee server_privatekey | wg pubkey > server_publickey
```
Then generate a client’s private & public keys:
```
wg genkey | tee peer1_privatekey | wg pubkey > peer1_publickey
```
To confirm the keys were generated and permissioned:
```
ls -la
```
Finally, output your new WireGuard keys to the console and save them (possibly in a text file, just be sure to delete it when we're done) for the next steps:
```
cat server_privatekey
cat server_publickey
cat peer1_privatekey
cat peer1_publickey
```
Lastly, `exit` root before continuing.

### Configure WireGuard Server
Create and edit the WireGuard configuration file:
```
sudo nano /etc/wireguard/wg0.conf
```
and replace the contents with the [WireGuard wg0.conf](wg0.conf) from this repository. You'll need to make the following changes:
```
[Interface]
Address = 10.9.0.1/24
```
This is the WireGuard interface, which will create a virtual subnet of `10.9.0.0` and assign itself an internal IP address of `10.9.0.1`. You can change this if you'd like, but you'll also need to change the internal IP of VPN clients as well.
```
# Default WireGuard port, change to anything that doesn’t conflict
ListenPort = 51820
```
The default port for WireGuard, which you can change if you'd like. You'll also need to open up this port on your router, otherwise incoming VPN traffic from outside your network *will not make it to WireGuard*. Information on how to do this is later in the guide.

**Note**: Some public wifi networks will block all ports other than `80` (TCP), `443` (TCP), and `53` (UDP) for HTTP, HTTPS, and DNS respectively. If you are connected to a public wifi network that does this, you will not be able to connect to your WireGuard VPN. One way around this is to set your WireGuard `ListenPort` to `53` and create a forward on your network's router on port `53`, thus circumventing the issue with blocked ports.
```
DNS = 192.168.x.x
```
Replace `192.168.x.x` with the static IP address of your Raspberry Pi.
```
PrivateKey = <server_privatekey>
```
Replace `<server_privatekey>` with the output of your `cat server_privatekey` command earlier.
```
PostUp = iptables -A FORWARD -i %i -j ACCEPT; iptables -A FORWARD -o %i -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i %i -j ACCEPT; iptables -D FORWARD -o %i -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE
```
We're using `eth0` here when the Raspberry Pi is connected over ethernet (wired), but you can replace both instances with `wlan0` if your Raspberry Pi is connected via wifi (wireless).

The next section of the WireGuard configuration file is for clients that connect to the VPN. For each client (device), you'll need to add another `[Peer]` section here and also create a separate client configuration file (details for that are next).
```
[Peer]
# Peer 1
PublicKey = <peer1_publickey>
```
Replace `<peer1_publickey>` with the output of your `cat peer1_publickey` command earlier.
```
AllowedIPs = 10.9.0.2/32
```
Using the virtual subnet created by WireGuard, give this device an internal IP address of `10.9.0.2`.

Once your WireGuard configuration file is compelete, exit the `nano` editor and save your changes.

### Configure a WireGuard Client
Now that WireGuard is configured, we'll need to create a client configuration file for each VPN client we want to connect to the network. First, create and edit your first client configuration file:
```
sudo nano /etc/wireguard/peer1.conf
```
and replace the contents with the [WireGuard client peer1.conf](peer1.conf) from this repository. You'll need to make the following changes:
```
[Interface]
Address = 10.9.0.2/32
```
Use the same virtual IP address that you used in the `wg0.conf` file earlier for this client.
```
DNS = 192.168.x.x
```
Replace `192.168.x.x` with the static IP address of your Raspberry Pi.
```
PrivateKey = <peer1_privatekey>
```
Replace `<peer1_privatekey>` with the output of your `cat peer1_privatekey` command earlier.
```
[Peer]
PublicKey = <server_publickey>
```
Replace `<server_publickey>` with the output of your `cat server_publickey` command earlier.
```
Endpoint = YOUR-PUBLIC-IP/DDNS:ListenPort
```
The `Endpoint` here refers to the public IP address and port number for incoming traffic to your network's router from the outside world. This is necessary so that devices outside your network know where to go to connect to your internal network's VPN. Your public IP address is available by visiting [IP Leak](http://www.ipleak.com), and the `ListenPort` should be set to the same port you set in your WireGuard's `wg0.conf` file (the default port is `51820`).
```
# For full tunnel use 0.0.0.0/0, ::/0 and for split tunnel use 192.168.1.0/24
# or whatever your router’s subnet is
AllowedIPs = 0.0.0.0/0, ::/0
```
For this use case, we're using a full tunnel rather than a [split tunnel](https://vpnbase.com/blog/what-is-vpn-split-tunneling/) (which allows some network traffic to come through outside of the VPN).

### Optional: Setup Dynamic DNS for Your Public IP address
If your ISP does not provide you with a static IP address (most don’t), and your IP changes for some reason (cable modem reboot, connectivity issues, etc.), your home network may be unreachable from the outside until you update it in your configuration files.  The solution is to use a DDNS (Dynamic DNS) service where you choose a readable domain name and can automatically notify the service when your public IP address changes.

So instead of worry about whether your public IP address is `98.10.200.11` or `98.10.200.42`, you can instead point a domain name like `username.us.to` at your public IP address and have the DDNS service update the domain record when your public IP address changes.

There are plenty of DDNS services out there, but I’m using [Afraid.org’s Free DNS](http://freedns.afraid.org) service because it doesn’t nag you to login every 30 days, even on the completely free plan.

#### Get a Free DNS Subdomain
First, create a Free DNS account at: http://freedns.afraid.org

After account creation, verify your account using the link you’ll receive in your email. Then go to **Add Subdomain** and enter a **Subdomain** (a username, your name, company name, etc.) and choose a **Domain** (there are many to choose from besides what’s listed, follow the instructions to find the rest). Leave **Type** set to an A record, **TTL** is set to 3600 for free accounts, and **Wildcard** functionality is only for paid accounts.

Your public IP address should be automatically filled in, but you can visit [IP Leak](http://www.ipleak.com) in a browser to get your public IP address if you need to.

Enter the CAPTCHA text and hit **Save** to continue. You should now have a shiny new subdomain pointing to your network’s public IP address! But what if your public IP address changes at some point?

#### Automatically Update Your Public IP Address
Having a domain name pointed to your public IP address is useless if that IP address changes in the future, which is why DDNS services exist in the first place. We'll need to update our Free DNS record if our network's public IP address changes, which is fairly simple to do.

While logged in at [Free DNS](http://freedns.afraid.org), go to the **Dynamic DNS** page and look for your new subdomain record. Right click the **Direct URL** link associated with your subdomain record and choose **Copy Link**. We'll use this to update your subdomain record directly from the Raspberry Pi.

On the Raspberry Pi, create a cronjob that runs every 5 minutes (replacing the XXXXX with the unique identifier in your Direct URL):
```
crontab -l | { cat; echo "*/5 * * * * curl http://freedns.afraid.org/dynamic/update.php?XXXXX”; } | crontab -
```
You’ll see `no crontab for ...` on the console, you can safely ignore that. You can change the timing from 5 minutes to 20 minutes (or whatever you'd like) by adjusting the `*/5 * * * *` part to `*/20 * * * *`. Verify that you’ve added the command correctly with:
```
crontab -l
```
Once you've finished, restart the cron service with:
```
sudo service cron restart
```
Now your DDNS subdomain will always point to the correct public IP address of your network, and VPN clients will be able to reach your network remotely regardless of whether your public IP address changes.

#### Using a Dynamic Subdomain Instead of a Public IP address
Go back to your WireGuard client configuration file and use your new DDNS subdomain with the `ListenPort` you set earlier and never worry about your public IP address changing! In the `/etc/wireguard/peer1.conf` file, edit the `Endpoint`:
```
Endpoint = username.us.to:51820
```
using the subdomain you chose on [Free DNS](http://freedns.afraid.org) and the `ListenPort` you set in your `/etc/wireguard/wg0.conf` file.

### Setting Up Your Phone to Use the VPN
Unlike IPSec or IKEv2, WireGuard isn’t built into the Android or iOS operating system (yet), so you’ll have to download [the WireGuard app](https://www.wireguard.com/install/) to each device to connect to your VPN. Here are some of the VPN clients available:

Android: https://play.google.com/store/apps/details?id=com.wireguard.android&hl=en_US
iOS: https://apps.apple.com/us/app/wireguard/id1441195209
macOS: https://apps.apple.com/us/app/wireguard/id1451685025?mt=12

#### Export Client Configuration with a QR Code
Rather than manually enter all the WireGuard configuration details into your phone, we can create a QR code directly on the Raspberry Pi console that your phone's native WireGuard app can scan and automatically fill out the details for you.

First, install the QR encoder on the Raspberry Pi:
```
sudo apt install qrencode
```
Become the root user in order to read the WireGuard client config:
```
sudo su
```
Create a QR code from the VPN client configuration we set up earlier:
```
qrencode -t ansiutf8 < /etc/wireguard/peer1.conf
```
**Note**: You may have to adjust the size of your Terminal/console window to properly show the QR code generated

#### Import Client Configuration Using the QR Code
Open the WireGuard app on your phone, tap **Add a tunnel** and select the **Create from QR code** option. Scan the QR code with your phone’s camera, give the tunnel a name, and allow WireGuard to add VPN configurations to your phone's operating system.

Now you can enable or disable VPN access directly through the WireGuard app! 

### Finish WireGuard Installation
On your Raspberry Pi, there are a few more steps needed to complete setup of the WireGuard VPN. First, allow WireGuard to start on boot:
```
sudo systemctl enable wg-quick@wg0
```
Set the correct permissions on the Wireguard configuration file with:
```
sudo chown -R root:root /etc/wireguard/
sudo chmod -R og-rwx /etc/wireguard/
```
On your Pi-Hole Web Interface, go to **Settings > DNS** and choose the **Listen on all interfaces, permit all origins** option under **Interface listening behavior**, then save your settings.

Start WireGuard now with:
```
sudo wg-quick up wg0
```
To check and see if the WireGuard interface was created successfully:
```
ifconfig wg0
```
Restart your Raspberry Pi with:
```
sudo reboot
```
Once the Raspberry Pi is done booting, check if WireGuard working:
```
sudo wg
```
You should see interface `wg0` and a peer.

### Open WireGuard VPN Port on Your Router
Without this step, you will not be able to reach your VPN from outside of the network. None of the write-ups I found mentioned this step, most likely assuming you would already know to do this (I didn't).

In order to reach your VPN from outside your network, you'll have to setup a port forward on your network's router that passes traffic on that port to an internal IP address of your choosing. Generally speaking, this might look like:
```
Description: WireGuard VPN
Public UDP Ports: 51820
Public TCP Ports: 51820
Private IP Address: 192.168.x.x
Private UDP Ports: 51820
Private TCP Ports: 51820
```
Where `192.168.x.x` is the internal static IP address of the Raspberry Pi running WireGuard, and `51820` is whatever you set the `ListenPort` to in your `/etc/wireguard/wg0.conf` file. Each router has different settings available, but the above are from my Apple Time Capsule's AirPort Utility in macOS.

**Note**: I'm passing through both TCP & UDP traffic on this port, which I assume is correct. That said, I'm not a network technician, so this may not be best practice.

Once you've added this port forwarding on your network's router, restart the device and now you should be able to connect to your WireGuard VPN from outside your network and enjoy the benefits of network-level ad-blocking from anywhere, at any time!

## References

There are several write-ups out there on how to do this, as well as install scripts to do it for you. Since the Raspberry Pi was meant to be a learning tool, I used this opportunity to figure things out on my own with the help of documentation from both software creators and the community. If it weren't for the latter, I doubt I would've been able to do this on my own. Thanks to everyone who has taken the time to share their knowledge, and experience, in setting up a Raspberry Pi.

https://www.sethenoka.com/build-your-own-wireguard-vpn-server-with-pi-hole-for-dns-level-ad-blocking/
https://github.com/harrypnyce/raspbian10-buster
https://github.com/crozuk/pi-hole-wireguard-privoxy
https://github.com/anudeepND/pihole-unbound
https://www.nlnetlabs.nl/documentation/unbound/howto-anchor/
https://github.com/ShaneCaler/EasyAsPiInstaller
