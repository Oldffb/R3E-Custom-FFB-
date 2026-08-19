# Changelog

## v0.22.0

**Added**
- **ABS vibration effect** — follows the simulator's own ABS signal, which pulses
  at around 10 Hz on its own. Enable, working frequency (0-100 Hz) and amplitude;
  no linearity, slew or damper, since the underlying signal is a simple on/off
  flag. Ships **disabled**, Mix at 0.
- Car database updated from the official source: **344 -> 356 cars**, **102 -> 105
  classes**, including the DTM 2026 class and the Alpine A110 GT4+.
- FFB profiles for 11 new cars.
- Logger records traction control state, level, setting and the raw integer
  values of the ABS and TC fields.

**Fixed**
- Profile warning was always shown in Portuguese, ignoring the selected language.
- Overlay: the device name was replaced by "No device" whenever translations
  refreshed, even with the wheel connected.

**Known limitation**
- A traction control effect is postponed: RaceRoom documents a "currently active"
  state for TC but never publishes it. Verified with 6332 frames of full throttle
  at maximum TC level.

## v0.21.5

- Licence section added to the terms of use, naming the third-party components
  bundled in the package. Because the text changed, the terms are shown again.
- `LICENSE` and `THIRD-PARTY-NOTICES.md` are now included in the download.
- About screen rewritten; the Buy Me a Coffee button and the QR code now share a
  single box instead of being stacked.

## v0.21.4

- The web content and the application icon now live inside the executable. The
  extracted package went from 36 files to 4.
- Documents that were never meant to be distributed were removed from the package.

## v0.21.3

- Discord link in the footer and in the About screen.
- Buy Me a Coffee button and QR code.
- The version number in the footer opens the About screen.

## v0.21.2

- First-run notice: the application now asks you to confirm that RaceRoom's own
  force feedback has been turned off before it starts.

## v0.21.1

- Language selector inside the terms screen, so the agreement can be read in your
  own language before accepting it.
- Default zoom raised to 120% for new installations.

## v0.21.0

- Terms of use shown on first run, with safety, warranty and liability sections.
- Accepting is recorded locally; declining closes the application.
