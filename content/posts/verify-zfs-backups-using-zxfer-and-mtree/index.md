---
title: "Verify ZFS Backups Using zxfer and mtree"
date: 2022-05-07T15:54:03-07:00
tags: ["Backups", "FreeBSD", "Linux", "mtree", "Raspberry Pi", "zxfer", "ZFS"]
---

Even if you trust your [backup process](/posts/create-a-centralized-backup-server-with-freebsd/),
the only way to make sure the archive contains what you expect is to perform an occasional *test restore* using a recent backup and compare it to the original source data.
Here's one way to do that on FreeBSD using [zxfer](/posts/back-up-zfs-to-a-removable-drive-using-zxfer/) to restore
and [mtree](https://www.freebsd.org/cgi/man.cgi?query=mtree) to compare.

<!--more-->

Both `zxfer` and `mtree` can target the local machine, making it possible to restore the backup data and
then verify it by moving a spare hard drive between the *backup* server and the *data* server.

However, it is also possible to perform both operations over the network using an intermediate host.
To do this, add a temporary Raspberry Pi to the network with a spare hard drive attached,
restore the data from the *backup* server to the Pi using `zxfer` over `ssh`,
then run `mtree` on the Pi and the original *data* server to compare the restored data to the original data.
Afterwards, remove the Pi from the network, wipe the spare drive and everything goes back to the way it was.

##### Create a destination zpool to hold the restored data

Attach a spare empty drive to the system and create a destination zpool named *restore*, then `export` it from the system for the next step.

{{< highlight txt >}}
C:\Users\ccammack
λ ssh backup.ccammack.com
[...]

$ su
Password:

root@backup:/usr/home/ccammack # dmesg
[...]
da0 at umass-sim0 bus 0 scbus2 target 0 lun 0
da0: <JMicron Tech 0508> Fixed Direct Access SPC-4 SCSI device
da0: Serial Number 000000000004
da0: 40.000MB/s transfers
da0: 3815447MB (7814037168 512 byte sectors)
da0: quirks=0x2<NO_6_BYTE>

root@backup:/usr/home/ccammack # gpart destroy -F da0
gpart: arg0 'da0': Invalid argument

root@backup:/usr/home/ccammack # gpart create -s gpt da0
da0 created

root@backup:/usr/home/ccammack # gpart add -a 1m -l restore -t freebsd-zfs "da0"
da0p1 added

root@backup:/usr/home/ccammack # gpart show -l da0
=>        40  7814037088  da0  GPT  (3.6T)
          40        2008       - free -  (1.0M)
        2048  7814033408    1  restore  (3.6T)
  7814035456        1672       - free -  (836K)

root@backup:/usr/home/ccammack # zpool create restore da0p1

root@backup:/usr/home/ccammack # zpool list restore
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
restore  3.62T   316K  3.62T        -         -     0%     0%  1.00x  ONLINE  -

root@backup:/usr/home/ccammack # zpool export restore
{{< /highlight >}}

---

##### Restore the data from the *backup* pool to the *restore* pool

There are a couple of ways to restore the backup data.
If the *restore* pool can be mounted on the same machine as the *backup* pool, use `zxfer` to restore the data directly from drive to drive.
Alternatively, the backup data can also be restored from one host to another over the LAN using `zxfer` and `ssh`.

###### Option 1: Restore the data from drive to drive on a single machine

Mount the *restore* pool on the same machine as the *backup* pool and run `zxfer` as *root* to restore the data.
Plug the destination drive directly into a SATA port if possible; external USB drive docks may not be as reliable, although that is what I use here.

If the *backup* pool contains data from multiple hosts, use `zfs list | grep` to find the right one; in this case,
I'm restoring the *zroot* pool from the *data* server, which is stored on the backup drive as `/backup/data/zroot/`.

Include a trailing `/` on the *source* pool when restoring with `zxfer` to make the output file hierarchy under `/restore/` parallel the one under `/backup/data/zroot/`.

Export the *restore* pool when finished to prepare for the `mtree` step.

{{< highlight txt >}}
$ su
Password:

root@backup:/usr/home/ccammack # hostname
backup.ccammack.com

root@backup:/usr/home/ccammack # zpool import restore

root@backup:/usr/home/ccammack # zpool list
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
backup   4.16T  2.12T  2.04T        -         -     0%    50%  1.00x  ONLINE  -
restore  3.62T   736K  3.62T        -         -     0%     0%  1.00x  ONLINE  -

root@backup:/usr/home/ccammack # zfs list | grep 'backup/.*zroot[^/]'
backup/data/zroot                                    11.0G  1.67T   156K  /backup/data/zroot
[...]

root@backup:/usr/home/ccammack # sh -x /usr/local/sbin/zxfer -deFPv -R backup/data/zroot/ restore
[...]
+ exit 0

root@backup:/usr/home/ccammack # ls /backup/data/zroot/
ROOT            reserved        usr
iocage          tmp             var

root@backup:/usr/home/ccammack # ls /restore/
ROOT            reserved        usr
iocage          tmp             var

root@backup:/usr/home/ccammack # zpool export restore
{{< /highlight >}}

While the restore runs, open a second terminal on the same machine and keep an eye on the progress by watching the *restore* pool's *ALLOC* size grow as the data is restored.
If using an external USB drive, also keep an eye on the *restore* pool's *HEALTH* to make sure it remains *ONLINE*.

{{< highlight txt >}}
$ zpool list restore
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
restore  3.62T   736K  3.62T        -         -     0%     0%  1.00x  ONLINE  -

[...]

$ zpool list restore
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
restore  3.62T  1.77T  3.62T        -         -     0%    48%  1.00x  ONLINE  -
{{< /highlight >}}

###### Option 2: Restore the data over the LAN

The data can also be restored over the LAN using `zxfer` and `ssh`.

It's generally better to perform backups in *pull* mode, allowing the *backup* server to log into each client and collect the data, rather than granting the clients any access to the *backup* server.
Similarly, when restoring the data, follow the same security policy; the *backup* server should log into the client and *push* the restored data to it rather than the reverse.

This example shows how to push the restored data from the *backup* server to a temporary [Raspberry Pi 4 running FreeBSD](https://wiki.freebsd.org/arm/Raspberry%20Pi) with an attached USB drive dock.
The [FreeBSD 13.1 image for RPI](https://download.freebsd.org/ftp/releases/arm64/aarch64/ISO-IMAGES/) includes default credentials, so it requires very little additional setup.
The only change required on the *backup* server will be to create a *new temporary key pair* to access the Pi, which can then be deleted afterwards.

To get started, log into the *backup* server, generate *new temporary keys* and transfer the public key to the Pi using the default user account `freebsd` and password `freebsd`.

{{< highlight txt >}}
C:\Users\ccammack
λ ssh backup.ccammack.com
[...]

$ ssh-keygen -f /tmp/zxfer_key -N ""
Generating public/private rsa key pair.
Your identification has been saved in /tmp/zxfer_key.
Your public key has been saved in /tmp/zxfer_key.pub.
[...]

$ ssh-copy-id -i /tmp/zxfer_key.pub freebsd@192.168.1.137
[...]
Password for freebsd@generic:

$ hostname
backup.ccammack.com
{{< /highlight >}}

Log into the Pi using the temporary key as the *freebsd* user, which should now work without asking for a password.
Switch to the *root* user using the default password `root` and append the `authorized_keys` from the *freebsd* account to the *root* account.

Change the `sshd` configuration to allow the *root* user to log in using keys only, then restart `sshd` and `exit` to the *backup* server.

{{< highlight txt >}}
$ ssh -i /tmp/zxfer_key freebsd@192.168.1.137
[...]

freebsd@generic:~ % su
Password:

root@generic:/home/freebsd # mkdir -p /root/.ssh

root@generic:/home/freebsd # cat /home/freebsd/.ssh/authorized_keys >> /root/.ssh/authorized_keys

root@generic:/home/freebsd # grep PermitRootLogin /etc/ssh/sshd_config
#PermitRootLogin no
[...]

root@generic:/home/freebsd # sed -i .tmp 's/^#PermitRootLogin no.*$/PermitRootLogin without-password/g' /etc/ssh/sshd_config

root@generic:/home/freebsd # grep PermitRootLogin /etc/ssh/sshd_config
PermitRootLogin without-password
[...]

root@generic:/home/freebsd # service sshd restart
[...]
Starting sshd.

root@generic:/home/freebsd # exit
exit
freebsd@generic:~ % exit
logout
Connection to 192.168.1.137 closed.

$ hostname
backup.ccammack.com
{{< /highlight >}}

Log into the Pi as *root* to make sure it works without asking for a password.
Install `zxfer` on the Pi, then import the destination *restore* pool and `exit` to the backup server.

{{< highlight txt >}}
$ hostname
backup.ccammack.com

$ ssh -i /tmp/zxfer_key root@192.168.1.137
[...]

root@generic:~ # pkg install -y zxfer
[...]

root@generic:~ # zpool import restore

root@generic:~ # zpool list restore
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
restore  3.62T   576K  3.62T        -         -     0%     0%  1.00x    ONLINE  -

root@generic:~ # exit
logout
Connection to 192.168.1.137 closed.

$ hostname
backup.ccammack.com
{{< /highlight >}}

On the *backup* server, `su` to *root* and use `zxfer` to push the backup data to the *restore* pool mounted on the Pi.

Make sure the correct source pool is chosen with `zfs list | grep`.

Include a trailing `/` on the *source* pool when restoring with `zxfer` to make the output file hierarchy under `/restore/` parallel the one under `/backup/data/zroot/`.

{{< highlight txt >}}
$ hostname
backup.ccammack.com

$ su
Password:

root@backup:/usr/home/ccammack # zfs list | grep 'backup/.*zroot[^/]'
backup/data/zroot                                    11.0G  1.67T   156K  /backup/data/zroot
[...]

root@backup:/usr/home/ccammack # sh -x /usr/local/sbin/zxfer -deFPv -T 'root@192.168.1.137 -i /tmp/zxfer_key' -R backup/data/zroot/ restore
[...]
+ exit 0
{{< /highlight >}}

While the restore runs, open a console on the Pi and keep an eye on the progress by watching the *restore* pool's *ALLOC* size grow as the data is restored.
Also keep an eye on the *restore* pool's *HEALTH* to make sure it remains *ONLINE*.

After the restore finishes, `su` to *root* on the Pi and export the *restore* pool to prepare for the `mtree` step.

{{< highlight txt >}}
C:\Users\ccammack
λ ssh freebsd@192.168.1.137
(freebsd@192.168.1.137) Password for freebsd@generic:
[...]

freebsd@generic:~ % hostname
generic

freebsd@generic:~ % zpool list restore
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
restore  3.62T   736K  3.62T        -         -     0%     0%  1.00x    ONLINE  -

[...]

freebsd@generic:~ % zpool list restore
NAME      SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
restore  3.62T  1.77T  1.85T        -         -     0%    48%  1.00x    ONLINE  -

freebsd@generic:~ % su
Password:

root@generic:/home/freebsd # zpool export restore
{{< /highlight >}}

---

##### Compare the restored data to the original data using `mtree`

The `mtree` utility appears to have been originally written for [NetBSD](https://www.freebsd.org/cgi/man.cgi?mtree) as an intrusion detection tool.
It has since been [ported](https://github.com/archiecobbs/mtree-port) to [Linux](https://manpages.ubuntu.com/manpages/hirsute/en/man8/fmtree.8.html),
although it doesn't seem to be very well known.

Unlike `diff -qr`, `mtree` can report much more detailed information about the differences between file two systems and
can operate over `ssh` if needed, making it a good option for comparing file systems on separate hosts.

###### Option A: Compare file systems on the same machine

To compare the original and restored data on a single machine, mount and `import` the *restore* pool on the host containing the original data.
In this example, I have mounted the *restore* pool on the *data* server and will use `mtree` to compare a specific dataset from both pools.

List the datasets in the *restore* pool and select one of them (`restore/usr/home`) for testing.

{{< highlight txt >}}
C:\Users\ccammack
λ ssh data.ccammack.com
[...]

$ su
Password:

root@data:/usr/home/ccammack # zpool import restore

root@data:/usr/home/ccammack # zfs list -r restore
NAME                                        USED  AVAIL  REFER  MOUNTPOINT
[...]
restore/usr/home                            312K  1.56T   240K  /restore/usr/home
[...]
{{< /highlight >}}

Select the corresponding dataset on the *data* server's *zroot* pool (`zroot/usr/home`) and note their *MOUNTPOINT* values (`/restore/usr/home` and `/usr/home`).

{{< highlight txt >}}
root@data:/usr/home/ccammack # zfs list -r restore/usr/home
NAME               USED  AVAIL  REFER  MOUNTPOINT
restore/usr/home   312K  1.56T   240K  /restore/usr/home

root@data:/usr/home/ccammack # zfs list -r zroot/usr/home
NAME             USED  AVAIL  REFER  MOUNTPOINT
zroot/usr/home  1.58M  1.42T   800K  /usr/home
{{< /highlight >}}

Make a small change to a local file in that dataset on the server and use `mtree` to compare the two file systems.
In this case, I added a single byte to the end of my `.cshrc` file and `mtree` reports the difference.

{{< highlight txt >}}
root@data:/usr/home/ccammack # echo "" >> .cshrc

root@data:/usr/home/ccammack # mtree -c -p /restore/usr/home | mtree -p /usr/home
[...]
ccammack/.cshrc:
        size (1054, 1055)
        modification time (Fri Nov  1 01:32:34 2019, Mon May  9 13:56:28 2022)
[...]
{{< /highlight >}}

###### Option B: Compare file systems over `ssh`

The `mtree` utility can also operate over `ssh`.
To allow the *data* server to reach the intermediate Pi without a password, generate *new temporary keys* and copy the public key to the Pi for use by the *freebsd* user.

{{< highlight txt >}}
C:\Users\ccammack
λ ssh data.ccammack.com
[...]

$ ssh-keygen -f /tmp/mtree_key -N ""
Generating public/private rsa key pair.
Your identification has been saved in /tmp/mtree_key.
Your public key has been saved in /tmp/mtree_key.pub.
[...]

$ ssh-copy-id -i /tmp/mtree_key.pub freebsd@192.168.1.137
[...]
Password for freebsd@generic:

$ hostname
data.ccammack.com
{{< /highlight >}}

Log into the Pi as the *freebsd* user using the temporary key, which should now work without asking for a password.
Switch to the *root* user using the password `root` and append the `authorized_keys` from the *freebsd* account to the *root* account.

Change the `sshd` configuration to allow the *root* user to log in using keys only, then restart `sshd` and `exit` back to the *data* server.

{{< highlight txt >}}
$ hostname
data.ccammack.com

$ ssh -i /tmp/mtree_key freebsd@192.168.1.137
[...]

freebsd@generic:~ % su
Password:

root@generic:/home/freebsd # mkdir -p /root/.ssh

root@generic:/home/freebsd # cat /home/freebsd/.ssh/authorized_keys >> /root/.ssh/authorized_keys

root@generic:/home/freebsd # grep PermitRootLogin /etc/ssh/sshd_config
#PermitRootLogin no
[...]

root@generic:/home/freebsd # sed -i .tmp 's/^#PermitRootLogin no.*$/PermitRootLogin without-password/g' /etc/ssh/sshd_config

root@generic:/home/freebsd # grep PermitRootLogin /etc/ssh/sshd_config
PermitRootLogin without-password
[...]

root@generic:/home/freebsd # service sshd restart
[...]
Starting sshd.

root@generic:/home/freebsd # exit
exit
freebsd@generic:~ % exit
logout
Connection to 192.168.1.137 closed.

$ hostname
data.ccammack.com
{{< /highlight >}}

Now that *root* can connect to the Pi using keys, list the datasets in the *restore* pool on the Pi, select one of them for testing (`restore/usr/home`) and
note its *MOUNTPOINT* (`/restore/usr/home`).

{{< highlight txt >}}
$ hostname
data.ccammack.com

$ su
Password:

root@data:/usr/home/ccammack # ssh -i /tmp/mtree_key root@192.168.1.137 zfs list -r restore
NAME                                        USED  AVAIL  REFER  MOUNTPOINT
[...]
restore/usr/home                            312K  1.56T   240K  /restore/usr/home
[...]
{{< /highlight >}}

Select the corresponding dataset on the *data* server (`zroot/usr/home`) and find its *MOUNTPOINT* (`/usr/home`).

{{< highlight txt >}}
root@data:/usr/home/ccammack # zfs list -r zroot/usr/home
NAME             USED  AVAIL  REFER  MOUNTPOINT
zroot/usr/home  1.58M  1.42T   800K  /usr/home
{{< /highlight >}}

Make a small change to a local file in that dataset on the server. In this case, I added a single byte to the end of my `.cshrc`.

To compare the file systems between hosts, run `mtree` on the remote host over `ssh` and pipe its output to `mtree` running on the local host.

{{< highlight txt >}}
root@data:/usr/home/ccammack # echo "" >> .cshrc

root@data:/usr/home/ccammack # ssh -i /tmp/mtree_key root@192.168.1.137 mtree -c -p /restore/usr/home | mtree -p /usr/home
[...]
ccammack/.cshrc:
        size (1054, 1055)
        modification time (Fri Nov  1 01:32:34 2019, Mon May  9 13:56:28 2022)
[...]
{{< /highlight >}}

---

##### Automate the `mtree` comparisons

To verify the entire backup, write a script to iterate the datasets in the *restore* pool and compare each of them to the corresponding dataset in the original *zroot* pool using `mtree`.

{{< highlight txt >}}
#!/bin/sh

# use mtree to generate a spec from the "restore" pool and
# pipe it to mtree running against the "zroot" pool to compare them

# uncomment the next line if the "restore" pool is on a remote system
#sshcmd="ssh -i /tmp/mtree_key root@192.168.1.137"
sshcmd=${sshcmd-""} # set sshcmd="" if not already assigned

# exit immediately if you're not root
if [ "$(id -u)" -ne 0 ]; then
  echo "Error: this must run as root"
  exit 1
fi

# create a temp file to hold the "excluded" paths (log, spool)
exclude=$(${sshcmd} mktemp /tmp/verify-backups.XXXXXX)
${sshcmd} /bin/sh << EOF
  echo log   >> "${exclude}"
  echo spool >> "${exclude}"
EOF

# read datasets and mountpoints from the "restore" pool
lines=$(${sshcmd} zfs list -rpH restore | \
  awk 'BEGIN { ORS = "|"; OFS = ":" } { print $1, $5 }'
)

# generate mtree spec from "restore" pool and compare it to "zroot" pool
echo "${lines}" |
awk -v sshcmd="${sshcmd}" -v exclude="${exclude}" 'BEGIN { RS = "|"; FS = ":" } {
  if ($1 != "" && $2 != "") {
    d1 = $1
    sub("restore", "zroot", d1)
    cmd = "zfs get -pH -o value mountpoint \"" d1 "\""
    cmd | getline m1
    close(cmd)
    sub(/\n/, "", m1)
    cmd = sshcmd
    cmd = cmd " mtree -c -j    -p \"" $2 "\" -X \"" exclude "\" |"
    cmd = cmd " mtree       -e -p \"" m1 "\""
    while (cmd | getline) print
    close(cmd)
  }
}'

# delete the temp "excluded" file
${sshcmd} rm "${exclude}"
{{< /highlight >}}
