---
title: "Configure a Fast Org Backlog Function in Doom Emacs"
date: 2025-08-16T12:33:54-04:00
tags: ["Emacs", "Org"]
---

My `projects.org` file tends to grow, but I'm pretty happy with its overall outline structure,
so I use this function to quickly refile the current subtree into the `someday.org` backlog file.

<!--more-->

Doing it this way is faster than running a standard `org-refile` operation,
and it automatically copies ancestor headings as needed to create the same outline path in the destination.
That way, if I ever go through the backlog file and find something to work on,
running the same command will bounce it back into the projects file in its original location.

In Doom, I have it mapped to `SPC m r b`.

{{< highlight elisp >}}
(setq my/org-refile-backlog-destinations
      (list (file-name-concat org-directory "projects.org")
            (file-name-concat org-directory "someday.org")))

(defun my/org-refile-backlog (&optional destinations)
  "Refile the current subtree to another file with a copy of its ancestor path.
The user will select the destination file from the list of filename strings
defined by DESTINATIONS or by those in my/org-refile-backlog-destinations"
  (interactive)
  (save-excursion
    (save-restriction
      (org-save-outline-visibility t
        (widen)
        (outline-show-all)
        (unless (derived-mode-p 'org-mode)
          (error "Buffer must be Org mode"))
        (unless (org-at-heading-p)
          (ignore-errors (org-back-to-heading t)))
        (unless (org-at-heading-p)
          (error "Point must be on a heading"))
        (let* ((src-buffer (current-buffer))

               ;; collect the point and heading for each ancestor
               (src-outline-lines (progn
                                    (save-excursion
                                      (let* ((result nil))
                                        (while (org-up-heading-safe)
                                          (let* ((head (org-get-heading t t t t))
                                                 (name (substring-no-properties (or head ""))))
                                            (push (list (pos-bol) name) result)))
                                        result))))

               ;; ask the user for the destination file
               (dst-files (seq-filter
                           (lambda (opt)
                             (not (string-equal opt (buffer-file-name))))
                           (or destinations
                               my/org-refile-backlog-destinations)))
               (dst-file (cond
                          ((null dst-files)
                           (error "No valid refile destinations available")
                           nil)
                          ((= 1 (length dst-files))
                           (car dst-files))
                          (t
                           (completing-read "Destination: " dst-files nil t))))
               (dst-buffer (ignore-errors
                             (or (find-buffer-visiting dst-file)
                                 (find-file-noselect dst-file)))))

          (when dst-buffer
            (let* ((path (list dst-file))
                   (mpos nil))

              ;; create missing destination headings
              (seq-doseq (line src-outline-lines)
                (let* ((pos (nth 0 line))
                       (name (nth 1 line))
                       (head (mapconcat 'identity path "/"))
                       (rfloc (list head dst-file nil mpos))
                       (olp (append path (list name))))

                  (when (null (ignore-errors (org-find-olp olp)))
                    (with-current-buffer src-buffer
                      (save-excursion
                      (goto-char pos)
                      (kill-ring-save (pos-bol) (pos-eol))))

                    (with-temp-buffer
                      (org-mode)
                      (yank)
                      (org-refile nil nil rfloc)))

                  (setq path (append path (list name)))
                  (setq mpos (marker-position (ignore-errors (org-find-olp path))))))

              ;; refile to destination
              (let* ((head (mapconcat 'identity path "/"))
                     (rfloc (list head dst-file nil mpos)))
                (with-current-buffer src-buffer
                  (org-refile nil nil rfloc))))))))))

(map! :after org
      (:map org-mode-map
       :localleader
       (:prefix ("r" . "+refile")
        :desc "Backlog" "b" #'my/org-refile-backlog)))
{{< /highlight >}}
