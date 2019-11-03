---
title: "Use iocage Plugins to Jail Common Applications"
date: 2019-07-16T18:20:58-07:00
tags: ["FreeBSD", "iocage", "jails"]
---

The [plugin](https://iocage.readthedocs.io/en/latest/plugins.html) feature in [iocage](../create-vnet-jails-in-freebsd-12-using-iocage) makes it easy
to create pre-configured container jails for many [common applications](https://raw.githubusercontent.com/freenas/iocage-ix-plugins/master/INDEX).

<!--more-->

Use **iocage list** to see the available applications.

{{< highlight txt >}}
$ su
Password:
# iocage -v
Version 1.1 RELEASE 2019/01
# iocage list --plugins --remote
'HTTPResponse' object has no attribute 'geturl'
{{< /highlight >}}

For me, the list command didn't work properly in version 1.1. Fortunately, it was easy to install the latest version from source.

To do so, first install the FreeBSD source tree into /usr/src.

{{< highlight txt >}}
# ls /usr/src/
# fetch -o /tmp ftp://ftp.freebsd.org/pub/`uname -s`/releases/`uname -m`/`uname -r | cut -d'-' -f1,2`/src.txz
# tar -C / -xvf /tmp/src.txz
# ls /usr/src/
.arcconfig              README.md               rescue
.arclint                UPDATING                sbin
.gitattributes          bin                     secure
.gitignore              cddl                    share
COPYRIGHT               contrib                 stand
LOCKS                   crypto                  sys
MAINTAINERS             etc                     targets
Makefile                gnu                     tests
Makefile.inc1           include                 tools
Makefile.libcompat      kerberos5               usr.bin
Makefile.sys.inc        lib                     usr.sbin
ObsoleteFiles.inc       libexec
README                  release
{{< /highlight >}}

Next, install the lastest master branch of iocage from source.

{{< highlight txt >}}
# cd /tmp
# pkg install python36 git-lite py36-cython py36-pip
# git clone --recursive https://github.com/iocage/iocage
# cd iocage && make install
{{< /highlight >}}

Try **iocage list** again.

{{< highlight txt >}}
# iocage -v
Version 1.2 RC
# iocage list -PR
Branch 12.0-RELEASE does not exist at https://github.com/freenas/iocage-ix-plugins.git!
Using "master" branch for plugin, this may not work with your RELEASE
+-------------------+-------------------+-------------------+------------------+
|       NAME        |    DESCRIPTION    |        PKG        |       ICON       |
+===================+===================+===================+==================+
| BackupPC          | BackupPC is a     | backuppc          | https://www.true |
|                   | high-performance, |                   | os.org/iocage-ic |
|                   | enterprise-grade  |                   | ons/backuppc.png |
|                   | system for        |                   |                  |
|                   | backing up Linux, |                   |                  |
|                   | WinXX and MacOSX  |                   |                  |
|                   | PCs and laptops   |                   |                  |
|                   | to a server's     |                   |                  |
|                   | disk.             |                   |                  |
+-------------------+-------------------+-------------------+------------------+
...
+-------------------+-------------------+-------------------+------------------+
| Nextcloud         | Access, share and | nextcloud         | https://www.true |
|                   | protect your      |                   | os.org/iocage-ic |
|                   | files, calendars, |                   | ons/nextcloud.pn |
|                   | contacts,         |                   | g                |
|                   | communication &   |                   |                  |
|                   | more at home and  |                   |                  |
|                   | in your           |                   |                  |
|                   | enterprise.       |                   |                  |
+-------------------+-------------------+-------------------+------------------+
...
{{< /highlight >}}

To create a jailed application, such as [NextCloud](https://nextcloud.com/), call **iocage fetch** and use the **-P** option to specify the **PKG** name from the list above.
To make the jail request its IP address using DHCP, also specify the **bpf** and **dhcp** parameters.

{{< highlight txt >}}
# iocage fetch -P nextcloud vnet="on" allow_raw_sockets="1" boot="on" bpf="yes" dhcp="on"
Plugin: Nextcloud
  Official Plugin: True
  Using RELEASE: 11.2-RELEASE
  Using Branch: 12.0-RELEASE
  Post-install Artifact: https://github.com/freenas/iocage-plugin-nextcloud.git
  These pkgs will be installed:
    - nextcloud-php71
    - nginx
    - mysql56-server
...
Admin Portal:
http://192.168.0.117
{{< /highlight >}}

Alternatively, to create a NextCloud jail using a static IP address, specify the **defaultrouter** and **ip4_addr** parameters.

{{< highlight txt >}}
# iocage fetch -P nextcloud vnet="on" allow_raw_sockets="1" boot="on" defaultrouter="192.168.0.1" ip4_addr="192.168.0.254/24"
Plugin: Nextcloud
  Official Plugin: True
  Using RELEASE: 11.2-RELEASE
  Using Branch: 12.0-RELEASE
  Post-install Artifact: https://github.com/freenas/iocage-plugin-nextcloud.git
  These pkgs will be installed:
    - nextcloud-php71
    - nginx
    - mysql56-server
...
Admin Portal:
http://192.168.0.254
{{< /highlight >}}

Finally, open the jailed address in a web browser.

{{< figure src="nextcloud-running-in-an-iocage-jail.png" alt="NextCloud Running in an iocage Jail">}}
