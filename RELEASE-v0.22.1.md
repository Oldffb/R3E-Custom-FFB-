# R3E Custom FFB v0.22.1

**Your settings now really survive an update, the FFB Constants screen speaks your
language, and when the wheel cannot be used the application finally says why.**

---

## New — settings and profiles live outside the application folder

Everything that belongs to you moved to `%LOCALAPPDATA%\R3E_FFB`: language, zoom,
window position, the four device shortcuts and the acceptance of the terms.

Until now these sat next to the executable. Extracting a new version into a
different folder lost all of it — the application asked you to accept the terms
again and forgot your wheel shortcuts. The move happens on first run, copies
rather than deletes, and can be repeated without risk.

**This also fixes custom vehicle profiles never being applied.** They were written
to one folder and read from another, so `Save` reported success, the file was
correct, and the profile never showed up or reached the FFB. Profiles already on
disk are picked up automatically — including any you created months ago and
thought were lost.

## New — FFB Constants in your language

The 41 fields and 9 groups of the constants catalogue used to arrive from the
backend in Portuguese, whatever language you had selected. They now go through the
translation system in all five languages.

The field descriptions are deliberately still in Portuguese: they are long
technical texts where a bad translation would mislead more than the original.

## New — error log file

When force feedback fails, the reason is now written to
`%LOCALAPPDATA%\R3E_FFB\logs\r3e-ffb-<date>.log`, kept for seven days.

Before, every device failure was swallowed silently — which is why "it doesn't
work on my machine" was impossible to diagnose from a distance. If you report a
problem, attach this file.

## New — a clear message when the wheel cannot be used

The application now asks Windows for the wheel in exclusive mode only, which is
what force feedback requires, and tells you what happened when that fails: another
program is holding the wheel, the device is not connected, the driver refused, and
so on. Each message says **what happened and what to do about it**, in all five
languages, both when you pick a device by hand and at start-up.

The old silent fallback is gone. It could not have worked anyway: it changed the
mode after the point where the real conflict appears, and force feedback cannot
run in the mode it fell back to — so it produced no feedback and no explanation.

## Fixed

**Vehicle profile categories were saved differently in each language.** A profile
created with the interface in German stored `Tourenwagen` where the Portuguese one
stored `Touring` — the same category, two different values on disk, and the
profile would not match.

**Saving a new profile failed silently.** With the name field empty, the button
simply did nothing. It now says what is missing and puts the cursor there.

**The About screen no longer contacts the internet.** The Buy Me a Coffee button
was loaded from a remote server, which revealed your address to it every time the
screen opened. The image is now bundled and the button works offline.

**The car list wasted half its width.** In FFB Constants, the Name and Profile
columns were roughly twice as wide as the longest text they ever showed. They are
now sized to their content; nothing is truncated.

**Lower overhead while driving.** The live data sent to the interface and the
overlay carried 190 fields, 60 times a second, of which only 96 were ever drawn.
The unused half is no longer sent — **51% fewer bytes per message**, with no
change to what you see, to the Logger or to the CSV export.

---

## Install

Download the `.zip`, extract it anywhere, and run **R3E FFB.exe**. There is no
installer. The .NET runtime is bundled — Windows 11 needs nothing else.

**RaceRoom's own force feedback must be off** (Controls → Force Feedback). The
application shows the instructions on first run.

Your settings are kept in `%LOCALAPPDATA%\R3E_FFB` and survive updates.

**If Windows Firewall asks for permission:** the application opens one port,
**5123/TCP, on loopback only** (`127.0.0.1` and `::1`), which is how its windows
load the interface. It never reaches the internet, and loopback traffic is not
filtered — denying the prompt should not stop it from working.

## Links

- [What it touches](docs/what-it-touches.md) — every file read or written, and why no game file is modified
- [Troubleshooting](docs/troubleshooting.md) — no force feedback, firewall prompt, stuttering, the log file
- [How the FFB is built](docs/how-it-works.md) — pipeline order and the formula behind each effect
- [Before you start](docs/before-you-start.md) — turning native FFB off
- [Discord](https://discord.gg/JGypJavdj) — bug reports and questions
