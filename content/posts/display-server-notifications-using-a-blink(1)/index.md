---
title: "Display Server Notifications Using a blink(1)"
date: 2020-04-19T20:24:57-07:00
tags: ["FreeBSD", "blink(1)"]
---

The [blink(1)](https://blink1.thingm.com/) is an open source USB notification light that can be controlled from the command line.

<!--more-->

{{< figure src="blink1-lit-green.jpg" alt="blink(1) set to full green">}}

I couldn't find a pre-built package for it in FreeBSD but it was easy to compile directly from source.
To do so, install a few dependencies and then clone the [repository](https://github.com/todbot/blink1-tool).

{{< highlight txt >}}
$ su
Password:

# pkg install -y gcc git gmake libiconv
[...]

# cd /tmp && git clone https://github.com/todbot/blink1-tool
[...]
{{< /highlight >}}

Compile	the	`blink1-tool` and install it on the system.

{{< highlight txt >}}
# cd blink1-tool && gmake && install blink1-tool /usr/local/bin
[...]

# which blink1-tool
/usr/local/bin/blink1-tool
{{< /highlight >}}

Insert the device into a USB port and make sure the system identifies it.

{{< highlight txt >}}
# dmesg
[...]
ugen0.3: <ThingM blink(1) mk3> at usbus0
uhid0 on uhub1
uhid0: <ThingM blink(1) mk3, class 0/0, rev 2.00/1.01, addr 5> on usbus0

# blink1-tool --list
blink(1) list:
id:0 - serialnum:3d5ee772 (mk3) fw version:303
{{< /highlight >}}

The [documentation](https://github.com/todbot/blink1/blob/master/docs/blink1-tool-tips.md) explains the program's basic usage.
For example, use `blink1-tool` to set the LED color to green and blink it 10 times.

{{< highlight txt >}}
# blink1-tool --green --blink 10
{{< /highlight >}}

Likewise, use `--red` to set the LED color to solid red. Reset the device using `--off`.

{{< highlight txt >}}
# blink1-tool --red
[...]

# blink1-tool --off
[...]
{{< /highlight >}}

Use `--help` to see more detailed usage examples.

{{< highlight txt >}}
# blink1-tool --help
[...]
{{< /highlight >}}
