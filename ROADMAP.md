# ROADMAP - master task file

> purpose: turn this ProperTree fork into a terminal plist editor. one item per
> session, finish it end to end (code + test + docs touch), then check it off.
> format: `[ ]` = open, `[x] = done, with the date noted under it.
> priority: (bold) = do next, (plain) = queue, (dim) = later / maybe never.

The repo is corpnewt/ProperTree (tkinter gui, BSD-3). The gui code still
works and stays until the terminal editor covers it. Everything below
reuses the parts worth keeping: Scripts/plist.py (xml + binary io), the
snapshot logic in plistwindow.py, the converter, the settings model.

## 0 - shape the project (do first, in order)

- [x] **0.1 pick the ui engine** - spike both, half a day each:
      1. stdlib curses, zero deps, full control of the W0lfSword look
      2. textual (pip dep, widgets, mouse for free)
      default is curses: ProperTree and W0lfSword are both zero-dep, and
      the aesthetic is hand-rolled ansi anyway. if the tree gets laggy
      with big plists or mouse support becomes a must-have, textual wins.
      record the decision here.
  _Done 2026-09-03: curses chosen. capability probe under a script(1) pty
  (TERM=xterm-256color): 256 colors, 65536 pairs, palette indexes 117/153/240
  init clean on default bg, unicode box glyphs + status glyphs render. note:
  this box has no controlling tty so direct pty runs fail on cbreak - script(1)
  is the test vehicle, same trick 5.2 will use. split: one-shot commands print
  plain ansi (this preview), the interactive editor gets curses. textual stays
  unspiked until big-plist perf or mouse support is a real need (the 0.1
  criteria), zero-dep ethos wins for now._
- [x] **0.2 layout** - decide where the cli code lives. option A: new
      propertreecli/ package next to Scripts/. option B: root-level
      propertreecli.py single file, mirroring ProperTree.py. default is
      B for the first cut (one file, easy to move later), then split
      when it passes ~1500 lines.
  _Done 2026-09-03: option B - root-level propertreecli.py (~380 lines),
  sections marked (palette / drawing / logo / glyphs / io / value fmt /
  tree render / cli) so the split is mechanical when it outgrows._
- [x] **0.3 entry point** - `propertreecli` runs from any directory:
      install.sh symlinks a wrapper into ~/.local/bin that resolves its
      real path and execs python3 on the script. wrapper must survive
      being moved, so resolve symlinks with readlink -f, not $0.
      handle: no args = open ./config.plist if it exists else error
      message listing usage; file args = open those.
  _Done 2026-09-03: symlink install (install.sh -> ~/.local/bin, verified
  from /tmp over PATH), sys.path resolves the repo root through the link so
  Scripts/ imports from anywhere. no args opens ./config.plist, else usage +
  red error, exit 1. multiple files, per-file exit code, broken pipe handled
  (head). since the editor did not exist yet, "open" = read-only styled tree
  preview via Scripts/plist.py load (xml + binary, auto-detected) - this is
  also the draw-layer scaffolding 2.1/2.2 build on. plain output when piped,
  for scripting. write path is still section 1.
  _Updated 2026-09-03 (v0.2.0): command renamed to `plist` (install.sh links
  plist + keeps propertreecli as an alias; the script file keeps the repo
  name so it can never shadow Scripts/plist.py). one file on a terminal now
  opens the curses editor; piped output still prints the tree; `plist edit
  <file>` forces the editor._
- [x] **0.4 palette + logo module** - single source of truth for the
      look, copied from W0lfSword's palette block:
        C_FROST 38;5;117 (accents, key column)
        C_DIM   38;5;240 (descriptions, hints, separators)
        C_GRN   0;32  (saved / ok)
        C_AMB   1;33  (warnings, changed type)
        C_RED   0;31  (errors, delete)
        B / NC  bold + reset
      plus a tree logo in the brand color (W0lfSword uses 38;5;153 for
      its wolf - pick one accent for the tree art, keep C_FROST for
      text). respect NO_COLOR and a --no-color flag, and disable
      automatically when stdout is not a tty.
  _Done 2026-09-03: palette block at the top of propertreecli.py, constants
  flip to empty strings on --no-color / NO_COLOR / non-tty stdout (verified
  piped output is ansi-free). C_BRAND 38;5;153 reserved for the tree art,
  C_FROST 38;5;117 for keys/accents. logo is a plist tree built from column
  math (root tag branching into dict/array/string) so branches always line
  up, drawn above the banner on tty runs. boxed banner + dim rule + the
  W0lfSword glyph helpers (ok/err/warn/info/hint) live here too. two
  palette-rule additions, both marked in code: array indexes render dim
  (structural, not keys) and booleans read as status (True grn, False dim),
  which reads well on config.plist._

## 1 - plist io (non-interactive first, testable without a tui)

- [x] **1.1 read/write roundtrip** - load and save xml + binary plists
      through Scripts/plist.py (load/dump, UID, data wrap). detect the
      format from the bplist00 magic on read; keep the file's format on
      save unless told otherwise.
  _Done 2026-09-03: write_plist() in propertreecli.py - dumps through
  Scripts/plist.py with sort_keys=False, re-parses the temp file to verify,
  then os.replace()s it in. a broken dump never touches the original. format
  kept per file unless convert says otherwise._
- [x] **1.2 ordering + types** - preserve key order in dicts (the gui
      already does OrderedDict; plist.py's dict_type param exists for
      this). full type set: string, number (int with 0x hex input, real),
      bool, date, data, uid. no silent type loss on write.
  _Done 2026-09-03: load keeps file order, dumps keep it (sort_keys=False);
  set appends new keys, never reorders. all plist types settable: -i parses
  0x hex, -f real, -b bool, -x data, -d iso date, -u uid. uid refuses xml
  writes with a clear error instead of silently corrupting (checked both on
  the flag and on the whole tree via _has_uid). set refuses to clobber a
  dict/array with a scalar. bool/int ordering handled (bool checked first)._
- [x] **1.3 scriptable one-shots** - `propertreecli get/set/del <file>
      <keypath>` for shell use, with a --json mode on get (W0lfSword has
      the same --json split). keypath = dot separated, n for array
      indexes. this is what makes the tool useful in scripts, not just
      interactive.
  _Done 2026-09-03: get/set/del live. keypaths are dot separated with
  backslash escaping for literal dots (com\\.apple\\.x), numeric segments
  index arrays. set auto-creates missing dicts along the path and appends at
  array index == len; -b with no value flips an existing boolean. get prints
  strings raw, bools lowercase, data as continuous hex (xxd -r -p friendly),
  dates iso; containers tree out or json with --json. --json and --no-color
  pre-scan from anywhere in argv, W0lfSword style. exit codes: 0 ok, 1 file
  or value error, 2 bad keypath. dash-prefixed values need -- (documented in
  -h). 42-check functional suite in /tmp/propertreecli_tests.py (repo tests
  come in 5.2)._
- [x] **1.4 convert** - `propertreecli convert <file> --to xml|binary`
      in place or to a second file. port of the gui's change_plist_type.
  _Done 2026-09-03: convert -t xml|binary, -o out for a second file, atomic
  verified write. binary->xml with uid values refused up front (xml cannot
  hold them) - the one place a conversion can genuinely lose data._

- [x] **1.5 create plist files** - `plist new <file>` makes an empty plist
      from scratch and drops you into the editor. creating plists on linux
      is a non-issue - a plist is just a file format, and the binary writer
      already runs here (the convert tests round-trip it daily).
  _Done 2026-09-03: plist new <file> [--binary] [--root dict|array], xml by
  default (config format honored), refuses to overwrite, opens the editor
  on a tty. verified: xml + binary + array root + overwrite guard._

- [x] **1.6 sample file + friendlier errors** - test.plist in the repo
      root so anyone can try the editor without hunting for a plist,
      and nicer failure messages throughout the cli.
  _Done 2026-09-03: test.plist (xml, EFI-flavored sample: nested dicts,
  arrays of dicts, data blob, bools, a date - 18 keys). missing files now
  say "create it with: plist new <path>", directories and non-plists get
  clear errors with a hint, running bare with no config.plist suggests
  plist test.plist / plist new / plist help, and `plist help` works as an
  alias for -h. hints go to stderr so --json and piped output stay clean._
- [x] **1.7 diff** - `plist diff <fileA> <fileB>` walks both trees and
      prints what differs as keypaths: + added, - removed, ~ changed,
      arrays by index and dicts by key, recursion stopping at the
      first difference so a whole subtree reads as one line. exit 0
      identical, 1 differences, 2 file error - scriptable, which is
      the whole point of the one-shots.
  _Done 2026-09-03: cmd_diff in propertreecli.py, wired into COMMANDS +
  help. output sorted by keypath, containers summarized as <dict N
  keys> / <array N items>, summary line at the end. test.plist-vs-edited
  copies covered in the /tmp suite._

## 2 - the editor (the main event)

- [x] **2.1 screen** - full-screen ansi renderer: alt screen on entry,
      cursor hidden while drawing, terminal state restored on exit
      (trap INT/TERM like W0lfSword's cleanup). handle SIGWINCH resize.
      layout, top to bottom:
        boxed banner (file name, format, dirty marker)
        tree: key / type / value columns
        status line (mode hints)
        footer (keybind legend, dim)
  _Done 2026-09-03: curses app in plist_tui.py (new module, loaded lazily so
  one-shots stay untouched). header row: file frost-bold + [xml/binary] +
  amber * when dirty + right-hand hints; dim rule under it; tree body;
  status row; dim footer legend. alt screen + full restore via
  curses.wrapper, cursor hidden outside prompts, KEY_RESIZE handled.
  gotchas hit: curses cbreak keeps IXON on, so ctrl+s was eaten as XOFF
  (termios now clears IXON+ISIG, staying in cbreak so keypad escape
  parsing keeps working - curses.raw() breaks arrow-key mapping); keypad
  sequences are application-mode (\\EOB), which tripped up the pty tests,
  not real terminals._
  _Updated 2026-09-03: folding was broken in the first cut - collapse only
  flipped the glyph because the visible row list was built once and children
  never left it. toggling now rebuilds the visible rows and keeps the
  selection on the toggled container (pty scenario C proves a folded Nest no
  longer intercepts j). initial expansion is config driven, see 2.10._
  _Updated 2026-09-03: keys are shaded by depth - top-level rows keep the
  plain frost blue they always had, nested keys desaturate in a straight
  line from frost toward gray (stock 256 indices 110 steel, 103 gray-blue,
  102 gray, then flat - red channel fixed, green and blue walk down
  together so the hue never detours). two earlier attempts used custom
  palette entries via init_color; that got reverted twice - first the
  0..255 rgb values were fed to curses raw (it wants 0..1000, everything
  rendered ~4x too dark), then palette redefinition turned out
  terminal-dependent and entries 16-21 rendered as black on terminals that
  ignore it, which looked like missing text. stock colors only now, same
  on every terminal._
- [x] **2.2 node model + draw** - in-memory tree of nodes over the
      plist, one node per key (dict) or index (array). collapse/expand
      dicts and arrays, indentation by depth, long values truncated to
      the column width. scroll when the tree outgrows the screen.
      current node always visible.
  _Done 2026-09-03: flat visible-row model rebuilt per frame from the live
  root + an expanded{} map keyed by path tuple; rows carry the live node so
  edits reflect instantly. containers default-expanded unless the file has
  >1200 nodes (then only depth <= 1 opens). ▸/▾ glyphs on containers, dim
  array indexes, column-aligned key/type/value, values truncated with …,
  bools colored (True grn / False dim) like the preview. selection stays on
  screen (scroll window follows it). huge files are the remaining question,
  flagged in 0.1._
- [x] **2.3 keybinds** - vi-style + arrows: j/k or up/down move, h/l or
      left/right collapse/expand and enter/leave nodes, enter edits the
      value, tab switches key/type/value focus, insert adds, delete
      removes (with confirm when the subtree is non-empty), ctrl+c quits
      with a save prompt when dirty, ? opens a help overlay listing
      every bind. ? keybinds documented in the footer hint.
  _Done 2026-09-03: arrows + j/k, g/G, pgup/pgdn; left/right/space fold,
  left on a folded container jumps to its parent row; enter edits values /
  toggles containers and booleans; i adds (type picker menu, then key name;
  into a container or after the current sibling); d deletes with a confirm
  dialog; r renames dict keys; t changes scalar type; < > reorder; c/x/p
  copy/cut/paste; u / ctrl+r undo/redo (ctrl+z also undoes); / find with
  n/N; R replace-all in string values; ctrl+s/F2 save; q/ctrl+c quit with a
  save dialog when dirty (defaults to yes); ? overlay lists everything.
  delete confirms even leaves (roadmap said containers only - safer, one
  keystroke more). tab focus model dropped: single editable value per row
  with an inline prompt, type changes via t.
  _Updated 2026-09-03: home/end top/bottom, ctrl+d / ctrl+u half a page,
  { } jump prev/next sibling, D duplicates in place (key gets a "copy"
  suffix, auto-increments on collision; array slots insert after)._
- [x] **2.4 edit operations** - port from plistwindow.py's command set:
      add key (type picker: key/boolean/data/date/number/uid/string),
      add child vs add sibling, delete, duplicate, rename key, reorder
      (W0lfSword menu_opt layout for the pickers), change a node's type.
  _Done 2026-09-03: add (9-type picker incl. dict/array, inserted after the
  anchor sibling or appended into a container, duplicate key refused), delete
  (whole subtree, dict rebuild or list pop), rename key (order-preserving
  dict rebuild), reorder < > (dict swap or list swap), change type (scalar
  conversions only, containers refused, uid gated to binary files). duplicate
  covered by c+x+p (copy to the /tmp clipboard, paste under a new key) - no
  separate D key._
- [x] **2.5 value editors** - per-type input with validation:
      ints accept decimal and 0x hex, bools cycle the gui's styles
      (True/False, YES/NO, On/Off, 1/0) - configurable default, data
      entered as hex or base64, dates parsed from iso. bad input = error
      flash, value unchanged, cursor back in the field.
  _Done 2026-09-03: one inline prompt per type, prefilled with the current
  value (caret-visible windowing, ctrl+u clears, esc cancels, left/right/
  home/end). commit parses through propertreecli.parse_value so editor and
  one-shots agree: ints decimal + 0x, real, data hex (spaces ok), iso dates,
  uid ints. booleans toggle on enter (style cycling needs the 4.1 config,
  deferred with the bool_style key). bad input flashes red, value untouched.
  unchanged input reports "unchanged" without a dirty marker._
- [x] **2.6 undo/redo** - full stack, 200 steps (the gui's max_undo).
      snapshots of the whole tree per op is fine at this size. ctrl+z /
      ctrl+y, counter on the status line.
  _Done 2026-09-03: deepcopy snapshots of the root per mutation, capped at
  200, redo cleared on new edits. u / ctrl+z undo, ctrl+r / ctrl+y redo,
  dirty flag set by every mutation (including the push itself, so nothing
  forgets). no step counter on the status line yet - the stack depth is not
  a useful number on screen, cut it._
- [x] **2.7 clipboard** - cut/copy/paste nodes between files (or
      instances) and within one. internal json clipboard file under
      XDG_RUNTIME_DIR or /tmp so a second propertreecli instance can
      paste into the first.
  _Done 2026-09-03: c/x/p work in-file and across instances - the clipboard
  is an xml plist file at $XDG_RUNTIME_DIR/plist-clipboard.plist (fallback
  /tmp) holding {"v": <payload>}, so fidelity is exact (plist in, plist
  out; a json file would mangle data/date/uid). paste into a container
  appends, after a row inserts a sibling, dict parents prompt for a key and
  refuse collisions. cut = copy + delete with a confirm when it is a
  subtree._
- [x] **2.8 search** - / opens a search line, n/N next/prev, highlight
      all matches in the tree. replace mode with the type filter the
      fork's recent commits added (find type defaults as a setting).
      search keys, values, or both.
  _Done 2026-09-03: / find + n/N cycle + amber match rows shipped, and R
  replaces the query inside string values (keys are left alone on
  purpose). scope picker closed the item: tab inside the find prompt
  cycles keys / values / both, the pick sticks for n/N and R, and the
  default comes from config find_scope (4.1). matches are substring over
  canonical value text; container rows match on their key only (the first
  cut matched every container row for any query - fixed). R refuses on a
  keys-scoped find instead of silently replacing value text. a second
  latent bug surfaced when scenario F finally drove / end to end: _goto
  was called by find but never defined, so every find crashed with an
  AttributeError caught by the run loop - find has now been exercised
  under a pty (scenario F) and works. deviation from the gui, noted: the
  tk dropdown filters by plist type (key/boolean/data/date/number/uid/
  string) - a terminal finder searches value text across types instead,
  which is a superset for substring use._
- [ ] **2.9 converter + templates** - the gui's ascii/base64/decimal/
      hex/binary converter as a keybind on a selected value (parity with
      the tk converter window), and insert-from-template using
      config_tex_info.py + menu.plist (OpenCore/Clover samples).
- [x] **2.10 editor view config** - a small config file decides how the
      editor opens files: everything expanded, everything collapsed, or
      the auto heuristic. plain key=value with # comments, template
      written on first run (W0lfSword config_schema trick).
  _Done 2026-09-03: ~/.config/propertreecli/config (XDG_CONFIG_HOME aware)
  with expand_mode = all | auto | none and format = xml | binary. all opens
  every container, none opens every container folded, auto keeps the old
  heuristic (expand unless the file exceeds ~1200 nodes, then only depth 1).
  created on first editor run with a comment template; load_config validates
  values and falls back to defaults. format is what plist new writes (1.5).
  still to come from 4.1: data/int/bool display styles, find type defaults,
  and a settings command to show/set values from the cli._

- [x] **2.11 movement polish** - holding j/k or the arrow keys
      accelerates smoothly: the step size drifts 1 -> 2 -> 3 -> 4 over
      a few seconds of holding, via a fractional accumulator fed by a
      slow ramp. no tiers, no half-page teleports.
  _Done 2026-09-03: streak-tiers (1/2/4/half-page) were the first cut;
  user feedback: jerky. reworked to ramp += 0.03 per repeat (cap 2.2) and
  step = the integer part of an accumulating 1.0 + ramp - so steps are
  mostly 1 for the first ~0.5s of a hold, then drift through a smooth
  1/2/3 mix to at most 4. a 240-repeat pty run crosses 500 rows around
  event 180, and the step sequence reads 1111111211121212... (verified in
  scenario E, which still passes). note: a cmatrix-style boot scramble was
  added and then removed the same session on user feedback (did not look
  good) - the editor now opens straight to the tree._

## 3 - snapshot mode (headless, flagship feature)

- [ ] **3.1 oc snapshot / oc clean snapshot** - port of plistwindow.py's
      oc_snapshot(clean=...): walk ACPI/Kexts/Tools/Drivers, add/remove
      entries, order kexts by OSBundleLibraries vs CFBundleIdentifier so
      dependencies load first, warn on duplicate CFBundleIdentifiers
      with MinKernel/MaxKernel/MatchKernel overlap checks, flag disabled
      parent kexts with enabled children.
- [ ] **3.2 schema detection** - md5 of OpenCore.efi matched against
      known hashes, falling back to the newest schema in
      Scripts/snapshot.plist. target version selectable in config, same
      as the gui's OC Snapshot Target Version menu.
- [ ] **3.3 dry-run mode** - `snapshot --dry-run` prints the diff
      (added/removed/reordered) instead of writing. exit 0 = clean, 1 =
      changes needed, 2 = error. this makes it usable in ci and in
      build scripts, which the gui never could be.

## 4 - config + settings

- [x] **4.1 config file** - ~/.config/propertreecli/config, plain
      key=value with # comments, template written on first run. schema
      doubles as the docs, same trick as W0lfSword's config_schema().
      keys: expand_mode (all/auto/none), format (xml/binary), find_scope
      (keys/values/both), theme (frost/red). validation lives in
      load_config; unknown or invalid values fall back to defaults
      instead of erroring.
  _Done 2026-09-03: the file itself landed with 2.10; the schema is now
  CONFIG_DEFAULTS + CONFIG_VALID in propertreecli.py, load_config
  validates every key against CONFIG_VALID, and `plist settings` (4.2)
  manages it from the cli. theme = red (added same day, user request)
  recolors the whole ui: the curses editor swaps its palette maps for
  red variants (accents 196, dim 124, depth shades 160/124/88) and the
  ansi one-shots swap their escapes when the config says so - every
  text color goes red, hints stay muted. keys that were planned and got
  cut, on purpose: animations on/off (the animation feature was removed
  from the product), prompt_symbol and logo/color toggles (the look is
  fixed, color already bows to --no-color / NO_COLOR / non-tty),
  data/int/bool display styles (the editor renders values readably
  as-is; revisit only if someone asks)._
- [x] **4.2 settings command** - `plist settings` shows the file,
      `settings set key value` validates and writes, `settings reset`
      restores defaults. no menu for it in the tui yet; editing by hand
      is fine.
  _Done 2026-09-03: settings show / set / reset live in cmd_settings(),
  wired into COMMANDS + help. set validates against CONFIG_VALID before
  writing, updates the config in place (comments and other keys
  preserved), reset rewrites the template. config file auto-created on
  first use, same as the editor._
- [ ] **4.3 version + update check** - single VERSION source in the
      script (W0lfSword style), --version prints it, update_check.py
      pattern reused against this repo's own version feed.

## 5 - finish

- [ ] **5.1 kill the gui** - drop ProperTree.py, plistwindow.py, the .bat
      and .command launchers, tkinter fallbacks in plist.py, py2
      shims. only once the editor covers the snapshot + converter
      features. keep Scripts/plist.py's io, minus the py2 branches.
- [ ] **5.2 tests** - plist roundtrips over a corpus (xml, binary, data,
      uid, dates, 0x ints, deep nesting), snapshot dry-run against a
      sample efi folder, keypath get/set/del, tui smoke test driving
      keys through a pty.
- [ ] **5.3 release** - README demo (asciinema), screenshots in the
      readme, v0.1.0 tag, install.sh verified from a fresh clone on a
      bare debian box.

## notes

- snapshot.plist + version.json are upstream's; version.json will
  become our own feed once 5.3 ships.
- the tree logo: draw it early (0.4) and let it set the tone, the same
  way the wolf does for W0lfSword. a plist is a tree - the logo should
  say that.
