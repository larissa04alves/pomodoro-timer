# Omadoro, the Omarchy pomodoro

Omadoro is a pomodoro timer plugin for omarchy-shell. A progress ring with the
countdown sits in the bar. A click opens a popup with two tabs, **Pomodoro**
(ring, pause, skip, restart) and **Config** (durations and auto-start). The
interface labels are in Portuguese.

There is no binary, no daemon of its own and no build step. The plugin runs
inside the shell you already use, with the logic in one JavaScript file and
the view in QML.

![Ring and countdown in the Omarchy bar, with the popup open on the Pomodoro tab](docs/img/barra.png)

| Pomodoro tab | Config tab |
| --- | --- |
| ![Popup on the Pomodoro tab showing the ring, 24:32, the Foco phase and the restart, pause and skip buttons](docs/img/popup-pomodoro.png) | ![Popup on the Config tab showing the focus, short break, long break and cycles sliders and the auto-start switch](docs/img/popup-config.png) |

## Install

Omadoro needs Omarchy 4 with omarchy-shell as the bar. It was tested on 4.0.3.

```bash
omarchy plugin add https://github.com/larissa04alves/omadoro.git --enable
```

With `--enable`, the installer asks which bar section gets the widget. The
default is `right`. To move it later:

```bash
omarchy bar move larissa04alves.omadoro --section center
```

## Use

The ring empties as the phase runs. The full theme accent color means focus,
half strength means a break, and the ring fades when the timer is stopped. A
new Omarchy theme changes the timer color with it.

| Input | Action |
| --- | --- |
| Left click on the bar | Opens or closes the popup |
| Right click on the bar | Pauses or resumes without opening the popup |
| Middle click on the bar | Restarts the current phase |
| `Esc` in the popup | Closes the popup |
| `Tab` in the popup | Moves to the next bar panel |
| Arrow keys in the popup | Switch tabs |
| `Enter` or `Space` in the popup | Pauses or resumes |

## Configure

The **Config** tab of the popup has four sliders and one switch:

| Option | UI label | Range |
| --- | --- | --- |
| Focus | Tempo de foco | 5 to 60 minutes |
| Short break | Pausa curta | 1 to 20 minutes |
| Long break | Pausa longa | 10 to 45 minutes |
| Cycles until the long break | Ciclos até a pausa longa | 2 to 8 |
| Auto-start the next phase | Auto-iniciar a próxima fase | on or off |

The plugin stores these options inline in its entry in
`~/.config/omarchy/shell.json`, the same file that holds the rest of the bar
layout. You can edit the entry by hand or through the bar CLI:

```bash
omarchy bar set larissa04alves.omadoro work 30
```

The entry looks like this:

```jsonc
{ "id": "larissa04alves.omadoro", "work": 25, "short": 5, "long": 15,
  "longEvery": 4, "autoStartNext": false }
```

The `sound` key takes the path to an audio file. When it is empty, the plugin
plays the freedesktop `complete.oga`.

The runtime state lives elsewhere. Phase, clock and cycle go to
`~/.local/state/larissa04alves.omadoro/state.json`, or under `$XDG_STATE_HOME`
when that variable is set.

## Shortcuts

The shell IPC exposes the same verbs the popup uses:

```bash
omarchy-shell omadoro <open|close|toggle|toggleRunning|pause|start|skip|restart|reset|health>
```

`open`, `close` and `toggle` control the window. `toggleRunning` switches
between running and stopped, `pause` only pauses, and `start` only starts a
stopped timer. `skip` jumps to the next phase, `restart` sets the phase back
to its full length and `reset` clears the cycle. `health` prints the phase,
whether it is running and the time left, as JSON.

To bind a key, add this to `~/.config/hypr/bindings.conf`:

```
bind = SUPER, P, exec, omarchy-shell omadoro toggleRunning
```

## Update and remove

```bash
omarchy plugin update larissa04alves.omadoro && omarchy restart shell
```

The `restart` is required. The plugin declares `keepLoaded: true` so the
timer keeps counting when the widget unmounts, and as a side effect the old
service survives the reload. The new code only runs after a shell restart.

```bash
omarchy plugin remove larissa04alves.omadoro
```

## How it works

The plugin has three parts. `Service.qml` runs once per shell. It owns the
timer, writes the state file, sends the notification and sound, and registers
the IPC target. `BarWidget.qml` runs once per monitor and draws the ring, the
MM:SS countdown and the popup. `Model.js` holds the pure logic, with no Qt,
and decides every transition.

Each phase stores the moment it ends, as an epoch in milliseconds, instead of
a counter. The time left is a subtraction. That choice gives these behaviors:

- If the machine suspends or shuts down for more than 120 seconds, the phase
  goes back to its full length, paused, with no notification. The phase did
  not finish, the machine was away.
- A shell restart resumes where the timer was. The plugin writes the state to
  disk every 30 seconds and on every user action.
- A skipped focus phase does not count toward the long break.
- A duration change only affects a paused phase that is still at its full
  length. Otherwise the change applies from the next cycle.
- The notification and sound only fire when a phase ends on its own. The
  notification comes from `omarchy-notification-send`, and the sound plays
  through whichever of `pw-play`, `paplay`, `mpv` or `ffplay` is installed.
  Clicking the notification starts the next phase.

The colors come from the Omarchy theme. The plugin has no palette of its own.

## Development

```bash
node test/model.test.js   # 36 tests of the pure logic, no Qt
scripts/verify.sh         # validate, qmllint, tests and ring render
scripts/verify.sh --live  # all of the above plus a smoke test on the running shell
scripts/dev.sh --restart  # copies the checkout into the plugin directory
```

`scripts/dev.sh` uses `rsync` into `~/.config/omarchy/plugins` because the
validator rejects any symlink inside the plugin directory. It accepts
`--enable` and `--restart`.

Saving `BarWidget.qml` or `Panel.qml` reloads the plugin within a few
milliseconds. `Service.qml` does not reload because of `keepLoaded`, so every
change to it needs `omarchy restart shell`.

`qmllint` and `qmltestrunner` live in `/usr/lib/qt6/bin`. Neither is on the
PATH of the development machine.

## Manual smoke test

After installing, check on screen that:

1. The bar shows the ring and the time as MM:SS.
2. A left click opens the popup on the Pomodoro tab.
3. Dragging a slider on the Config tab does not close the popup.
4. With the timer running, `omarchy restart shell` resumes the countdown where
   it was, with no stray notification.

## License

MIT. See [LICENSE](LICENSE).
