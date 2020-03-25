---
title: "Label GPT Partitions With Their Physical Locations"
date: 2020-03-24T21:02:40-07:00
tags: ["FreeBSD", "ZFS"]
---

[Lucas and Jude](https://www.amazon.com/FreeBSD-Mastery-ZFS-Book-ebook/dp/B00Y32OHNM) recommend using GPT labels on each partition to indicate the hard drive's physical location and serial number.
This would obviously simplify things when working with large numbers of drives in a remote location, but it seems just as useful to me for a personal sever. Also, it's really easy to do.

<!--more-->

The first step is to devise a numbering scheme for the physical drive locations in the case.
For example, my test server contains a stack of 8 drive bays, which I have labeled from 00 at the bottom to 07 at the top.

Next, shut down the server, pull each drive out one at a time and record the serial number from each along with the physical drive bay where it resides.

|Drive Bay|HDD Serial Number|
|---------|-----------------|
|00       |WD-WMATV0942547  |
|01       |JP2921HQ0A6ZJA   |
|02       |9VP9Q4SD         |

GPT labels are restricted to 15 characters in length, so devise a naming scheme for the labels that includes both the drive bay id and serial number.
In my case, I'm using the prefix *zfs-* followed by two digits for the bay number and the last 8 characters of the serial number, giving labels such as *zfs-00-V0942547*.

The test machine for this example is a 3-drive ZFS mirror and the system has currently assigned device nodes *ada0*-*ada2* to the drives.
Use the **diskinfo** or **geom** command to find the serial numbers for each device node and match them up with the serial numbers collected earlier.

{{< highlight txt >}}
$ su
Password:

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

# diskinfo -v ada0 ada1 ada2 | grep ident
        WD-WMATV0942547 # Disk ident.
        JP2921HQ0A6ZJA  # Disk ident.
        9VP9Q4SD        # Disk ident.

# geom disk list ada0 ada1 ada2 | grep ident
   ident: WD-WMATV0942547
   ident: JP2921HQ0A6ZJA
   ident: 9VP9Q4SD
{{< /highlight >}}

|Drive Bay|HDD Serial Number|Device Node|
|---------|-----------------|-----------|
|00       |WD-WMATV0942547  |ada0       |
|01       |JP2921HQ0A6ZJA   |ada1       |
|02       |9VP9Q4SD         |ada2       |

> In this example, the device node numbers happened to match the drive bay numbers, but this ordering is not guaranteed.
Hardware faults or swapped cables may cause the device nodes to be numbered in a different order, so it's important to pair up the device nodes with the HDD serial numbers rather than the drive bay numbers.

Use **gpart modify** to change the GPT label on the third partition of each drive to include the drive bay id and serial number.

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

=>        40  1953525088  ada2  GPT  (932G)
          40        1024     1  gptboot2  (512K)
        1064         984        - free -  (492K)
        2048     4194304     2  swap2  (2.0G)
     4196352  1949327360     3  zfs2  (930G)
  1953523712        1416        - free -  (708K)

# gpart modify -i3 -l zfs-00-V0942547 ada0
ada0p3 modified
# gpart modify -i3 -l zfs-01-HQ0A6ZJA ada1
ada1p3 modified
# gpart modify -i3 -l zfs-02-9VP9Q4SD ada2
ada2p3 modified

# gpart show -l
=>        40  1953525088  ada0  GPT  (932G)
          40        1024     1  gptboot0  (512K)
        1064         984        - free -  (492K)
        2048     4194304     2  swap0  (2.0G)
     4196352  1949327360     3  zfs-00-V0942547  (930G)
  1953523712        1416        - free -  (708K)

=>        40  1953525088  ada1  GPT  (932G)
          40        1024     1  gptboot1  (512K)
        1064         984        - free -  (492K)
        2048     4194304     2  swap1  (2.0G)
     4196352  1949327360     3  zfs-01-HQ0A6ZJA  (930G)
  1953523712        1416        - free -  (708K)

=>        40  1953525088  ada2  GPT  (932G)
          40        1024     1  gptboot2  (512K)
        1064         984        - free -  (492K)
        2048     4194304     2  swap2  (2.0G)
     4196352  1949327360     3  zfs-02-9VP9Q4SD  (930G)
  1953523712        1416        - free -  (708K)
{{< /highlight >}}

That's it for the labels.

To simulate a minor problem, take drive 1 offline with **zpool offline**.
Note that **zpool status** now says that the *OFFLINE* drive **was ada1**.
Also note that the device node **ada1** has not been reassigned to any of the other *ONLINE* drives.

{{< highlight txt >}}
# zpool offline zroot ada1p3.eli

# zpool status
  pool: zroot
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: resilvered 823M in 0 days 00:00:16 with 0 errors on Mon Mar 23 22:27:10 2020
config:

        NAME                     STATE     READ WRITE CKSUM
        zroot                    DEGRADED     0     0     0
          mirror-0               DEGRADED     0     0     0
            ada0p3.eli           ONLINE       0     0     0
            4185812879084831896  OFFLINE      0     0     0  was /dev/ada1p3.eli
            ada2p3.eli           ONLINE       0     0     0

errors: No known data errors
{{< /highlight >}}

For cases like this, where the problem drive can still be detected by the system and the device node has not been reassigned,
run **gpart show** on the *OFFLINE* partition to get the faulty drive's bay id and serial number from the GPT label.

{{< highlight txt >}}
# gpart show -l ada1 | grep zfs
     4196352  1949327360     3  zfs-01-HQ0A6ZJA  (930G)
{{< /highlight >}}

To simulate a more serious problem, power down the server, disconnect drive 1 completely and restart the server.

Run **zpool status** again and note that ZFS still indicates that the *OFFLINE* drive **was ada1**, but also note that device node **ada1** has been reassigned to another drive.

{{< highlight txt >}}
$ su
Password:

# zpool status
  pool: zroot
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: resilvered 823M in 0 days 00:00:16 with 0 errors on Mon Mar 23 22:27:10 2020
config:

        NAME                     STATE     READ WRITE CKSUM
        zroot                    DEGRADED     0     0     0
          mirror-0               DEGRADED     0     0     0
            ada0p3.eli           ONLINE       0     0     0
            4185812879084831896  OFFLINE      0     0     0  was /dev/ada1p3.eli
            ada1p3.eli           ONLINE       0     0     0

errors: No known data errors
{{< /highlight >}}

For cases like this, where the system cannot detect the problem drive at all and reassigns the device node to another drive,
use **gpart show** to list all of the partitions and look for the one that's missing from the list to find the faulty drive.

{{< highlight txt >}}
# gpart show -l | grep zfs
     4196352  1949327360     3  zfs-00-V0942547  (930G)
     4196352  1949327360     3  zfs-02-9VP9Q4SD  (930G)
{{< /highlight >}}

To restore the system to its original state after the test, power down the server, reattach the drive, power up, bring it back online and wait for it to resilver.

{{< highlight txt >}}
$ su
Password:

# zpool status
  pool: zroot
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: scrub repaired 0 in 0 days 00:00:13 with 0 errors on Wed Mar 25 09:24:16 2020
config:

        NAME                     STATE     READ WRITE CKSUM
        zroot                    DEGRADED     0     0     0
          mirror-0               DEGRADED     0     0     0
            ada0p3.eli           ONLINE       0     0     0
            4185812879084831896  OFFLINE      0     0     0  was /dev/ada1p3.eli
            ada2p3.eli           ONLINE       0     0     0

errors: No known data errors

# zpool online zroot 4185812879084831896

# zpool status
  pool: zroot
 state: ONLINE
status: One or more devices is currently being resilvered.  The pool will
        continue to function, possibly in a degraded state.
action: Wait for the resilver to complete.
  scan: resilver in progress since Wed Mar 25 09:34:38 2020
        824M scanned at 74.9M/s, 506M issued at 46.0M/s, 825M total
        494M resilvered, 61.36% done, 0 days 00:00:06 to go
config:

        NAME            STATE     READ WRITE CKSUM
        zroot           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            ada0p3.eli  ONLINE       0     0     0
            ada1p3.eli  ONLINE       0     0     0
            ada2p3.eli  ONLINE       0     0     0

errors: No known data errors
{{< /highlight >}}

After the resilver finishes, run **zpool scrub** and then clear the detected errors with **zpool clear**.

{{< highlight txt >}}
# zpool scrub zroot

# zpool status
  pool: zroot
 state: ONLINE
status: One or more devices has experienced an unrecoverable error.  An
        attempt was made to correct the error.  Applications are unaffected.
action: Determine if the device needs to be replaced, and clear the errors
        using 'zpool clear' or replace the device with 'zpool replace'.
   see: http://illumos.org/msg/ZFS-8000-9P
  scan: scrub repaired 0 in 0 days 00:00:12 with 0 errors on Wed Mar 25 09:44:58 2020
config:

        NAME            STATE     READ WRITE CKSUM
        zroot           ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            ada0p3.eli  ONLINE       0     0     0
            ada1p3.eli  ONLINE       0     0     6
            ada2p3.eli  ONLINE       0     0     0

errors: No known data errors

# zpool clear zroot
{{< /highlight >}}
