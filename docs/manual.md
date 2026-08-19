# Manual — R3E Custom FFB

> Generated from the application's own texts by `tests/scripts/gerar-manual.mjs`.
> Every description below is the exact wording shown in the program, so the manual
> cannot drift away from the software.

## Contents

- [How the FFB is built](#how-the-ffb-is-built)
- [The effects](#the-effects)
- [Settings](#settings)
- [Where things are stored](#where-things-are-stored)

---

## How the FFB is built

Before the individual descriptions, it helps to know **how the effects interact**,
because two settings that look unrelated often affect each other.

The order is: each effect is computed from telemetry, shaped by its own linearity
and mix, and only then combined. The combination is not a plain sum —
**understeer multiplies** the front-end group (self-aligning torque, lateral G,
gyroscopic, oversteer), so they fade together as the front loses grip. Weight
transfer is faded harder still, down to zero. Suspension and downforce stay
outside that chain and are purely additive.

Only after everything is combined does the output chain run:

    equalizer -> slew limiter -> low-pass -> global linearity
    -> clamp -> invert -> Output FFB scale

This is why the equalizer cannot isolate a single effect: it acts on the sum. And
why Output FFB is the right control for "too strong overall", but the wrong one
for fixing the balance between effects.

Three consequences worth keeping in mind while reading the sections below:

- **Understeer is a global control.** Raising its Mix quietens four other effects.
- **A quiet effect may not be its own fault** — understeer may be scaling it down.
- **Linearity is applied before Mix**, so Mix changes how much you feel, never the
  character of the effect.

📖 **[Full pipeline walkthrough with pseudo-code](how-it-works.md)** — every stage,
the exact filter order, and the combination formula.

---

## The effects

Each effect is derived from telemetry and mixed into the final force. The **Mix**
column in the main table sets how much of it reaches the wheel; **Linearity**
reshapes its response curve; the remaining columns are specific to each effect.

Two of them are **dampers** rather than forces: Weight Transfer and Downforce
resist *moving* the wheel instead of pushing it in a direction. Understeer is
**multiplicative** — it scales what is already there instead of adding to it.

### Self-Aligning Torque

Self-Aligning Torque — the force with which the front tyres try to straighten the wheel. It is your main cue for front grip: it builds with lateral load and fades as the tyre passes its slip peak.

**How it is computed**

```
lateral force ÷ reference G × pneumatic trail (peaks at the slip angle, then decays) × tyre load × speed fade.
```

---

### Lateral G-Force

Lateral G-Force — the raw sideways acceleration of the car, fed straight to the wheel. Simpler and more immediate than SAT: it tells you how hard the car is cornering, not how close the tyre is to letting go.

**How it is computed**

```
lateral acceleration from the chassis, scaled by mix and held between physics frames.
```

---

### Weight Transfer

Weight Transfer — a damper that resists MOVING the wheel under braking, when load shifts onto the front axle. It does not pull towards any angle; it just makes the wheel heavier to turn, like a loaded front end.

**How it is computed**

```
−gain × normalised steering speed (EMA) × brake gate.
```

---

### Gyroscopic

Gyroscopic — the resistance the spinning front wheels put up against being steered, growing with speed and yaw rate. It is what makes the wheel feel planted at high speed and light when the car is slow.

**How it is computed**

```
average front wheel rotation × yaw rate × inertia constant.
```

---

### Oversteer

Oversteer — a predictive cue for the rear letting go. It rises BEFORE the car actually rotates, by combining rear grip loss with the rotation difference between the two rear wheels (asymmetric wheelspin, typical of throttle oversteer).

**How it is computed**

```
rear grip loss × weight + rear wheel speed difference × weight, filtered and signed by yaw direction.
```

---

### Understeer

Understeer — compares the yaw rate the steering angle ASKS for (Ackermann) with the yaw the car actually delivers. When the front washes out, the wheel goes light — the same cue you get in a real car.

**How it is computed**

```
(expected yaw − actual yaw), normalised and gated by front grip; applied multiplicatively, so it attenuates the whole force.
```

---

### Slow Suspension

Slow suspension band: body roll/pitch and weight sway (low frequency). Additive texture.

---

### Fast Suspension

Fast suspension band: bumps and kerbs (high frequency). Additive texture.

---

### Downforce

Downforce — an aerodynamic damper: the faster you go, the heavier the wheel gets. Scaled by the car's real downforce, so it is exactly zero when stopped and saturates at formula-car load.

**How it is computed**

```
−gain × normalised steering speed (EMA) × clamp(downforce ÷ reference).
```


---

## Settings

### Output chain

| Setting | What it does |
|---|---|
| `invert ffb` | Invert FFB — Signal polarity Reverses the direction of the base FFB signal and additive effects. Use if your wheel pulls in the wrong direction in corners. Note: G-Forces are automatically corrected when this is active to maintain physical coherence. |
| `linearity` | FFB Linearity Curve — Response shaping Adjusts the mapping between the raw FFB signal and the wheel output. 0 = linear (no change) − = gamma curve: compresses light forces, preserves peaks (reduces the 'numb center' feel) + = log curve: boosts light forces, adds detail at low steering angles Range: −0.5 to +0.5 |
| `output rate` | Output Rate — SetForce call frequency Controls how often the force value is sent to the wheel hardware. Auto = matches R3E telemetry update rate (~346 Hz) 512 Hz = upsampled with linear interpolation 900 Hz = maximum rate — ⚠ high CPU load, use only on powerful systems Higher rates can feel smoother on direct-drive wheels. |
| `post slew hz` | Post-SRL Low Pass Filter — Output smoothing Applied after the Slew Rate Limiter. Removes high-frequency noise left by the SRL without affecting lower frequencies. 0 Hz = bypass (disabled) 60–190 Hz = useful range Lower values = smoother but more delayed output. Nyquist limit at pipeline rate (~200 Hz): values ≥200 Hz auto-bypass. |
| `post slew q` | Q Factor — Filter resonance Controls the resonance of the Post-SRL low-pass filter. 0.707 = Butterworth (maximally flat, no peak) — recommended <0.5 = overdamped (very gradual rolloff) >1.0 = underdamped (slight peak near cutoff, adds 'edge') Range: 0.3 – 1.5 |
| `reaction mode` | Reaction mode — latency frame budget applied to all effects. Low Latency=0 (max response) · Directo=3 (recommended) · Médio=4 · Suave=5. Saved globally. |
| `slew rate` | Slew Rate Limiter — Spike protection Limits the maximum change in the FFB signal between frames. 0% = off — full dynamics, no limiting 100% = maximum smoothing (~50 ms for a full sweep) Use to tame sudden sharp spikes (kerbs, bumps) without dulling sustained cornering forces. |

### Self-aligning torque

| Setting | What it does |
|---|---|
| `sat os atten` | How much SAT is reduced when Oversteer is present |
| `sat us atten` | How much SAT is reduced when Understeer is present |

### Lateral and vertical G

| Setting | What it does |
|---|---|
| `glat enabled` | G Lateral — Enable lateral G-force effect Injects lateral acceleration (AccelX) into the FFB signal. Simulates the steering resistance change felt in corners as lateral G increases. Signal: AccelX ÷ 20 m/s² (normalized to ±1 at ~2G) Filter: low-pass at Freq LP before injection. |
| `glat freq lp` | G Lateral Freq LP — Low-pass cutoff Filters the AccelX signal before injection, removing high-frequency noise from the lateral acceleration data. Lower = smoother, only sustained cornering forces Higher = faster response, more variation Default: 10 Hz. Typical range: 2–15 Hz. Note: uses SM update rate (~100 Hz) as the filter sample rate. |
| `glat gain` | G Lateral Gain — Injection strength Scales how much lateral G contributes to the FFB signal within the budget allocated to this effect. 0% = effect disabled 100% = full budget used Note: overall contribution also depends on the Lv.1 budget bar. |
| `gvert enabled` | G Vertical — Enable vertical G-force effect Injects vertical acceleration (AccelY) into the FFB signal. Simulates weight transfer over bumps and kerbs through the steering column. Signal: AccelY ÷ 15 m/s² (normalized to ±1 at ~1.5G) Filter: low-pass at Freq LP before injection. |
| `gvert freq lp` | G Vertical Freq LP — Low-pass cutoff Filters the AccelY signal before injection. Default: 15 Hz. Typical range: 2–20 Hz. Note: uses SM update rate (~100 Hz) as the filter sample rate. |
| `gvert gain` | G Vertical Gain — Injection strength Scales how much vertical G contributes to the FFB signal within the budget allocated to this effect. 0% = effect disabled 100% = full budget used |

### Suspension

| Setting | What it does |
|---|---|
| `susp aa fast` | AA Fast — Anti-alias fast cutoff Low-pass filter applied to the fast suspension component before capping. Removes high-frequency noise that could create harsh artifacts at the wheel. Default: 3.5 Hz. Range: 1–10 Hz. |
| `susp blend` | Rear blend (RearBlend%, 0–50%). How much of the rear axle average is mixed into the front: raw = (front + k·rear)/(1+k), k = %/100. 0% = front only. |
| `susp compress k` | Compress k — Tanh soft-clip strength Applies a tanh() soft-clipping curve to the suspension signal before injection. Prevents hard clipping at ±1. 0 = bypass (linear, no compression) 1.5 = moderate compression (default) 3.0 = heavy compression (strong limiting) Higher k = more compressed, less dynamic range. |
| `susp damp` | Extra dampening (Damp%). Adds a low-pass after the band: fc = FreqHi·(1−p) + 2·p, p = %/100. 0% = transparent, 100% = very soft (~2 Hz). Raise if the band feels too harsh. |
| `susp ema fc` | EMA Fc — Slow component cutoff Cutoff frequency of the exponential moving average (EMA) that extracts the 'slow' suspension movement (body roll). Lower = smoother, captures only chassis lean Higher = faster, includes more suspension detail Default: 5 Hz. Useful range: 2–10 Hz. |
| `susp enabled` | Suspension Effects — Enable/disable Adds suspension velocity (wheel travel speed) as an additive effect to the base FFB signal. When enabled, bumps and road texture are felt through the steering column independently of the base rack force. |
| `susp fast cap` | Fast Cap — Fast component limit Clamps the fast suspension component (wheel transients) to this fraction of the signal range. Controls impact intensity from kerbs and sharp bumps. Default: 12% (0.12). Range: 0–100%. |
| `susp fast inf` | Fast Inf — Fast component injection Scales how much of the fast suspension component (wheel transients) is mixed into the final signal. Default: 40% (0.40). Range: 0–100%. |
| `susp freqHi` | Band-pass high cut (low-pass @ FreqHi). Sets the upper limit of the band — together with FreqLo defines its 'colour'. FreqHi ≥ 190 Hz = no low-pass. |
| `susp freqLo` | Band-pass low cut (high-pass @ FreqLo). Sets where the band starts: SLOW = low ondulations/body movement, FAST = high texture. FreqLo ≤ 0 = no high-pass. |
| `susp lin` | Band linearity (response shaping). Reshapes the band signal before output: − = compress light, keep peaks; + = boost light detail. 0 = linear. |
| `susp minact` | Activation deadzone (MinAct%). Minimum signal to pass: |x|<t → 0, else sign·(|x|−t)/(1−t), t = %/100. Raise to cut micro-noise/chatter, let only significant impacts through. |
| `susp mix` | Band weight (Mix%). Final gain of the band in the additive mix (SLOW and FAST summed). Pure additive, outside the US chain. Default 0% (inert until enabled). |
| `susp slew` | Slew rate limiter (anti-spike). Limits the per-frame change of the band's output to tame sudden impacts. 0% = off, full dynamics. |
| `susp slow cap` | Slow Cap — Slow component limit Clamps the slow suspension component (body movement) to this fraction of the signal range. Prevents chassis lean from dominating the FFB signal. Default: 20% (0.20). Range: 0–100%. |
| `susp slow inf` | Slow Inf — Slow component injection Scales how much of the slow suspension component (body roll) is mixed into the final signal. Default: 35% (0.35). Range: 0–100%. |

### Vibration

| Setting | What it does |
|---|---|
| `vib amp` | Left = minimum activation · right = amplitude |
| `vib dir` | Rising / falling amplitude |
| `vib enable` | Enable vibration |
| `vib range` | Value / range (Start–End on the same bar) |
| `vib range toggle` | Toggle value / range (Start–End on the same bar) |

### Equalizer

| Setting | What it does |
|---|---|
| `eq b1` | EQ Band 1 — 5 Hz Boosts or cuts the FFB signal around 5 Hz. Very slow forces: chassis sway, long-radius yaw, slow understeer. Rarely needs adjustment — use carefully. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 5 Hz) |
| `eq b10` | EQ Band 10 — 160 Hz Highest adjustable band. Ultra-fine surface detail and fast oscillations near the fixed LP cutoff at 180 Hz. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 160 Hz) |
| `eq b2` | EQ Band 2 — 10 Hz Boosts or cuts the FFB signal around 10 Hz. Slow weight transfer, gradual cornering load buildup. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 10 Hz) |
| `eq b3` | EQ Band 3 — 15 Hz Boosts or cuts the FFB signal around 15 Hz. Cornering forces and moderate weight transfer. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 15 Hz) |
| `eq b4` | EQ Band 4 — 20 Hz Boosts or cuts the FFB signal around 20 Hz. Sustained cornering loads and lateral G felt through the steering column. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 20 Hz) |
| `eq b5` | EQ Band 5 — 25 Hz Boosts or cuts the FFB signal around 25 Hz. Transition between sustained and dynamic forces. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 25 Hz) |
| `eq b6` | EQ Band 6 — 30 Hz Boosts or cuts the FFB signal around 30 Hz. Suspension bumps, road undulations and mid-frequency road surface feedback. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 30 Hz) |
| `eq b7` | EQ Band 7 — 50 Hz Boosts or cuts the FFB signal around 50 Hz. Kerb strikes, sharp bumps and the impact character of road imperfections. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 50 Hz) |
| `eq b8` | EQ Band 8 — 80 Hz Boosts or cuts the FFB signal around 80 Hz. Fine suspension detail and medium-high frequency impacts from surface texture. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 80 Hz) |
| `eq b9` | EQ Band 9 — 130 Hz Boosts or cuts the FFB signal around 130 Hz. Fine road texture, high-frequency vibrations and surface grain detail. +dB amplifies · −dB attenuates · 0 dB = bypass Q = 1.5 (affects roughly ±1 octave around 130 Hz) |
| `eq gap q` | Gap Q · lower Q = wider bell = shallower dip between bands |

