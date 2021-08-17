---
title: "Schedule Street Cleaning Reminders in Org-mode"
date: 2018-10-27T23:04:21-07:00
tags: ["Emacs", "Org-mode"]
---

[comment]: # ( https://stackoverflow.com/questions/53078035/diary-float-query-for-emacs-diary											)
[comment]: # ( https://stackoverflow.com/questions/16946220/emacs-org-mode-how-to-set-date-and-time-on-the-first-wednesday-of-month		)

[Org-mode](https://orgmode.org/) can parse [Emacs diary entries](https://www.gnu.org/software/emacs/manual/html_node/emacs/Sexp-Diary-Entries.html)
that represent concepts such as the *4th Tuesday of the month*, which makes it possible to schedule agenda reminders for periodic events like street sweeping.

<!--more-->

For example, the `diary-float` entry below represents the *4th Tuesday of each month*:

* the value **t** is a placeholder that represents *all months*
* the value **2** represents *Tuesday* (Sunday=0, Monday=1, ..., Saturday=6)
* the value **4** represents *4th*

{{< highlight txt >}}
%%(diary-float t 2 4) Today is the 4th Tuesday of the month
{{< /highlight >}}

Use two separate entries to represent both the *2nd and 4th Tuesdays of each month*.

{{< highlight txt >}}
%%(diary-float t 2 2) Today is the 2nd Tuesday of the month
%%(diary-float t 2 4) Today is the 4th Tuesday of the month
{{< /highlight >}}

To get a reminder one day ahead of time, wrap the diary-float expression with `diary-remind` and specify **-1** days as the last parameter.

{{< highlight txt >}}
%%(diary-remind '(diary-float t 2 4) -1) Monday before the 4th Tuesday
{{< /highlight >}}

In my case, the city cleans the nearby blocks on the 2nd and 4th Tuesday-Friday of each month.
Putting it all together, the street cleaning section of my org file looks like this when the car is parked on the *Friday* side of the street.

{{< highlight txt >}}
* Street Cleaning
** COMMENT Move the car before Tuesday at 10
   <%%(diary-remind '(diary-float t 2 2) -1)>
   <%%(diary-remind '(diary-float t 2 4) -1)>
** COMMENT Move the car before Wednesday at 11
   <%%(diary-remind '(diary-float t 3 2) -1)>
   <%%(diary-remind '(diary-float t 3 4) -1)>
** COMMENT Move the car before Thursday at 12
   <%%(diary-remind '(diary-float t 4 2) -1)>
   <%%(diary-remind '(diary-float t 4 4) -1)>
** TODO Move the car before Friday at 11
   <%%(diary-remind '(diary-float t 5 2) -1)>
   <%%(diary-remind '(diary-float t 5 4) -1)>
{{< /highlight >}}

Any entry marked as a **COMMENT** will not appear in the agenda.
Each time the car moves to a new location, toggle the COMMENT off for the new location's entry using `Ctrl-C ;` and toggle it on for the rest of the entries using `Ctrl-C ;`.

Marking the new location's entry as **TODO** isn't required, but it makes the reminder stand out more in the agenda view.

{{< figure src="agenda-view.png" caption="Street cleaning reminders as they appear in the agenda">}}

Although this approach requires one to manually toggle the COMMENT on and off to hide and show entries in the agenda, it should be possible to write
[custom code](https://stackoverflow.com/questions/13555385/org-mode-how-to-schedule-repeating-tasks-for-the-first-saturday-of-every-month/13755627#13755627)
that combines the standard Emacs state transitions with diary entries to accomplish the same thing in a more elegant fashion.
