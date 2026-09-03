# ProperTreeCLI

Edit plists the way you edit config files on a server: from the
terminal, over ssh, inside tmux, on a machine with no display at all.

    plist test.plist

opens a plist as a tree you can walk with the arrow keys, fold like
a file explorer, edit inline, and save with ctrl+s. Styled after
W0lfSword - frost blue on grey, boxed banners, nothing on screen that
does not earn its place. If that look is good enough for a kernel
exploit toolkit, it is good enough for an EFI folder.

ProperTreeCLI is a fork of corpnewt's ProperTree, a tkinter app. The
gui still works; the point of this repo is the command line twin. It
keeps the parts worth keeping (Scripts/plist.py handles xml + binary
plists, the snapshot logic stays for later) and adds what a terminal
tool should have: scriptable one-shots, exit codes, and no window.

test.plist in the repo root is a small sample config (nested dicts,
arrays, data, a date) for trying it out. No file yet? plist new
my.plist starts one from scratch.

## Try it in a minute

    git clone https://github.com/kaffeindecaf/ProperTreeCLI
    cd ProperTreeCLI
    ./install.sh          # symlinks `plist` into ~/.local/bin
    plist test.plist      # sample editor (q quits, ? shows keys)
    plist --version

No pip, no venv, no tkinter. Stdlib only. Run plist from any
directory afterwards.

Piped output is plain text: no terminal means no colors and no editor,
just the tree, so it greps and scripts.

## The editor

    plist test.plist



/ finds. n and N cycle the matches, esc clears. tab inside the find
prompt picks what to search - keys, values, or both - and the choice
sticks for the session. R replaces the query inside string values
(when the search is scoped to keys it refuses; replacing keys is a
rename, do it by hand with r).

^t (ctrl+t) opens the value converter on the selected entry: read the
text as ascii, base64, decimal, hex, or binary and render it as
another. pick from and to, tweak the text if you like, and enter on
the result writes it back as the entry's own kind - pasting foreign
base64 into a data field and entering stores the decoded bytes in one
trip.

T inserts from the OpenCore/Clover preset library: pick a section,
then a path (Kernel/Add, ACPI/Patch...), then a preset such as a blank
entry or a ready kext pack. Missing containers are created along the
way; it asks before clobbering anything.

Hold j/k or the arrows and movement accelerates smoothly: one row per
repeat at first, then gradually more, no sudden jumps.

Writes are atomic and verified by re-reading; the file keeps its
format and key order.

## One-shots

    plist get  config.plist Misc.Boot.Timeout
    plist set  config.plist Misc.Boot.Timeout 5 -i
    plist set  config.plist Kernel.Quirks.EnableWriteUnprotector false -b
    plist del  config.plist Wifi
    plist diff config.plist config-clean.plist    what changed between two
    plist convert config.plist -t binary
    plist new  new-config.plist            start from an empty plist
    plist settings                          show the config values
    plist settings set theme red            every text color red

diff walks both trees and prints added/removed/changed keypaths;
it exits 0 when the files match and 1 when they differ, so it works
in a build script or a pre-push check.

set creates missing keys along the way and keeps the file's format
and key order. get prints strings raw, booleans lowercase, data as
hex; add --json for machine output. Keypaths are dotted, array
elements are indexes: Drivers.0.Path. Values that start with a dash
need -- first:

    plist set config.plist Boot-args -- -v keepsyms=1

Run plist help for the whole list.

## Config

The editor reads ~/.config/propertreecli/config (created on first
run, XDG_CONFIG_HOME aware):

- expand_mode = all, auto, or none: whether containers open expanded
  or folded
- format = xml or binary: what plist new writes
- find_scope = keys, values, or both: what / searches by default
- theme = frost or red: the palette. red turns every text color red

plist settings shows and changes these without hand-editing the file.

## State of things

The editor and one-shots sit on the parts of ProperTree worth keeping
instead of rewriting them:

- Scripts/plist.py handles xml and binary plists, data, uid, 0x ints
- the oc snapshot code walks ACPI/Kexts/Tools/Drivers, orders kexts
  by dependency, and detects the schema from OpenCore.efi's md5 - a
  headless snapshot command is next, tracked in ROADMAP.md
- find/replace is ported, as are the value converter and the
  OpenCore/Clover insert-from-template presets

Progress lives in ROADMAP.md. It is a checklist, not a plan: one item
per session, done end to end or not done.

## Running the original gui

    python3 ProperTree.py [file.plist]

needs python 3 with tkinter (apt install python3-tk on debian/ubuntu).
That dependency goes away with the gui.

## Credit

ProperTree by CorpNewt, BSD 3-Clause. Forked and repurposed by
kaffeindecaf.
