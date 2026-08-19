# How the FFB is built

This page explains **how the effects interact** — the order in which they are
computed, the formula behind each one, and where every filter sits in the chain.
Read it before the per-effect descriptions: knowing the order explains why two
settings that look unrelated affect each other.

Everything here is taken from the pipeline source and the authors' own comments,
so it describes what the application actually does.

---

## Contents

- [The short version](#the-short-version)
- [Stage 1 — Reading the telemetry](#stage-1--reading-the-telemetry)
- [Stage 2 — Computing the effects](#stage-2--computing-the-effects)
- [Stage 3 — Per-effect shaping](#stage-3--per-effect-shaping)
- [Stage 4 — Combining, and where understeer acts](#stage-4--combining-and-where-understeer-acts)
- [Stage 5 — Blending with the native FFB](#stage-5--blending-with-the-native-ffb)
- [Stage 6 — The output chain](#stage-6--the-output-chain)
- [Stage 7 — Output](#stage-7--output)
- [Practical consequences](#practical-consequences)

---

## The short version

```
telemetry ─► normalise with reference constants
          ─► compute each effect independently
          ─► per-effect: normGain ─► linearity ─► (vibration) ─► mix
                                                                  │
              understeer multiplies the others  ◄─────────────────┤  ← last thing
                                                                  ▼      to act
                                                    one combined signal
                                                                  │
                                    blend with the simulator's native FFB
                                                                  │
                       equalizer ─► slew limiter ─► low-pass ─► linearity
                                                                  │
                             clamp ─► invert ─► Output FFB ─► wheel
```

Two things explain most of the surprises:

**Understeer is applied last, not summed.** It is *computed* alongside the other
effects, but it is *applied* at the very end of the effect stage — the final
operation before the native blend. It **multiplies** the others instead of being
added to them.

**Every filter acts on the combined signal.** The equalizer, the slew limiter and
the low-pass never see individual effects. Cutting 50 Hz affects everything with
energy there.

---

## Stage 1 — Reading the telemetry

Each cycle reads RaceRoom's shared memory into a snapshot (`PhysicsState`). These
are the fields the effects actually consume:

| Field | Unit | Used by |
|---|---|---|
| `VelocityKmh` | km/h | speed fades in almost every effect |
| `AccelX` | m/s² | lateral G, self-aligning torque |
| `AccelY` | m/s² | vertical G |
| `AccelZ` | m/s² | weight transfer (deceleration gate) |
| `SteeringAngle` | rad | SAT modulation, weight transfer, downforce |
| `OrientRoll` | rad | banking removal (lateral) |
| `OrientPitch` | rad | road gradient removal (longitudinal) |
| `ChassisSlipAngleDeg` | deg | SAT pneumatic trail |
| `YawRate` | rad/s | understeer, oversteer, gyroscopic |
| `TireGripFL…RR` | 0–1 (−1 = N/A) | understeer / oversteer, via the axle averages |
| `TireRpsFL…RR` | rad/s | gyroscopic, oversteer |
| `TireLoadFL…RR` | N | front load ratio |
| `SuspVelFL…RR` | m/s | suspension bands |
| `BrakePedal` · `BrakePressure` | 0–1 | weight transfer gate |
| `Downforce` | N | downforce gate |
| `SteeringForcePct` | −1…+1 | the native FFB, for the blend |

### Conversions before anything is computed

Raw telemetry is in physical units, and the effects work in normalised −1…+1.
Three kinds of conversion happen first:

**Reference constants.** Each physical quantity is divided by a reference value
that represents "full scale" for that car:

```
latNorm  = AccelX / LatGRef            # LatGRef = 28 m/s² by default (GT3)
longNorm = AccelZ / LongGRef           # LongGRef = 10 m/s²
gVert    = AccelY / 15                 # fixed constant
gyrNorm  = rawTorque / GyrFullscale    # in N·m
usNorm   = yawDelta / UsYawFullscale
```

`LatGRef` is **overwritten by the active vehicle profile**. This is what makes a
GT3 and a formula car feel comparable at the same Mix: the reference changes, not
the formula.

**Derived quantities.** Several names in the formulas below are not telemetry
fields — they are computed first. Every one of them is defined here, so no formula
depends on something unexplained.

```
# ── the two accelerometer versions ────────────────────────────────────────
AccelXClean = AccelX          # currently passed through unfiltered
AccelZClean = AccelZ          # (the low-pass exists in the code but is disabled)

# ── banking removal, used by self-aligning torque ─────────────────────────
rollFiltered = OrientRoll                        if latency mode = LowLatency
             = lowPass(OrientRoll, bankingLpHz)  otherwise
bankRoll     = |rollFiltered| > 0.05 rad ? rollFiltered : 0      # ~3° deadzone

AccelXNet = sign(AccelXClean) × ( |AccelXClean| × cos(bankRoll)
                                  − 9.81 × |sin(bankRoll)| )

# ── road gradient removal, longitudinal ───────────────────────────────────
pitchFiltered = lowPass(OrientPitch, 10 Hz)
AccelZNet     = AccelZClean − 9.81 × sin(pitchFiltered)

# ── front load share, used by self-aligning torque ────────────────────────
fzFront        = TireLoadFL + TireLoadFR
fzTotal        = fzFront + TireLoadRL + TireLoadRR
FrontLoadRatio = fzTotal > 10 N ? fzFront / fzTotal : 0.5     # 0.5 = fallback

# ── axle grip, used by understeer and oversteer ───────────────────────────
GripFL…RR    = TireGripFL…RR < 0 ? 1.0 : clamp(TireGripFL…RR, 0, 1.5)   # −1 = N/A
GripFrontAvg = (GripFL + GripFR) / 2
GripRearAvg  = (GripRL + GripRR) / 2

# ── misc ──────────────────────────────────────────────────────────────────
velMs   = VelocityKmh / 3.6
slipMag = |ChassisSlipAngleDeg|
```

Three of these deserve a note.

**The banking correction is not a plain subtraction.** An earlier version used
`AccelXClean − 9.81·sin(roll)`, which on banked ovals *added* to the signal instead
of removing gravity, because the sign was inverted — it inflated the self-aligning
torque exactly where it should have reduced it. The current form works on
magnitudes and reapplies the original sign, with a ~3° deadzone so that flat track
noise does not trigger a correction at all.

**`FrontLoadRatio` has a guard.** Below 10 N of total load the car is not really on
the ground, and dividing would produce nonsense, so it falls back to 0.5 — an even
front/rear split.

**Grip uses −1 as "not available"**, which is why the normalisation maps negatives
to `1.0` (full grip) rather than clamping them to zero. A car that does not publish
grip therefore reads as *gripping*, never as *sliding* — the safe direction, since
understeer and oversteer both key off grip *loss*.

**Which version of AccelX each effect uses is deliberate.** Lateral G uses
`AccelXClean`, **keeping** banking, because the driver already feels banking relief
through `g·sin(roll)` and removing it would double-count. Self-aligning torque uses
`AccelXNet`, **without** banking, because it models tyre force.

### The latency modes, and the α in the formulas

Several formulas below contain a first-order filter written as
`state += α × (input − state)`. That α is not fixed — it comes from the latency
mode, expressed as a **budget of frames of delay**:

| Mode | Budget | α |
|---|---|---|
| Low Latency | 0 frames | 1.000 — pass-through, same frame |
| Direct | 3 frames | 0.250 |
| Medium | 4 frames | 0.200 |
| Smooth | 5 frames | 0.167 |

Each effect declares its **natural** filter length, and the resolver takes
`min(natural, budget)` — so a filter already faster than the budget is left alone,
and the mode only ever makes things quicker, never slower than designed. Changing
mode mid-session ramps α gradually, so there is no step in the force.

### Two clock rates

The simulator publishes physics at roughly **400 Hz** and the pipeline runs its own
loop, so a new telemetry frame is not available every cycle. Effects that depend on
rate of change only recompute on a genuinely new frame (`newPhysicsFrame`); the
rest run every cycle.

### Native bypass

If the blend is at 100% native, the pipeline **skips every effect computation** —
nothing in Stage 2 or 3 runs at all. A real bypass, not a gain of zero, and the
reason 100% native has the lowest possible latency.

---

## Stage 2 — Computing the effects

Each effect is computed independently from the same `PhysicsState`. None can see
another's output.

### Self-Aligning Torque

Pacejka-inspired, stateless. The force with which the front tyres try to
straighten the wheel:

```
latForceNorm = AccelXNet / LatGRef

# pneumatic trail: peaks at SlipPeak, decays linearly to zero at SlipDecay
if      slipMag <= SlipPeak :  trailPneu = TrailPneu × (slipMag / SlipPeak)
elif    slipMag <= SlipDecay:  trailPneu = TrailPneu × (1 − (slipMag − SlipPeak)
                                                      / (SlipDecay − SlipPeak))
else                     :  trailPneu = 0

trailNorm = (TrailBase + trailPneu) / (TrailBase + TrailPneu)
speedFade = min(1, VelocityKmh / SatSpeedFadeKmh)
loadFactor = FrontLoadRatio / SatFrontLoadRef
steerMod   = 1 + SteerModAmp × tanh(SteeringAngle / SteerModRad)

SAT = −latForceNorm × trailNorm × loadFactor × speedFade × steerMod
```

The negative sign is what makes the wheel return to centre. The trail term is the
important one: it **rises to the slip peak and then falls away**, which is why the
wheel goes light exactly when the front tyre passes its limit.

### Lateral G-Force

The simplest of them all — raw sideways acceleration, held between physics frames:

```
GLat = AccelXClean / LatGRef
```

### Gyroscopic

Precession torque from the spinning front wheels:

```
speedFade = min(1, VelocityKmh / GyrSpeedFadeKmh)
rpsAvg    = (TireRpsFL + TireRpsFR) / 2
rawTorque = rpsAvg × YawRate × GyrInertia        # τ = ω_wheel × ω_yaw × I
Gyr       = (rawTorque / GyrFullscale) × speedFade
```

Felt as extra weight in the steering through fast corners.

### Oversteer

Predictive — it rises *before* the car actually rotates:

```
gripLossRear = 1 − GripRearAvg                        # primary signal
rearDiff     = |TireRpsRL − TireRpsRR|                # asymmetric wheelspin
rearDiffNorm = rearDiff / OsRearSlipFullscale

osRaw   = gripLossRear × OsGripWeight + rearDiffNorm × OsSlipWeight
osLp   += α × (osRaw − osLp)                          # α from the latency mode
yawNorm = −(YawRate / OsYawFullscale)                 # negated: R3E sign convention

OS = osLp × yawNorm × (OsInvert ? −1 : +1)
```

Two sources on purpose: grip loss covers the classic slide, wheel-speed difference
catches throttle oversteer in rear-wheel-drive cars.

### Understeer

Compares the yaw the steering *asks for* with the yaw the car *delivers*:

```
if velMs < UsMinSpeedMs: return (filters advance toward zero)

gripLossFront = 1 − GripFrontAvg
frontGripLp  += αFront × (gripLossFront − frontGripLp)     # ~2 Hz
frontGate     = frontGripLp × UsFrontGateGain

expectedYaw = (velMs / WheelBase) × SteeringAngle × KSteerToYaw   # Ackermann
yawDelta    = max(0, |expectedYaw| − |YawRate|)                   # sub-response

usRaw = (yawDelta / UsYawFullscale) × frontGate
usLp += αUs × (usRaw − usLp)                                # ~4 Hz
```

The `frontGate` matters: a yaw mismatch alone is not understeer — the front axle
must also be **losing grip**. Without the gate, tight low-speed corners would
register as understeer.

### Weight Transfer

A **damper**: it resists *moving* the wheel, it does not push it anywhere.

```
decelMag  = clamp(lowPass(AccelZClean, WtDecelLpHz) / WtDecelRef, 0, 1)
pedalGate = clamp(BrakePedal / WtBrakeEps, 0, 1)
speedFade = clamp((WtStandstillKmh − VelocityKmh) / WtStandstillKmh, 0, 1) × WtParkingScale

gate     = max(decelMag × pedalGate, speedFade)
steerPrev = SteeringAngle of the previous frame
rateEma  += α × ((SteeringAngle − steerPrev) − rateEma)
rateNorm  = clamp(rateEma / WtSteerRateRef, −1, +1)

WT = −WtDamperGain × rateNorm × gate
```

The hybrid gate is deliberate: `decelMag` gives the physical proportion of the
braking effort, while `pedalGate` gives an **instant release** when you lift — the
deceleration filter alone would leave a tail of engine braking. `speedFade` adds
parking effort when stopped.

### Downforce

The same damper idea, but gated by **aerodynamic** load instead of braking:

```
gate     = clamp(Downforce / DownforceRef, 0, 1)      # exactly 0 when stopped
rateNorm = clamp(rateEma / DfSteerRateRef, −1, +1)

DF = −DfDamperGain × rateNorm × gate
```

Because aero is not grip, downforce is **not** killed by understeer — see Stage 4.

### Suspension (two bands)

Both bands run the same eight steps over `SuspVel` from the four wheels:

```
1. input      frontAvg = (SuspVelFL + SuspVelFR) / 2
              rearAvg  = (SuspVelRL + SuspVelRR) / 2
              raw = (frontAvg + k × rearAvg) / (1 + k)      k = RearBlend% / 100
2. bandpass   high-pass at FreqLo → low-pass at FreqHi
3. normalise  x = clamp(bp / NormFs, −1, +1)
4. deadzone   |x| < t → 0, else x = sign × (|x| − t) / (1 − t)
5. dampening  extra low-pass at FreqHi × (1 − p) + 2 × p    p = Damp% / 100
6. linearity  exponent 1 − L
7. slew       limit Δ per frame
8. mix        × Mix% / 100
```

Slow carries body movement, Fast carries kerbs and bumps. The deadzone in step 4
is continuous — it rescales what survives instead of leaving a step at the
threshold.

---

## Stage 3 — Per-effect shaping

Every effect then goes through the same sequence:

```
value = raw × normGain           # calibration, from the vehicle profile
value = linearity(value)         # your per-effect curve, −0.5 … +0.5
value = vibration(value)         # oversteer and understeer only, if enabled
value = value × (mix / 100)
value = clamp(value, −1, +1)     # understeer clamps to 0…1
```

**Linearity is applied before Mix.** The curve shapes the full-scale signal and
the Mix scales the already-curved result, so Mix never changes an effect's
character — only how much of it reaches the wheel.

The vibration overlay exists only on oversteer and understeer, and sits **between**
linearity and mix. That is what produces the slip tremble instead of a smooth
force.

---

## Stage 4 — Combining, and where understeer acts

This is the last thing that happens to the effects, and it is not a plain sum:

```
usMixed   = the understeer signal after Stage 3 (0…1)
usFull    = max(0, 1 − usMixed)          # no floor: can reach exactly zero
usFloored = max(UsFloor, usFull)         # with floor: never fully mutes

signal = (SAT + GLat + Gyr + OS) × usFloored     # front-end group
       + WT × usFull                             # damper, faded harder
       + SuspSlow + SuspFast + DF                # additive, untouched
```

Three different treatments, each deliberate:

**The front-end group** fades together as the front loses grip. `UsFloor` stops
the wheel going completely dead, so it keeps breathing at the limit.

**Weight transfer** uses `usFull`, **without** the floor, so it can reach exactly
zero. Holding a damper while the front is sliding would feel like a mechanical
fault.

**Suspension and downforce stay outside** the understeer chain. A kerb strike is
real regardless of front grip, and aerodynamic load is not grip.

---

## Stage 5 — Blending with the native FFB

```
t       = blendPct / 100
blended = native + (effects − native) × t
```

A linear interpolation: 0% pure native, 100% pure effects, 50% an even mix.

One exception. **Below 50 km/h, and only when the blend is under 50%**, the result
is progressively pulled back toward native as speed approaches zero:

```
if speed < 50 and blendPct < 50:
    wStopped = max(0, 1 − speed / 50)
    blended  = blended + (native − blended) × wStopped
```

This keeps the wheel sane in the pits and on the grid, where several
telemetry-derived effects are meaningless at a standstill.

---

## Stage 6 — The output chain

Only now do the filters act, always in this order, on the **single combined
signal**:

```
x = equalizer(x)
x = slewLimiter(x)
x = lowPass(x)
x = linearity(x)        # global, separate from per-effect linearity
```

### Equalizer

Six peaking filters in series:

```
5 Hz → 10 Hz → 20 Hz → 50 Hz → 80 Hz → 120 Hz
```

A band at 0 dB is bypassed and costs nothing. Each band takes its Q from the gaps
around it, using the **narrower** of the two neighbours, so a sharp setting never
bleeds into a band you left wide.

### Slew rate limiter

Caps how fast the output may change between cycles. Tames spikes, at the cost of
blunting genuinely fast transients.

### Low-pass

Applied **after** the slew limiter, to clean up the steps it leaves behind. 0 Hz
bypasses it.

### Global linearity

A final curve on the whole signal — separate from the per-effect one.

---

## Stage 7 — Output

```
if not finite(x): x = 0
x = clamp(x, −1, +1)
if not onTrack:   x = 0
if inverted:      x = −x
force = x × outputScale        # the Output FFB percentage
```

**Output FFB is the last thing applied**, scaling everything uniformly. It is the
right control for "too strong overall" and the wrong one for fixing the balance
between effects.

---

## Practical consequences

**Order of tuning.** Output FFB first, then the blend, then per-effect Mix, and
only then the equalizer. Backwards means retuning the equalizer every time you
move an earlier control.

**Understeer is a global control.** Raising its Mix quietens four other effects at
once, and kills weight transfer entirely. If everything went quiet through corners,
look here first.

**The equalizer cannot separate effects.** It acts on the sum. To cut vibration
from one effect, use that effect's own settings.

**A silent effect may not be its own fault.** Self-aligning torque disappearing
mid-corner may be understeer multiplying it down.

**Vehicle profiles change the references, not the formulas.** If a car feels wrong
at settings that work elsewhere, the profile's reference values are the place to
look.

**Filters cost latency.** The slew limiter and the low-pass both delay the signal.
Direct leaves them mostly out of the way; the smoother modes trade delay for
smoothness.
