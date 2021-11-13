---
title: "Configure a Wi-Fi Client on FreeBSD"
date: 2021-11-13T10:05:09-08:00
tags: ["FreeBSD", "networking"]
---

If you need a GUI-based Wi-Fi manager for a FreeBSD desktop environment, consider using [networkmgr](https://www.freshports.org/net-mgmt/networkmgr)
or [wifimgr](https://www.freshports.org/net-mgmt/wifimgr). For console-based servers or IOT devices,
the `bsdconfig wireless` command should allow one to configure a supported wireless device on FreeBSD after installation,
but I have never been able to get it to work properly on my devices.

<!--more-->

Fortunately, it's easy to set up Wi-Fi by hand for [supported devices](https://www.freebsd.org/releases/13.0R/hardware/)
following the instructions in the [handbook](https://docs.freebsd.org/en/books/handbook/advanced-networking/#network-wireless).

First, use `sysctl` and `pciconf` to see if the system has detected a wireless device.

{{< highlight txt >}}
$ su
Password:

# uname -a
FreeBSD laptop.ccammack.com 13.0-RELEASE-p4 FreeBSD 13.0-RELEASE-p4

# sysctl net.wlan.devices
net.wlan.devices: iwn0

# pciconf -lv | grep -B3 network
[...]
iwn0@pci0:3:0:0:	class=0x028000 rev=0x34 hdr=0x00 vendor=0x8086 device=0x0085 subvendor=0x8086 subdevice=0x1311
    vendor     = 'Intel Corporation'
    device     = 'Centrino Advanced-N 6205 [Taylor Peak]'
    class      = network
{{< /highlight >}}

In this case, the system detected an *Intel Centrino Advanced-N 6205* Wi-Fi card as wireless device **iwn0**.

To find out more about available drivers for your wireless hardware, use the `apropos` and `man` commands.

{{< highlight txt >}}
# apropos intel | grep wireless
iwm, if_iwm(4) - Intel IEEE 802.11ac wireless network driver
iwn, if_iwn(4) - Intel IEEE 802.11n wireless network driver

# man iwn
[...]
{{< /highlight >}}

Support for new wireless devices often takes a while on FreeBSD, so if your hardware is not recognized, the easiest solution is to just buy an inexpensive USB Wi-Fi dongle that is already supported.
Search the [FreeBSD Release Notes](https://www.freebsd.org/releases/13.0R/hardware/) for the word *wireless* to see a list of supported hardware.

Once the system recognizes a wireless device, use either `ee` or `echo >>` to add two lines to the bottom of */etc/rc.conf* to configure it.
Replace **iwn0** in the example below with the appropriate driver name given by the `sysctl` command.

{{< highlight txt >}}
# ee /etc/rc.conf
[...]

# echo 'wlans_iwn0="wlan0"' >> /etc/rc.conf
# echo 'ifconfig_wlan0="WPA SYNCDHCP"' >> /etc/rc.conf

# cat /etc/rc.conf
[...]
wlans_iwn0="wlan0"
ifconfig_wlan0="WPA SYNCDHCP"

{{< /highlight >}}

Next, use `ee` or `wpa_passphrase >>` to add a properly-formatted *network* configuration block to the file */etc/wpa_supplicant.conf* for each Wi-Fi hotspot.

{{< highlight txt >}}
# ee /etc/wpa_supplicant.conf
[...]

# wpa_passphrase "SSID" "WPA2passphrase" >> /etc/wpa_supplicant.conf

# cat /etc/wpa_supplicant.conf
network={
        ssid="SSID"
        #psk="WPA2passphrase"
        psk=8f0022337a28144c1ee2ae9b7f570d978c9c014b9e3b6e7ad3bfaf816d272f60
}
{{< /highlight >}}

The *wpa_supplicant.conf* file can contain as many hotspots as needed. The longer *psk* value (without double-quotes) is just a hash of the commented-out *#psk* value that contains the raw passphrase.

If you need to change the passphrase for an existing hotspot, you can delete the line with the longer *psk* hash, uncomment the line with the shorter *psk* value and change it to the new raw passphrase.
The system will recognize either *psk* format.

Finally, restart the network service and `ping` to check the connection.

{{< highlight txt >}}
# service netif restart
[...]

# ifconfig
[...]
wlan0: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        ether 8c:70:5a:bd:39:3c
        inet 192.168.0.101 netmask 0xffffff00 broadcast 192.168.0.255
        groups: wlan
        ssid SSID channel 2 (2417 MHz 11g ht/20) bssid 1c:af:f7:dd:9c:3b
        regdomain FCC country US authmode WPA2/802.11i privacy ON
        deftxkey UNDEF TKIP 3:128-bit txpower 30 bmiss 10 scanvalid 60
        protmode CTS ampdulimit 64k ampdudensity 8 -amsdutx amsdurx shortgi
        -stbc -ldpc -uapsd wme roaming MANUAL
        parent interface: iwn0
        media: IEEE 802.11 Wireless Ethernet MCS mode 11ng
        status: associated
        nd6 options=29<PERFORMNUD,IFDISABLED,AUTO_LINKLOCAL>

# ping google.com
PING google.com (142.250.191.78): 56 data bytes
64 bytes from 142.250.191.78: icmp_seq=0 ttl=117 time=9.805 ms
64 bytes from 142.250.191.78: icmp_seq=1 ttl=117 time=10.081 ms
64 bytes from 142.250.191.78: icmp_seq=2 ttl=117 time=10.059 ms
^C
--- google.com ping statistics ---
3 packets transmitted, 3 packets received, 0.0% packet loss
round-trip min/avg/max/stddev = 9.805/9.982/10.081/0.125 ms
{{< /highlight >}}

To connect to a different hotspot in a new location, use `ifconfig scan` to scan for nearby hotspots and then add the desired SSID to */etc/wpa_supplicant.conf* using either `ee` or `wpa_passphrase >>` as before.

{{< highlight txt >}}
# ifconfig wlan0 list scan | more
SSID/MESH ID                      BSSID              CHAN RATE    S:N     INT CAPS
[...]
GuestWiFi                         1c:af:f7:dd:9c:3b    2   54M  -66:-95   100 EPS  RSN WPA WME HTCAP ATH WPS
[...]

# wpa_passphrase "GuestWiFi" "WPA2passphrase" >> /etc/wpa_supplicant.conf

# cat /etc/wpa_supplicant.conf
[...]
network={
        ssid="GuestWiFi"
        #psk="WPA2passphrase"
        psk=8f0022337a28144c1ee2ae9b7f570d978c9c014b9e3b6e7ad3bfaf816d272f60
}

# service netif restart
[...]

# ping google.com
[...]
{{< /highlight >}}

For reference, here are several related command line Wi-Fi scripts (and the original FreeBSD source),
but I haven't studied them enough to see which ones work on FreeBSD or could be adapted work, so I can't make any recommendations about them.

* [script to start/stop wireless connections](https://forums.freebsd.org/threads/useful-scripts.737/page-8#post-203539)
* [FreeBSD Network Management with network.sh Script](https://vermaden.wordpress.com/2018/03/24/freebsd-network-management-with-network-sh-script/)
* [gihnius/freebsd-wifi](https://github.com/gihnius/freebsd-wifi)
* [goulov/wifimgr](https://github.com/goulov/wifimgr)
* [freebsd/freebsd-src/usr.sbin/bsdconfig/networking/](https://github.com/freebsd/freebsd-src/tree/main/usr.sbin/bsdconfig/networking)
* [freebsd/freebsd-src/usr.sbin/bsdinstall/scripts/](https://github.com/freebsd/freebsd-src/tree/main/usr.sbin/bsdinstall/scripts)
