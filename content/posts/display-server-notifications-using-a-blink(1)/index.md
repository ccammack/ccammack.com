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

Specify `--green` to set the LED color to solid green.

{{< highlight txt >}}
# blink1-tool --green
{{< /highlight >}}

Specify `--red --blink 10` to set the LED color to red and flash it 10 times.

{{< highlight txt >}}
# blink1-tool --red --blink 10
{{< /highlight >}}

Reset the device using `--off`.

{{< highlight txt >}}
# blink1-tool --off
{{< /highlight >}}

Use `--playpattern <patternstr>` to create more complicated patterns.

The`<patternstr>` parameter consists of a list of values separated by commas.
The first value in the list is a repeat count indicating how many times the entire sequence should be repeated or 0 to indicate that the sequence should repeat forever.
Following the repeat count is a list of triples, each of which contains a hexidecimal RGB to specify color,
a float to specify the event length in seconds and an integer between 0 and 2 to indicate which LEDs to use.

For example, a sample pattern can be built up like this:

* `'` begin the pattern string
* `0,` repeat forever
* `#ff0000,1.0,0,` display *red* for 1 second on both LEDs
* `#000000,1.0,0,` display nothing for 1 second on both LEDs
* `#00ff00,2.0,1,` display *green* for 2 seconds on the bottom LED
* `#000000,1.0,0,` display nothing for 1 second on both LEDs
* `#0000ff,3.0,2,` display *blue* for 3 seconds on the top LED
* `#000000,1.0,0` display nothing for 1 second on both LEDs
* `'` end the pattern string

Set the transition time between states to 0ms using `-m 0` and the entire command looks like this:

{{< highlight txt >}}
# blink1-tool --playpattern '0,#ff0000,1.0,0,#000000,1.0,0,#00ff00,2.0,1,#000000,1.0,0,#0000ff,3.0,2,#000000,1.0,0' -m 0
{{< /highlight >}}

Use `--help` to see more detailed usage examples.

{{< highlight txt >}}
# blink1-tool --help
{{< /highlight >}}
