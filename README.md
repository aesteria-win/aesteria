# aesteria

External tooling for **AION 2** — one project, two platforms.

An overlay and a menu that show you the world, and a movement boost that leans on your velocity
instead of teleporting you. Built by measuring the live client, not by assuming. **Where a result
is not proven, this file says so instead of smoothing it over.**

| | **Windows** | **Linux** (Steam / Proton) |
|---|---|---|
| Status | New — single `.exe`, released Oct 2026 | Measured since late 2025 |
| Menu + overlay | ✅ | ✅ |
| Movement boost | ✅ | ✅ |
| Runs as | one program, no Python needed | Python 3.11+ beside the game |
| Download | [`aesteria-windows-v0.2.zip`](../../releases/latest) | [`dist/aesteria-linux-v0.1.zip`](dist/aesteria-linux-v0.1.zip) |

## Download

**Windows** — [`aesteria-windows-v0.2.zip`](../../releases/latest) (~21 MB). Unzip anywhere.
No installer, no Python, no dependencies. Right-click → **Run as administrator**.

**Linux** — see [Running it on Linux](#running-it-on-linux) below. The Linux tools are source; they
read `/proc/<pid>/mem` from outside the process.

## What it does

- **The menu.** `PgDn` opens it. Every control is live — change the range and the boxes move out on
  the next pass; change a boost rate and the next step uses it. Position and size are remembered.
- **The world overlay.** Mobs, other players and collectible objects, each with a distance. Drawn
  whether the menu is open or not, because an overlay you have to open a menu to see is not an
  overlay. Players of the other faction come up red.
- **The movement boost.** A step along your live velocity, as a fraction of your current speed,
  capped separately per mode — on foot, mounted, and in flight. It pauses itself when the server
  corrects you and steps the rate down if that keeps happening. **It starts off** and stays off
  until you arm it (`F6`).

### What it writes

Everything is a **read** except the boost. The boost writes **your own character's position**, and
only while armed.

- No injection, no code patching, no hooks, no DLLs.
- Two fields per step, nothing else.
- It never touches another player, the server's ledger, items, currency or world state.

## Running it on Windows

1. Unzip, then **right-click `aesteria.exe` → "Run as administrator."**
   The client runs at High integrity; a normal process cannot even read it, and Windows refuses
   that before any anti-cheat is involved.
2. Start the game and get **in-world** (not the character select screen).
3. It finds the client itself. If it picks the wrong thing: `aesteria.exe --pid 1234`
4. Run the game **borderless or windowed.** Exclusive fullscreen hides every overlay that exists,
   this one included.

**Check the window works first** — opens the menu with no game attached, ten seconds:

    aesteria.exe --demo

Run it from **your own logged-in desktop**. If you see an error 1459 note about "no interactive
window station", you launched it from a service, scheduled task or remote session that has no
desktop to receive keys — not a broken tool.

| key | does |
|---|---|
| `PgDn` | open / close the menu |
| `F6` | arm / disarm the boost |
| `Ctrl+C` | quit (from its console) |

## Running it on Linux

The game runs under Steam/Proton; these tools run beside it, as your own user.

**Requirements**

* Python 3.11+
* `numpy`
* For the overlay only: **GTK 4** + `gtk4-layer-shell` and their GObject bindings
  (Arch: `python-gobject gtk4 gtk4-layer-shell`; Debian/Ubuntu:
  `python3-gi gir1.2-gtk-4.0 gir1.2-gtk4layershell-1.0`)
* Permission to read `/proc/<pid>/mem` of your own game process. On most distributions Yama
  restricts this — either `sudo sysctl kernel.yama.ptrace_scope=0` for the session, or run the
  tool with `sudo`. Nothing here needs root for any other reason.

**The overlay:**

```sh
python3 tools/aesteria_overlay.py --screen 1920x1080          # set YOUR resolution
python3 tools/aesteria_overlay.py --screen 1920x1080 --hp     # + HP bars
python3 tools/aesteria_overlay.py --screen 1920x1080 --all    # + every interactable, labelled
```

Run the game **borderless**, not exclusive fullscreen. The overlay is a `wlr-layer-shell` surface,
so it needs a Wayland compositor that implements that protocol (sway, Hyprland, KDE Plasma;
**GNOME does not**). It is click-through and never takes focus.

**Without the overlay**, every reader works standalone and prints to the terminal:

```sh
python3 tools/aesteria_radar.py --mesh Protect_01 --state 1 --watch   # uncollected feathers
python3 tools/aesteria_dungeons.py                                    # entrances from the paks
python3 tools/aesteria_route.py                                       # a loop over both
python3 tools/aesteria_hp.py                                          # own + nameplate HP
python3 tools/aesteria_fps.py                                         # frametimes
```

Offsets are per client build. After a patch, re-derive with `tools/aesteria_registry.py` rather
than editing constants by hand — the registry finds classes by shape, so it survives patches that
move things.

## What's actually proven

This is the part most projects skip. Read it before you decide how much to trust the tool.

**Linux — well measured.** A year of runs against the live client, across client patches. Reads,
overlay and boost have each been exercised repeatedly, including runs where the boost
self-corrected.

**Windows — new, and narrower than the Linux side.** Measured on **one machine, one client build**:
the read path, the overlay render, and 16 boost steps in 65 seconds in flight and mounted with the
client alive throughout. Everything else on Windows is inference from the Linux side.

**The one thing still unexplained:** one correction fired while mounted and the cause is not
known. It may have been a speed higher than the rate was tuned for; it may be the server tightening
what it allows. **If your character gets pulled back or shoved off the mount, tell me — that is the
single most useful report you can send.** Don't assume you did something wrong.

Practical consequence: **arm the boost in flight first.** That is the case with the most
measurement behind it and the one where a mistake is cheapest.

## Privacy

**No telemetry. Nothing here phones home.** Not on launch, not on a timer, not ever — on either
platform.

- **No analytics, no usage statistics, no crash reporting, no version check, no licence check.**
- **No account, character, or machine identifiers are collected** — because nothing is collected.
- **Nothing leaves your machine.** These tools never open a network connection, to us or to anyone.

Don't take that on trust. The Windows binary ships as a frozen build, so here is the check against
the source instead:

```sh
grep -rn 'import socket\|urllib\|requests\.\|winhttp\|InternetOpen' tools framework
```

No hits in the product path. On the Linux side every hit is in `aesteria_netcap.py`, and every one
of them *reads* — the game's own sockets out of `/proc`, and `socket.inet_ntop` to format an
address into text. **No socket is ever created.**

- **Local files only.** Config and state live under `$XDG_CONFIG_HOME` and `$XDG_STATE_HOME`
  (normally `~/.config/aesteria` and `~/.local/state/aesteria`), plus a launch log at
  `/tmp/aesteria-launch.log`. On Windows, the boost log is `%LOCALAPPDATA%\aesteria\boost.log`.
  Delete them and nothing remembers you.
- **`aesteria_netcap.py` is the one tool that handles network data, and it is opt-in and local.**
  It captures *your own* traffic to a file on *your own* disk, for inspecting the game's protocol.
  Its default filter is deliberately narrow — the game's own endpoints — specifically so it does
  not record your browsing, your mail, or the client's TLS connections. `--all` widens it, and
  warns you when it does.

This isn't a statement of intent you have to believe. It's a property of the source you're reading.

⚠ One word that looks alarming in this tree and isn't: `framework/mirage/publisher.py` has a
field called `telemetry`. It means **tick and error counters written to a local state file** so the
overlay can read them. It is not network telemetry and the name is a poor choice.

## Safety notes

- Tools are read-only unless a mode says `--apply` or the boost is armed. Linux writes go through
  `/proc/<pid>/mem`; Windows writes use process memory access on your own character.
- Offsets and layouts are build-specific. Re-derive them after a client patch.
- This runs against a live online game. **You are responsible for what that means for your
  account — use an account you can afford to lose.**

## What's in the repo

- `tools/` — Python tools that read `/proc/<pid>/mem` from outside the process (no injection):
  UObject/GNames/GObjects discovery, a blind class registry that survives patches, entity and player
  readers, HP read from the UI, a feather/dungeon radar and route solver, a frame-rate meter and
  live cap, and an overlay driver.
- `src/inject/` — a passive `LD_PRELOAD` probe and a Vulkan implicit layer (overlay) with Makefiles.
- `spikes/` — throwaway experiments, with their findings.

The **Windows** product is currently distributed as a build rather than as source. Ask and I will
send the source for the part you want to read.

## Tests

Offline tests live in `tools/test_*.py` and `tools/test_wrap_modes.sh`. `test_registry.py` needs a
real process capture and skips (loudly) without one.

## Licence

MIT. See [`LICENSE`](LICENSE).
