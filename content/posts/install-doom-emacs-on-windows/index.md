---
title: "Install Doom Emacs on Windows"
date: 2021-07-24T21:35:54-07:00
tags: ["Emacs", "Windows", "Chocolatey"]
---

I recently noticed that [Xah had trouble](https://www.youtube.com/watch?v=3qDoHd-6NOQ) installing [Doom Emacs](https://github.com/hlissner/doom-emacs) on Windows.
Here's how to install it on Windows 7, 8 and 10+ using [Chocolatey](https://chocolatey.org/) in case it helps someone else.

<!--more-->

For Windows 10+, using [WSL and Ubuntu](https://github.com/hlissner/doom-emacs/blob/develop/docs/getting_started.org#with-wsl--ubuntu-1804-lts) makes the
installation similar to other operating systems, but if you don't want to use WSL or are running an earlier version of Windows, Chocolatey works the same for all of them.

##### Install Chocolatey

To install [Chocolatey](https://chocolatey.org/) itself, press the *Windows* key to open the *Start* menu,
type `power` on the keyboard to filter the options and then right-click *Windows PowerShell* to run it using the *Administrator* account.

{{< figure src="run-powershell-as-administrator.png" alt="Run PowerShell as Administrator">}}

Chocolatey's [web installation](https://chocolatey.org/install#individual) requires at least PowerShell v3.0+ and .NET Framework v4.5+, so check the installed versions and upgrade if needed.

{{< highlight txt >}}
Windows PowerShell
Copyright (C) 2012 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32> Get-Host | Select-Object Version

Version
-------
3.0


PS C:\Windows\system32> Get-ChildItem 'HKLM:\SOFTWARE\Microsoft\NET Framework Setup\NDP' -Recurse | Get-ItemProperty -Name version -EA 0 | Where { $_.PSChildName -Match '^(?!S)\p{L}'} | Select PSChildName, Version

PSChildName                                                 Version
-----------                                                 -------
v2.0.50727                                                  2.0.50727.5420
v3.0                                                        3.0.30729.5420
Windows Communication Foundation                            3.0.4506.5420
Windows Presentation Foundation                             3.0.6920.5011
v3.5                                                        3.5.30729.5420
Client                                                      4.8.03761
Full                                                        4.8.03761
Client                                                      4.0.0.0
{{< /highlight >}}

Install Chocolatey using the command given in [the instructions](https://chocolatey.org/install#individual) and then close the console with `exit`.

{{< highlight txt >}}
Windows PowerShell
Copyright (C) 2012 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32> Set-ExecutionPolicy Bypass -Scope Process -Force; [System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072; iex ((New-Object System.Net.WebClient).DownloadString('https://community.chocolatey.org/install.ps1'))
[...]
Getting Chocolatey from https://community.chocolatey.org/api/v2/package/chocolatey/0.12.1.
[...]
WARNING: It's very likely you will need to close and reopen your shell before you can use choco.
[...]
You may need to shut down and restart powershell and/or consoles first prior to using choco.
[...]
PS C:\Windows\system32> exit
{{< /highlight >}}

Close the PowerShell console and restart it again using the *Administrator* account.

##### Install Emacs

In the PowerShell *Administrator* console, use Chocolatey to install the Windows-native [packages](https://community.chocolatey.org/packages) for `git`, `emacs`, `ripgrep`, `llvm` and `fd`.
I also like to install `hunspell` to get spell-checking support. Use `exit` to close the PowerShell console afterwards.

{{< highlight txt >}}
Windows PowerShell
Copyright (C) 2012 Microsoft Corporation. All rights reserved.

PS C:\Windows\system32> choco install git emacs ripgrep llvm fd hunspell.portable -y
Chocolatey v0.12.1
Installing the following packages:
git;emacs;ripgrep;llvm;fd;hunspell.portable
[...]
Installed:
 - hunspell.portable v1.7.0
 - emacs v27.2.0.20210423
 - git.install v2.35.1.2
 - ripgrep v13.0.0.20210621
 - llvm v13.0.1
 - fd v8.3.2
 - git v2.35.1.2
[...]
PS C:\Windows\system32> exit
{{< /highlight >}}

##### Install Doom

To install the Doom package, use the *Start* menu to run the Windows `cmd` prompt as the regular user and then `cd` to the appropriate directory depending on the Windows version.

On Windows 7, the configuration files should be installed into the user's `%APPDATA%` directory.

{{< highlight txt >}}
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

C:\Users\ccammack>echo %APPDATA%
C:\Users\ccammack\AppData\Roaming

C:\Users\ccammack>cd %APPDATA%

C:\Users\ccammack\AppData\Roaming>
{{< /highlight >}}

On Windows 8+, the configuration files should be installed into the user's `%USERPROFILE%` directory.

{{< highlight txt >}}
Microsoft Windows [Version 6.2.9200]
(c) 2012 Microsoft Corporation. All rights reserved.

C:\Windows\System32>echo %USERPROFILE%
C:\Users\ccammack

C:\Windows\System32>cd %USERPROFILE%

C:\Users\ccammack>
{{< /highlight >}}

Next, run `git clone` and `doom install` to clone and install the required configuration packages,
approximately following the installation [instructions](https://github.com/hlissner/doom-emacs#install),
making adjustments for the different treatment of the `~` character and path separators in Windows.

During the installation, answer `y` to the prompt *Generate an envvar file?*.

Note any error messages and instructions that appear at the end of the installation output.

{{< highlight txt >}}
Microsoft Windows [Version 6.1.7601]
Copyright (c) 2009 Microsoft Corporation.  All rights reserved.

[...]

C:\Users\ccammack\AppData\Roaming>git clone --depth 1 https://github.com/hlissner/doom-emacs .emacs.d
Cloning into '.emacs.d'...
remote: Enumerating objects: 1090, done.
remote: Counting objects: 100% (1090/1090), done.
remote: Compressing objects: 100% (1036/1036), done.
remote: Total 1090 (delta 29), reused 566 (delta 15), pack-reused 0
Receiving objects: 100% (1090/1090), 1007.70 KiB | 4.16 MiB/s, done.

Resolving deltas: 100% (29/29), done.

C:\Users\ccammack\AppData\Roaming>.emacs.d\bin\doom install
> Installing straight...
  > Cloning use-package...
> Executing 'doom install' with Emacs 27.2 at 2021-24-07 09:50:08
[...]
Generate an envvar file? (see `doom help env` for details) (y or n) y
[...]
Doom cannot install all-the-icons' fonts on Windows!
You'll have to do so manually:
   1. Launch Doom Emacs
   2. Execute 'M-x all-the-icons-install-fonts' to download the fonts
   3. Open the download location in windows explorer
   4. Open each font file to install them

Finished! Doom is ready to go!

But before you doom yourself, here are some things you should know:

1. Don't forget to run 'doom sync', then restart Emacs, after modifying
   ~/.doom.d/init.el or ~/.doom.d/packages.el.

   This command ensures needed packages are installed, orphaned packages are
   removed, and your autoloads/cache files are up to date. When in doubt, run
   'doom sync'!

2. If something goes wrong, run `doom doctor`. It diagnoses common issues with
   your environment and setup, and may offer clues about what is wrong.

3. Use 'doom upgrade' to update Doom. Doing it any other way will require
   additional steps. Run 'doom help upgrade' to understand those extra steps.

4. Access Doom's documentation from within Emacs via 'SPC h d h' or 'C-h d h'
   (or 'M-x doom/help')

Have fun!

Finished in 8m51s
{{< /highlight >}}

##### Run Doom Emacs

The Windows executable for Emacs is `runemacs.exe` and is located in `C:\ProgramData\chocolatey\bin\runemacs.exe` when installed via Chocolatey.

Either create a new shortcut with the `Target:` set to `C:\ProgramData\chocolatey\bin\runemacs.exe` or run it directly from the console.

{{< figure src="doom-emacs-shortcut.png" alt="Doom Emacs Shortcut">}}

{{< figure src="doom-emacs-first-run.png" alt="Doom Emacs First Run">}}

##### Install Required Fonts

The little orange square at the bottom left corner of the screen that says `E897` indicates that the correct font for that character has not been installed.

To download the missing fonts inside Emacs, press the *space bar* followed by the `:` character and enter the search text `install fo`.
In the documentation style for Doom Emacs, this command would look like `SPC : install fo`.

Entering this command shortcut inside Emacs will begin a so-called *Meta-x* command (`M-x`) and find the command matching the search string `install fo`.
Doing this will cause Emacs to suggest the command `all-the-icons-install-fonts`, which is the correct command to run.

Press `Enter` to run the selected command.

{{< figure src="doom-emacs-install-fonts.png" alt="Doom Emacs Install Fonts">}}

Emacs will then ask for confirmation with the prompt *This will download and install fonts, are you sure you want to do this? (y or n)*.

Enter `y` to continue.

{{< figure src="doom-emacs-confirm-install-fonts.png" alt="Doom Emacs Confirm Install Fonts">}}

Next, Emacs will ask for a directory where the fonts should be downloaded.
Use the up and down arrow keys or enter filter text to select matching directories from the list.

If needed, enter `../` to move up one directory and list the directories there.

{{< figure src="doom-emacs-request-download-directory.png" alt="Doom Emacs Request Download Directory">}}

For example, if the current directory is `c:/ProgramData/chocolatey/bin`, type `../../Dow` to go up two directories and select the Windows Downloads directory.

Press `Enter` to finalize the selection and download the required fonts.

{{< figure src="doom-emacs-select-download-directory.png" alt="Doom Emacs Select Download Directory">}}

{{< figure src="doom-emacs-finished-download.png" alt="Doom Emacs Finished Download">}}

Press the *space bar* followed by `q` `q` (`SPC q q`) to exit Emacs, then open the Downloads folder and right-click each font file to install them.

{{< figure src="doom-emacs-install-downloaded-windows-fonts.png" alt="Doom Emacs Install Downloaded Windows Fonts">}}

Restart Emacs and note that the little orange square previously at the bottom left corner now appears as a lock, indicating that the required fonts have been installed.

{{< figure src="doom-emacs-corrected-fonts.png" alt="Doom Emacs Corrected Fonts">}}
