# freesync

A friendly terminal tool for keeping a local editing folder and a folder on the
photo server in sync, in both directions.

It wraps `rsync` so you don't retype long paths, and it **always shows you what
it's about to do before it does it**.

## Install

```bash
ln -s "$PWD/freesync" /opt/homebrew/bin/freesync
```

Then open a new terminal and run `freesync`.

## Everyday use

Just run:

```bash
freesync
```

You get a menu of your saved folders, then a menu of what to do:

```
freesync v1.0.0

  1) trip-day2  (last used)
  2) portraits

  a) add a new folder pair     l) show details     q) quit

Which folder? [1] 

  1) Pull   server → your computer   (get photos to work on)
  2) Push   your computer → server   (send your edits back)
  3) Both   pull, then push

What would you like to do? [1] 
```

freesync then previews the change and waits for you:

```
Checking what would change (nothing is copied yet)…

Would copy 34 file(s), 1.9 GB:
  DSC_0431.NEF
  DSC_0432.NEF
  …and 14 more

Copy these to your computer? [y/N]
```

If you prefer typing, all of these work too:

```bash
freesync pull                 # uses the folder you used last
freesync push trip-day2
freesync both trip-day2       # pull, then push
freesync -n push              # preview only, never copies
freesync -y pull              # no preview, no questions (for when you're sure)
```

## Adding a folder pair

```bash
freesync add
```

It asks for a short name, your local folder, and the server folder. **Tip: drag
a folder from Finder into the terminal** to paste its path — freesync handles the
backslashes Finder adds, and quotes, and trailing slashes.

Or in one line:

```bash
freesync add trip-day2 ~/Pictures/editing "/Volumes/photos/2026/trip/Day2/raw"
```

Other commands:

```bash
freesync list             # show saved pairs
freesync remove NAME      # forget a pair (never touches any files)
freesync mode NAME new|changed
freesync edit-excludes    # files that are never copied
freesync edit-options     # extra rsync options
```

## The two modes

| Mode | What it does |
| --- | --- |
| `new` (default) | Copies only files the other side doesn't have. An existing file is **never** overwritten. |
| `changed` | Also overwrites a file when the other side has a **newer** version of it. |

`new` is the safe default: it is `rsync --update --ignore-existing`.

Switch a pair to `changed` when you want edits made *in place* to travel back —
sidecar `.xmp` files, Capture One/Lightroom adjustments, or a re-saved JPEG that
keeps its original filename. In `new` mode those edits stay on your laptop
forever, because the file already exists on the server.

```bash
freesync mode trip-day2 changed
```

Even in `changed` mode, older files never overwrite newer ones (`--update`), so
you can't clobber a fresh edit with a stale copy.

## Safety

- **Nothing is ever deleted**, in either direction. `--delete` is not used, and
  there is no flag to turn it on.
- **Preview first, always.** freesync does a dry run and asks before copying.
  Only `-y` skips that.
- **Unmounted server = stop.** If the server folder is on `/Volumes/…` and that
  volume isn't connected, freesync refuses to run instead of quietly filling up
  your local disk with a folder that looks like the server.
- **Nested folders are rejected**, so you can't sync a folder into itself.
- **Interrupted transfers are fine.** `--partial` keeps what was copied; just
  run the same command again.

## Folder mapping (worth knowing once)

freesync mirrors the *contents* of the two folders. If the pair is:

```
local   ~/Pictures/editing
server  /Volumes/photos/…/Day2/raw
```

then `raw/DSC_0431.NEF` ⇄ `editing/DSC_0431.NEF`. Pull and push are exact
mirrors of each other.

