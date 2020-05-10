---
title: "Install Cygwin and apt-cyg (and SSH)"
date: 2018-03-30T23:41:01-07:00
tags: ["Cygwin", "Windows"]
---

This is the easiest way I know of to install [Cygwin](https://www.cygwin.com) and [apt-cyg](https://github.com/transcode-open/apt-cyg) on Windows.

<!--more-->

1. Open the Start menu, type `cmd` to filter, and select `cmd` to open a console window.
{{< figure src="open-cmd-prompt.png" alt="Open the Windows command prompt">}}

1. Use the console to create a root folder for Cygwin, such as `C:\cygwin`, then download and install Cygwin and `wget`.
{{< highlight bat >}}
C:\Windows\System32> mkdir c:\cygwin
C:\Windows\System32> explorer https://www.cygwin.com/setup-x86_64.exe
C:\Windows\System32> mv %USERPROFILE%\Downloads\setup-x86_64.exe c:\cygwin
C:\Windows\System32> c:\cygwin\setup-x86_64.exe -q -R c:\cygwin -s http://cygwin.mirror.constant.com -l c:\cygwin\packages -P wget
{{< /highlight >}}

1. After the installation finishes, a new shortcut called `Cygwin64 Terminal` will appear on the desktop.
{{< figure src="cygwin64-shortcut.png" alt="Cygwin64 Terminal desktop shortcut">}}

1. Right-click the `Cygwin64 Terminal` shortcut and select `Run as administrator` to run Cygwin as root.

1. In the Cygwin root console, use `wget` to download and install the `apt-cyg` package manager.
{{< highlight txt >}}
$ wget rawgit.com/transcode-open/apt-cyg/master/apt-cyg
$ install apt-cyg /bin
$ rm apt-cyg
{{< /highlight >}}

1. Follow the instructions for [apt-cyg](https://github.com/transcode-open/apt-cyg) to install additional software as needed.
For example, to install `SSH`, run the `Cygwin64 Terminal` as `Administrator` and enter the command `apt-cyg install openssh`.
{{< highlight txt >}}
$ apt-cyg install openssh
Installing openssh
openssh-7.6p1-1.tar.xz: OK
Unpacking...
Package openssh requires the following packages, installing:
bash csih cygrunsrv cygwin diffutils libcrypt0 libedit0 libgcc1 libgssapi_krb5_2 libkrb5_3 libopenssl100 libssp0 zlib0
Package bash is already installed, skipping
Package csih is already installed, skipping
Package cygrunsrv is already installed, skipping
Package cygwin is already installed, skipping
Package diffutils is already installed, skipping
Package libcrypt0 is already installed, skipping
Package libedit0 is already installed, skipping
Package libgcc1 is already installed, skipping
Package libgssapi_krb5_2 is already installed, skipping
Package libkrb5_3 is already installed, skipping
Package libopenssl100 is already installed, skipping
Package libssp0 is already installed, skipping
Package zlib0 is already installed, skipping
Running /etc/postinstall/openssh.sh
Package openssh installed
$ ssh
usage: ssh [-46AaCfGgKkMNnqsTtVvXxYy] [-b bind_address] [-c cipher_spec]
           [-D [bind_address:]port] [-E log_file] [-e escape_char]
           [-F configfile] [-I pkcs11] [-i identity_file]
           [-J [user@]host[:port]] [-L address] [-l login_name] [-m mac_spec]
           [-O ctl_cmd] [-o option] [-p port] [-Q query_option] [-R address]
           [-S ctl_path] [-W host:port] [-w local_tun[:remote_tun]]
           [user@]hostname [command]
{{< /highlight >}}

1. To periodically update an existing Cygwin installation, open the Start menu, type `cmd` to filter, select `cmd` to open a console, and run the setup program again.
{{< highlight bat >}}
C:\Windows\System32> C:\cygwin\setup-x86_64.exe --no-desktop --no-shortcuts --no-startmenu --quiet-mode
{{< /highlight >}}
