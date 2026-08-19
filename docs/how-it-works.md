# How the FFB is built

This page explains **how the effects interact** — the order in which they are
computed, how they are combined, and where each filter sits in the chain. Read it
before the per-effect descriptions: knowing the order explains why two settings
that look unrelated affect each other.

Everything here is taken from the pipeline source, not from a design document, so
it describes what the application actually does.

---

## The short version

```
telemetry ─► compute each effect ─► apply linearity ─► apply mix
                                                          │
                              understeer modulates ◄───────┤
                                                          ▼
                                            sum into one signal
                                                          │
                          blend with the simulator's native FFB
                                                          │
                    equalizer ─► slew limiter ─► low-pass ─► linearity
                                                          │
                              clamp ─► invert ─► output scale ─► wheel
```

Two things are worth noticing straight away, because they explain most of the
surprises:

**The effects are not independent.** Understeer is not added to the signal — it
*multiplies* the others. When understeer rises, self-aligning torque, lateral G,
gyroscopic and oversteer all fade together, which is what a real front-end losing
grip feels like.

**Everything is mixed before the output chain.** The equalizer, the slew rate
limiter and the low-pass act on the **final combined signal**, not on individual
effects. Cutting 50 Hz in the equalizer affects every effect that has energy
there, not just the one you had in mind.

---

## Stage 1 — Reading the telemetry

Each cycle starts by reading RaceRoom's shared memory into a snapshot: speed,
accelerations, steering angle, slip angle, tyre loads and wheel angular speeds,
grip front and rear, suspension velocities, brake pressure, downforce, yaw rate.

Two rates coexist here. The simulator publishes physics at roughly **400 Hz**, and
the pipeline runs its own loop, so a new telemetry frame is not available on every
cycle. Effects that depend on rate of change only recompute when a genuinely new
frame arrives (`newPhysicsFrame`); the rest run every cycle.

**Native FFB bypass.** If the blend is set to 100% native, the pipeline **skips
every effect computation** — no effect is calculated at all. This is not just a
gain of zero: it is a real bypass, and it is why 100% native has the lowest
possible latency.

---

## Stage 2 — Computing the effects

With the snapshot ready, each effect is computed independently. None of them can
see the others' output; they all read the same physics state.

```
satRaw   = SelfAligningTorque(state)
gLatRaw  = LateralG(state)
wtRaw    = WeightTransfer(state)
gyrRaw   = Gyroscopic(state)
osRaw    = Oversteer(state)
usRaw    = Understeer(state)
dfRaw    = Downforce(state)
suspSlow, suspFast = Suspension(state, dt)
```

Each result is a normalised value, mostly in the range −1 to +1. Understeer is the
exception: it produces **0 to 1**, because it is a *loss* factor rather than a
force.

---

## Stage 3 — Per-effect shaping

Every effect goes through the same three steps, in this order:

```
value = raw × normGain          # calibration, per vehicle profile
value = linearity(value)        # your per-effect curve, −0.5 … +0.5
value = value × (mix / 100)     # your per-effect Mix
value = clamp(value, −1, +1)
```

The order matters. **Linearity is applied before Mix**, so the curve shapes the
full-scale signal and the Mix scales the already-curved result. Changing Mix
therefore never changes the *character* of an effect, only how much of it reaches
the wheel.

`normGain` comes from the active vehicle profile and is what makes a GT3 and a
formula car feel comparable at the same Mix setting.

**Oversteer and understeer have an extra step.** After linearity and before Mix,
an optional vibration is layered onto them — this is what produces the slip
tremble rather than a smooth force:

```
osNorm = vibration(linearity(osRaw × normGain))
usNorm = vibration(linearity(usRaw × normGain))
```

---

## Stage 4 — Combining

This is where the interaction happens. The combination is **not** a plain sum:

