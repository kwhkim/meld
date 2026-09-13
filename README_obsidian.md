# Using this Meld with Obsidian (or any other editor)

This branch (`readonly-gtk3`) is a personal fork of Meld, based on the last
commit before the upstream GTK4/libadwaita rewrite. It targets the GTK3 +
GtkSourceView 4 stack that's already installed on this system (matching the
same dependencies as the distro's `meld` package), so it runs without pulling
in any new libraries.

The goal: let you keep a file open in Meld for comparison/review while it's
being actively edited by another program (Obsidian, in particular) — without
Meld ever writing to that file, and without the two programs racing to
overwrite each other's changes.

## Why this exists

Opening the same file in two editors is normally risky: whichever one saves
last silently overwrites whatever the other program wrote. This fork removes
that risk entirely on Meld's side, and layers on some usability changes so
that watching a file that's changing underneath you doesn't feel painful:

- Meld is put into a strict read-only mode: it can never save, so it can
  never be the program that clobbers your edit.
- When the file changes on disk, Meld reloads it automatically and moves the
  cursor to wherever the change happened, instead of resetting to the top of
  the file every time.
- The reload doesn't cause any visible flash or jump to the wrong place
  first — the pane holds its content steady until the correct, final state
  is ready.
- A "File changed on disk" notice can be silenced during a burst of rapid
  saves (e.g. autosave-on-every-keystroke), so it doesn't pop up and vanish
  constantly while you're actively writing.

None of this requires anything from Obsidian's side — it's purely a Meld
change.

## How to run it

There's a wrinkle on this machine: the `python3` on your `PATH` resolves to
Miniforge/conda's interpreter, which doesn't have the GTK bindings (`gi`)
installed. Always invoke Meld with the system interpreter explicitly:

```sh
/usr/bin/python3 /home/gogpu/git/meld2/bin/meld --read-only file1 file2
```

For a real Obsidian workflow, that's typically two versions of the same note
(e.g. comparing your working copy against a backup or another branch):

```sh
/usr/bin/python3 /home/gogpu/git/meld2/bin/meld --read-only \
    ~/notes/vault/some-note.md ~/notes/vault-backup/some-note.md
```

Make sure you're on the `readonly-gtk3` branch of this checkout — the `main`
branch is the GTK4 rewrite and needs GtkSourceView 5, which isn't installed
here.

## Command-line options added by this fork

### `-r`, `--read-only`

Opens every comparison read-only, regardless of the underlying files'
on-disk permissions. No pane in the comparison can ever be edited or saved.
This is the flag that makes it safe to point Meld at a file another program
is actively writing to.

This flag also disables GTK's UI animations for the process, so that the
cursor/scroll jumps described below happen as an instant snap rather than an
animated glide.

### `--auto-reload=<value>`

Controls the "File _ changed on disk / Reloaded automatically" notice bar
that appears when `--read-only` auto-reloads a file. **The reload itself,
and the cursor jump to the changed line, always happen** no matter what you
set this to — this option only affects whether/how often you're told about
it. `<value>` can be:

- **A number of seconds**, e.g. `--auto-reload=10` (the default is `10`).
  This is the minimum pause since the last notice before showing another
  one. If you're saving repeatedly faster than this, only the first notice
  in the burst is shown; once you pause for longer than this, the next
  reload shows the notice again.
- **`Inf`** (or `inf`/`infinity`) — shows the very first "changed on disk"
  notice ever for a pane, and never again after that, no matter how long you
  wait. Reload and cursor-tracking are unaffected.
- **`no-msg`** — never shows the notice at all, not even the first time.

Examples:

```sh
# Default: a notice at most every 10 seconds
/usr/bin/python3 bin/meld --read-only file1 file2

# Only ever see the notice once
/usr/bin/python3 bin/meld --read-only --auto-reload=Inf file1 file2

# Never see the notice, just silent tracking
/usr/bin/python3 bin/meld --read-only --auto-reload=no-msg file1 file2

# Custom pause: at most one notice every 30 seconds
/usr/bin/python3 bin/meld --read-only --auto-reload=30 file1 file2
```

## What "auto-reload and jump to the change" means in practice

With `--read-only` set, whenever the file changes on disk (Obsidian saves
it):

1. Meld detects the change (via its existing file-watcher) and debounces
   rapid bursts of change notifications into a single reload, so a save that
   triggers several low-level filesystem events only causes one reload.
2. It reloads the file's content.
3. It compares the pane's content immediately before and after the reload,
   finds the first line that actually differs, and moves the cursor there —
   scrolling it into view with a bit of context above it, rather than
   leaving you at line 1 or wherever the cursor happened to be.
4. All of this happens without any visible flash of the wrong location or
   the pane going blank in between — the display only updates once, to the
   final, correct state.

If you keep editing the same spot repeatedly (e.g. steadily writing a
paragraph with autosave), the view simply stays where it is across reloads,
since that's already the location of the latest change.

## Notes

- This is all scoped to `--read-only` mode. Without that flag, Meld behaves
  exactly like upstream: editable panes, no auto-reload, manual "Reload?"
  prompts on external changes.
- The changes live in `meld/filediff.py` (the reload/scroll/notice logic)
  and `meld/meldapp.py` (the new CLI options).
