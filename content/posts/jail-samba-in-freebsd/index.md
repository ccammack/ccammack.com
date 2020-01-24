---
title: "Jail Samba in FreeBSD"
date: 2019-11-03T16:40:06-08:00
tags: ["FreeBSD", "iocage", "jails", "Samba"]
---

The release of FreeBSD 12 with VNET support has made it easy to jail a Samba file server using [iocage](../create-vnet-jails-in-freebsd-12-using-iocage).

<!--more-->

To create a jail that uses DHCP to request an IP address from the router, call **iocage create** and specify the **bpf** and **dhcp** parameters.
Alternatively, to create a jail with a static IP address, call **iocage create** and specify the **defaultrouter** and **ip4_addr** parameters.

In this example, I create a new jail named **media** that relies on DHCP to reserve IP address 192.168.0.**108** on startup.

{{< highlight txt >}}
$ su
Password:

# iocage create -n "media" -r latest vnet="on" allow_raw_sockets="1" boot="on" bpf="yes" dhcp="on"
media successfully created!
media: vnet requires defaultrouter, using 192.168.0.1
* Starting media
  + Started OK
  + Using devfs_ruleset: 5
  + Configuring VNET OK
  + Using IP options: vnet
  + Starting services OK
  + Executing poststart OK
  + DHCP Address: 192.168.0.108/24

# iocage list
+-----+-------+-------+--------------+------+
| JID | NAME  | STATE |   RELEASE    | IP4  |
+=====+=======+=======+==============+======+
| 1   | media | up    | 12.1-RELEASE | DHCP |
+-----+-------+-------+--------------+------+
{{< /highlight >}}

Inside the jail, use **pkg search** to find the latest version of Samba.

{{< highlight txt >}}
# iocage exec media pkg search samba
The package management tool is not yet installed on your system.
Do you want to fetch and install it now? [y/N]: y
Bootstrapping pkg from pkg+http://pkg.FreeBSD.org/FreeBSD:12:amd64/quarterly, please wait...
Verifying signature with trusted certificate pkg.freebsd.org.2013102301... done
[media] Installing pkg-1.12.0...
[media] Extracting pkg-1.12.0: 100%
pkg: Repository FreeBSD missing. 'pkg update' required
p5-Samba-LDAP-0.05_2           Manage a Samba PDC with an LDAP Backend
p5-Samba-SIDhelper-0.0.0_3     Create SIDs based on G/UIDs
samba-nsupdate-9.14.2_1        nsupdate utility with GSS-TSIG support
samba410-4.10.8                Free SMB/CIFS and AD/DC server and client for Unix
samba48-4.8.12_4               Free SMB/CIFS and AD/DC server and client for Unix
{{< /highlight >}}

Samba **4.10.8** seems to be the latest package, so use **pkg install** to install it.

{{< highlight txt >}}
# iocage exec media pkg install -y samba410
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
The following 53 package(s) will be affected (of 0 checked):

New packages to be INSTALLED:
        samba410: 4.10.8
...

Message from samba410-4.10.8:
--
How to start: http://wiki.samba.org/index.php/Samba4/HOWTO
* Your configuration is: /usr/local/etc/smb4.conf
* All the relevant databases are under: /var/db/samba4
* All the logs are under: /var/log/samba4
* Provisioning script is: /usr/local/bin/samba-tool

For additional documentation check: http://wiki.samba.org/index.php/Samba4

Bug reports should go to the: https://bugzilla.samba.org/
{{< /highlight >}}

Samba configuration is notoriously confusing. For this example, I want to achieve these results:

* create a single user account for Samba access
* require a password to connect to the Samba shares
* deny access to all other users
* create the Samba shares in the user's home directory
* create new shares as needed
* configure shares to be read-only by default
* configure particular shares to be writable as needed

Create a user account for Samba access and assign it a password.

{{< highlight txt >}}
# iocage exec media adduser
Username: ccammack
Full name:
Uid (Leave empty for default):
Login group [ccammack]:
Login group is ccammack. Invite ccammack into other groups? []: wheel
Login class [default]:
Shell (sh csh tcsh nologin) [sh]:
Home directory [/home/ccammack]:
Home directory permissions (Leave empty for default):
Use password-based authentication? [yes]:
Use an empty password? (yes/no) [no]:
Use a random password? (yes/no) [no]:
Enter password:
Enter password again:
Lock out the account after creation? [no]:
Username   : ccammack
Password   : *****
Full Name  :
Uid        : 1001
Class      :
Groups     : ccammack wheel
Home       : /home/ccammack
Home Mode  :
Shell      : /bin/sh
Locked     : no
OK? (yes/no): yes
adduser: INFO: Successfully added (ccammack) to the user database.
Add another user? (yes/no): no
Goodbye!
{{< /highlight >}}

Create the folder you intend to share, along with an extra folder and file for testing, then change their owner to the Samba user.

{{< highlight txt >}}
# iocage exec media mkdir -p /home/ccammack/media/test
# iocage exec media touch /home/ccammack/media/test/test.txt
# iocage exec media chown -R ccammack:ccammack /home/ccammack/media
{{< /highlight >}}

