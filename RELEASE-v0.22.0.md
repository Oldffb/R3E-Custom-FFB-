# R3E Custom FFB v0.22.0

**New ABS vibration effect and updated car list.**

---

## New — ABS vibration

A vibration in the wheel while the anti-lock system is releasing the brakes, so
you can feel the limit of braking without looking at anything.

The effect **follows the simulator's own ABS signal** instead of running a fixed
oscillator. Measured across three logs and two cars, that signal pulses on its own
at around **10 Hz** — the cadence of a real ABS — with pulses of 14 to 35 ms. The
effect inherits that rhythm, so a light intervention feels different from a heavy
one.

The new row sits below Fast Suspension and has only three controls: **enable**,
**working frequency** (0–100 Hz) and **amplitude**. There is no linearity, slew,
actuation frequency or damper — the underlying signal is a simple on/off flag, so
none of those would mean anything.

**It ships disabled, with Mix at 0.** Nothing changes for you until you turn it
on. Start with a low Mix and raise it until the vibration is informative without
masking the road.

## Updated car list

The car database was refreshed from the official source: **344 → 356 cars** and
**102 → 105 classes**, including the whole **DTM 2026** class and the **Alpine
A110 GT4+**. Cars that previously showed as unknown now have their proper names.

Eleven cars also received FFB profiles. The seven DTM 2026 entries are mapped as
GT3, following the rule already used throughout the file: **a car is classified by
what it is, not by the championship it races in** — the same as the existing Audi
and BMW GT3 DTM entries.

## Fixed

**Profile warning was always in Portuguese.** The message shown when a car has no
defined profile ignored the selected language. It is now translated in all five
languages.

**Overlay: device name disappeared.** The device name was replaced by "No device"
whenever the interface refreshed translations, even with the wheel connected and
working.

---

## Install

Download the `.zip`, extract it anywhere, and run **R3E FFB.exe**. There is no
installer. The .NET runtime is bundled — Windows 11 needs nothing else.

**RaceRoom's own force feedback must be off** (Controls → Force Feedback). The
application shows the instructions on first run.

Your settings are kept in `%LOCALAPPDATA%\R3E_FFB` and survive updates.

## Links

- [What it touches](docs/what-it-touches.md) — every file read or written, and why no game file is modified
- [How the FFB is built](docs/how-it-works.md) — pipeline order and the formula behind each effect
- [Before you start](docs/before-you-start.md) — turning native FFB off
- [Discord](https://discord.gg/JGypJavdj) — bug reports and questions
