---
title: "Create a Centralized Backup Server With FreeBSD"
date: 2020-04-26T13:24:37-07:00
tags: ["FreeBSD", "ZFS", "Backups", "Samba", "SFTP", "restic", "Duplicity", "zxfer"]
draft: true
---

To back up data on the cheap, using removable drives that you swap and carry off-site,
install a 5.25" to 3.5" hot swap bay in an old machine and centralize the backup process for all of your hosts onto a single server running FreeBSD 12.1 on ZFS.

<!--more-->

The hardware I'm using for this is a [Mini-ITX Intel J1900](https://www.ebay.com/sch/i.html?_nkw=Mini-ITX+Intel+J1900)
inside an [Inwin Development BM639 Mini ITX Slim Case](https://www.in-win.com/en/computer-chassis/bm-series/APAC),
which has internal bays for the OS and one external 5.25" bay that can hold a hot swap drive tray.
This same approach should work just as well with an external USB drive dock.

The J1900 is slow and outdated and [requires setting a couple of kernel options](https://bugs.freebsd.org/bugzilla/show_bug.cgi?id=230172#c48)
to boot FreeBSD properly, but it's perfectly capable of shuffling data from the network port to the backup drive.

In the motherboard settings, make sure *AHCI is enabled* for the SATA port attached to the removable drive to allow the hot swap feature to operate.
My own machine is a little quirky and will not enable the hot swap feature unless there is a drive present in the hot swap bay as the system boots.
Once booted, the feature operates properly and the drive can be hot swapped as many times as needed.

##### Prepare the Removable Backup Drive

To prepare the removable drive, insert it into the hot swap bay and create the backup partition, geli container and backup zpool.
In this example, the system assigns device node **ada1** to the drive.
Use a different password for the backup drives than those used for the system itself.

{{< highlight txt >}}
$ su
Password:

# dmesg
[...]
ada1 at ahcich1 bus 0 scbus1 target 0 lun 0
ada1: <Hitachi HUS724040ALE641 MJAOA5F0> ATA8-ACS SATA 3.x device
ada1: Serial Number P4H3KUPC
ada1: 300.000MB/s transfers (SATA 2.x, UDMA6, PIO 8192bytes)
ada1: Command Queueing enabled
ada1: 3815447MB (7814037168 512 byte sectors)

# gpart destroy -F ada1
ada1 destroyed

# gpart destroy -F ada1
gpart: arg0 'ada1': Invalid argument

# gpart create -s gpt ada1
ada1 created

# gpart add -a 1m -l backup -t freebsd-zfs "ada1"
ada1p1 added

# gpart show -l ada1
=>        40  7814037088  ada1  GPT  (3.6T)
          40        2008        - free -  (1.0M)
        2048  7814033408     1  backup  (3.6T)
  7814035456        1672        - free -  (836K)

# grep "geli init" /var/log/bsdinstall_log
DEBUG: zfs_create_boot: geli init -bg -e AES-XTS -J - -l 256 -s 4096 "ada0p3"

# geli init -e AES-XTS -l 256 -s 4096 "/dev/gpt/backup"
Enter new passphrase:
Reenter new passphrase:
[...]

# geli attach /dev/gpt/backup
Enter passphrase:

# geli status
[...]
gpt/backup.eli  ACTIVE  gpt/backup

# zpool create backup gpt/backup.eli

# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
backup  3.62T   336K  3.62T        -         -     0%     0%  1.00x  ONLINE  -
zroot    117G  2.98G   114G        -         -     3%     2%  1.00x  ONLINE  -
{{< /highlight >}}

After preparing the drive, `export` the backup pool, `detach` the geli device and **remove the drive** from the system.

{{< highlight txt >}}
# zpool export backup

# zpool list
NAME    SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
zroot   117G  2.98G   114G        -         -     3%     2%  1.00x  ONLINE  -

# geli status
[...]
gpt/backup.eli  ACTIVE  gpt/backup

# geli detach gpt/backup.eli

# geli status
[...]
{{< /highlight >}}

---

##### Detect Drive Insertion

FreeBSD uses the `devd` system to detect hardware changes such as inserting a drive into the hot swap bay.
Use `nc` to snoop the messages on `devd.pipe` and then **insert the drive** and wait for the `GEOM` `CREATE` messages to appear after a few seconds.
**Remove the drive** again and wait for the corresponding `GEOM` `DESTROY` messages, then press `Ctrl-C` to get the command prompt back.

{{< highlight txt >}}
# nc -U /var/run/devd.pipe
[...]
!system=GEOM subsystem=DEV type=CREATE cdev=gpt/backup
[...]
!system=GEOM subsystem=DEV type=DESTROY cdev=gpt/backup
^C
{{< /highlight >}}

The default configuration file for `devd` uses *directory* directives to indicate where the configuration files should be stored.
Use `grep` to see what the default configuration file expects and note that `/etc/devd` is the default location for config files created by the system.
User config files should be added to the secondary location, which is specified as `/usr/local/etc/devd`.

Consult the [man pages for `devd.conf`](https://www.freebsd.org/cgi/man.cgi?query=devd.conf) for more information about detecting hardware events
and the configuration file format.

{{< highlight txt >}}
# grep directory /etc/devd.conf
        # Each "directory" directive adds a directory to the list of
        directory "/etc/devd";
        directory "/usr/local/etc/devd";
{{< /highlight >}}

To configure `devd` to trigger specific actions when the drive is inserted and ejected, create a configuration file `/usr/local/etc/devd/backup.conf`
and add two `notify` statements, one each for drive insertion and ejection.
Inside each statement, add sub-statements to `match` the values given by the final `GEOM` `CREATE` and `GEOM` `DESTROY` events read from `devd.pipe`.
Finally, for each `notify` statement, create an `action` sub-statement that writes a message to the system log using `logger`.

{{< highlight txt >}}
# mkdir -p /usr/local/etc/devd

# ee /usr/local/etc/devd/backup.conf
[...]

# cat /usr/local/etc/devd/backup.conf
notify 100 {
    # insert backup drive
    match "system" "GEOM";
    match "subsystem" "DEV";
    match "type" "CREATE";
    match "cdev" "gpt/backup";
    action "logger Backup drive inserted";
};

notify 100 {
    # eject backup drive
    match "system" "GEOM";
    match "subsystem" "DEV";
    match "type" "DESTROY";
    match "cdev" "gpt/backup";
    action "logger Backup drive ejected";
};
{{< /highlight >}}

To test the configuration, restart the `devd` service and run `tail -f` to watch the system log.
Next, **insert** the drive and wait a few seconds for the message *Backup drive inserted* to appear in the log.
Then, **eject** the drive and wait a few seconds for the message *Backup drive ejected* to appear in the log.
Finally, press `Ctrl-C` to get the command prompt back.

{{< highlight txt >}}
# service devd restart
Stopping devd.
Waiting for PIDS: 540.
Starting devd.

# tail -f /var/log/messages
[...]
May 30 01:01:30 server ccammack[53176]: Backup drive inserted
[...]
May 30 01:01:41 server ccammack[53177]: Backup drive ejected
^C
{{< /highlight >}}

Once the basic `devd` functionality works, change the `action` sub-statements in the config file to call a shell script elsewhere in the system and restart the `devd` service again.

{{< highlight txt >}}
# ee /usr/local/etc/devd/backup.conf
[...]

# cat /usr/local/etc/devd/backup.conf
notify 100 {
    [...]
    action "/home/ccammack/bin/backup.sh --on-insert &";
};

notify 100 {
    [...]
    action "/home/ccammack/bin/backup.sh --on-eject &";
};

# service devd restart
Stopping devd.
Waiting for PIDS: 43499.
Starting devd.
{{< /highlight >}}

---

##### Create an Empty Backup Script

Create an empty backup script that can be called by `devd`.
To allow the script to automount the `geli` container when the drive is inserted, `echo` the backup drive's `geli` password into a text file in the same folder.
Spend a few moments in quiet reflection, doubting the wisdom of that decision.

{{< highlight txt >}}
# mkdir -p /home/ccammack/bin
# touch /home/ccammack/bin/backup.sh
# chmod +x /home/ccammack/bin/backup.sh
# echo "1234" > /home/ccammack/bin/backup.pass
# chmod go= /home/ccammack/bin/backup.pass
{{< /highlight >}}

##### Add Blinkenlights

I'm using a [blink(1)](/posts/display-server-notifications-using-a-blink1/) to give visual feedback during the backup process,
so the first part of the backup script will specify some colors and functions to start and stop a repeating pattern of flashes.
Calling the `blink1-tool` with `--playpattern <patternstr>` seems to be the easiest way to specify a repeating sequence of color flashes on the device.
Include an `exit` at the top of the script to prevent it from running by accident until it is ready for testing.

{{< highlight txt >}}
# ee /home/ccammack/bin/backup.sh
[...]

# cat /home/ccammack/bin/backup.sh
#!/bin/sh

exit

blink1="/usr/local/bin/blink1-tool"

OFF="#000000,0.25,0,"
RED="#ff0000,0.25,0,${OFF}"
GREEN="#00ff00,0.25,0,${OFF}"
BLUE="#0000ff,0.25,0,${OFF}"
ORANGE="#ff8000,0.25,0,${OFF}"
CYAN="#00ffff,0.25,0,${OFF}"
MAGENTA="#ff00ff,0.25,0,${OFF}"
WHITE="#ffffff,0.25,0,${OFF}"

blink_clear_pattern() {
  pkill blink1-tool ; $blink1 --off >/dev/null 2>&1
}

blink_play_pattern() {
  # usage: blink_play_pattern "$RED$GREEN$BLUE"
  blink_clear_pattern
  BEGIN="'0," ; END="#000000,1.5,0'"
  p=${BEGIN}${1}${END}
  $blink1 --playpattern "$p" -b 64 -m 0 -q >/dev/null 2>&1 &
}

[...]
{{< /highlight >}}

##### Outline the Backup Script

My backup procedure calls for swapping the drives every morning. When a fresh drive is inserted, the system will automatically mount it and begin making backups.
Backups will repeat periodically until early the next morning and then the system will automatically unmount the drive so it can be swapped for the next one.

To do this, the backup script relies on three functions: the *initialize* and *terminate* functions will mount and unmount the backup drive, preparing for backup and cleaning up afterwards;
the *backup* function performs the backup commands, sleeping for a while between each iteration.
All three functions redirect their command outputs to the system logger and emit one or more colors to *stdout* so the script can display the results of each step on the blink(1).
If any command inside the *initialize* function fails, the *backup* function will be skipped, but the *terminate* function will still run for cleanup.

The second half of the backup script implements these three functions plus a couple of helper functions.

{{< highlight txt >}}
# ee /home/ccammack/bin/backup.sh
[...]

# cat /home/ccammack/bin/backup.sh
[...]

log() {
  logger "$(basename "$0"): ${1}" >/dev/null 2>&1
}

run() {
  # redirect stdout and stderr from command to log and return $?
  output=$("$@" 2>&1) ; error=$? ; log "$output" ; return $error
}

initialize() {
  # abort if prerequisites are already running
  run geli status gpt/backup.eli ||
    run zpool list backup ||
    run service samba_server onestatus
  [ $? -eq 0 ] &&
    log "prerequisites are already running" &&
    printf %s $RED && return 2

  # TODO: find a better way to manage the geli password
  run geli attach -j /home/ccammack/bin/backup.pass /dev/gpt/backup &&
    run zpool import backup &&
    run service samba_server onestart
  [ $? -ne 0 ] &&
    printf %s $RED && return 1

  printf %s $GREEN && return 0
}

backup() {
  # repeat until 3AM tomorrow
  end=$(date -v+1d +"%Y%m%d030000")
  until [ "$(date +"%Y%m%d%H%M%S")" -gt "$end" ]
  do
    # perform backup commands
    blink_play_pattern "$BLUE"

    # sleep 15 minutes between backup cycles
    sleep 900
  done

  return 0
}

terminate() {
  run service samba_server onestop ; a=$?
  run zpool export backup ; b=$?
  run geli detach gpt/backup.eli ; c=$?
  error=$(( a + b + c ))
  [ $error -ne 0 ] && printf %s $RED && return 1
  printf %s $GREEN && return 0
}

if [ "$(pgrep -fl "$(basename "$0")" | grep -cv -e "^$$")" -gt 1 ]; then
  # abort if the script is already running
  procs="$(pgrep -fl "$(basename "$0")" | tr '\n' ' ')"
  log "already running as ${procs}"
  echo "$(basename "$0"): already running as ${procs}"
elif [ "$1" = "--on-insert" ]; then
  # process on-insert
  p=$(initialize) ; error=$?
  [ $error -gt 1 ] && blink_play_pattern "$p" && exit
  [ $error -gt 0 ] && p=${p}${RED} || p=${p}$(backup)
  p=${p}$(terminate)
  blink_play_pattern "$p"
elif [ "$1" = "--on-eject" ]; then
  # process on-eject
  blink_clear_pattern
fi
{{< /highlight >}}

---

##### Insert the Backup Drive

Assuming that the backup script contains an `exit` at the top, it should be safe to insert the backup drive now and mount it by hand to configure the first backup client.

{{< highlight txt >}}
# dmesg
[...]
ada1 at ahcich1 bus 0 scbus1 target 0 lun 0
ada1: <Hitachi HUS724040ALE641 MJAOA5F0> ATA8-ACS SATA 3.x device
ada1: Serial Number P4H3KUPC
ada1: 300.000MB/s transfers (SATA 2.x, UDMA6, PIO 8192bytes)
ada1: Command Queueing enabled
ada1: 3815447MB (7814037168 512 byte sectors)

# geli attach /dev/gpt/backup
Enter passphrase:

# geli status
[...]
gpt/backup.eli  ACTIVE  gpt/backup

# zpool import backup

# zpool list
NAME     SIZE  ALLOC   FREE  CKPOINT  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
backup  3.62T  2.43M  3.62T        -         -     0%     0%  1.00x  ONLINE  -
[...]
{{< /highlight >}}

##### Create a New User Group for the Backup Clients

Before adding the first backup client to the server, create a new user group named **backup** that they will all share to allow them to be treated as a group if needed.

{{< highlight txt >}}
# pw groupadd backup
{{< /highlight >}}

---

##### Configure Samba Access

Samba network shares are widely supported by Windows, Mac and Unix-like systems and are a good default backup destination unless there is a reason to use something else.
The *initialize* and *terminate* functions start and stop the `samba` service so other machines on the LAN can perform their backups over Samba.
Create a separate user account and Samba share for each client to prevent rogue clients from deleting each other's backups.

Find and install the latest version of Samba.

{{< highlight txt >}}
# pkg search samba
[...]
samba410-4.10.15               Free SMB/CIFS and AD/DC server and client for Unix

# pkg install -y samba410
[...]
{{< /highlight >}}

Create a user account for the first client, which is a Windows machine called **desktop** in this example.
Remember to add the user to the **backup** group.

{{< highlight txt >}}
# adduser
Username: desktop
Full name:
Uid (Leave empty for default):
Login group [desktop]:
Login group is desktop. Invite desktop into other groups? []: backup
Login class [default]:
Shell (sh csh tcsh git-shell nologin) [sh]:
Home directory [/home/desktop]:
Home directory permissions (Leave empty for default):
Use password-based authentication? [yes]:
Use an empty password? (yes/no) [no]:
Use a random password? (yes/no) [no]:
Enter password:
Enter password again:
Lock out the account after creation? [no]:
[...]
OK? (yes/no): yes
adduser: INFO: Successfully added (desktop) to the user database.
Add another user? (yes/no): no
Goodbye!
{{< /highlight >}}

Create a Samba share folder for the client on the backup drive, using the **same name** as the client's user name.

{{< highlight txt >}}
# zfs create backup/desktop

# zfs list
NAME                 USED  AVAIL  REFER  MOUNTPOINT
backup              1.67M  3.51T    88K  /backup
backup/desktop        88K  3.51T    88K  /backup/desktop
[...]

# chown -R desktop:desktop /backup/desktop

# ls -alg /backup
[...]
drwxr-xr-x   2 desktop  desktop   2 Jun  5 00:20 desktop
{{< /highlight >}}

Edit the Samba configuration file `/usr/local/etc/smb4.conf` to configure global and client share settings.

{{< highlight txt >}}
# ee /usr/local/etc/smb4.conf
[...]

# cat /usr/local/etc/smb4.conf
[global]
workgroup = WORKGROUP
netbios name = BACKUP
security = user
passdb backend = tdbsam
encrypt passwords = yes
max log size = 512
hosts allow = 192.168.1.
bind interfaces only = yes
socket options = TCP_NODELAY
writable = no

[desktop]
path = /backup/desktop
force user = desktop
writable = yes
{{< /highlight >}}

Use `pdbedit` to map the user account to the Samba database.
Samba uses a separate password database from the system, so `pdbedit` will ask for a new Samba-specific password.
I won't tell anyone if you decide to use the same password for both Samba and the user account.

{{< highlight txt >}}
# pdbedit --create --user=desktop
new password:
retype new password:
[...]
{{< /highlight >}}

To test Samba by itself, start the service and make sure the share appears on the client machine.
If needed, enable the checkbox that causes the system to remember the user name and password for future connections.

{{< highlight txt >}}
# service samba_server onestart
Performing sanity check on Samba configuration: OK
Starting nmbd.
Starting smbd.
Starting winbindd.
{{< /highlight >}}

{{< figure src="testing-samba.png" alt="Testing Samba">}}

Once Samba works properly, `stop` the service, `export` the backup pool and `geli detach`.
Edit the backup script again and comment out or remove the `exit` command at the top of the script.
Finally, physically eject the hard drive from the server.

{{< highlight txt >}}
# service samba_server onestop
[...]

# zpool export backup

# geli detach gpt/backup.eli

# ee /home/ccammack/bin/backup.sh
[...]

# head /home/ccammack/bin/backup.sh
#!/bin/sh

#exit

blink1="/usr/local/bin/blink1-tool"
[...]
{{< /highlight >}}

##### First Insert/Eject Test with Samba

At this point, the backup server should be ready to use, assuming that Samba is the only requirement.

Use `tail -f` again to keep an eye on the log and then **insert** the removable drive.
The backup script should auto-mount the drive and start Samba and then the blink(1) should begin flashing *blue* every few seconds.
The Samba share should be reachable from the client machine and will remain available until 3AM the following day so the client can save files to it.

After 3AM the following day, the drive will automatically dismount and the blink(1) will begin repeating three *green* flashes to indicate that the *initialize*, *backup* and *terminate* functions all ran without error.
Any color other than green indicates an error in the corresponding function, which will appear in the system log.
If there are **any flashing red lights, do not physically eject the drive** from the server until you have properly entered the commands to dismount the drive manually.

#### Cancel a Running Backup Script and Dismount Manually

Use `pkill` to interrupt the backup script and clear the blink(1) if needed. Remember to stop the `samba` service and dismount the drive in the OS before physically ejecting it from the case.

{{< highlight txt >}}
# pkill -f backup.sh ; pkill blink1-tool

# service samba_server onestop ; zpool export backup ; geli detach gpt/backup.eli
[...]

# service samba_server onestatus ; zpool list ; geli status
[...]
{{< /highlight >}}

#### Test a Windows Backup Over Samba Using restic

The example below shows what a test run would look like on Windows using client-side backup software like [restic](https://restic.net/):

1. Insert the backup drive into the server and wait for the LED to start flashing *blue*
1. Create a new temporary folder and file on the Windows machine
1. Create a new backup repository using restic
1. Back up the temporary folder using restic
1. Restore the temporary folder to a new location on the Windows machine using restic
1. Generate file hashes for the original and restored files to make sure they match

{{< highlight txt >}}
C:\Users\ccammack> mkdir C:\test & cd C:\test

C:\test> powershell $out = new-object byte[] 1048576; (new-object random).nextbytes($out); [io.file]::writeallbytes('random.bin', $out)

C:\test> restic init --repo \\BACKUP\desktop\restic-repo
[...]
created restic repository df32e0f92d at \\BACKUP\desktop\restic-repo
[...]

C:\test> restic -r \\BACKUP\desktop\restic-repo --verbose backup c:\test
[...]
snapshot 96f75888 saved

C:\test> restic -r \\BACKUP\desktop\restic-repo restore latest --target c:\test
[...]
restoring <Snapshot 96f75888 of [c:\test] at 2020-06-08 22:39:50.69799 -0700 PDT by desktop\ccammack@desktop> to c:\test

C:\test> certutil -hashfile c:\test\random.bin MD5
MD5 hash of file c:\test\random.bin:
16 93 6d 4b d5 2a b5 d6 82 1d e5 fd 37 67 4c fe
[...]

C:\test> certutil -hashfile c:\test\c\test\random.bin MD5
MD5 hash of file c:\test\c\test\random.bin:
16 93 6d 4b d5 2a b5 d6 82 1d e5 fd 37 67 4c fe
[...]
{{< /highlight >}}

---

#### Configure SFTP Access

Aside from Samba, another commonly supported way to send backups over the network is to use the *SSH File Transfer Protocol*.
This example sets up SFTP access for backups transferred from a laptop running Ubutntu 20.04.

To configure the backup server for SFTP, first create a new client user account called **laptop** and add it to the **backup** group.

{{< highlight txt >}}
# adduser
Username: laptop
Full name:
Uid (Leave empty for default):
Login group [laptop]:
Login group is laptop. Invite laptop into other groups? []: backup
Login class [default]:
Shell (sh csh tcsh git-shell nologin) [sh]:
Home directory [/home/laptop]:
Home directory permissions (Leave empty for default):
Use password-based authentication? [yes]:
Use an empty password? (yes/no) [no]:
Use a random password? (yes/no) [no]:
Enter password:
Enter password again:
Lock out the account after creation? [no]:
[...]
OK? (yes/no): yes
adduser: INFO: Successfully added (laptop) to the user database.
Add another user? (yes/no): no
Goodbye!
{{< /highlight >}}

Next, create a new directory for the **laptop** user account where it can store its backup files, using the **same name** as the user account.

{{< highlight txt >}}
# zfs create backup/laptop

# zfs list
NAME                 USED  AVAIL  REFER  MOUNTPOINT
backup              2.02M  3.51T    88K  /backup
backup/dekstop        88K  3.51T    88K  /backup/dekstop
backup/laptop         88K  3.51T    88K  /backup/laptop

# chown -R laptop:laptop /backup/laptop

# ls -alg /backup
[...]
drwxr-xr-x   2 laptop   laptop    2 Jun  9 23:18 laptop
{{< /highlight >}}

To ensure that the client can log into the backup server initially with a user name and password, edit `/etc/ssh/sshd_config` to set `PasswordAuthentication` to `yes` and then restart `sshd`.

{{< highlight txt >}}
# grep PasswordAuth /etc/ssh/sshd_config
PasswordAuthentication yes

# service sshd restart
[...]
{{< /highlight >}}

It should now be possible to log into the backup server from another machine using the **laptop** user name and password created earlier.

{{< highlight txt >}}
λ ssh laptop@192.168.1.112
laptop@192.168.1.112's password:
FreeBSD 12.1-RELEASE-p3 GENERIC

Welcome to FreeBSD!
[...]
{{< /highlight >}}

#### Test an Ubuntu Backup Over SFTP Using Duplicity

Some client backup software supports SFTP logins using user names and passwords.

For example, to test the default backup program in Ubuntu 20.04, open the application called **Backups**,
then select **Storage location** to specify the server and destination folder for the backup files.

In this example, the **Network Location** of the backup server is *ssh://192.168.1.112* and the destination **Folder** is the absolute path */backup/laptop/duplicity*.

{{< figure src="duplicity-set-storage-location.png" alt="Duplicity: Set storage location">}}

Next, select **Overview > Back Up Now...** to open the **Back Up** dialog.

{{< figure src="duplicity-back-up-now.png" alt="Duplicity: Back up now...">}}

On the **Back Up** dialog, enter the **Username** and **Password** and press the **Forward** button.
Select the option to encrypt the backup files with a password if desired and then press the **Forward** button again to start the backup.

{{< figure src="duplicity-user-pass.png" alt="Duplicity: Enter user name and password">}}
{{< figure src="duplicity-encrypt.png" alt="Duplicity: Encrypt with password">}}
{{< figure src="duplicity-starting-backup.png" alt="Duplicity: Starting backup">}}

#### Prefer SSH Keys Over Passwords

Some backup software might require the use of *SSH keys* rather than passwords, and it's generally better to use them rather than passwords if possible.
There are many options for SSH key management, but they all follow the same basic procedure:

1. Generate *SSH keys* on the client machine. This can be done in the terminal using `ssh-keygen` or in a graphical keyring manager like the *Password and Encryption Keys* tool in Ubuntu 20.04.
1. Copy the newly generated *public* SSH key from the client machine to the backup server and append it to the end of the user's *authorized_keys* file,
located under the user's home directory at `~/.ssh/authorized_keys`.
1. On the client machine, before connecting to the backup server, decrypt the *private* key into RAM by running `ssh-agent` and `ssh-add` in the terminal
or by relying on a graphical keyring manager to handle the whole procedure automatically.
1. The client can now securely connect to the backup server as needed without manually entering a user name and password each time.

See [SSH Mastery](https://www.amazon.com/SSH-Mastery-OpenSSH-PuTTY-Tunnels-ebook/dp/B079NL1L9K) for more information.

#### Restrict Each Client to Its Own Backup Directory

With either login method, once the client can automatically log into the SFTP server to save the backups, it's good to restrict each
client to its own directory on the server to prevent rogue clients from accidentally overwriting each other's backup files.

To do this, add a new `Match` block to the bottom of the *sshd* configuration that matches each member of the *backup* group
and uses `ChrootDirectory` and `ForceCommand` to restrict each of them to their own backup directory.

Restart the `sshd` service to put the new configuration into effect.

{{< highlight txt >}}
# ee /etc/ssh/sshd_config
[...]

# tail /etc/ssh/sshd_config
[...]
# restrict each <user> in the backup group to directory /backup/<user>
Match Group backup
        ChrootDirectory /backup/
        ForceCommand internal-sftp -d %u

# service sshd restart
Performing sanity check on sshd configuration.
Stopping sshd.
Waiting for PIDS: 846.
Performing sanity check on sshd configuration.
Starting sshd.
{{< /highlight >}}

After making this change, also change the destination **Folder** in the backup software to be relative (*duplicity*) rather than absolute (*/backup/laptop/duplicity*) .

{{< figure src="duplicity-set-relative-storage-location.png" alt="Duplicity: Set relative storage location">}}

-------

##### Configure the Backup Server to Back Itself Up

Since the backup server is running FreeBSD on ZFS, back up the backup server to itself using
[zfs-auto-snapshot](/posts/schedule-zfs-snapshots-using-zfs-auto-snapshot/) and
[zxfer](/posts/back-up-zfs-to-a-removable-drive-using-zxfer/).

First, set up [zfs-auto-snapshot](/posts/schedule-zfs-snapshots-using-zfs-auto-snapshot/) and then use `zfs list -t` to make sure snapshots are being generated.

{{< highlight txt >}}
# zfs list -t snapshot
NAME                                                         USED  AVAIL  REFER  MOUNTPOINT
zroot@zfs-auto-snap_monthly-2020-06-01-00h28                    0      -    88K  -
zroot@zfs-auto-snap_weekly-2020-06-07-00h14                     0      -    88K  -
zroot@zfs-auto-snap_daily-2020-06-11-00h07                      0      -    88K  -
[...]
{{< /highlight >}}

Next, install [zxfer](/posts/back-up-zfs-to-a-removable-drive-using-zxfer/) and create a new dataset for the host's backup files.

{{< highlight txt >}}
# pkg search zxfer
zxfer-1.1.7                    Easily and reliably transfer ZFS filesystems

# pkg install -y zxfer
[...]

# zfs create backup/`hostname`

# zpool list
[...]

# zfs list
[...]
{{< /highlight >}}

Finally, call `zxfer` from inside the script's `backup` function.
The script will count the number of attempts that complete without error using the `self_` variables and
will report that the overall process succeeded if at least one attempt worked during the daily cycle.

{{< highlight txt >}}
backup() {
  # track backup attempts and successes
  self_try=0 ; self_win=0

  # repeat until 3AM tomorrow
  end=$(date -v+1d +"%Y%m%d030000")
  until [ "$(date +"%Y%m%d%H%M%S")" -gt "$end" ]
  do
    # perform backup commands
    blink_play_pattern "$BLUE"
    self_try=$((self_try+1))
    run /usr/local/sbin/zxfer -dFkPv -g 376 \
      -I com.sun:auto-snapshot -R zroot backup/"$(hostname)"
    [ $? -eq 0 ] && self_win=$((self_win+1))

    # sleep 15 minutes between backup cycles
    sleep 900
  done

  log "$(hostname) successfully backed up ${self_win}/${self_try} times"
  [ $self_win -gt 0 ] && printf %s $GREEN || printf %s $RED

  return 0
}
{{< /highlight >}}

-------

##### Cancel Backup If the Drive Is Too Full

Use `zpool list` to check the used space on the backup drive and cancel before starting if it's over 80%.

{{< highlight txt >}}
backup() {
  # abort if the backup drive is too full (capacity: 0%-100%)
  [ "$(zpool list -H -o capacity backup | sed -r 's/%//g')" -gt 80 ] &&
    log "backup disk is low on space" && 
    printf %s $ORANGE && return 1
  
  # track backup attempts and successes
  self_try=0 ; self_win=0
  
  [...]
}
{{< /highlight >}}

-------

##### Scrub for ZFS Errors

To scrub the backup drive for ZFS errors, add two new helpers and call them from `backup`.

{{< highlight txt >}}
scrub_start() {
  # run zfs scrub on Saturdays
  [ "$(date +"%u")" -eq 6 ] ; scrub=$?
  [ "$scrub" -eq 1 ] &&
    log "starting zfs scrub backup" &&
    run zpool scrub backup
  return $scrub
}

scrub_wait() {
  blink_play_pattern "$WHITE"
  while zpool status backup |
      grep -E " scan: +scrub in progress since " >/dev/null 2>&1; do
    sleep 60
  done
  blink_clear_pattern
  results=$(zpool status backup |
            sed -n 's/^.*scan: scrub repaired *//p' |
            sed -n 's/ *in .*with */;/p' |
            sed -n 's/ *errors on .*$//p')
  repaired=$(echo "$results" | cut -d ';' -f1)
  errors=$(echo "$results" | cut -d ';' -f2)
  [ "$repaired" -eq 0 ] && [ "$errors" -eq 0 ] &&
    printf %s $GREEN && return 0

  log "zfs scrub repaired: ${repaired} with errors: ${errors}" &&
  printf %s $ORANGE && return 0
}

backup() {
  # abort if the backup drive is too full (capacity: 0%-100%)
  [ "$(zpool list -H -o capacity backup | sed -r 's/%//g')" -gt 80 ] &&
    log "backup disk is low on space" && 
    printf %s $ORANGE && return 1

  # start zfs scrub
  scrub_start ; scrub=$?

  # track backup attempts and successes
  self_try=0 ; self_win=0

  [...]
 
  # wait for zfs scrub to finish
  [ "$scrub" -eq 0 ] && scrub_wait || printf %s $GREEN

  return 0
}
{{< /highlight >}}


