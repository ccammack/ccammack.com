---
title: "Create (or Expand) a ZFS Mirror in FreeBSD"
date: 2020-03-20T21:42:46-07:00
tags: ["FreeBSD", "ZFS"]
---

Here's a quick guide to convert a system created by the FreeBSD installer from a single-drive stripe into a 2-drive mirror. The same procedure also works to expand an existing mirror by adding additional drives.

<!--more-->

The test system was partitioned by selecting **Auto (ZFS)** in the installer and configured with the options **stripe: 1 disk**, **Encrypt Disks**, and **Encrypt Swap**.

{{< figure src="freebsd-installer-partitioning.png" alt="FreeBSD Installer > Partitioning">}}

{{< figure src="freebsd-installer-zfs-configuration.png" alt="FreeBSD Installer > ZFS Configuration">}}

First, back up your data.

Second, use **zpool status** to examine the current ZFS pool. In this example, the **zroot** pool consists of the third partition (**p3**) of drive **ada0** and it has been encrypted with GELI (**.eli**).

{{< highlight txt >}}
$ su
Password:

# zpool status
  pool: zroot
 state: ONLINE
  scan: none requested
config:

        NAME          STATE     READ WRITE CKSUM
        zroot         ONLINE       0     0     0
          ada0p3.eli  ONLINE       0     0     0

errors: No known data errors
{{< /highlight >}}

Next, use **swapinfo** to examine the swap partition and notice that the swap uses the second partition (**p2**) of the same drive (**ada0**) and that it is also GELI-encrypted (**.eli**).

{{< highlight txt >}}
# swapinfo
Device          1K-blocks     Used    Avail Capacity
/dev/ada0p2.eli   2097152        0  2097152     0%
{{< /highlight >}}

Also, use **geom** or **camcontrol** to get the drive's serial number for reference.

{{< highlight txt >}}
# geom disk list ada0 | grep ident
   ident: WD-WMATV0942547

# camcontrol identify ada0 | grep serial
serial number         WD-WMATV0942547
{{< /highlight >}}

Finally, run **gpart show** to display the complete partition layout for drive ada0.

{{< highlight txt >}}
# gpart show ada0
=>        40  1953525088  ada0  GPT  (932G)
          40        1024     1  freebsd-boot  (512K)
        1064         984        - free -  (492K)
        2048     4194304     2  freebsd-swap  (2.0G)
     4196352  1949327360     3  freebsd-zfs  (930G)
  1953523712        1416        - free -  (708K)
{{< /highlight >}}

At this point, copy the serial number of the new drive to be added to the system from the drive label.
Insert the new drive into an empty bay and run **dmesg** to find the device node assigned to it.
In this case, the new drive has been assigned **ada1**.

{{< highlight txt >}}
# dmesg
[...]
ada1 at ahcich1 bus 0 scbus1 target 0 lun 0
ada1: <Hitachi HDS721010CLA332 JP4OA25C> ATA8-ACS SATA 2.x device
ada1: Serial Number JP2921HQ0A6ZJA
ada1: 300.000MB/s transfers (SATA 2.x, UDMA6, PIO 8192bytes)
ada1: Command Queueing enabled
ada1: 953869MB (1953525168 512 byte sectors)
{{< /highlight >}}

Delete any existing partitions on the new drive using **gpart destroy**. The command will give an error if the drive doesn't contain any partitions.

{{< highlight txt >}}
# gpart destroy -F ada1
ada1 destroyed

# gpart destroy -F ada1
gpart: arg0 'ada1': Invalid argument
{{< /highlight >}}

The easiest way to copy the partition layout from drive ada0 to drive ada1 is to use the  **gpart backup** and **restore** commands.

Run **gpart backup** to examine the partition layout on drive **ada0**.

{{< highlight txt >}}
# gpart backup ada0
GPT 128
1   freebsd-boot         40       1024 gptboot0
2   freebsd-swap       2048    4194304 swap0
3    freebsd-zfs    4196352 1949327360 zfs0
{{< /highlight >}}

The last column in the output lists the automatically generated GPT labels for each of the three partitions on drive ada**0**, which is why the installer appended a **0** to the end of each GPT label.

Since the new drive has been assigned the device node ada**1**, run the command again and pipe the output into **sed** to replace the last **0** on each line with a **1** to generate new GPT label names for the new drive.

{{< highlight txt >}}
# gpart backup ada0 | sed -r 's/(^[[:digit:]].*)0/\11/'
GPT 128
1   freebsd-boot         40       1024 gptboot1
2   freebsd-swap       2048    4194304 swap1
3    freebsd-zfs    4196352 1949327360 zfs1
{{< /highlight >}}

If the output from sed looks correct, run the last command again and pipe those results into **gpart restore** to copy the partition layout to ada1.

{{< highlight txt >}}
# gpart backup ada0 | sed -r 's/(^[[:digit:]].*)0/\11/' | gpart restore -lF ada1
{{< /highlight >}}

Use **gpart show** to see that both drives now have the same layout.