### Detail levels

| Setting | What it does |
|---|---|
| `lv1 budget` | Lv.1 Budget Bar — Shared output range The 4 segments share the full output range [−1, +1]. Drag the dividers to redistribute the budget. Each effect's contribution is scaled by its budget fraction: Base FFB × (budget%) contributes to the sum Effects are injected within their allocated share Minimum 3% per segment. Total always = 100%. |
| `lv1 klat` | k_lat — G Lateral cross-attenuation Reduces suspension contribution when lateral G is high. Physical basis: in loaded cornering the suspension variation is already captured in the lateral G signal. 0 = no attenuation (default, same as current behavior) 0.5 = G lat at 1.0 reduces suspension by 50% 1 = G lat at 1.0 fully suppresses suspension Formula: suspWeight = clamp(1 − k_lat × |gLat| − k_vert × |gVert|, 0, 1) |
| `lv1 kvert` | k_vert — G Vertical cross-attenuation Reduces suspension contribution when vertical G is high. Physical basis: a kerb hit is already expressed in the vertical G signal — adding full suspension on top would double-count the same event. 0 = no attenuation 0.3 = G vert at 1.0 reduces suspension by 30% (default) 1 = G vert at 1.0 fully suppresses suspension Formula: suspWeight = clamp(1 − k_lat × |gLat| − k_vert × |gVert|, 0, 1) |
| `lv2 attack hz` | EMA Attack Hz — Onset speed Controls how quickly the US/OS effect responds when grip loss begins (score rising). Higher = faster onset, more immediate response Lower = slower onset, more gradual buildup Default: 12 Hz (~83 ms to respond) Physical basis: grip loss happens abruptly — fast attack is realistic. |
| `lv2 os enable` | Oversteer — Enable/disable When enabled, adds a directional force in the direction of vehicle rotation when rear grip is lost. Simulates the steering 'loading up' during a snap oversteer. Formula: rawValue += sign(YawRate) × smoothedOsScore × OsGain Score: yawAccel×0.5 + rearSlip×0.3 + gripLossRear×0.2 |
| `lv2 os gain` | Oversteer Gain — Maximum addition Controls how much directional force is added at maximum oversteer score. 0 = no effect 0.5 = score 1.0 → adds ±0.5 to the signal 1 = score 1.0 → adds ±1.0 (maximum) Amplifies G Lateral that already points in the same direction. |
| `lv2 release hz` | EMA Release Hz — Recovery speed Controls how slowly the US/OS effect fades when grip is recovered (score falling). Higher = faster recovery Lower = slower fade, longer 'hangover' after grip recovery Default: 1 Hz (~1 s to decay) Physical basis: grip recovery is progressive — slow release is realistic. |
| `lv2 us enable` | Understeer — Enable/disable When enabled, reduces the total FFB signal proportionally to front grip loss. Simulates the wheel going light when the front tyres lose traction. Formula: rawValue *= (1 − smoothedUsScore × UsGain) Score: gripLossFront × steeringAngleFactor |
| `lv2 us gain` | Understeer Gain — Maximum attenuation Controls how much the signal is reduced at maximum understeer score. 0 = no effect 0.5 = score 1.0 → signal reduced by 50% 1 = score 1.0 → signal zeroed completely Acts on the full signal: Base FFB + G-Forces + Suspension. |