> This is the classic hand-rolled-rsync trap: it's easy to write a pull command
> with no trailing slash on the source (which nests a `raw/` subfolder inside
> your local folder) and a push command *with* one (which sends your local
> folder's contents straight into `raw/`). Run the two in sequence and you get
> `…/Day2/raw/raw/`. freesync always mirrors, in both directions.

## What gets skipped

macOS and Windows clutter, editable at `freesync edit-excludes`:

```
.DS_Store         .Spotlight-V100    .fseventsd
._*               .Trashes           .DocumentRevisions-V100
Thumbs.db         .TemporaryItems    *.lrcat-journal
desktop.ini                          *.lrcat.lock
```

## If the server complains about permissions

WebDAV and SMB shares often can't store Unix permissions or ownership, so rsync
may print errors like `failed to set permissions`. Fix it once:

```bash
freesync edit-options
```

and uncomment these lines:

```
--no-perms
--no-owner
--no-group
```

## Where settings live

```
~/.config/freesync/profiles.tsv      your saved folder pairs
~/.config/freesync/excludes.txt      files never copied
~/.config/freesync/rsync-extra.txt   extra rsync options
~/.config/freesync/last              the pair you used most recently
```

Plain text — safe to edit, copy to another Mac, or back up.

## Requirements

- **rsync 3.1.0 or newer** — install it with `brew install rsync`.
- macOS. Works with the stock `/bin/bash` 3.2, so nothing else to install.

### The rsync that ships with macOS is not enough

`/usr/bin/rsync` on macOS is **openrsync**, which identifies itself as
"rsync version 2.6.9 compatible". It rejects two options freesync relies on
(`--info=progress2` and `--no-human-readable`), so freesync will stop with an
`unrecognized option` error instead of syncing. Older macOS shipped the real
rsync 2.6.9 from 2006, which behaves the same way.

Check which one you have:

```bash
rsync --version | head -1
```

| Output | Verdict |
| --- | --- |
| `rsync  version 3.5.0  protocol version 32` | good |
| `openrsync: protocol version 29` | too old — `brew install rsync` |
| `rsync  version 2.6.9  protocol version 29` | too old — `brew install rsync` |

`freesync doctor` prints the same thing along with the exact binary in use.

freesync picks the first rsync it finds, in this order:

1. `/opt/homebrew/bin/rsync` (Homebrew on Apple Silicon)
2. `/usr/local/bin/rsync` (Homebrew on Intel)
3. whatever `rsync` is on your `PATH`

So once Homebrew's rsync is installed, freesync uses it automatically — you
don't need to change your `PATH` or edit anything. **Each computer needs its
own `brew install rsync`**; a machine that has never had it will fall through to
openrsync at step 3.

Why 3.1.0 specifically: `--info=progress2` (the single overall progress bar)
landed in rsync 3.1.0, and `--no-human-readable` in 3.0.0. Anything newer is
fine; Homebrew currently ships 3.5.x.

## Troubleshooting

**"the server folder is not usable" — but I can `cd` into it**

Run `freesync doctor`. It walks the path, shows the first component that is
really missing, and lists what that folder actually contains. The usual causes
all look identical on screen:

| Cause | Why `cd` still works |
| --- | --- |
| A carriage return or trailing space in the saved path | You typed/tab-completed the real name; the config holds `…/raw\r`, which prints as `…/raw`. freesync strips these automatically now. |
| Capitalisation | Your Mac's disk ignores case, most servers don't, so `Day2` and `day2` are one folder locally and two on the server. |
| Accented or non-Latin names | macOS stores `í` decomposed (NFD), servers usually send it precomposed (NFC). Identical on screen, not equal to `test`. Copy the name out of `ls` instead of typing it. |
| A different mount name on that machine | macOS appends `-1` when a stale mount exists, so the share lands at `/Volumes/photos-1`. `freesync doctor` lists every mounted volume. |
| A wrong component deeper in the path | `Q3` vs `Q4`, `Day2` vs `Day 2`. The doctor points at the exact part. |

Profiles are per-machine. Copying `profiles.tsv` between computers is what
produces most of the above — `freesync add` on each machine is safer.

**`rsync: unrecognized option '--info=progress2'` (or `--no-human-readable`)**

That machine is falling back to macOS's built-in openrsync. Run
`brew install rsync` on it, then `freesync doctor` to confirm the binary in use
changed. See [Requirements](#requirements).

## License

MIT — see [LICENSE](LICENSE).
