---
title: "Jail Gitea in FreeBSD"
date: 2020-01-18T16:07:26-08:00
tags: ["FreeBSD", "iocage", "jails", "git"]
---

Here's the bare minimum required to run a local instance of [**Gitea**](https://gitea.com/) inside a FreeBSD jail.

<!--more-->

First, use `iocage` to create a jail called *git*.

{{< highlight txt >}}
$ su
Password:

# iocage create -n "git" -r latest vnet="on" allow_raw_sockets="1" boot="on" bpf="yes" dhcp="on"
git successfully created!
git: vnet requires defaultrouter, using 192.168.0.1
* Starting git
  + Started OK
  + Using devfs_ruleset: 5
  + Configuring VNET OK
  + Using IP options: vnet
  + Starting services OK
  + Executing poststart OK
  + DHCP Address: 192.168.0.112/24
{{< /highlight >}}

Find and install the latest Gitea package inside the jail.

{{< highlight txt >}}
# iocage exec git pkg search gitea
The package management tool is not yet installed on your system.
Do you want to fetch and install it now? [y/N]: y
Bootstrapping pkg from pkg+http://pkg.FreeBSD.org/FreeBSD:12:amd64/quarterly, please wait...
Verifying signature with trusted certificate pkg.freebsd.org.2013102301... done
[git] Installing pkg-1.12.0...
[git] Extracting pkg-1.12.0: 100%
pkg: Repository FreeBSD missing. 'pkg update' required
gitea-1.10.3                   Compact self-hosted Git service

# iocage exec git pkg install -y gitea
Updating FreeBSD repository catalogue...
FreeBSD repository is up to date.
All repositories are up to date.
Updating database digests format: 100%
The following 36 package(s) will be affected (of 0 checked):
[...]
{{< /highlight >}}

After the installation finishes, enter the jail's console for configuration.

{{< highlight txt >}}
# iocage console git
FreeBSD 12.1-RELEASE r354233 GENERIC

Welcome to FreeBSD!

Release Notes, Errata: https://www.FreeBSD.org/releases/
Security Advisories:   https://www.FreeBSD.org/security/
FreeBSD Handbook:      https://www.FreeBSD.org/handbook/
FreeBSD FAQ:           https://www.FreeBSD.org/faq/
Questions List: https://lists.FreeBSD.org/mailman/listinfo/freebsd-questions/
FreeBSD Forums:        https://forums.FreeBSD.org/

Documents installed with the system are in the /usr/local/share/doc/freebsd/
directory, or can be installed later with:  pkg install en-freebsd-doc
For other languages, replace "en" with a language code like de or fr.

Show the version of FreeBSD installed:  freebsd-version ; uname -a
Please include that output and any error messages when posting questions.
Introduction to manual pages:  man man
FreeBSD directory layout:      man hier

Edit /etc/motd to change this login announcement.
{{< /highlight >}}

The configuration file */usr/local/etc/gitea/conf/app.ini* requires minimal changes before running the program, but make a backup copy of it first.

{{< highlight txt >}}
root@git:~ # cp /usr/local/etc/gitea/conf/app.ini /usr/local/etc/gitea/conf/app.ini.bak
{{< /highlight >}}

Replace the **HTTP_ADDR** with the IP address of the jail and disable user registrations on the web interface by setting **DISABLE_REGISTRATION** to *true*.

{{< highlight txt >}}
root@git:~ # sed -i .tmp 's/^HTTP_ADDR.*=.*$/HTTP_ADDR = 192.168.0.112/g' /usr/local/etc/gitea/conf/app.ini
root@git:~ # sed -i .tmp 's/^DISABLE_REGISTRATION.*=.*$/DISABLE_REGISTRATION = true/g' /usr/local/etc/gitea/conf/app.ini
{{< /highlight >}}

Use `gitea generate secret` to replace the three secret configuration values.

{{< highlight txt >}}
root@git:~ # sed -i .tmp 's/^JWT_SECRET.*=.*$/JWT_SECRET = '`gitea generate secret JWT_SECRET`'/g' /usr/local/etc/gitea/conf/app.ini
root@git:~ # sed -i .tmp 's/^INTERNAL_TOKEN.*=.*$/INTERNAL_TOKEN = '`gitea generate secret INTERNAL_TOKEN`'/g' /usr/local/etc/gitea/conf/app.ini
root@git:~ # sed -i .tmp 's/^SECRET_KEY.*=.*$/SECRET_KEY = '`gitea generate secret SECRET_KEY`'/g' /usr/local/etc/gitea/conf/app.ini
{{< /highlight >}}

Finally, `diff` the backup and current versions of the configration file to make sure the changes look correct.

{{< highlight txt >}}
root@git:~ # diff /usr/local/etc/gitea/conf/app.ini.bak /usr/local/etc/gitea/conf/app.ini

52c52
< JWT_SECRET = D56bmu6xCtEKs9vKKgMKnsa4X9FDwo64HVyaS4fQ4mY
---
> JWT_SECRET = IL7AcvDb77_jtvZymGR1G8xQxm2hSHn3hGSItYEp73s

70,71c70,71
< INTERNAL_TOKEN = 1FFhAklka01JhgJTRUrFujWYiv4ijqcTIfXJ9o4n1fWxz+XVQdXhrqDTlsnD7fvz7gugdhgkx0FY2Lx6IBdPQw==
< SECRET_KEY   = ChangeMeBeforeRunning
---
> INTERNAL_TOKEN = eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJuYmYiOjE1Nzk5MjMwMjh9.84_14lNPr2lfXxtZZ_S4m4v2I67a-ZUxfwHipri5sQU
> SECRET_KEY = PW9GTLmuKMcnrWE05oYhxyaZ3SG46RZuLTwbfcTBoinse5RP7IVsBy1R5CFkIGXM

79c79
< HTTP_ADDR    = 127.0.0.1
---
> HTTP_ADDR = 192.168.0.112

91c91
< DISABLE_REGISTRATION   = false
---
> DISABLE_REGISTRATION = true
{{< /highlight >}}

Enable and start the *gitea service*, then check the log to make sure the expected IP address appears at the end.

{{< highlight txt >}}
root@git:~ # sysrc gitea_enable=YES
gitea_enable:  -> YES

root@git:~ # service gitea start

root@git:~ # service gitea status
gitea is running as pid 14836.

root@git:~ # tail -1 /var/log/gitea/gitea.log
2020/01/24 19:52:18 ...ce/gracehttp/http.go:142:Serve() [I] Serving 192.168.0.112:3000 with pid 14836
{{< /highlight >}}

To register new users at the command line, switch to the *git* user and call [`gitea admin create-user`](https://docs.gitea.io/en-us/command-line/).
Specify the required *user name*, *password* and *email address* parameters.

{{< highlight txt >}}
root@git:~ # su git

$ gitea admin create-user --username ccammack --password 1234 --email ccammack@example.com --admin -c /usr/local/etc/gitea/conf/app.ini

2020/01/24 19:55:01 ...dules/setting/git.go:87:newGit() [I] Git Version: 2.24.1, Wire Protocol Version 2 Enabled
2020/01/24 19:55:01 .../xorm/session_raw.go:76:queryRows() [I] [SQL] SELECT count(*) FROM `user` WHERE (type=0) - took: 4.149963ms
[...]
New user 'ccammack' has been successfully created!
{{< /highlight >}}

Open [http://192.168.0.112:3000](http://192.168.0.112:3000) in a broswer and sign in with the username and password.

{{< figure src="gitea-main-page.png" alt="Gitea main page">}}
