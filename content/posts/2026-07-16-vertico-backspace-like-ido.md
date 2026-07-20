---
title: "Making backspace in vertico behave like ido"
slug: vertico-backspace-like-ido
date: 2026-07-16
tags: [emacs]
summary: "After many years of ido muscle memory, the one thing that kept biting me after switching to vertico was backspace: ido's file history drops you into a directory, so backspace goes up one directory at a time right away, while vertico's recalls the full file path — and backspace first has to delete the file name character by character.  A two-dozen-line replacement for vertico-directory-delete-char restores the ido feel — and improves on it."
description: "Restoring ido's backspace behavior — delete the whole path component — in vertico."
---

My Emacs file-finding habits go back a long way.  I lived with vanilla `find-file` until I discovered ido-mode, and that was a big step up in UX.  Later I moved most completion to [helm](https://github.com/emacs-helm/helm) — but `C-x C-f` stayed on ido the whole time: when `helm-mode` takes over the standard completion commands, `helm-completing-read-handlers-alist` lets you add per-command exceptions, and mine said `(find-file . ido)`.  Last December I finally rewrote my config around the modern stack — [vertico](https://github.com/minad/vertico), [orderless](https://github.com/oantolin/orderless), [consult](https://github.com/minad/consult) — and the [vertico wiki](https://github.com/minad/vertico/wiki#make-vertico-and-vertico-directory-behave-more-like-ivyido) even has a recipe for making it behave more like ido.  Most of my muscle memory survived the move.

Except backspace.

Here is the difference.  In `ido-find-file`, `M-p`/`M-n` move you between the *directories* you previously opened files in: you land in a directory, and from there each backspace takes you one level up.  It is navigation, not text editing.  Vertico's `M-p` instead recalls the full path of a previously opened *file*, inserted into the minibuffer as ordinary editable text — which I actually like more: the file I want is often one I already opened.  And when it is not, a recalled path is still my starting point.  In a project I know well, `project-find-file` is only half of how I move around; just as often I navigate relative to a file I touched recently — a habit formed years before project.el or projectile existed: recall it, delete its name, go a level or two up or into a sibling directory.  Which means pressing backspace — and this is where it breaks.  Ordinary text gets ordinary editing: backspace deletes one character, except right after a slash, where `vertico-directory-delete-char` removes a whole path component.  So I hold backspace to erase the long file name; it eats the name character by character — and the moment it crosses the slash, the still-held key starts deleting whole directories.  Release a beat too late and half the path is gone.

The fix is a small replacement for `vertico-directory-delete-char`: when the point is at the end of the minibuffer and the input as a whole is an existing file path, treat backspace as "delete the last component" — exactly what my fingers expect after ido.  This assumes the `vertico-directory` extension (shipped with vertico) is loaded, as in the wiki recipe above.

```elisp
(defun my/-vertico-directory-delete-char (n)
  "Delete N directories or characters before point.
Like `vertico-directory-delete-char', but when point is at the end
of the minibuffer and the input is an existing file path, delete the
entire path component instead of a single character."
  (interactive "p")
  (cond
   ;; Active region: let `delete-backward-char' handle it.
   ((and (use-region-p) delete-active-region (= n 1))
    (with-no-warnings (delete-backward-char n)))
   ;; At a directory boundary, try to go up a directory.  Also at end of
   ;; input, if the whole path is an existing local file, treat DEL as
   ;; directory-up (delete the last path component).  Skip remote paths to
   ;; avoid blocking on network round-trips.
   ((and (or (eq (char-before) ?/)
             (and (eobp)
                  (let ((mb (minibuffer-contents-no-properties)))
                    (and (not (file-remote-p mb))
                         (file-exists-p mb)))))
         (vertico-directory-up n)))
   ;; Otherwise, delete a character.
   (t
    (with-no-warnings (delete-backward-char n)))))
```

And the binding, replacing the stock one:

```elisp
(keymap-set vertico-map "DEL" #'my/-vertico-directory-delete-char)
```

The `file-exists-p` check is the heart of it, not a safety net.  It is what distinguishes a path you just recalled from history (it exists — backspace removes the whole name) from a name you are typing right now (it does not exist yet — backspace fixes the typo, character by character).  Orderless filter input like `init compl` is likewise unaffected: such input rarely names an existing file.  Two more details: remote paths are skipped on purpose, because `file-exists-p` on a TRAMP path is a network round-trip, and backspace is the last key that should ever block on the network; and mid-path behavior is stock — after a slash, backspace deletes a whole path component; inside a name, a single character.

One honest limitation: while erasing a typed name character by character, you can land on an existing prefix (deleting `Makefile2` and reaching `Makefile`), and the next backspace deletes a whole component.  In practice, I have yet to be bitten by it.

The result: recall a path from history, press backspace once — the file name is gone; every further press climbs one directory, deliberately.  No more holding the key down, no more runaway deletion, and years of finger habits work again.

History recall is where the stock behavior hurt the most, but the same rule pays off in ordinary navigation too: while the name you are typing matches no existing file, backspace edits it character by character; the moment the input is a complete file name — typed in full or completed with TAB — a single backspace removes it whole.  Funnily enough, ido is worse here: whenever there is text in the minibuffer, it deletes character by character.  My ido habits, it turns out, were really about directory-oriented history — and where the two behaviors differ, I like the new one more.

To be clear, none of this is a critique of the stock `vertico-directory` behavior — my replacement is simply what fits my habits better, and I hope it may be useful if your years of ido left you with the same directory-oriented reflexes.

Here is [the commit](https://github.com/vshender/emacs-config/commit/0516e63138c3abcc3b1e30c3352bebde34597d23) in my config, with the `use-package` wiring around it.

**Update (2026-07-20):** Daniel Mendler, the author of vertico, [responded](https://github.com/vshender/emacs-config/commit/0516e63138c3abcc3b1e30c3352bebde34597d23#commitcomment-192882975): the stock behavior is deliberate — chosen precisely because of the `Makefile`-prefix case above.  My replacement is just a compromise that works better for me — as noted above, that case has yet to bite me in practice.  He also suggested [consult-dir](https://github.com/karthink/consult-dir), and it is a nice alternative for the same need: quickly pick a recent directory and, if needed, continue navigating from there — exactly the workflow I was trying to streamline.  I am keeping my replacement — it works well with my habits — but I will be using consult-dir alongside it: a great alternative way to do the same thing, and quite possibly a more efficient one.
