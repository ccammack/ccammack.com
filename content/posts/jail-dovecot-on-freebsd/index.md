---
title: "Jail Dovecot on FreeBSD"
date: 2023-05-06T10:50:52-07:00
tags: ["FreeBSD", "iocage", "jails", "dovecot", "imap", "email", "archive", "backup", "self-hosted"]
---

There are many software packages designed to archive email,
but it's pretty simple to just run a full [IMAP](https://en.wikipedia.org/wiki/Internet_Message_Access_Protocol)
server on local hardware and move messages to the archive using an email client.
Because IMAP is a [standard email protocol](https://en.wikipedia.org/wiki/Comparison_of_mail_servers),
the archive will be accessible to any device on the network
running practically any email client one might want to use.

<!--more-->

##### Installation

On FreeBSD, [*Dovecot*](https://www.dovecot.org/) is commonly used 
and it can be jailed using `iocage` as follows.
First, list the iocage releases and create a new jail named *imap*.

{{< highlight txt >}}
$ su
Password:

# iocage list -r
+---------------+
| Bases fetched |
+===============+
| 13.2-RELEASE  |
+---------------+

# iocage create -n "imap" -r 13.2-RELEASE --thickjail vnet="on" allow_raw_sockets="1" boot="on" bpf="yes" dhcp="on"
imap successfully created!
No default gateway found for ipv6.
* Starting imap
  + Started OK
  + Using devfs_ruleset: 1000 (iocage generated default)
  + Configuring VNET OK
  + Using IP options: vnet
  + Starting services OK
  + Executing poststart OK
  + DHCP Address: 192.168.1.165/24
{{< /highlight >}}

Next, enter the jail’s console to find and install the latest `dovecot` package.
Note the installation notes printed at the bottom of the output.

{{< highlight txt >}}
# iocage console imap
[...]
Welcome to FreeBSD!
[...]

root@imap:~ # hostname
imap

root@imap:~ # pkg search dovecot
[...]
dovecot-2.3.21_2               Secure, fast and powerful IMAP and POP3 server
[...]

root@imap:~ # pkg install -y dovecot
[...]

New packages to be INSTALLED:
        dovecot: 2.3.21_2
        liblz4: 1.9.4,1
        zstd: 1.5.5

[...]

You must create the configuration files yourself. Copy them over
 to /usr/local/etc/dovecot and edit them as desired:

        cp -R /usr/local/etc/dovecot/example-config/* \
                /usr/local/etc/dovecot

 The default configuration includes IMAP and POP3 services, will
 authenticate users agains the system's passwd file, and will use
 the default /var/mail/$USER mbox files.

 Next, enable dovecot in /etc/rc.conf:

        dovecot_enable="YES"

[...]
{{< /highlight >}}

---

##### Basic Configuration

As explained in the installation notes, the default configuration will *authenticate users against the system's passwd file*.
Create a new user account in the jail to receive the email archive and assign it a temporary password.

> The next section explains how to configure Dovecot using plaintext authentication,
so the chosen password will be sent in the clear until the configuration is properly secured.
Consider using a temporary password for the initial configuration and change it to a better one after securing the connection.
In any case, avoid reusing existing passwords.

{{< highlight txt >}}
root@imap:~ # pw useradd -n ccammack -m

root@imap:~ # passwd ccammack
Changing local password for ccammack
New Password:
Retype New Password:
{{< /highlight >}}

Next, copy the example configuration files to `/usr/local/etc/dovecot` and add `dovecot_enable="YES"` to `/etc/rc.conf`.

{{< highlight txt >}}
root@imap:~ # cp -R /usr/local/etc/dovecot/example-config/* /usr/local/etc/dovecot

root@imap:~ # sysrc dovecot_enable=YES
dovecot_enable:  -> YES

root@imap:~ # cat /etc/rc.conf
[...]
dovecot_enable="YES"
{{< /highlight >}}

Use `doveconf -a | more` to display the current Dovecot configuration settings.

{{< highlight txt >}}
root@imap:~ # doveconf -a | more
# 2.3.21 (47349e2482): /usr/local/etc/dovecot/dovecot.conf
[...]
{{< /highlight >}}

The installation notes indicate that Dovecot will store the email for each user in
an [mbox](https://en.wikipedia.org/wiki/Mbox) file named `/var/mail/$USER` by default
if the `mail_location` variable is not defined.

{{< highlight txt >}}
root@imap:~ # doveconf -a | grep mail_location
mail_location =

root@imap:~ # grep mail_location /usr/local/etc/dovecot/conf.d/*
[...]
/usr/local/etc/dovecot/conf.d/10-mail.conf:#mail_location =
[...]
{{< /highlight >}}


Dovecot also supports the
[Maildir++](https://doc.dovecot.org/admin_manual/mailbox_formats/maildir/#directory-structure) directory layout,
which is a much better option for storage.
Set `mail_location = maildir:~/Maildir` in the `10-mail.conf` file
to store email in a per-user `~/Maildir` directory.

{{< highlight txt >}}
root@imap:~ # ee /usr/local/etc/dovecot/conf.d/10-mail.conf
[...]

root@imap:~ # doveconf -a | grep mail_location
mail_location = maildir:~/Maildir
{{< /highlight >}}

---

##### Configure Plaintext IMAP Connections



Check the `protocols` configuration variable and note that the default list includes `imap`, which is the traditional unencrypted protocol.

{{< highlight txt >}}
root@imap:~ # doveconf -a | grep protocols
protocols = imap pop3 lmtp
{{< /highlight >}}

To verify that the unencrypted `imap` protocol uses port 143 by default, check the configuration values for `inet_listener imap`.

{{< highlight txt >}}
root@imap:~ # doveconf -a | grep -A7 "inet_listener imap"
  inet_listener imap {
    address =
    haproxy = no
    port = 143
    reuse_port = no
    ssl = no
  }
[...]
{{< /highlight >}}

The default configuration values look reasonable, so try starting the `dovecot` service.

{{< highlight txt >}}
root@imap:~ # service dovecot start
Starting dovecot.
doveconf: Fatal: Error in configuration file /usr/local/etc/dovecot/conf.d/10-ssl.conf line 12: ssl_cert: Can't open file /etc/ssl/certs/dovecot.pem: No such file or directory
/usr/local/etc/rc.d/dovecot: WARNING: failed to start dovecot
{{< /highlight >}}

The default configuration is looking for the `ssl-cert` file on line 12 (and also the `ssl_key` file on line 13), but they don't exist.
The [Dovecot documentation](https://doc.dovecot.org/admin_manual/ssl/certificate_creation/#self-signed-certificate-creation) indicates that
*[b]inary installations ususally create the certificate automatically when installing Dovecot*, but that dosen't seem to be the case on FreeBSD.

For the moment, disable the need for those missing files by editing `10-ssl.conf` to comment out the lines for `ssl_cert` and `ssl_key` on lines 12 and 13.

{{< highlight txt >}}
root@imap:~ # ee /usr/local/etc/dovecot/conf.d/10-ssl.conf
[...]

root@imap:~ # grep dovecot.pem /usr/local/etc/dovecot/conf.d/10-ssl.conf
#ssl_cert = </etc/ssl/certs/dovecot.pem
#ssl_key = </etc/ssl/private/dovecot.pem
{{< /highlight >}}

Also make sure Dovecot is configured to use only plain text [authentication](https://doc.dovecot.org/configuration_manual/dovecot_ssl_configuration/#dovecot-ssl-configuration)
by checking the configuration values for `auth_mechanisms` and `disable_plaintext_auth`,
which appears in the file `10-auth.conf`.

{{< highlight txt >}}
root@imap:~ # doveconf -a | grep auth_mechanisms
auth_mechanisms = plain

root@imap:~ # doveconf -a | grep disable_plaintext_auth
disable_plaintext_auth = yes

root@imap:~ # grep disable_plaintext_auth /usr/local/etc/dovecot/conf.d/*
/usr/local/etc/dovecot/conf.d/10-auth.conf:#disable_plaintext_auth = yes
[...]
{{< /highlight >}}

In my case, I had to edit `10-auth.conf` and set `disable_plaintext_auth = no` to allow plain text authentication.

{{< highlight txt >}}
root@imap:~ # ee /usr/local/etc/dovecot/conf.d/10-auth.conf
[...]

root@imap:~ # grep disable_plaintext_auth /usr/local/etc/dovecot/conf.d/*
/usr/local/etc/dovecot/conf.d/10-auth.conf:disable_plaintext_auth = no
[...]

root@imap:~ # doveconf -a | grep disable_plaintext_auth
disable_plaintext_auth = no
{{< /highlight >}}

Try starting `dovecot` again, successfully this time.

{{< highlight txt >}}
root@imap:~ # service dovecot start
Starting dovecot.
{{< /highlight >}}

---

##### Test Plaintext Connection

From another machine on the network, `ping` the IMAP server and make sure it answers.

{{< highlight txt >}}
C:\Users\ccammack
λ ping imap.ccammack.com

Pinging imap.ccammack.com [192.168.1.165] with 32 bytes of data:
Reply from 192.168.1.165: bytes=32 time<1ms TTL=64
Reply from 192.168.1.165: bytes=32 time<1ms TTL=64
Reply from 192.168.1.165: bytes=32 time<1ms TTL=64
Reply from 192.168.1.165: bytes=32 time<1ms TTL=64

Ping statistics for 192.168.1.165:
    Packets: Sent = 4, Received = 4, Lost = 0 (0% loss),
Approximate round trip times in milli-seconds:
    Minimum = 0ms, Maximum = 0ms, Average = 0ms
{{< /highlight >}}

Next, try to log into the server using `curl`.

{{< highlight txt >}}
C:\Users\ccammack
λ curl -v --url "imap://imap.ccammack.com/" --user "ccammack"
Enter host password for user 'ccammack':
*   Trying 192.168.1.165:143...
* Connected to imap.ccammack.com (192.168.1.165) port 143 (#0)
< * OK [CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE LITERAL+ STARTTLS AUTH=PLAIN] Dovecot ready.
> A001 CAPABILITY
< * CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE LITERAL+ STARTTLS AUTH=PLAIN
< A001 OK Pre-login capabilities listed, post-login capabilities have more.
> A002 AUTHENTICATE PLAIN AGNjYW1tYWNrAHBhc3N3b3Jk
< * CAPABILITY IMAP4rev1 SASL-IR LOGIN-REFERRALS ID ENABLE IDLE SORT SORT=DISPLAY THREAD=REFERENCES THREAD=REFS THREAD=ORDEREDSUBJECT MULTIAPPEND URL-PARTIAL CATENATE UNSELECT CHILDREN NAMESPACE UIDPLUS LIST-EXTENDED I18NLEVEL=1 CONDSTORE QRESYNC ESEARCH ESORT SEARCHRES WITHIN CONTEXT=SEARCH LIST-STATUS BINARY MOVE SNIPPET=FUZZY PREVIEW=FUZZY PREVIEW STATUS=SIZE SAVEDATE LITERAL+ NOTIFY SPECIAL-USE
< A002 OK Logged in
> A003 LIST "" *
< * LIST (\HasNoChildren) "." INBOX
* LIST (\HasNoChildren) "." INBOX
< A003 OK List completed (0.001 + 0.000 + 0.001 secs).
* Connection #0 to host imap.ccammack.com left intact
{{< /highlight >}}

On the server, check the Dovecot log for errors using `tail /var/log/maillog`.

{{< highlight txt >}}
root@imap:~ # tail /var/log/maillog
[...]
May 19 14:37:05 imap dovecot[77310]: imap-login: Login: user=<ccammack>, method=PLAIN, rip=192.168.1.100, lip=192.168.1.165, mpid=77326, session=<d+Ub39IYemjAqAFk>
May 19 14:37:05 imap dovecot[77310]: imap(ccammack)<77326><d+Ub39IYemjAqAFk>: Disconnected: Logged out in=27 out=576 deleted=0 expunged=0 trashed=0 hdr_count=0 hdr_bytes=0 body_count=0 body_bytes=0
{{< /highlight >}}

If the connection works, it should now be possible to add the IMAP server to an email client and move messages to it over the LAN.

{{< figure src="configure-email-client-imap-settings.1.png" alt="Configure Windows Live Mail Client IMAP Settings Step 1">}}
{{< figure src="configure-email-client-imap-settings.2.png" alt="Configure Windows Live Mail Client IMAP Settings Step 2">}}