{{< highlight txt >}}
# gpart show -l
=>        40  1953525088  ada0  GPT  (932G)
          40        1024     1  gptboot0  (512K)
        1064         984        - free -  (492K)
        2048     4194304     2  swap0  (2.0G)
     4196352  1949327360     3  zfs0  (930G)
  1953523712        1416        - free -  (708K)

=>        40  1953525088  ada1  GPT  (932G)
          40        1024     1  gptboot1  (512K)
        1064         984        - free -  (492K)
        2048     4194304     2  swap1  (2.0G)
     4196352  1949327360     3  zfs1  (930G)
  1953523712        1416        - free -  (708K)
{{< /highlight >}}

Since the root partition was GELI-encrypted during the installation, check the installer log to find out what options were used for the **geli init** and **attach** commands.

{{< highlight txt >}}
# grep geli /var/log/bsdinstall_log
[...]
DEBUG: zfs_create_boot: geli init -bg -e AES-XTS -J - -l 256 -s 4096 "ada0p3"
[...]
DEBUG: zfs_create_boot: geli attach -j - "ada0p3"
[...]
{{< /highlight >}}

Run the **geli init** command for the new drive using slightly different options. Remove the **-J -** option to make the command interactive and change **"ada0p3"** to **"ada1p3"** to match the new drive.

When requested, enter the same GELI password currently used to boot the system.

{{< highlight txt >}}
# geli init -bg -e AES-XTS -l 256 -s 4096 "ada1p3"

Enter new passphrase:
Reenter new passphrase:

Metadata backup for provider ada1p3 can be found in /var/backups/ada1p3.eli
and can be restored with the following command:

        # geli restore /var/backups/ada1p3.eli ada1p3
{{< /highlight >}}

Mount the new GELI partition using the **geli attach** command, enter the password when requested and view the active containers with **geli status**.

{{< highlight txt >}}
# geli attach "ada1p3"
Enter passphrase:

# geli status
      Name  Status  Components
ada0p3.eli  ACTIVE  ada0p3
ada0p2.eli  ACTIVE  ada0p2
ada1p3.eli  ACTIVE  ada1p3
{{< /highlight >}}

Use **zpool attach** to attach the new root partition to the existing stripe and convert it into a mirror. Write the boot code to the boot partition on the new drive as recommended by the output text.

{{< highlight txt >}}
# zpool status
  pool: zroot
 state: ONLINE
  scan: none requested
config:

        NAME          STATE     READ WRITE CKSUM
        zroot         ONLINE       0     0     0
          ada0p3.eli  ONLINE       0     0     0

errors: No known data errors

# zpool attach zroot ada0p3.eli ada1p3.eli
Make sure to wait until resilver is done before rebooting.

If you boot from pool 'zroot', you may need to update
boot code on newly attached disk 'ada1p3.eli'.

Assuming you use GPT partitioning and 'da0' is your new boot disk
you may use the following command:

        gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 da0

# gpart bootcode -b /boot/pmbr -p /boot/gptzfsboot -i 1 ada1
partcode written to ada1p1
bootcode written to ada1
{{< /highlight >}}

At this point, **/etc/fstab** contains an entry for the swap partition on drive ada0, which was added by the installer, but no entry for the swap partition on the newly added drive.
Edit the file or use sed to add an entry for the new drive.

{{< highlight txt >}}
# cat /etc/fstab
# Device                Mountpoint      FStype  Options         Dump    Pass#
/dev/ada0p2.eli         none    swap    sw              0       0

# sed -n 's/ada0p2/ada1p2/p' /etc/fstab
/dev/ada1p2.eli         none    swap    sw              0       0

# sed -n 's/ada0p2/ada1p2/p' /etc/fstab >> /etc/fstab

# cat /etc/fstab
# Device                Mountpoint      FStype  Options         Dump    Pass#
/dev/ada0p2.eli         none    swap    sw              0       0
/dev/ada1p2.eli         none    swap    sw              0       0
{{< /highlight >}}

Wait for the drive to finish resilvering and check the status of the mirror.

{{< highlight txt >}}
# zpool status
  pool: zroot
 state: ONLINE
  scan: resilvered 823M in 0 days 00:00:19 with 0 errors on Mon Mar 23 21:56:30 2020
config:

        NAME            STATE     READ WRITE CKSUM
        zroot           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            ada0p3.eli  ONLINE       0     0     0
            ada1p3.eli  ONLINE       0     0     0

errors: No known data errors
{{< /highlight >}}

That's it. The same procedure can be repeated to add more drives as needed, but drives are reliable enough that it's hard to justify mirroring more than three.

{{< highlight txt >}}
# dmesg
[...]

# zpool attach zroot ada0p3.eli ada2p3.eli
[...]

# zpool status
  pool: zroot
 state: ONLINE
  scan: resilvered 823M in 0 days 00:00:16 with 0 errors on Mon Mar 23 22:27:10 2020
config:

        NAME            STATE     READ WRITE CKSUM
        zroot           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            ada0p3.eli  ONLINE       0     0     0
            ada1p3.eli  ONLINE       0     0     0
            ada2p3.eli  ONLINE       0     0     0

errors: No known data errors
{{< /highlight >}}