### Overlay

| Setting | What it does |
|---|---|
| `frame compo` | Show/hide the FFB Composition panel (effects) |
| `frame eq` | Show/hide the Equalizer panel |
| `frame logger` | Show/hide the Logger panel |
| `frame next` | Next frame |
| `frame output` | Show/hide the Output FFB panel |
| `frame prev` | Previous frame |
| `toggle graph` | Show/hide the FFB composition graph |
| `zoom in` | Zoom in 5% |
| `zoom out` | Zoom out 5% |

### Profiles, logger and shortcuts

| Setting | What it does |
|---|---|
| `load config` | Load a saved profile — available on track |
| `load profile` | Load profile |
| `log raw mode` | 1:1 mode (real output): logs one row per loop tick instead of only on a new input frame — measures the REAL output rate even with static input. |
| `log start ontrack` | Available only with the simulator on track |
| `save profile` | Save effect settings for the current car |
### Other settings

| Setting | What it does |
|---|---|
| `about` | About this app |
| `density mode` | Control density — Basic: Mix only · Intermediate: + Linearity · Advanced: + Vibration, EQ, Post and Graph |
| `discord` | R3E FFB Discord |
| `dyn damp deadzone` | Steering angular-velocity deadzone before Dynamic Damping starts |
| `reset type defaults` | Restore the built-in values for this vehicle type |
| `settings` | Settings |
| `support` | Support the project on Buy Me a Coffee (opens in your browser) |

---

## Where things are stored

| What | Where |
|---|---|
| Global settings, zoom, language, device shortcuts | `appsettings.json`, next to the executable |
| Car and track profiles | `%LOCALAPPDATA%R3E_FFBconfigscars` |
| Terms acceptance | `terms-accepted.json`, next to the executable |
| First-run notice acknowledgement | `notice-ack.json`, next to the executable |
| Logger CSV exports | wherever you choose when exporting |

Nothing is transmitted anywhere. Deleting `terms-accepted.json` makes the terms
screen appear again on the next start.
