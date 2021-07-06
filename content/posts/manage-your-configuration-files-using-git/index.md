---
title: "Manage Your Configuration Files Using Git"
date: 2021-03-20T19:45:53-07:00
tags: ["dotfiles", "git", "Windows", "Ubuntu"]
---

There are [dozens of popular ways](https://dotfiles.github.io/) to manage [user dotfile configurations](https://github.com/webpro/awesome-dotfiles),
but I think the simplest might be to just use a [bare git repository](https://www.atlassian.com/git/tutorials/dotfiles) for each platform.

<!--more-->

This approach is easy to set up, requires no additional tools beyond `git` and allows any file below the user's home directory to be tracked.

For example, to share my emacs config between Windows and Linux, I created a bare git repo called `.dot-all`, which I use to manage any configuration that is to be shared between platforms.
I also have a Windows-only repo (`.dot-win`) and one for configs shared between Linux and FreeBSD (`.dot-unix`)

{{< highlight txt >}}
C:\Users\ccammack
λ cd %USERPROFILE%

C:\Users\ccammack
λ git init --bare .dot-all
Initialized empty Git repository in C:/Users/ccammack/.dot-all/

C:\Users\ccammack
λ ls .dot-all\
config  description  HEAD  hooks/  info/  objects/  refs/
{{< /highlight >}}

To manage files in one directory using a bare repository in a different directory, `git` relies on the `--work-tree` and `--git-dir` parameters to specify those two directories for each `git` operation.

To ensure that both of these parameters are specified properly when working with configuration files, [most articles](https://www.atlassian.com/git/tutorials/dotfiles) recommend using a shell alias to create an alternate version of the `git` command that assigns these parameters their proper values.

However, I think a [less-common approach](https://dev.to/bowmanjd/store-home-directory-config-files-dotfiles-in-git-using-bash-zsh-or-powershell-the-bare-repo-approach-35l3) works better;
rather than creating a shell alias for each bare repo, create a separate git [alias](https://git-scm.com/book/en/v2/Git-Basics-Git-Aliases) for each bare repo instead.

For example, to run `git` commands on the bare `.dot-all` repo, create a git alias that sets the parameters to the correct values.

{{< highlight txt >}}
C:\Users\ccammack
λ git config --global --replace-all alias.dot-all "!git --git-dir=C:\\Users\\ccammack\\.dot-all --work-tree=C:\\Users\\ccammack"

C:\Users\ccammack
λ git config --global --list
[...]
alias.dot-all=!git --git-dir=C:\\Users\\ccammack\\.dot-all --work-tree=C:\\Users\\ccammack
{{< /highlight >}}

Now, to operate on files managed by the `.dot-all` repo, use `git dot-all` rather than `git.`

{{< highlight txt >}}
C:\Users\ccammack
λ git dot-all status
[...]
On branch master

No commits yet

Untracked files:
  (use "git add <file>..." to include in what will be committed)
		[...]

nothing added to commit but untracked files present (use "git add" to track)
{{< /highlight >}}

Home directories usually have a lot of files in them and the default `status` command will show all of them.
To fix this, change the `local` configuration for the repo to disable `showUntrackedFiles` and try again.

{{< highlight txt >}}
C:\Users\ccammack
λ git dot-all config --local status.showUntrackedFiles no

C:\Users\ccammack
λ git dot-all config --list
[...]
status.showuntrackedfiles=no

C:\Users\ccammack
λ git dot-all status
On branch master

No commits yet

nothing to commit (create/copy files and use "git add" to track)
{{< /highlight >}}

Now that the setup is complete, add and commit the files that are to be tracked using the `.dot-all` repo.

{{< highlight txt >}}
C:\Users\ccammack
λ git dot-all add .doom.d\*

C:\Users\ccammack
λ git dot-all status
On branch master

No commits yet

Changes to be committed:
  (use "git rm --cached <file>..." to unstage)
        new file:   .doom.d/config.el
        new file:   .doom.d/init.el
        new file:   .doom.d/packages.el

Untracked files not listed (use -u option to show untracked files)

C:\Users\ccammack
λ git dot-all commit -m "Add emacs configuration."
[master (root-commit) a0508e6] Add emacs configuration.
 3 files changed, 310 insertions(+)
 create mode 100644 .doom.d/config.el
 create mode 100644 .doom.d/init.el
 create mode 100644 .doom.d/packages.el

C:\Users\ccammack
λ git dot-all remote add origin http://git.ccammack.com:3000/ccammack/dot-all.git

C:\Users\ccammack
λ git dot-all push -u origin master
[...]
{{< /highlight >}}

---

To replicate the shared configuration onto another machine, `git clone` the repo using the `--bare` option, set up the global `dot-all` git alias and locally disable `showUntrackedFiles`.

For example, the following commands replicate the `.dot-all` repo and its files onto an Ubuntu desktop machine.

{{< highlight txt >}}
ccammack@ubuntu:~$ cd $HOME

ccammack@ubuntu:~$ git clone --bare http://git.ccammack.com:3000/ccammack/dot-all.git $HOME/.dot-all
Cloning into bare repository '/home/ccammack/.dot-all'...
remote: Enumerating objects: 6, done.
remote: Counting objects: 100% (6/6), done.
remote: Compressing objects: 100% (5/5), done.
remote: Total 6 (delta 0), reused 0 (delta 0), pack-reused 0
Unpacking objects: 100% (6/6), 6.17 KiB | 6.17 MiB/s, done.

ccammack@ubuntu:~$ git config --global --replace-all alias.dot-all '!git --git-dir=$HOME/.dot-all --work-tree=$HOME'

ccammack@ubuntu:~$ git config --global --list
alias.dot-all=!git --git-dir=$HOME/.dot-all --work-tree=$HOME

ccammack@ubuntu:~$ git dot-all config --local status.showUntrackedFiles no

ccammack@ubuntu:~$ git dot-all config --list
[...]
status.showuntrackedfiles=no

ccammack@ubuntu:~$ git dot-all checkout
error: The following untracked working tree files would be overwritten by checkout:
        .doom.d/config.el
        .doom.d/init.el
        .doom.d/packages.el
Please move or remove them before you switch branches.
Aborting
{{< /highlight >}}

The `checkout` will fail if local copies of the repository files already exist. If the local changes should be preserved, copy them to a temporary location, perform the `checkout` again and
then overwrite the checked out files with the temporary copies.

If the local changes are not important, simply overwrite them with `checkout -f`.

{{< highlight txt >}}
ccammack@ubuntu:~$ git dot-all checkout -f

ccammack@ubuntu:~$ git dot-all log
commit a0508e6d89880d744ec7122cecb1da27ddc62488 (HEAD -> master)
Author: ccammack <ccammack@example.com>
Date:   Sun Jun 20 20:07:47 2021 -0700

    Add emacs configuration.
{{< /highlight >}}
