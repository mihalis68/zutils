Z Utils
=======

This is a tiny collection of tools for leveraging Multicast DNS (aka
Zeroconf or Bonjour) on a mixed network of Apple Macs and other Unix
machines. The focus is on making it easier to get around (find and
login remotely to other machines) starting from a Mac workstation,
since Apple's mDNS is less command-line friendly than the Linux
version (avahi).

My setup tends to be that I work from a Mac and login to other macs
and non-mac unix machines using Remote Desktop, file sharing and ssh a
lot. I like the fact that Apple's Finder automatically finds (using
mDNS) all local hosts that advertise their presence. Launching a
remote desktop or file sharing session to another Mac directly from
Finder is very convenient. This is so convenient that I even get my
non-Mac Unix hosts to advertise themselves so that they show up in
Finder too (Linux has a fine zeroconf implementation too). This is
particularly handy for headless hosts that come and go on
DHCP-assigned IP addresses, for example my growing collection of
Raspberry Pis.

However, I was finding it annoying if I want to ssh to local hosts I
have to type their exact hostname, which is often long on a Mac. I
wanted some kind of substring matching, so I only need to provide
something unambiguous and let the machine work it out. 

Further, using raw mDNS to ssh to Macs, the .local hostname is not
even the same string as the displayed name, thus "Christopher's
MacBook Pro" is converted to "Christophers-MacBook-Pro" in mDNS (the
apostrophe is removed and spaces are converted to hyphens). You can
see this in action on the System Preference Sharing menu where
"Computer Name" and the hostname in .local have to both be shown.

This all makes ssh a bit of a poor relation (IMO) compared to the GUI
tools.

I started to look into scripting easier use of mDNS from the
command-line to allow me to get into my machines more easily, but
another irritation arose. Apple's documentation for mDNS service
discovery (see man dns-sd) tries to scare you away from calling it
from scripts. Firstly the output format is not stable. Second, the
commands are meant to be consumed as a stream. That's ok for a GUI
application such as Finder which runs for a relatively long time. Not
ok for a quick and dirty comand-line lookup tool.

Apple instead recommends using language-specific bindings for the
language of your choice to call the underlying DNS-SD API. All very
well for some serious application, but overkill for a small
utility. So I built Z Utils instead. 

Z Utils for now consists of a lookup tool which looks up those hosts
which advertise ssh, a clone of the Unix "host" command which
leverages the lookup for simple queries and also an SSH wrapper which
looks up the desired host and then invokes ssh to the resulting
hostname or IP address to easily connect you to the target machine.

* zeroconf-ssh-host

This utility finds a host or hosts that advertise ssh locally and
returns their details from mDNS as a series of lines. Bash
sub-processes and timeouts are used to hide the fact that dns-sd does
not terminate, and the output of Apple's tool is parsed with simple
easy to hack unix editing pipelines. This is the meat of the Z utils
project.

* zhost

This utility approximates the behavior of host, but with the substring
matching described previously.

* zssh

This utility provides a way to connect to any ssh-advertising host by
providing any substring of its name provided that that host can be
resolved in mDNS. The provided name is looked up using
zeroconf-ssh-host and the first host that matches will be the one
connected to.

Usage examples
==============

In the following examples there are two hosts, my MacBook Pro on wifi,
and a raspberry pi, hostname raspberrypi1 which is connected by both
ethernet and wifi. The rpi is advertising ssh via mDNS, see below for
how to configure that.

* Lookup all ssh-advertising hosts

```
Christophers-MacBook-Pro:zu cmorgan$ zeroconf-ssh-host 
15:48:45.731  Add     2  4 raspberrypi1.local.                    10.0.1.38                                    120
15:48:45.731  Add     3  4 raspberrypi1.local.                    10.0.1.7                                     120
15:48:44.721  Add     2  4 Christophers-MacBook-Pro.local.        10.0.1.13                                    120
Christophers-MacBook-Pro:zu cmorgan$ 
```

* Run a single command on the raspberry pi

```
Christophers-MacBook-Pro:zu cmorgan$ zssh pi hostname
raspberrypi1
Christophers-MacBook-Pro:zu cmorgan$ 
```

* Log into the raspberry pi interactively

```
Christophers-MacBook-Pro:zu cmorgan$ zssh pi
Linux raspberrypi1 3.18.11+ #781 PREEMPT Tue Apr 21 18:02:18 BST 2015 armv6l

The programs included with the Debian GNU/Linux system are free software;
the exact distribution terms for each program are described in the
individual files in /usr/share/doc/*/copyright.

Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
permitted by applicable law.
No mail.
Last login: Sat Dec 26 20:47:27 2015 from christophers-macbook-pro.local
cmorgan@raspberrypi1 ~ $ 
```

Detailed Command-line options
=============================

The full syntax for invoking zssh is

```
zssh <flags> <hostname> <ssh params>
```

The flags are as follows :

-v Verbose
-h Login via hostname

So to verbosely ssh to the rasperry pi using its hostname in
.local. (instead of merely connecting to its IP address) you would do
the following:

```
Christophers-MacBook-Pro:zu cmorgan$ ./zssh -v -h pi /sbin/iwconfig
Found raspberrypi1.local. in local domain
Output of "ssh raspberrypi1.local. /sbin/iwconfig" :
lo        no wireless extensions.x

eth0      no wireless extensions.

wlan0     IEEE 802.11bgn  ESSID:"Chris Morgan's Upstate Network"  
          Mode:Managed  Frequency:2.437 GHz  Access Point: 28:CF:DA:B4:03:6D   
          Bit Rate=65 Mb/s   Tx-Power=20 dBm   
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:off
          Link Quality=70/70  Signal level=-19 dBm  
          Rx invalid nwid:0  Rx invalid crypt:0  Rx invalid frag:0
          Tx excessive retries:2  Invalid misc:102   Missed beacon:0
```

Advertising ssh from linux
==========================

Assuming you copy the provided ssh.service file onto a Linux machine,
and assuming it's a system with apt-get (Ubuntu, or raspbian as I use
on my Raspberry Pi machines) the following commands would configure
that machine to advertise ssh via mDNS :

```
sudo apt-get install libnss-mdns avahi-daemon
sudo cp ssh.service /etc/avahi/services
sudo service avahi-daemon restart
```

Explanation:

* libnss-mdns adds mDNS as a name resolution backend to GNU libC
* avahi-daemon is the mDNS (zeroconf) implementation for Linux
* ssh.service is a service fragment that tells avahi to advertise ssh
  on port 22
* restarting the daemon forces it to notice the new service it should advertise

N.B. Also install avahi-utils if you want to be able to browse the
mDNS fields from the linux host, e.g. avahi-browse --all

Limitations
===========

This project started off as utilities for connecting *from* Apple Macs
*to* assorted Linux machines. On Linux the situation is much better
(it is easier to probe and parse mDNS info), so a zssh would be easier
to implemenet, but I haven't made this work at all yet.
