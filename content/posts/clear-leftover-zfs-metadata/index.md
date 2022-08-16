---
title: "Clear Leftover ZFS Metadata"
date: 2022-07-30T11:18:29-07:00
tags: ["FreeBSD", "ZFS"]
---

Redeploying an old drive that previously had zpools on it can potentially cause name conflicts when trying to import the new pool into the system
unless the leftover ZFS metadata has been cleared first.

<!--more-->

For example, I recently repartitioned and GELI-encrypted an old drive to add to the backup rotation, but ZFS failed to import the new `backup` zpool.

{{< highlight txt >}}
# zpool import backup
cannot import 'backup': more than one matching pool
import by numeric ID instead
{{< /highlight >}}

That was unexpected. Running `zpool import` showed that there were actually two pools on the system named `backup`.

{{< highlight txt >}}
# zpool import
   pool: backup
     id: 9534841782064508861
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        backup            ONLINE
          gpt/backup.eli  ONLINE

   pool: backup
     id: 10113335167323929010
  state: UNAVAIL
status: One or more devices are missing from the system.
 action: The pool cannot be imported. Attach the missing
        devices and try again.
   see: https://openzfs.github.io/openzfs-docs/msg/ZFS-8000-3C
 config:

        backup        UNAVAIL  insufficient replicas
          gpt/backup  UNAVAIL  cannot open
{{< /highlight >}}

I was expecting to see the first pool, on `gpt/backup.eli`, since I had just created it, but the second one, on `gpt/backup`, was left over from some long-forgotten test.

Fortunately, ZFS provides the `zdb` command which can be used to display the metadata for the unexpected pool on the `gpt/backup` device.
Note the `hostname` and `path` fields, which show the machine and partition on which the pool was originally created.

{{< highlight txt >}}
# zdb -l /dev/gpt/backup
------------------------------------
LABEL 0
------------------------------------
    version: 5000
    name: 'backup'
    state: 1
    txg: 14352
    pool_guid: 5882229770500143841
    hostid: 4108274352
    hostname: 'freebsd-test'
    top_guid: 8420073982120704520
    guid: 8420073982120704520
    vdev_children: 1
    vdev_tree:
        type: 'disk'
        id: 0
        guid: 8420073982120704520
        path: '/dev/da0p1'
        whole_disk: 1
        metaslab_array: 67
        metaslab_shift: 35
        ashift: 12
        asize: 4000780386304
        is_log: 0
        create_txg: 4
    features_for_read:
        com.delphix:hole_birth
        com.delphix:embedded_data
    labels = 0 1 2 3
{{< /highlight >}}

Contrast that with the `hostname` and `path` recorded on the new device, which indicates that it was created on the backup server and GELI-encrypted as expected.

{{< highlight txt >}}
# zdb -l /dev/gpt/backup.eli
------------------------------------
LABEL 0
------------------------------------
    version: 5000
    name: 'backup'
    state: 1
    txg: 48
    pool_guid: 15581526701847492683
    errata: 0
    hostid: 676551432
    hostname: 'backup.ccammack.com'
    top_guid: 16613588697789622250
    guid: 16613588697789622250
    vdev_children: 1
    vdev_tree:
        type: 'disk'
        id: 0
        guid: 16613588697789622250
        path: '/dev/gpt/backup.eli'
        whole_disk: 1
        metaslab_array: 256
        metaslab_shift: 34
        ashift: 12
        asize: 4000780124160
        is_log: 0
        create_txg: 4
    features_for_read:
        com.delphix:hole_birth
        com.delphix:embedded_data
    labels = 0 1 2 3
{{< /highlight >}}

This conflict happens because ZFS stores two copies of its metadata in the first 512K of each device and two copies in the last 512K of each device,
and since those areas are not overwritten when repartitioning, new pools may conflict with the leftover metadata from the old ones if they are assigned the same names.

Don't do this on a data drive you care about unless you have good backups, but it's easy to fix:
use `zpool labelclear` to wipe out the metadata on the older device and check it again with `zdb` to see that all four areas have been cleared.

{{< highlight txt >}}
# zpool labelclear -f /dev/gpt/backup

# zdb -l /dev/gpt/backup
failed to unpack label 0
failed to unpack label 1
failed to unpack label 2
failed to unpack label 3
{{< /highlight >}}

Check for importable pools again and note that ZFS only finds the new one, which can be imported as usual.

{{< highlight txt >}}
# zpool import
   pool: backup
     id: 15581526701847492683
  state: ONLINE
 action: The pool can be imported using its name or numeric identifier.
 config:

        backup            ONLINE
          gpt/backup.eli  ONLINE

# zpool import backup

# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP    HEALTH  ALTROOT
backup  3.62T  1.13M  3.62T        -         -     0%     0%  1.00x    ONLINE  -
zroot    117G  18.5G  98.5G        -         -    22%    15%  1.00x    ONLINE  -

# zpool status backup
  pool: backup
 state: ONLINE
config:

        NAME              STATE     READ WRITE CKSUM
        backup            ONLINE       0     0     0
          gpt/backup.eli  ONLINE       0     0     0

errors: No known data errors
{{< /highlight >}}

Finis.
