# ProperTreeCLI

A plist editor that lives in the terminal. ProperTree's job - editing
config.plist, kext ordering, oc snapshots - without a window, so it
works over ssh, in tmux, or on a box with no display at all.

This is a fork of corpnewt's ProperTree (a tkinter app). The gui still
works; the point of this repo is the command that runs from any
directory:

    plist test.plist

That opens a plist as a tree in your terminal, styled like W0lfSword:
frost blue on grey, boxed banners, dim hints, nothing on screen that
does not earn its place. If that look is good enough for a kernel
exploit toolkit, it is good enough for an EFI folder.

test.plist in the repo root is a small sample config (nested dicts,
arrays, data, a date) for trying it out. No file yet? plist new
my.plist starts one from scratch.

The same command works piped: no terminal means no colors, no editor,
just the tree as plain text, so it greps and scripts.

## Quick start

    git clone https://github.com/kaffeindecaf/ProperTreeCLI
    cd ProperTreeCLI
    ./install.sh          # links `plist` into ~/.local/bin, stdlib only
    plist test.plist      # open the sample editor
    plist --version

That is the whole install. No pip, no venv. Run plist from any
directory afterwards.

## The editor

j/k or the arrows move, home/end top/bottom, ctrl+d / ctrl+u half a
page, { } jump between siblings, left/right fold containers, enter
edits a value (booleans toggle), i adds an entry (pick the type, then
name it), D duplicates, d deletes, r renames a key, t changes a
value's type, c/x/p copy/cut/paste, u undoes (ctrl+r redoes, 200
steps), ctrl+s saves, q quits (it asks when the file is dirty). ?
shows every keybind.

/ finds. n and N cycle the matches, esc clears. tab inside the find
prompt picks what to search - keys, values, or both - and the choice
sticks for the session. R replaces the query inside string values
(when the search is scoped to keys it refuses; replacing keys is a
rename, do it by hand with r). Top-level keys keep the plain frost
blue, nested keys stay the same hue but slightly desaturated, so nesting
reads at a glance. Hold j/k or the arrows and movement accelerates
smoothly: one row per repeat at first, then gradually more, no sudden
jumps.

The file keeps its format and key order, and writes are atomic and
verified by re-reading.

## One-shots

    plist get  config.plist Misc.Boot.Timeout
    plist set  config.plist Misc.Boot.Timeout 5 -i
    plist set  config.plist Kernel.Quirks.EnableWriteUnprotector false -b
    plist del  config.plist Wifi
    plist convert config.plist -t binary
    plist new  new-config.plist            start from an empty plist
    plist settings                          show the config values
    plist settings set format binary        change one (validated)

set creates missing keys along the way and keeps the file's format and
key order. get prints strings raw, booleans lowercase, data as hex;
add --json for machine output. Keypaths are dotted, array elements are
indexes: Drivers.0.Path. Values that start with a dash need -- first:

    plist set config.plist Boot-args -- -v keepsyms=1

Run plist help for the whole list.

## Config

The editor reads ~/.config/propertreecli/config (created on first run,
XDG_CONFIG_HOME aware):

- expand_mode = all, auto, or none: whether containers open expanded
  or folded
- format = xml or binary: what plist new writes
- find_scope = keys, values, or both: what / searches by default

plist settings shows and changes these without hand-editing the file.

## State of things

The editor and one-shots sit on the parts of ProperTree worth keeping
instead of rewriting them:

- Scripts/plist.py handles xml and binary plists, data, uid, 0x ints
- the oc snapshot code walks ACPI/Kexts/Tools/Drivers, orders kexts by
  dependency, and detects the schema from OpenCore.efi's md5 - the
  headless snapshot command is the next big thing, tracked in
  ROADMAP.md
- find/replace and the ascii/hex/base64 converter from the tk window
  (the converter is still pending too)

Progress lives in ROADMAP.md. It is a checklist, not a plan: one item
per session, done end to end or not done.

## Running the original gui

    python3 ProperTree.py [file.plist]

needs python 3 with tkinter (apt install python3-tk on debian/ubuntu).
That dependency goes away with the gui.

## Credit

ProperTree by CorpNewt, BSD 3-Clause. Forked and repurposed by
kaffeindecaf.