```
usFull    = max(0, 1 − usMixed)          # no floor: can reach zero
usFloored = max(usFloor, usFull)         # with floor: never fully mutes

signal = (sat + gLat + gyr + os) × usFloored     # front-end group, faded by understeer
       + wt × usFull                            # weight transfer, faded harder
       + suspSlow + suspFast + df                # additive, untouched by understeer
```

Three different treatments, and each is deliberate:

**The front-end group** — self-aligning torque, lateral G, gyroscopic and
oversteer — is multiplied by `usFloored`. As the front loses grip, these fade
together. The floor (`usFloor`) stops the wheel going completely dead, so it keeps
"breathing" at the limit instead of falling silent.

**Weight transfer** is multiplied by `usFull`, **without** the floor. It can reach
exactly zero. Weight transfer is a damper that resists moving the wheel, and
holding it while the front is sliding would feel like a mechanical fault.

**Suspension and downforce are purely additive.** They stay outside the understeer
chain entirely, because a kerb strike or aero load is real regardless of front
grip.

---

## Stage 5 — Blending with the native FFB

The combined signal is now mixed with the simulator's own force:

```
t       = blendPct / 100
blended = native + (effects − native) × t
```

A plain linear interpolation: 0% is pure native, 100% is pure effects, 50% is an
even mix.

One exception exists. **Below 50 km/h, and only when the blend is under 50%**, the
result is progressively pulled back toward native as speed approaches zero:

```
if speed < 50 and blendPct < 50:
    wStopped = max(0, 1 − speed / 50)
    blended  = blended + (native − blended) × wStopped
```

This keeps the wheel behaving normally in the pits and on the grid, where several
telemetry-derived effects are meaningless at a standstill.

---

## Stage 6 — The output chain

Now — and only now — the filters act, always in this order, on the **single
combined signal**:

```
x = equalizer(x)        # 6 peaking bands
x = slewLimiter(x)      # maximum rate of change
x = lowPass(x)          # if enabled
x = linearity(x)        # global curve, separate from per-effect linearity
```

### Equalizer

Six peaking filters in series, at fixed frequencies:

```
5 Hz → 10 Hz → 20 Hz → 50 Hz → 80 Hz → 120 Hz
```

A band at 0 dB is bypassed, costing nothing. The Q of each band is taken from the
gaps around it, using the **narrower** of the two neighbours, so a sharp setting
never bleeds into a band you left wide.

### Slew rate limiter

Caps how fast the output may change between cycles. It tames sudden spikes, at the
cost of blunting genuinely fast transients.

### Low-pass

Applied **after** the slew limiter, to clean up the steps the limiter leaves
behind. The cutoff comes from the web interface setting; if that is off, the
legacy setting is used. 0 Hz bypasses it.

### Global linearity

A final curve on the whole signal. This is **separate** from per-effect linearity:
that one shapes a single effect before mixing, this one shapes everything after.

---

## Stage 7 — Output

```
if not finite(x): x = 0
x = clamp(x, −1, +1)
if not onTrack: x = 0
if inverted:    x = −x
force = x × outputScale        # the Output FFB percentage
```

The clamp is also a safety guard: a non-finite value becomes zero rather than
reaching the wheel.

**Output FFB is the last thing applied.** It scales everything uniformly, which is
why it is the right control for "too strong overall" — and the wrong one for
fixing the balance between effects.

---

## Practical consequences

**Order of tuning.** Set Output FFB first, then the blend, then per-effect Mix,
and only then the equalizer. Doing it backwards means retuning the equalizer every
time you move an earlier control.

**Understeer is a global control.** Raising its Mix quietens four other effects at
once. If everything went quiet through corners, look here before touching each
effect.

**The equalizer cannot separate effects.** It acts on the sum. To cut vibration
from a single effect, use that effect's own settings.

**A silent effect may not be its own fault.** If self-aligning torque disappears
mid-corner, the cause may be understeer multiplying it down — not the SAT settings.

**Filters cost latency.** The slew limiter and the low-pass both delay the signal.
The Direct latency mode leaves them mostly out of the way; the smoother modes trade
delay for smoothness.