Use **ifconfig** to get the name of the jail's interface, which is **epair0b** in this case.

{{< highlight txt >}}
# iocage exec media ifconfig
lo0: flags=8049<UP,LOOPBACK,RUNNING,MULTICAST> metric 0 mtu 16384
        options=680003<RXCSUM,TXCSUM,LINKSTATE,RXCSUM_IPV6,TXCSUM_IPV6>
        inet6 ::1 prefixlen 128
        inet6 fe80::1%lo0 prefixlen 64 scopeid 0x1
        inet 127.0.0.1 netmask 0xff000000
        groups: lo
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
epair0b: flags=8843<UP,BROADCAST,RUNNING,SIMPLEX,MULTICAST> metric 0 mtu 1500
        options=8<VLAN_MTU>
        ether 28:92:4a:db:46:19
        hwaddr 02:0f:6a:76:7e:0b
        inet6 fe80::2a92:4aff:fedb:4619%epair0b prefixlen 64 scopeid 0x2
        inet 192.168.0.108 netmask 0xffffff00 broadcast 192.168.0.255
        groups: epair
        media: Ethernet 10Gbase-T (10Gbase-T <full-duplex>)
        status: active
        nd6 options=21<PERFORMNUD,AUTO_LINKLOCAL>
{{< /highlight >}}

Inside the jail, create the samba config file (**/usr/local/etc/smb4.conf**) using **ee** or **vi**.

{{< highlight txt >}}
# iocage exec media ee /usr/local/etc/smb4.conf
...

# iocage exec media cat /usr/local/etc/smb4.conf
[global]
workgroup = WORKGROUP
netbios name = MEDIA
security = user
passdb backend = tdbsam
encrypt passwords = yes
force user = ccammack
max log size = 512
hosts allow = 192.168.0.
interfaces = epair0b
bind interfaces only = yes
socket options = TCP_NODELAY
writable = no

[media]
path = /home/ccammack/media
#writable = yes
{{< /highlight >}}

Use **pdbedit** to map the user account and password to the Samba database.

{{< highlight txt >}}
# iocage exec media pdbedit --create --user=ccammack
new password:
retype new password:
Unix username:        ccammack
NT username:
Account Flags:        [U          ]
User SID:             S-1-5-21-1722310244-3315895870-785234628-1000
Primary Group SID:    S-1-5-21-1722310244-3315895870-785234628-513
Full Name:            User &
Home Directory:       \\media\ccammack
HomeDir Drive:
Logon Script:
Profile Path:         \\media\ccammack\profile
Domain:               MEDIA
Account desc:
Workstations:
Munged dial:
Logon time:           0
Logoff time:          9223372036854775807 seconds since the Epoch
Kickoff time:         9223372036854775807 seconds since the Epoch
Password last set:    Mon, 11 Nov 2019 01:09:37 PST
Password can change:  Mon, 11 Nov 2019 01:09:37 PST
Password must change: never
Last bad password   : 0
Bad password count  : 0
Logon hours         : FFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFF
{{< /highlight >}}

Use **sysrc** to add **samba_server_enable=YES** to the jail's **rc.conf** and start the samba server immediately.

{{< highlight txt >}}
# iocage exec media sysrc samba_server_enable=YES
# iocage exec media service samba_server start
Performing sanity check on Samba configuration: OK
Starting nmbd.
Starting smbd.
{{< /highlight >}}

From another machine, browse to the **\\\\MEDIA** share, enter the Samba user name and password when requested, and note that the contents of **\\\\MEDIA\media** are read-only.
With the current Samba configuration, the user cannot make any changes to the **test** folder or the **test.txt** file inside it.

{{< figure src="media-share.png" alt="Browse to Network MEDIA share">}}
{{< figure src="samba-user-and-password.png" alt="Enter Samba user name and password">}}

To allow changes inside the **\\\\MEDIA\\media** folder, edit the configuration file again, uncomment the line at the bottom that says **#writable = yes** and restart Samba.

{{< highlight txt >}}
# iocage exec media ee /usr/local/etc/smb4.conf
...

# iocage exec media cat /usr/local/etc/smb4.conf
[global]
workgroup = WORKGROUP
netbios name = MEDIA
security = user
passdb backend = tdbsam
encrypt passwords = yes
force user = ccammack
max log size = 512
hosts allow = 192.168.0.
interfaces = epair0b
bind interfaces only = yes
socket options = TCP_NODELAY
writable = no

[media]
path = /home/ccammack/media
writable = yes

# iocage exec media service samba_server restart
Performing sanity check on Samba configuration: OK
Stopping smbd.
Waiting for PIDS: 5906.
Stopping nmbd.
Waiting for PIDS: 5901.
Performing sanity check on Samba configuration: OK
Starting nmbd.
Starting smbd.
{{< /highlight >}}

The **\\\\MEDIA\media** folder should now allow changes to the **test** folder and **test.txt** file.
