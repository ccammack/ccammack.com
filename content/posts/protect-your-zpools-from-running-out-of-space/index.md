---
title: "Protect Your zpools From Running Out of Space"
date: 2020-09-20T21:30:41-07:00
tags: ["FreeBSD", "ZFS"]
---

One quirk of ZFS is that deleting a file actually uses a small amount of additional drive space to record the deletion,
which causes significant problems if a zpool is ever allowed to fill up 100%.

<!--more-->

Fortunately, it's easy to prevent. As Benedict says:

{{< highlight txt >}}
Don't let your zpool fill up completely by creating a dataset with
reservation.

# zfs create -o refreservation=<5% of total pool space> <poolname>/reserved

You can always shrink the reserve if you need the space, but your pool will
always have space left this way.

                -- Benedict Reuschling [...]
{{< /highlight >}}

Easy.

{{< highlight txt >}}
$ su
Password:

# zpool list -H -o size backup
3.62T

# bc
3.62 * 0.05 * 1024
184.32
^C
Broken pipe

# zfs create -o refreservation=185G backup/reserved
{{< /highlight >}}
	
Thanks, Benedict.
