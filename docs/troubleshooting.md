# Troubleshooting

Start here if something is not working. The problems below are ordered by how
often they turn out to be the cause.

---

## No force feedback at all

### 1. Is a wheel selected?

Settings → Device. Without a device selected there is no force feedback, and this
is the most common cause by a wide margin.

It is worth checking even if you selected one before. On first run Windows may
show the firewall prompt (see below), and that dialog **takes focus** — if it
appeared while you were choosing your wheel, the selection may not have gone
through.

### 2. Is RaceRoom's own force feedback off?

Controls → Force Feedback, in the game. Both programs send *constant force* to
the same axis, and two sources commanding one motor fight each other. The
application shows these instructions on first run.

### 3. Is another program holding the wheel?

Wheelbase companion software, SimHub, or another FFB tool can hold the device in
**exclusive mode**. Force feedback requires exclusive access, so when that happens
the application cannot drive your wheel at all.

Since v0.22.1 it **tells you so**, in your own language, with the reason and what to
do about it. Earlier versions fell back to a shared mode that looked like it had
worked and silently produced no force — which is why some reports said the wheel
"just stopped working" with no explanation.

Settings also shows the mode actually in use:

| Shown | Meaning |
|---|---|
| `Exclusive+Background` | normal — the application has the wheel and keeps it while you drive |
| `(falhou)` | acquisition failed; the message on screen names the cause |

Close the other program **and restart R3E Custom FFB** — the mode is decided when
the device is acquired, not continuously.

### 4. It worked, then stopped mid-session

The device can be lost when Windows shows a UAC prompt, when the screen locks, or
when another program grabs it. The application detects this and re-acquires
automatically. If it does not come back, restart the application.

---

## The firewall prompt on first run

Windows may ask to *"allow access"* the first time you run the application.

**This is not about your wheel.** Force feedback goes through DirectInput, which
talks to the device driver and never touches the network. **The firewall cannot
block a USB device.**

The prompt appears because the interface you see is a web page, served by the
application to itself over a **local port**:

| | |
|---|---|
| Port | **5123** (TCP) |
| Address | `127.0.0.1` and `::1` — **loopback only** |
| Reaches the internet | **no** |

Loopback traffic is not filtered by Windows Firewall, so **denying the prompt
should not stop the application from working**. If you did deny it and something
feels wrong, the likely cause is the interruption itself — the dialog is modal and
steals focus, so it may have cut short whatever you were doing, such as selecting
your wheel.

If port 5123 is already taken by another program, pass a different one as a
command-line argument.

---

## Stuttering while driving

If the game stutters with the overlay enabled but is smooth without it, the
overlay is the cause. It is a second browser window composited over the game, and
on a GPU already busy with the simulator that costs frames.

Options, in order of how much they help:

- **Turn the overlay off** and keep the main window on a second monitor
- Lower the overlay zoom, or hide the frames you are not watching
- Close other overlays running at the same time

---

## The wheel jolts when ABS engages

The ABS effect should vibrate, not pull. If it pulls the wheel to one side during
braking, you are on **v0.22.0** — this was fixed in v0.22.1. Update.

---

## Settings are not remembered

Settings live in `%LOCALAPPDATA%\R3E_FFB`. If they are not being kept:

- Check that the folder exists and is writable
- If you extracted the application to `Program Files` or another protected
  location, move it somewhere under your user folder

Deleting that folder resets the application to a fresh install — sometimes the
fastest fix if the configuration got into a strange state.

---

## The log file

Since v0.22.1 the application keeps a diagnostic log:

```
%LOCALAPPDATA%\R3E_FFB\logs\r3e-ffb-<date>.log
```

One file per day, the last seven kept. It records the startup, which device was
acquired and in which mode, and **every failure with the exact step and error code
from the driver**. Attaching the file to a report turns "it doesn't work" into
something that can actually be diagnosed.

Nothing is sent anywhere — the file stays on your machine until you choose to share it.

---

## Still stuck?

Ask on [Discord](https://discord.gg/JGypJavdj). What helps most:

- **Which version** — bottom right corner of the window
- **Your wheelbase** and its software, if any
- **What the wheel actually did**, as opposed to what you expected
- The **acquisition mode** from Settings (see above)
- A **CSV from the Logger tab**, if the problem happens while driving — it
  records the telemetry and every force the application computed, which turns a
  description into something that can be analysed
