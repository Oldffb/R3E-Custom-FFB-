# What the application touches on your system

A fair question before running any third-party tool next to a simulator you care
about — especially one that talks to your wheelbase. This page lists **every file
and system resource** R3E Custom FFB reads or writes, and shows how the claims can
be verified rather than taken on trust.

**Short answer: no file belonging to RaceRoom is opened for writing, ever.** The
game's telemetry is read through a **read-only** memory handle, and the operating
system itself refuses any write through it.

---

## Files the application writes

Everything the application creates lives in **two folders**, both of them yours:

### 1. `%LOCALAPPDATA%\R3E_FFB\`

Your settings and everything the app needs to remember between sessions:

| Path | Contents |
|---|---|
| `configs\cars\global.json` | Base settings, used when no profile matches |
| `configs\cars\<carId>.json` | Per-car profile |
| `configs\cars\cars\<carId>\<track>\…` | Per-car-and-track profile |
| `configs\cars\ui-layout.json` | Panel positions in the main window |
| `device.json` | Which wheel you selected |
| `WebView2\` · `WebView2_Overlay\` | Browser engine cache for the two windows |

Deleting this folder resets the application to a fresh install. Nothing else
breaks.

### 2. `Documents\RRE_FFB_Logs\`

Only when **you** press record in the Logger tab. One CSV per session, containing
telemetry and the FFB values computed from it. Nothing is written here otherwise,
and nothing is ever sent anywhere.

### 3. Next to the executable

| File | Written when |
|---|---|
| `appsettings.json` | Settings that apply before the interface loads — zoom, language, start-with-Windows |
| `terms-accepted.json` | You accept the terms on first run |
| `notice-ack.json` | You confirm the initial instructions |

If you extracted the application to a read-only location, these fall back to
defaults rather than failing.

### 4. The Windows temporary folder — only during CSV replay

If you replay a telemetry CSV (a development feature, not part of normal driving),
the file is staged as `%TEMP%\replay_<random>.csv` and **deleted as soon as the
replay ends**, including when it fails. Nothing else uses the temporary folder.

---

## The Windows registry

**One value, and only if you ask for it.**

```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Run
    R3E Custom FFB = "<path to the exe>" [/minimized]
```

Written when you enable **Start with Windows**, removed when you disable it.
`HKEY_CURRENT_USER` is your own user hive — no machine-wide key, no service, no
driver, nothing that needs administrator rights.

---

## Files the application reads

| Source | Access |
|---|---|
| RaceRoom shared memory (`$R3E`) | **read-only**, see below |
| Its own `configs\` and `appsettings.json` | read/write |
| Its own bundled resources (interface, car list, icon) | read, from inside the executable |

That is the complete list. **No file inside the RaceRoom installation folder is
opened at all** — not for reading, not for writing. The application does not know
where the game is installed and never asks.

---

## How the telemetry is read

RaceRoom publishes a shared-memory block for third-party tools. This is a
documented, supported interface — the same one dashboards and telemetry apps use.
The application opens it like this:

```c
handle = OpenFileMapping(FILE_MAP_READ, false, "$R3E");
view   = MapViewOfFile(handle, FILE_MAP_READ, 0, 0, 0);
```

`FILE_MAP_READ` is the whole point. The handle is opened **read-only**, so any
attempt to write through it is refused by Windows itself — this is enforced by the
operating system, not by our own code being careful. The application copies bytes
out of that block and never writes a single one back.

**No game files are modified, patched or replaced. No DLL is injected into the
game process. No plugin is installed. No hook is placed in the game.** The
application is a separate process that reads a memory block the game deliberately
publishes.

---

## The overlay window

The overlay is a normal, transparent, always-on-top window of our own, drawn over
the game. It **does not modify the game window**: it does not reparent it, resize
it or move it. It reads the game window's position so it can align itself, and it
returns focus to whatever had it before, so clicking the overlay does not pull you
out of the game.

---

## Force feedback output

The application sends force to the wheel through **DirectInput**, the standard
Windows API, using a single effect:

| | |
|---|---|
| Effect | **Constant Force** |
| Axis | **X** (the steering axis) |
| Direction | `0` |
| Duration | Infinite — created at start, magnitude updated continuously |
| Update rate | Up to ~400 Hz |
| On exit | Magnitude set to **0**, effect stopped, device released |

Only the **X axis** is driven, because that is the steering axis — the only one a
wheel exposes for force feedback. Pedals, handbrakes and button boxes are never
touched, even when they belong to the same device.

This is exactly the mechanism the game itself uses for its own force feedback,
which is precisely why **RaceRoom's own FFB must be turned off**: two programs
sending constant-force effects to the same axis will fight each other. That is not
a conflict between our application and your hardware — it is two sources
commanding the same actuator.

On exit — and if the application crashes — the force is set to zero and the device
is released, so the wheel never stays loaded.

### If your wheelbase software also runs

Some bases have companion software that can also send DirectInput effects. If you
run one of those with its own effects enabled, the same conflict applies. Use the
base software for the settings the application does not touch (rotation, maximum
torque, hardware filters) and let one source drive the force.

---

## Verifying this yourself

None of the above has to be taken on faith:

**Windows Resource Monitor** — open it, find `R3E FFB.exe`, and look at the *Disk*
tab while the application runs. You will see the two folders listed above and
nothing else.

**Process Monitor** (Sysinternals) — filter by `Process Name is R3E FFB.exe` and
`Operation is WriteFile`. Every write lands in `%LOCALAPPDATA%\R3E_FFB`, in
`Documents\RRE_FFB_Logs` when logging, or next to the executable.

**Check the game folder** — note the modification dates of the RaceRoom
installation before and after running the application. They do not change.

**Steam** — "Verify integrity of game files" reports nothing wrong after using
this application, because nothing in the installation was touched.

---

## If something looks wrong with your wheel

Force feedback problems are almost always **two sources driving the same axis**.
Before suspecting file corruption:

1. Confirm RaceRoom's own FFB is **off** (Controls → Force Feedback)
2. Check whether your wheelbase software has its own effects or game profile active
3. Close this application entirely and test the game alone
4. Reintroduce one source at a time

If a problem persists with this application closed, it does not come from here —
and the sections above explain why it cannot: nothing outside your own settings
folder is ever written.

Reports are welcome on [Discord](https://discord.gg/JGypJavdj), especially with
the base model and what the wheel actually did.
