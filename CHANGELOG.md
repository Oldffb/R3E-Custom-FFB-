# Changelog

## v0.22.1

**Added**
- Settings, device shortcuts, terms acceptance and vehicle profiles moved to
  `%LOCALAPPDATA%\R3E_FFB`, so they survive an update instead of being lost when
  the new version is extracted elsewhere.
- FFB Constants: the 41 fields and 9 groups of the catalogue are translated in all
  five languages (field descriptions remain in Portuguese on purpose).
- Error log file in `%LOCALAPPDATA%\R3E_FFB\logs\`, kept for seven days. Device
  failures used to be swallowed silently.
- A message explaining why force feedback could not start — another program
  holding the wheel, device not connected, driver refusal — with what to do about
  it, in all five languages.

**Fixed**
- Custom vehicle profiles were saved to one folder and read from another, so they
  were never loaded or applied. Existing files are picked up automatically.
- Vehicle profile categories were stored in the interface language, so the same
  category produced different values on disk depending on the language.
- Saving a new profile with an empty name failed silently.
- The About screen loaded the Buy Me a Coffee button from a remote server; the
  image is now bundled and the application no longer contacts the internet.
- FFB Constants: Name and Profile columns were about twice as wide as their
  content.
- Live data sent to the interface and the overlay carried 190 fields of which 96
  were drawn; the unused half is no longer sent (51% fewer bytes per message).

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
