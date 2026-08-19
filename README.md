# R3E Custom FFB

**Custom force feedback for RaceRoom Racing Experience.** Free, for Windows.

[![Download](https://img.shields.io/badge/download-v0.22.0-brightgreen?style=for-the-badge)](../../releases/latest)
[![Discord](https://img.shields.io/badge/Discord-join-5865F2?style=for-the-badge&logo=discord&logoColor=white)](https://discord.gg/JGypJavdj)

---

## What it is

R3E FFB, as the name suggests, is an application that combines RaceRoom's native FFB with customisable effects generated from the telemetry shared-memory data. You can reshape the current FFB through a range of adjustments, filters and vibration effects, configurable effect by effect. It also adds tuning options that would not be available in hardware across several manufacturers, and lets you fine-tune some characteristics of the simulator, much like other tools do.

## How it works

This application replaces the simulator's native FFB output by reading the
shared-memory telemetry block that RaceRoom publishes for third-party tools. The
simulator appears to update that data at around **400 Hz**, which is what keeps
the latency close to native FFB. In the **Logger** tab you can export a CSV to
analyse response times and telemetry yourself, but the latency should land
between roughly **2 ms and 10 ms**, depending on your filter settings and whether
the smooth latency mode is selected.

Because the application drives the wheel directly, RaceRoom's own force feedback
**must be turned off** — see [Before you start](#before-you-start).

---

## ⚠️ Safety first

Direct-drive bases can produce enough torque to cause injury. Never run the application with the base unsecured or the wheel free to spin near you. Keep hands, hair and clothing clear while testing, start with reduced output when trying new effect settings, keep the base power switch within reach, and do not let unsupervised users operate it.

---

## Before you start

Before using this application you must turn Force Feedback OFF in RaceRoom, under Controls → Force Feedback, as shown in the images below. Only then is your device free for R3E Custom FFB to use: the application reads the data directly from the shared-memory telemetry, so there is no conflict between the two modes driving the same device.

Once you have made the change, confirm using the button below.

| | |
|---|---|
| ❌ RaceRoom — Force Feedback ON (must be changed) | ✓ RaceRoom — Force Feedback OFF (correct) |

---

## Install

1. Download the latest `.zip` from [Releases](../../releases/latest)
2. Extract it anywhere you like — there is no installer
3. Run **R3E FFB.exe**

The package contains just the executable and a settings file; the .NET runtime is
bundled inside. **Windows 11** already includes everything else. On older Windows
you may need the [WebView2 Runtime](https://developer.microsoft.com/microsoft-edge/webview2/).

On first run you will be asked to accept the terms of use and to confirm that you
have turned RaceRoom's force feedback off. Your settings are stored in
`%LOCALAPPDATA%R3E_FFB`.

---

## Using it

### Basic mode

Pick your wheel under **Settings → Device**, then set **Output FFB** to a level
you are comfortable with. The defaults are deliberately gentle. Leave **Output
Type** on native FFB until you want to start blending in telemetry effects.

### The effects

Each effect is generated from telemetry and mixed into the final signal. The
**Mix** column sets how much of it reaches the wheel.

| Effect | What it does |
|---|---|
| **Self-Align. Torque** | the force with which the front tyres try to straighten the wheel. It is your main cue for front grip: it builds with lateral load and fades as the tyre passes its slip peak. |
| **Lateral G-Force** | the raw sideways acceleration of the car, fed straight to the wheel. Simpler and more immediate than SAT: it tells you how hard the car is cornering, not how close the tyre is to letting go. |
| **Weight Transfer** | a damper that resists MOVING the wheel under braking, when load shifts onto the front axle. It does not pull towards any angle; it just makes the wheel heavier to turn, like a loaded front end. |
| **Gyroscopic** | the resistance the spinning front wheels put up against being steered, growing with speed and yaw rate. It is what makes the wheel feel planted at high speed and light when the car is slow. |
| **Oversteer** | a predictive cue for the rear letting go. It rises BEFORE the car actually rotates, by combining rear grip loss with the rotation difference between the two rear wheels (asymmetric wheelspin, typical of throttle oversteer). |
| **Understeer** | compares the yaw rate the steering angle ASKS for (Ackermann) with the yaw the car actually delivers. When the front washes out, the wheel goes light — the same cue you get in a real car. |
| **Slow Suspension** | Slow suspension band: body roll/pitch and weight sway (low frequency). Additive texture. |
| **Fast Suspension** | Fast suspension band: bumps and kerbs (high frequency). Additive texture. |
| **Downforce** | an aerodynamic damper: the faster you go, the heavier the wheel gets. Scaled by the car's real downforce, so it is exactly zero when stopped and saturates at formula-car load. |

### Equalizer

Six bands (5, 10, 20, 50, 80 and 120 Hz) with an adjustable Q per gap, so you can
cut a resonance or lift a range without touching the effects themselves. The blue
curve behind the bars is the real response, computed the same way the audio filter
computes it.

### Logger

Records every frame to CSV — telemetry, per-effect contributions and the final
output — so you can check response times and see exactly what produced a given
force.

### Overlay

A transparent window over the simulator with the same panels, so you can adjust
things without leaving the car. Frames can be moved, toggled and zoomed, and the
positions are saved per resolution.

### Device shortcuts

Bind buttons, POV hats or keys from any DirectInput device to raise or lower
Blend and Output FFB in 1% or 10% steps, without taking your hands off the wheel.

---

## Documentation

| Document | |
|---|---|
| [How the FFB is built](docs/how-it-works.md) | Pipeline order, effect interaction and filter chain |
| [What it touches](docs/what-it-touches.md) | Every file read or written — and why no game file is modified |
| [Manual](docs/manual.md) | Every effect and every setting, in detail |
| [Terms of use](docs/terms.md) | The agreement shown on first run |
| [Before you start](docs/before-you-start.md) | Turning RaceRoom's FFB off |
| [About](docs/about.md) | Who is behind this |
| [Changelog](CHANGELOG.md) | What changed in each version |
| [Licence](LICENSE) | Free, proprietary, non-commercial |
| [Third-party notices](THIRD-PARTY-NOTICES.md) | Bundled components |

---

## Licence

This application is free but not open source: it is proprietary software licensed for personal, non-commercial use. The package bundles the .NET 8 runtime, SharpDX.DirectInput, YamlDotNet and the WebView2 SDK, each under its own licence — see LICENSE and THIRD-PARTY-NOTICES.md in the download.

---

## Support the project

If the application is useful to you and you would like to contribute, join our Discord or become a supporter through the link below, to help bring updates and new features.

- **Discord** — https://discord.gg/JGypJavdj
- **Buy me a beer** — https://www.buymeacoffee.com/Mic7udIzNy
- **E-mail** — oooldffb@gmail.com

See you on track.

---

<sub>This is an independent, non-commercial project. It is not affiliated with, sponsored by, or endorsed by KW Studios, Sector3 Studios or RaceRoom Racing Experience, nor by any wheelbase manufacturer. All trademarks are the property of their respective owners.</sub>
