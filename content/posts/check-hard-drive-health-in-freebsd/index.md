---
title: "Check Hard Drive Health in FreeBSD"
date: 2020-03-28T18:29:24-07:00
tags: ["FreeBSD"]
---

It's wise to check a new hard drive for [S.M.A.R.T.](https://en.wikipedia.org/wiki/S.M.A.R.T.) errors before using it the first time and also to keep an eye on it after it goes into production.
In FreeBSD, the [Smartmontools](https://www.smartmontools.org/) package provides a way to check drives for errors on demand using `smartctl` and on schedule using `smartd`.

<!--more-->

To get started, find and install the latest version.

{{< highlight txt >}}
$ su
Password:

# pkg search smartmontools
smartmontools-7.0_2            S.M.A.R.T. disk monitoring tools
ssd_report-smartmontools-0.4   SSD health report

# pkg install -y smartmontools
[...]
smartmontools has been installed

To check the status of drives, use the following:

        /usr/local/sbin/smartctl -a /dev/ad0    for first ATA/SATA drive
        /usr/local/sbin/smartctl -a /dev/da0    for first SCSI drive
        /usr/local/sbin/smartctl -a /dev/ada0   for first SATA drive

To include drive health information in your daily status reports,
add a line like the following to /etc/periodic.conf:
        daily_status_smart_devices="/dev/ad0 /dev/da0"
substituting the appropriate device names for your SMART-capable disks.

To enable drive monitoring, you can use /usr/local/sbin/smartd.
A sample configuration file has been installed as
/usr/local/etc/smartd.conf.sample
Copy this file to /usr/local/etc/smartd.conf and edit appropriately

To have smartd start at boot
        echo 'smartd_enable="YES"' >> /etc/rc.conf
{{< /highlight >}}

Use `smartctl --scan` to list the drives recognized by smartctl.

{{< highlight txt >}}
# smartctl --scan
/dev/ada0 -d atacam # /dev/ada0, ATA device
/dev/ada1 -d atacam # /dev/ada1, ATA device
/dev/ada2 -d atacam # /dev/ada2, ATA device
/dev/ada3 -d atacam # /dev/ada3, ATA device
{{< /highlight >}}

In this system, drives **ada0**-**ada2** make up a 3-drive ZFS mirror and drive **ada3** is a removable drive that will be used for backups.

To check the SMART status of a new drive before putting it into service the first time, run the *long* test using `smartctl -t long`.

{{< highlight txt >}}
# smartctl -t long /dev/ada3
smartctl 7.0 2018-12-30 r4883 [FreeBSD 12.1-RELEASE amd64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF OFFLINE IMMEDIATE AND SELF-TEST SECTION ===
Sending command: "Execute SMART Extended self-test routine immediately in off-line mode".
Drive command "Execute SMART Extended self-test routine immediately in off-line mode" successful.
Testing has begun.
Please wait 167 minutes for test to complete.
Test will complete after Sat Mar 28 22:25:41 2020

Use smartctl -X to abort test.
{{< /highlight >}}

The software estimates and reports how long it will take to run as it begins.  Use `smartctl -a` to keep an eye on its progress.

{{< highlight txt >}}
# smartctl -a /dev/ada3 | grep remaining
                                        90% of test remaining.
{{< /highlight >}}

After the test finishes, examine the full results.

{{< highlight txt >}}
# smartctl -a /dev/ada3
smartctl 7.0 2018-12-30 r4883 [FreeBSD 12.1-RELEASE amd64] (local build)
Copyright (C) 2002-18, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
[...]
{{< /highlight >}}

From what I can find, these are the most important attributes to monitor for an early warning of potential drive failure.

|ID  |Attribute Name               |
|----|-----------------------------|
|  5 |Reallocated Sector Count     |
|187 |Reported Uncorrectable Errors|
|188 |Command Timeout              |
|197 |Current Pending Sector Count |
|198 |Offline Uncorrectable        |

SMART error 199 (*UDMA CRC Error Count*) might indicate a problem with the drive cable or port, so it's worth checking that one too for a new drive.

{{< highlight txt >}}
# smartctl -f brief -a /dev/ada3 | grep -E '^(ID#| {2}5|187|188|197|198|199)'
ID# ATTRIBUTE_NAME          FLAGS    VALUE WORST THRESH FAIL RAW_VALUE
  5 Reallocated_Sector_Ct   PO--CK   100   100   005    -    0
187 Reported_Uncorrect      -O--CK   100   100   000    -    0
197 Current_Pending_Sector  -O--C-   100   100   000    -    0
198 Offline_Uncorrectable   ----C-   100   100   000    -    0
199 UDMA_CRC_Error_Count    -OSRCK   200   200   000    -    0
{{< /highlight >}}

The exact meaning of each SMART attribute is not standardized, and this particular drive only reports five of those six attributes, but the *RAW_VALUE*s in the right column are the ones to watch.

On this removable drive, all five attributes report a *RAW_VALUE* of 0, so I'll put this one into production.

---

The output text given by the installer also explains how to set up `smartd` for regular automatic reporting of SMART attributes.

To configure the system to run `smartd`, create the configuration file */usr/local/etc/smartd.conf* and add the system drives to it.

{{< highlight txt >}}
# touch /usr/local/etc/smartd.conf

# echo "/dev/ada0" >> /usr/local/etc/smartd.conf
# echo "/dev/ada1" >> /usr/local/etc/smartd.conf
# echo "/dev/ada2" >> /usr/local/etc/smartd.conf

# cat /usr/local/etc/smartd.conf
/dev/ada0
/dev/ada1
/dev/ada2
{{< /highlight >}}

> The installation also includes an extensive sample configuration file at /usr/local/etc/smartd.conf.sample, but the most basic setup just requires a list of drives to monitor.

Next, use `sysrc` to add `smartd_enable=YES` to **rc.conf** and then start the service manually.

{{< highlight txt >}}
# sysrc smartd_enable=YES
smartd_enable:  -> YES

# service smartd start
Starting smartd.
{{< /highlight >}}

Monitor */var/log/messages* regularly to see the results of the SMART analysis.

{{< highlight txt >}}
# grep smartd /var/log/messages
Mar 28 21:41:48 server smartd[11127]: Device: /dev/ada2, WARNING: A firmware update for this drive may be available,
Mar 28 21:41:48 server smartd[11127]: see the following Seagate web pages:
Mar 28 21:41:48 server smartd[11127]: http://knowledge.seagate.com/articles/en_US/FAQ/207931en
Mar 28 21:41:48 server smartd[11127]: http://knowledge.seagate.com/articles/en_US/FAQ/213891en
{{< /highlight >}}
