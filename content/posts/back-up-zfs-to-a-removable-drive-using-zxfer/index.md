---
title: "Back Up ZFS to a Removable Drive Using zxfer"
date: 2020-03-30T21:01:50-07:00
tags: ["FreeBSD", "ZFS", "Backups", "zxfer"]
---

The [`zxfer`](https://www.freebsd.org/cgi/man.cgi?query=zxfer) shell script wraps the `zfs send` and `zfs receive` commands and provides one of the easiest ways to back up a ZFS-based system to a remote server.
It also works just as well to back up data to a local hard drive. Run `zpool scrub` on the backup drives to detect bit rot, rotate through several drives to maintain multiple copies of the data
and store them off-site to create a reliable and inexpensive backup solution.

<!--more-->

This example assumes that the system generates snapshots using [`zfs-auto-snapshot`](/posts/schedule-zfs-snapshots-using-zfs-auto-snapshot/) and it will not work properly without them.
It also assumes that the backup partition will be GELI-encrypted, which is wise for backups stored at an off-site location.
Finally, it assumes that the backup target will be a ZFS pool.
If ZFS cannot be used on the target device for some compelling reason, it would be better to select another backup solution or possibly even [`rsync`](https://www.freebsd.org/cgi/man.cgi?query=rsync).

To get started, confirm that the system contains some snapshots using `zfs list -t`, then find and install `zxfer`.

{{< highlight txt >}}
$ su
Password:

# zfs list -t snapshot
NAME                                                         USED  AVAIL  REFER  MOUNTPOINT
zroot@zfs-auto-snap_frequent-2020-04-03-00h45                   0      -    88K	 -
zroot/ROOT/default@zfs-auto-snap_frequent-2020-04-03-00h45      0      -   964M	 -
zroot/usr/home@zfs-auto-snap_frequent-2020-04-03-00h45        64K      -   124K	 -
zroot/var/audit@zfs-auto-snap_frequent-2020-04-03-00h45         0      -    88K	 -
zroot/var/log@zfs-auto-snap_frequent-2020-04-03-00h45           0      -   348K	 -
zroot/var/mail@zfs-auto-snap_frequent-2020-04-03-00h45          0      -   100K	 -
[...]

# pkg search zxfer
zxfer-1.1.7                    Easily and reliably transfer ZFS filesystems

# pkg install -y zxfer
[...]
{{< /highlight >}}

Insert the removable backup drive, run `dmseg` to find its device node name (**ada3**) and use `gpart destroy` to remove any old partition table that might be on the drive.

{{< highlight txt >}}
# dmesg
[...]
ada3 at ahcich5 bus 0 scbus5 target 0 lun 0
ada3: <ST3500630AS 3.AHG> ATA-7 SATA 2.x device
ada3: Serial Number 9QG2A9ET
ada3: 300.000MB/s transfers (SATA 2.x, UDMA5, PIO 8192bytes)
ada3: Command Queueing enabled
ada3: 476940MB (976773168 512 byte sectors)

# gpart destroy -F ada3
ada3 destroyed

# gpart destroy -F ada3
gpart: arg0 'ada3': Invalid argument
{{< /highlight >}}

Create a new ZFS partition with the GPT label **backup**.

{{< highlight txt >}}
# gpart create -s gpt ada3
ada3 created

# gpart add -a 1m -l backup -t freebsd-zfs "ada3"
ada3p1 added

# gpart show -l ada3
=>       40  976773088  ada3  GPT  (466G)
         40       2008        - free -  (1.0M)
       2048  976771072     1  backup  (466G)
  976773120          8        - free -  (4.0K)
{{< /highlight >}}

Run `geli init` and `geli attach` to encrypt and mount the partition.

{{< highlight txt >}}
# grep "geli init" /var/log/bsdinstall_log
DEBUG: zfs_create_boot: geli init -bg -e AES-XTS -J - -l 256 -s 4096 "ada0p3

# geli init -e AES-XTS -l 256 -s 4096 "/dev/gpt/backup"
Enter new passphrase:
Reenter new passphrase:
[...]

# geli attach /dev/gpt/backup
Enter passphrase:

# geli status
[...]
gpt/backup.eli  ACTIVE  gpt/backup
{{< /highlight >}}

Create a new zpool in the GELI partition called **backup**.
It's possible to use the **backup** pool as the destination for zxfer directly, but this example instead creates a new dataset inside the pool, named after the source machine's **hostname**.
This will help identify backup data and allow multiple hosts to back up to the same drive.
Since the source machine's hostname is **server** in this example, the backup destination for zxfer will be **backup/server**.

>Note that the [man page](https://www.freebsd.org/cgi/man.cgi?query=zxfer) warns that **the usage of spaces in zfs(8) filesystem names is NOT supported**,
so do not create datasets with spaces in their names when using `zfs create`.

{{< highlight txt >}}
# zpool create backup gpt/backup.eli

# zfs create backup/`hostname -s`

# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
backup   464G   776K   464G        -         -     0%     0%  1.00x  ONLINE  -
zroot    928G   880M   927G        -         -     0%     0%  1.00x  ONLINE  -

# zfs list
NAME                 USED  AVAIL  REFER  MOUNTPOINT
backup               464K   449G    88K  /backup
backup/server         88K   449G    88K  /backup/server
zroot                879M   898G    88K  /zroot
[...]
{{< /highlight >}}

Finally, run `zpool export` and `geli detach`, then `exit` the root user and pretend that you have removed the drive from the system to practice the complete backup procedure.

{{< highlight txt >}}
# zpool export backup

# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zroot   928G   879M   927G        -         -     0%     0%  1.00x  ONLINE  -

# geli detach gpt/backup.eli

# geli status
[...]

# exit
exit
$
{{< /highlight >}}

---

To begin the backup procedure, insert the drive into the system and run `geli attach` and `zpool import` to mount the pool.

{{< highlight txt >}}
$ su
Password:

# geli attach /dev/gpt/backup
Enter passphrase:

# geli status
[...]
gpt/backup.eli  ACTIVE  gpt/backup

# zpool import backup

# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
backup   464G   776K   464G        -         -     0%     0%  1.00x  ONLINE  -
zroot    928G   880M   927G        -         -     0%     0%  1.00x  ONLINE  -

# zfs list
NAME                 USED  AVAIL  REFER  MOUNTPOINT
backup               464K   449G    88K  /backup
backup/server         88K   449G    88K  /backup/server
zroot                879M   898G    88K  /zroot
[...]
{{< /highlight >}}

The [zxfer man page](https://www.freebsd.org/cgi/man.cgi?query=zxfer) provides several usage examples in the bottom half of the text,
and the first example (**Ex1 - Backup	a pool (including snapshots and	properties)**) offers a good model for the use case described here,
which is to replicate the entire *zroot* pool from the host system to the backup drive.

>This example relies on `zfs-auto-snapshot` to create snapshots for any dataset that has the ZFS property **com.sun:auto-snapshot** set to **true**.
Because of this, when running `zxfer` to copy snapshots to a *locally-mounted* backup pool, it is critical to specify the option **-I com.sun:auto-snapshot** to prevent that property from being copied to the backup data.
If this option is not specified, zxfer will copy the property to the data in the backup pool and the system will begin taking snapshots of the backup data, which can prevent files from replicating properly.
This option may not be necessary for replication to a remote server or if the backup is only applied to specific datasets rather than the entire zroot.

To run the backup, use `zfs list | awk` to confirm that none of the dataset names contain spaces,
then run the `zxfer` command to perform the actual backup operation followed by `zpool scrub` to make sure the backup pool is free of data errors.

Use `zpool list` to compare the *ALLOC* sizes of the source and destination pools to see that they are approximately the same size after the backup.

{{< highlight txt >}}
# zfs list -H | cut -f1 | awk '/[[:space:]]/{printf("Error! Dataset name contains spaces: %s\n",$0)}'

# zxfer -dFkPv -g 376 -I com.sun:auto-snapshot -R zroot backup/`hostname -s`
[...]

# zpool scrub backup

# zpool status backup
  pool: backup
 state: ONLINE
  scan: scrub in progress since Fri Apr  3 14:13:20 2020
        967M scanned at 107M/s, 680M issued at 75.5M/s, 967M total
        0 repaired, 70.27% done, 0 days 00:00:03 to go
config:

        NAME              STATE     READ WRITE CKSUM
        backup            ONLINE       0     0     0
          gpt/backup.eli  ONLINE       0     0     0

errors: No known data errors

# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
backup   464G   880M   463G        -         -     0%     0%  1.00x  ONLINE  -
zroot    928G   880M   927G        -         -     0%     0%  1.00x  ONLINE  -
{{< /highlight >}}

>If the amount of data to back up is very large, the snapshot cron job might run during the backup process and delete some of the older snapshots that zxfer was about to copy.
If this happens, zxfer will give a warning and abort the backup.
To continue, simply restart the backup as many times as needed to catch up with the current set of snapshots.
After the initial backup completes, future runs will finish more quickly and this warning will be less likely to occcur.
{{< highlight txt >}}
# zxfer -dFkPv -g 376 -I com.sun:auto-snapshot -R zroot backup/`hostname -s`
[...]
WARNING: could not send zroot/iocage/download/12.1-RELEASE@zfs-auto-snap_frequent-2020-04-07-00h30: does not exist
cannot receive: failed to read from stream
Error when zfs send/receiving.

# zxfer -dFkPv -g 376 -I com.sun:auto-snapshot -R zroot backup/`hostname -s`
[...]
Writing backup info to location /backup/server/.zxfer_backup_info.zroot
{{< /highlight >}}

---

Before testing the restore process, use `zpool list` to make sure *zroot* has enough space to hold the entire *backup*,
then add a new temporary file under **/usr/home** to create a known difference between the backup and host data.

{{< highlight txt >}}
# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
backup   464G   880M   463G        -         -     0%     0%  1.00x  ONLINE  -
zroot    928G   880M   927G        -         -     0%     0%  1.00x  ONLINE  -

# ls /usr/home/ccammack
.cshrc          .login_conf     .mailrc         .shrc
.login          .mail_aliases   .profile

# touch /usr/home/ccammack/hello

# ls /usr/home/ccammack
.cshrc          .login_conf     .mailrc         .shrc
.login          .mail_aliases   .profile        hello
{{< /highlight >}}

---

To restore the pool from backup, follow the third example (**Ex3 - Restore a pool**) on the manpage and restore the data to **zroot/tmp**,
which has snapshots *disabled* and will therefore prevent the system from making backups of the backup data.

{{< highlight txt >}}
# zfs get com.sun:auto-snapshot zroot/tmp
NAME       PROPERTY               VALUE                  SOURCE
zroot/tmp  com.sun:auto-snapshot  false                  local

# zxfer -deFPv -R backup/`hostname -s`/zroot zroot/tmp
[...]
{{< /highlight >}}

---

After the files are restored, use `diff -qr` to compare the contents of **/usr/home** and **/tmp/zroot/usr/home** to demonstrate their differences.
In this example, the temporary file named *hello* will only appear on the host and will be missing from the restored backup data.

Destroy the restored **/tmp** dataset, then wait for the next execution of the [snapshot cronjob](/posts/schedule-zfs-snapshots-using-zfs-auto-snapshot/), which is 15 minutes (900 seconds) in this example.

Run the `zxfer` *back up* command again to back up the new snapshots (including the new *hello* file), followed by the `zxfer` *restore* command to restore them to the **/tmp** folder again.

Use `diff -qr` again to compare the contents of **/usr/home** and **/tmp/zroot/usr/home** to demonstrate that there are no differences and that the new *hello* file has been properly backed up.

{{< highlight txt >}}
# diff -qr /usr/home/ /tmp/zroot/usr/home/
Only in /usr/home/ccammack: hello

# zfs destroy -r zroot/tmp/zroot

# sleep 900

# zxfer -dFkPv -g 376 -I com.sun:auto-snapshot -R zroot backup/`hostname -s`
[...]

# zxfer -deFPv -R backup/`hostname -s`/zroot zroot/tmp
[...]

# diff -qr /usr/home/ /tmp/zroot/usr/home/
{{< /highlight >}}

The entire host file system can also be compared to the backup using `diff -qr`.
In this example, doing this confirms that several directories are correctly excluded from the snapshot set and that one of the log files has changed since the last backup.

{{< highlight txt >}}
# diff -qr / /tmp/zroot
Only in /: .cshrc
[...]
Only in /tmp: .ICE-unix
[...]
Only in /usr: bin
[...]
Only in /var: account
[...]
Files /var/log/cron and /tmp/zroot/var/log/cron differ
[...]
{{< /highlight >}}

To clean up, destroy the temporary dataset and remove the test file.

{{< highlight txt >}}
# ls /tmp/
.ICE-unix       .X11-unix       .XIM-unix       .font-unix       zroot

# zfs destroy -r zroot/tmp/zroot

# ls /tmp/
.ICE-unix       .X11-unix       .XIM-unix       .font-unix

# ls /usr/home/ccammack
.cshrc          .login_conf     .mailrc         .shrc
.login          .mail_aliases   .profile        hello

# rm /usr/home/ccammack/hello

# ls /usr/home/ccammack
.cshrc          .login_conf     .mailrc         .shrc
.login          .mail_aliases   .profile
{{< /highlight >}}

---

Finally, to remove the drive for off-site storage, run `zpool export` followed by `geli detach` and then remove the drive from the system.

{{< highlight txt >}}
# zpool export backup

# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zroot   928G   879M   927G        -         -     0%     0%  1.00x  ONLINE  -

# geli detach gpt/backup.eli

# geli status
[...]
{{< /highlight >}}
