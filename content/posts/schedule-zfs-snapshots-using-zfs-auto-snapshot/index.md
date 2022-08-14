---
title: "Schedule ZFS Snapshots Using zfs-auto-snapshot"
date: 2019-11-16T16:35:05-08:00
tags: ["FreeBSD", "ZFS", "Backups"]
---

The [ZFS Tools](https://github.com/bdrewery/zfstools) package offers an easy way to automatically create a rotating set of ZFS snapshots for backup purposes.
Here's how to set it up.

<!--more-->

{{< highlight txt >}}
$ su
Password:

# pkg install zfstools
[...]

Message from zfstools-0.3.6_1:
--
To enable automatic snapshots, place lines such as these into /etc/crontab:

    PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin
    15,30,45 * * * * root /usr/local/sbin/zfs-auto-snapshot frequent  4
    0        * * * * root /usr/local/sbin/zfs-auto-snapshot hourly   24
    7        0 * * * root /usr/local/sbin/zfs-auto-snapshot daily     7
    14       0 * * 7 root /usr/local/sbin/zfs-auto-snapshot weekly    4
    28       0 1 * * root /usr/local/sbin/zfs-auto-snapshot monthly  12

This will keep 4 15-minutely snapshots, 24 hourly snapshots, 7 daily snapshots,
4 weekly snapshots and 12 monthly snapshots. Any resulting zero-sized snapshots
will be automatically cleaned up.

Enable snapshotting on a dataset or top-level pool with:

    zfs set com.sun:auto-snapshot=true DATASET

Children datasets can be disabled for snapshot with:

    zfs set com.sun:auto-snapshot=false DATASET

Or for specific intervals:

    zfs set com.sun:auto-snapshot:frequent=false DATASET

See website and command usage output for further details.
{{< /highlight >}}

During installation, the `zfstools` installer prints a set of `cron` commands to create a basic snapshot rotation.
It says that the lines can be added to **/etc/crontab**, which is the system crontab, but the 
[FreeBSD handbook](https://www.freebsd.org/doc/handbook/configtuning-cron.html) warns that the system crontab should not be modified.
Instead, the handbook recommends the creation of a separate user crontab to run cron jobs as **root**.

> Note that the cron text given by the zfstools installer is formatted for the *system crontab*, in which the 6th column specifies the user **who** should run the cron job.
To use those lines in a *user crontab* instead, delete the 6th column of text from each line, which is the column containing the word **root**.

To create a user crontab for the root user, run `crontab -e -u root`, then add the header lines specified in section
[11.3.1 of the handbook](https://www.freebsd.org/doc/handbook/configtuning-cron.html),
followed by the crontab lines (without the 6th column) copied from the zfstools installer output.
After creating the root user crontab, double-check the schedule using `crontab -l`.

{{< highlight txt >}}
# crontab -e -u root
[...]

# crontab -l
SHELL=/bin/sh
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin

# Order of crontab fields
# minute hour mday month wday command

# Periodic ZFS Snapshots
PATH=/etc:/bin:/sbin:/usr/bin:/usr/sbin:/usr/local/bin
15,30,45 *    *    *     *    /usr/local/sbin/zfs-auto-snapshot frequent  4
0        *    *    *     *    /usr/local/sbin/zfs-auto-snapshot hourly   24
7        0    *    *     *    /usr/local/sbin/zfs-auto-snapshot daily     7
14       0    *    *     7    /usr/local/sbin/zfs-auto-snapshot weekly    4
28       0    1    *     *    /usr/local/sbin/zfs-auto-snapshot monthly  12
{{< /highlight >}}

The script will only create snapshots for datasets that have their **com.sun:auto-snapshot** property set to **true**, and
the snapshot property will automatically be inherited by all descendants of that dataset unless disabled further down the tree.

It is not necessary to snapshot every dataset on the system; [Michael W. Lucas](https://www.amazon.com/FreeBSD-Mastery-ZFS-Book-ebook/dp/B00Y32OHNM) recommends [disabling snapshots](https://mwl.io/archives/2140)
for datasets that can be easily recreated if needed and for those for which old versions of the files are not generally useful:

* /tmp
* /usr/obj
* /usr/ports
* /usr/ports/distfiles
* /usr/ports/packages
* /usr/src
* /var/crash
* /var/empty
* /var/run
* /var/tmp

I also disable snapshots on `/zroot/iocage/images` and `zroot/reserved`.

After disabling snapshots for the non-essential datasets, some of which might not exist, enable them for **zroot**.

{{< highlight txt >}}
# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zroot  3.62T  10.3G  3.61T        -         -     0%     0%  1.00x  ONLINE  -

# zfs list
NAME                                      USED  AVAIL  REFER  MOUNTPOINT
zroot                                    10.3G  3.50T    96K  /zroot
zroot/ROOT                               4.01G  3.50T    96K  none
zroot/ROOT/default                       4.01G  3.50T  4.01G  /
[...]

# zfs set com.sun:auto-snapshot=false zroot/tmp
# zfs set com.sun:auto-snapshot=false zroot/usr/obj
# zfs set com.sun:auto-snapshot=false zroot/usr/ports
# zfs set com.sun:auto-snapshot=false zroot/usr/ports/distfiles
# zfs set com.sun:auto-snapshot=false zroot/usr/ports/packages
# zfs set com.sun:auto-snapshot=false zroot/usr/src
# zfs set com.sun:auto-snapshot=false zroot/var/crash
# zfs set com.sun:auto-snapshot=false zroot/var/empty
# zfs set com.sun:auto-snapshot=false zroot/var/run
# zfs set com.sun:auto-snapshot=false zroot/var/tmp

# zfs set com.sun:auto-snapshot=false zroot/iocage/images
# zfs set com.sun:auto-snapshot=false zroot/reserved

# zfs set com.sun:auto-snapshot=true zroot

# zfs get com.sun:auto-snapshot
NAME                PROPERTY               VALUE    SOURCE
zroot               com.sun:auto-snapshot  true     local
zroot/ROOT          com.sun:auto-snapshot  true     inherited from zroot
zroot/ROOT/default  com.sun:auto-snapshot  true     inherited from zroot
zroot/iocage/images com.sun:auto-snapshot  false    local
zroot/reserved      com.sun:auto-snapshot  false    local
zroot/tmp           com.sun:auto-snapshot  false    local
zroot/usr           com.sun:auto-snapshot  true     inherited from zroot
zroot/usr/home      com.sun:auto-snapshot  true     inherited from zroot
zroot/usr/ports     com.sun:auto-snapshot  false    local
zroot/usr/src       com.sun:auto-snapshot  false    local
zroot/var           com.sun:auto-snapshot  true     inherited from zroot
zroot/var/audit     com.sun:auto-snapshot  true     inherited from zroot
zroot/var/crash     com.sun:auto-snapshot  false    local
zroot/var/log       com.sun:auto-snapshot  true     inherited from zroot
zroot/var/mail      com.sun:auto-snapshot  true     inherited from zroot
zroot/var/tmp       com.sun:auto-snapshot  false    local
{{< /highlight >}}

That should be all it takes to get things running.

Within 15 minutes or so, the first set of snapshots should begin to appear in the system and can be viewed with `zfs list -t snapshot`.
Old snapshots will be automatically deleted from the system when their replacements arrive.

{{< highlight txt >}}
# zfs list -t snapshot
NAME
 USED  AVAIL  REFER  MOUNTPOINT
zroot@zfs-auto-snap_frequent-2019-11-18-23h15
    0      -    96K  -
zroot/ROOT/default@zfs-auto-snap_frequent-2019-11-18-23h15
    0      -  4.01G  -
[...]
{{< /highlight >}}
