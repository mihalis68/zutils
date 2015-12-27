Z Utils
=======

This is a tiny collection of tools for leveraging Multicast DNS aka
Zeroconf aka Bonjour on a network of mixed Apple Macs and other Unix
machines. 

My setup tends to be that I work from a Mac and login to other macs
and non-mac unix machines using Remote Desktop, file sharing and ssh a
lot. I like the fact that Apple's Finder automatically finds (using
mDNS) all local hosts that advertise their presence. Launching a
remote desktop or file sharing session to another Mac direcly from
Finder is very convenient. This is so convenient that I even get my
non-Mac unix hosts to advertise themselves so that they show up in
Finder. This is particularly handy for headless hosts that come and go
on DHCP-assigned IP addresses, for example my growing collection of
Raspberry Pis.

However, I was finding it annoying if I want to ssh to local hosts I
have to type their exact hostname, which is often long on a
Mac. Further, using mDNS to ssh to Macs, the .local hostname is not
even the same string as the "Display Name", thus "Christopher's
MacBook Pro" is converted to "Christophers-MacBook-Pro" in mDNS. The
apostrophe is removed and spaces are converted to hyphens. This all
makes ssh a bit of a poor relation compared to the GUI tools.

I started to look into scripting the use of mDNS from the command-line
to allow me to get into my machines more easily, but another
irritation arose. Apple's documentation for mDNS service discovery
(see man dns-sd) tries to scare you away from calling it from
scripts. Firstly the output format is not stable. Second, the commands
are meant to be consumed as a stream. That's ok for a GUI application
such as Finder which runs for a relatively long time. Not ok for a
quick and dirty comand-line lookup tool.

Apple instead recommends using language-specific bindings for the
language of your choice to call the underlying DNS-SD API. All very
well for some serious application, but overkill for a small
utility. So I built this, "Z Utils". Z Utils for now consists of a
lookup tool which looks up those hosts which advertise ssh, together
with an SSH wrapper which looks up the desired host using the lookup
tool and then invokes ssh to the resulting hostname or IP address to
easily connect you to the target machine.

* zeroconf-ssh-host

This utility finds a host or hosts that advertise ssh locally and
returns their details from mDNS as a series of lines.

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
and a raspberry pi, hostname raspberrypi which is connected by both
ethernet and wifi. The rpi is advertising ssh via mDNS, see below for
how to configure that.

* Lookup all ssh-advertising hosts

```
Christophers-MacBook-Pro:zu cmorgan$ zeroconf-ssh-host 
15:48:45.731  Add     2  4 raspberrypi1.local.                    10.0.1.38                                    120
15:48:45.731  Add     3  4 raspberrypi1.local.                    10.0.1.7                                     120
15:48:44.721  Add     2  4 Christophers-MacBook-Pro.local.        10.0.1.13                                    120
Christophers-MacBook-Pro:zu cmorgan$ ```

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

zssh <flags> <hostname> <ssh params>

The flags are as follows :

-v Verbose
-h Login via hostname

So to verbosely ssh to the rasperry pi using its hostname in .local.
you would do the following:

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

