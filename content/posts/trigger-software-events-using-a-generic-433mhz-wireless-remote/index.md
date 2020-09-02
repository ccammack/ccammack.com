---
title: "Trigger Software Events Using a Generic 433MHz Wireless Remote"
date: 2020-07-25T20:54:14-07:00
tags: ["Windows", "Ubuntu", "HTPC", "IoT", "Home Automation"]
---

Combine an inexpensive [software-defined radio](https://en.wikipedia.org/wiki/Software-defined_radio) with
a generic key fob remote and some [open source software](https://github.com/merbanan/rtl_433) to trigger events on a computer.

<!--more-->

{{< figure src="sdr-dongle-and-key-fob-remote.jpg" alt="SDR dongle and key fob remote">}}

This post explains how to install the SDR drivers on Windows and Ubuntu and provides sample scripts that can be customized for use with
IoT, media playback, photography or any other situation where a small remote is more convenient to use than a keyboard.

Remotes suitable for this use are sold as *433MHz universal fixed code cloning learning remotes* and are intended for use with gates and garage doors.
Avoid remotes designed to pair only with specific brands because they implement a *rolling code* security feature that broadcasts a different signal each time a button is pressed.

---

##### Windows Installation

Follow the [SDR# (SDRSharp) Set Up Guide](https://www.rtl-sdr.com/rtl-sdr-quick-start-guide/) to install the required drivers on Windows.

1. Download the [Windows SDR Software Package](https://airspy.com/download/)
2. Unzip the contents to a folder on the computer, such as `C:\SDRSharp`
3. Double-click `C:\SDRSharp\install-rtlsdr.bat` to download the required drivers
4. Plug in the SDR dongle and wait a few seconds for the system to detect the device
5. Right-click `C:\SDRSharp\zadig.exe` and select **Run as adminstrator**
6. In the *Zadig* menu, select **Options > List All Devices**
	{{< figure src="list-all-devices.png" alt="Select Options > List All Devices">}}
7. Select the SDR device, which will appear as something like **RTL2832UHIDIR**, **RTL2832U** or **Bulk-In, Interface (Interface 0)** from the drop down list
	{{< figure src="select-device.png" alt="Select the SDR device">}}
8. Press the **Replace Driver** button to install the appropriate drivers on the device
	{{< figure src="replace-driver.png" alt="Press the Replace Driver button">}}
	{{< figure src="installation-success.png" alt="Drivers have been successfully installed">}}

Next, download a recent Windows release of [rtl_433](https://github.com/merbanan/rtl_433) from [bintray.com](https://bintray.com/chzu/dist/rtl_433#files)

For my 64-bit version of Windows 8, I selected a recently updated *win-x64* release,
which for me was the file [rtl_433-20.02-94-gf03710a-win-x64.zip](https://bintray.com/chzu/dist/download_file?file_path=rtl_433-20.02-94-gf03710a-win-x64.zip).

After downloading, unzip the file and copy the static executable, **rtl_433_64bit_static.exe**, to a folder elsewhere on the computer, such as `C:\Users\ccammack\bin`.

Open the console and run `rtl_433_64bit_static.exe` from the command line:

{{< highlight txt >}}
C:\Users\ccammack\bin (master)
λ rtl_433_64bit_static.exe
rtl_433 version 20.02-94-gf03710a branch  at 202007201253 inputs file rtl_tcp RTL-SDR
Use -h for usage help and see https://triq.org/ for documentation.
Trying conf file at "C:\Users\ccammack\bin\rtl_433.conf"...
Trying conf file at "C:\Users\ccammack\AppData\Local\rtl_433\rtl_433.conf"...
Trying conf file at "C:\ProgramData\rtl_433\rtl_433.conf"...
Registered 130 out of 158 device decoding protocols [ 1-4 8 11-12 15-17 19-21 23 25-26 29-36 38-60 63 67-71 73-100 102-105 108-116 119 121 124-128 130-149 151-158 ]
Found Rafael Micro R820T tuner
Exact sample rate is: 250000.000414 Hz
[R82XX] PLL not locked!
Sample rate set to 250000 S/s.
Tuner gain set to Auto.
Tuned to 433.920MHz.
{{< /highlight >}}

Press each button on the remote, noting the output that appears in the console.
My remote has four buttons, labeled `A`-`D`, which produced the following output.

{{< highlight txt >}}
time      : 2020-07-24 16:27:38
model     : Generic-Remote                         House Code: 21845
Command   : 12           Tri-State : ZZZZZZZZ0010
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-07-24 16:27:40
model     : Generic-Remote                         House Code: 21845
Command   : 48           Tri-State : ZZZZZZZZ0100
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-07-24 16:27:50
model     : Generic-Remote                         House Code: 21845
Command   : 192          Tri-State : ZZZZZZZZ1000
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-07-24 16:27:52
model     : Generic-Remote                         House Code: 21845
Command   : 3            Tri-State : ZZZZZZZZ0001
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
{{< /highlight >}}

In population-dense areas, the SDR dongle may also pick up messages from other devices operating on the same frequency such as remote controls for ceiling fans as well as devices like home weather stations and wireless door bells.

For example, while testing, my SDR also received a message from one of the tire sensors on a passing Toyota indicating that it is over-inflated to nearly 38 PSI.

{{< highlight txt >}}
time      : 2020-07-24 16:45:45
model     : Schrader     type      : TPMS          flags     : 07
ID        : 0CA376E
Pressure  : 260.0 kPa    Temperature: 28 C         Integrity : CRC
{{< /highlight >}}

To filter out these extraneous messages, create a wrapper script around `rtl_433_64bit_static.exe` that looks for the button-specific `Command` messages and triggers the desired events.
For my own use, I created a script called `remote.bat` and placed it in `C:\Users\ccammack\bin` along with `rtl_433_64bit_static.exe`.

It might be possible to write this as a pure batch script using `FOR` and `FINDSTR`,
but a [polyglot](https://en.wikipedia.org/wiki/Polyglot_(computing)) of [batch and Powershell](https://stackoverflow.com/a/49122891) was much easier to get working.
Replace the calls to `Write-Host` with your own functionality and adjust the length of the `debounce` variable as needed to ignore duplicate events.

{{< highlight ps1 >}}
C:\Users\ccammack\bin (master)
λ cat remote.bat
<#  :cmd header for PowerShell script
@   set dir=%~dp0
@   set ps1="%TMP%\%~n0-%RANDOM%-%RANDOM%-%RANDOM%-%RANDOM%.ps1"
@   copy /b /y "%~f0" %ps1% >nul
@   powershell -NoProfile -ExecutionPolicy Bypass -File %ps1% %*
@   del /f %ps1%
@   goto :eof
#>
$ps = New-Object System.Diagnostics.Process
$ps.StartInfo.Filename = "rtl_433_64bit_static.exe"
$ps.StartInfo.RedirectStandardError = $True
$ps.StartInfo.RedirectStandardOutput = $True
$ps.StartInfo.UseShellExecute = $False
[void] $ps.Start()
$stopWatch = [System.Diagnostics.Stopwatch]::StartNew()
$debounce = 750
$prevLine = ""
while (!$ps.HasExited) {
	[string] $line = $ps.StandardOutput.ReadLine();
	if ($stopWatch.ElapsedMilliseconds -gt $debounce -and
		$prevLine -like "*House Code: 21845") {
		switch -wildcard ($line) {
			"Command   : 12*"	{ Write-Host "Button A" }
			"Command   : 48*"	{ Write-Host "Button B" }
			"Command   : 192*"	{ Write-Host "Button C" }
			"Command   : 3*"	{ Write-Host "Button D" }
		}
		$stopWatch.Restart()
	}
	$prevLine = $line
}
{{< /highlight >}}

To test, run the batch file, press each button on the remote to see the feedback and then press `Ctrl-C` to exit.

{{< highlight txt >}}
C:\Users\ccammack\bin (master)
λ remote.bat
Button A
Button B
Button C
Button D
Terminate batch job (Y/N)? y
{{< /highlight >}}

---

##### Ubuntu Installation

Installation on Ubuntu is much simpler. Simply plug in the SDR dongle, install `rtl-433` and run a quick test to make sure everything works.

{{< highlight txt >}}
~$ sudo apt -y install rtl-433
[...]

~$ rtl_433 
[...]
Tuned to 433.920MHz.
Allocating 15 zero-copy buffers
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-07-24 01:13:14
model     : Generic-Remote                         House Code: 21845
Command   : 12           Tri-State : ZZZZZZZZ0010
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-07-24 01:13:15
model     : Generic-Remote                         House Code: 21845
Command   : 48           Tri-State : ZZZZZZZZ0100
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-07-24 01:13:17
model     : Generic-Remote                         House Code: 21845
Command   : 192          Tri-State : ZZZZZZZZ1000
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-07-24 01:13:18
model     : Generic-Remote                         House Code: 21845
Command   : 3            Tri-State : ZZZZZZZZ0001
^CSignal caught, exiting!
Reattached kernel driver
{{< /highlight >}}

Write a shell script to filter the output.

{{< highlight txt >}}
~/bin$ cat ./remote.sh
#!/bin/sh

stopWatch=$(date +%s%3N)
debounce=750
prevLine=""
rtl_433 2>/dev/null | while read line; do
  elapsed=$(( $(date +%s%3N) - stopWatch ))
  if [ "$elapsed" -gt "$debounce" ] &&
     [ "$prevLine" = "House Code: 21845" ]; then
    case "$line" in
      "Command   : 12")  echo "Button A"
      ;;
      "Command   : 48")  echo "Button B"
      ;;
      "Command   : 192") echo "Button C"
      ;;
      "Command   : 3")   echo "Button D"
      ;;
    esac
    stopWatch=$(date +%s%3N)
  fi
  prevLine="$line"
done 
{{< /highlight >}}

Run the script and press the remote buttons.

{{< highlight txt >}}
~/bin$ chmod +x ./remote.sh

~/bin$ ./remote.sh
Button A
Button B
Button C
Button D
^C
{{< /highlight >}}

---

##### Need More Buttons?

If the key fob doesn't have enough buttons for your needs, check the list of **Supported device protocols** on the [rtl_433 page](https://github.com/merbanan/rtl_433#running)
and the corresponding [test folder for each device](https://github.com/merbanan/rtl_433_tests/tree/master/tests) for some other possibilities.

One good option is the **DirecTV RC66RX Remote Control**, which can be had for under $10 and works well with this setup.

{{< figure src="directv-rc66rx-remote-control.jpg" alt="DirecTV RC66RX Remote Control">}}

To [enable RF mode](https://github.com/merbanan/rtl_433_tests/tree/master/tests/directv#description) on the remote:

1. Move the input select slider all the way to the left
1. Press and hold `Mute` and `SELECT` until the LED flashes twice
1. Enter `9` `6` `1` and the LED will flash twice
1. Press `CHAN|PAGE Up` to enter RF mode and the LED will flash twice
1. Enter any 6-digit code you want to identify the remote
1. Press `SELECT` and the LED will flash twice

If the LED does not flash twice in the last step, repeat the procedure without any delay between steps.

Run `rtl_433` again and press some buttons to see the output.

{{< highlight txt >}}
C:\Users\ccammack\bin (master)
λ rtl_433_64bit_static.exe
[...]
Tuned to 433.920MHz.
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-08-16 00:54:49
model     : DirecTV-RC66RX                         id        : 123456
button_id : 0x30         button_name: [VCR PLAY]   event     : INITIAL
mic       : CHECKSUM
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-08-16 00:54:49
model     : DirecTV-RC66RX                         id        : 123456
button_id : 0x30         button_name: [VCR PLAY]   event     : REPEAT
mic       : CHECKSUM
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-08-16 00:55:03
model     : DirecTV-RC66RX                         id        : 123456
button_id : 0x31         button_name: [VCR STOP]   event     : INITIAL
mic       : CHECKSUM
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-08-16 00:55:03
model     : DirecTV-RC66RX                         id        : 123456
button_id : 0x31         button_name: [VCR STOP]   event     : REPEAT
mic       : CHECKSUM
Signal caught, exiting!
{{< /highlight >}}

The `VOL`, `Mute`, `TV INPUT` and `TV POWER` buttons do not generate unique RF signals because they were intended to control the functions on a TV or stereo using IR.
Instead, they all broadcast a single common RF signal, which appears as `DTV: IR ALERT` in the output stream.

The remaining ~40 buttons all generate unique RF signals. In addition, `rtl_433` will automatically distinguish the `INITIAL` and `REPEAT` button events in the output.

{{< highlight txt >}}
time      : 2020-08-16 00:48:54
model     : DirecTV-RC66RX                         id        : 123456
button_id : 0x5B         button_name: [DTV: IR ALERT]	event     : INITIAL
mic       : CHECKSUM
_ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _ _
time      : 2020-08-16 00:48:54
model     : DirecTV-RC66RX                         id        : 123456
button_id : 0x5B         button_name: [DTV: IR ALERT]	event     : REPEAT
mic       : CHECKSUM
{{< /highlight >}}
