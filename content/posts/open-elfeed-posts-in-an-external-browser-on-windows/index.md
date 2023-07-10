---
title: "Open Elfeed Posts in an External Browser on Windows"
date: 2023-06-17T12:35:00-07:00
tags: ["Emacs", "RSS", "Windows"]
---

[Elfeed](https://github.com/skeeto/elfeed) includes a command to open feed items an an external browser,
but doing so causes the browser to steal the focus from Emacs.
I don't know of any way to force the browser to remain in the background on Windows,
but it's simple enough to configure Emacs to immediately steal the focus back using an external AutoHotkey script.

First, install [AutoHotkey](https://www.autohotkey.com/) from the main site or by using [Chocolatey](https://community.chocolatey.org/packages?q=autohotkey),
then write a short AutoHotkey script to wait for Emacs to lose focus and then immediately steal it back.

{{< highlight txt >}}
C:\Users\ccammack\.config\doom
λ cat restore-focus.ahk

#SingleInstance force

; Wait for Emacs to lose focus and steal it back

; Get Emacs HWND
hwnd := WinExist("ahk_class Emacs")
If (hwnd == 0x0)
{
    MsgBox, 16, Error, Could not find a running Emacs instance
    Exit, 255
}

; Wait for Emacs to lose focus and steal it back
WinWaitNotActive, ahk_id %hwnd%
WinActivate, ahk_id %hwnd%
{{< /highlight >}}

Next, in the Emacs configuration, write a short function to run the external script and then advise the *elfeed-search-browse-url* function to call it *before* opening each item.

{{< highlight txt >}}

C:\Users\ccammack\.config\doom
λ cat config.el

[...]

;; launch an ahk script to restore focus before opening an elfeed link
(after! elfeed
  (defun cc/restore-focus (&rest ignored)
    (if (eq system-type 'windows-nt)
        (call-process-shell-command (concat doom-user-dir "restore-focus.ahk &") nil 0)))
  (advice-add 'elfeed-search-browse-url :before #'cc/restore-focus))

[...]

{{< /highlight >}}

There will be a brief cursor flash in Emacs as the focus bounces back and forth when opening an item.
It might be possible to break the script by opening many items in a large region at once, but in normal use it works well enough.
