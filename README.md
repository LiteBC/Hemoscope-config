# Hemoscope-config

Cloud source-of-truth for HemoScope bench configuration.

The HemoScope app fetches the files in this repo at startup, caches them locally,
and falls back to the bundled copy shipped with the install only when this repo is
unreachable. **This repo is the live config — edit here, not in the HemoScope source tree.**

## Repository layout

Three folders, one per config layer (lowest precedence first):

```
Location/   — per-site settings (e.g. Lab.config, 10k.config)
Computer/   — per-PC overrides  (e.g. Moriya-LT.config, HEMO-SERVER1.config)
Hemoscope/  — per-instrument overrides (e.g. Hemoscope1.config, HemoscopeTest.config)
```

At runtime each bench reads the file matching its environment variables:

| Layer    | Env var              | File looked up                           |
|----------|----------------------|------------------------------------------|
| Location | `HEMOSCOPE_LOCATION` | `Location/<HEMOSCOPE_LOCATION>.config`   |
| Computer | `HEMOSCOPE_COMPUTER` | `Computer/<HEMOSCOPE_COMPUTER>.config` (defaults to machine name) |
| Device   | `HEMOSCOPE_DEVICE`   | `Hemoscope/<HEMOSCOPE_DEVICE>.config`    |

`HEMOSCOPE_LOCATION` and `HEMOSCOPE_DEVICE` are required — the app throws at startup
if either is unset. There is no `Computer/Default.config` fallback: a machine with no
matching file fails with "Configuration file not found".

Full precedence, lowest first:

```
App.config (shipped in the app)  <  Location  <  Computer  <  Device  <  user.config
```

`user.config` is local to each machine, git-ignored, and never synced — it is where a
developer overrides settings for their own box. Only `appSettings`
`<add key="..." value="..."/>` entries are read.

## Arduino.Pins — the LED and trigger map

Each device file describes its LED and camera wiring in a single key, `Arduino.Pins`:
one entry per Arduino pin, keyed by pin number.

```xml
<add key="Arduino.Pins" value='{
    1:  { Role: "Led", Camera: "CXP", LedGroup: 0, Controller: "Gardasoft",
          Position: "LB", Illumination: "OBLIQUE", Wavelength: "530" },
    5:  { Role: "Led", Camera: "USB", LedGroup: 0, Controller: "Mightex", ControllerChannel: 1,
          Name: "Kohler 450nm", State: "Triggered", TriggerType: "HighEdge",
          Power: 1000, MaxPower: 3500, LedDelayUs: 0, LedOnTimeUs: 1000,
          Position: "RT", Illumination: "Kohler", Wavelength: "450" },
    7:  { Role: "Led", Connected: false, Position: "RB", Illumination: "OBLIQUE", Wavelength: "530" },
    11: { Role: "CameraTrigger", Camera: "USB" }
}' />
```

| Field | Meaning |
|---|---|
| `Role` | `Led`, `CameraTrigger` or `Motor` (default `Led`). An unrecognised value fails the whole config load. |
| `Connected` | `false` when nothing is wired to this pin; the LED Pattern tab omits the column (default `true`) |
| `Camera` | `CXP` / `USB` — the camera this pin fires, or that this LED illuminates |
| `LedGroup` | which physical LED this pin belongs to within its camera |
| `Controller` | `Mightex` / `Gardasoft` — the LED controller driving this pin |
| `ControllerChannel` | the channel this pin occupies on that controller |
| `Name` | free-text label, shown in the GUI and logs |
| `State` | `Constant` or `Triggered` |
| `TriggerType` | `None` / `HighEdge` / `LowEdge` (required when `Triggered`) |
| `Power`, `MaxPower` | drive current and this LED's ceiling, mA |
| `LedDelayUs` | trigger edge to LED on, us (optional, default 0) |
| `LedOnTimeUs` | LED on time, us (required when `Triggered`) |
| `Position` | LED position label, e.g. `RT` |
| `Illumination` | e.g. `OBLIQUE`, `Kohler` |
| `Wavelength`, `Constant` | LED wavelength, and the constant-on wavelength, nm |

Notes:

- **Pins sharing a `Camera` and `LedGroup` are one physical LED** wired across several
  channels — the old `led_groups:[[1,2,3]]` form. The number also fixes group order,
  which the app indexes into, so keep it 0-based and contiguous per camera.
- **Gardasoft has no driver yet.** Its pins are described so the wiring is recorded;
  the app logs that it is not driving them.
- **`Motor` pins are recorded, not yet driven.** The role exists so the wiring can be
  described; nothing fires a motor from the Arduino pattern yet. Such a pin is simply
  not treated as an LED, and it carries no `Position` / `Illumination` / `Wavelength`.
  If `MainLed0` / `MainLed1` still point at a pin that is no longer an LED, its position
  and illumination are recorded as `Unknown` rather than failing the capture.
- **A pin marked `Connected: false` can contain anything.** Nothing reads its fields, so
  they are not validated: an unrecognised `Role`, a bad number, any leftover text is
  logged and the pin ignored. Entries are parsed one pin at a time, so a broken
  disconnected pin never affects its neighbours. A pin that is *wired* is still validated
  strictly and a bad value fails the load, naming the pin.
- `Connected: false` hides a column, it does not disable the pin. The entry keeps its
  metadata, and a pin used by the active pattern is shown regardless so that hiding it
  can never silently drop it from the pattern.

`Arduino.Pins` replaces six keys that each described the same pins from a different
angle. When it is absent the app falls back to them, so a bench migrates on its own
schedule:

| Superseded key | Now |
|---|---|
| `WideCamera.Arduino.PinRoles` | `Role` / `Camera` / `LedGroup` |
| `LedController.Device` | `Controller`, per pin |
| `LedController.Channels` | `ControllerChannel` and the power/trigger fields |
| `LedPositionsWide` | `Position` |
| `IlluminationWide` | `Illumination` |
| `BacklightWavelengths`, `ConstantBacklightWavelengths` | `Wavelength`, `Constant` |

Migrated: `Hemoscope1`, `Hemoscope2`, `Hemoscope3`.
Still on the legacy keys: `HemoscopeTest`, `Dummy`.

The LED cycle itself stays separate, in `Default.Arduino.LedColorsPattern` and
`<Sequence>.Arduino.LedColorsPattern` — a list of cycles, each listing the Arduino pins
lit together.

## How HemoScope picks up changes

For each of the three files, on every app launch HemoScope tries, in order:

1. **Cloud** — fetch the latest from this repo. On success, the local cache is refreshed.
2. **Cache** — the last successful copy under `%LOCALAPPDATA%\HemoScope\ConfigCache\` (used when offline or GitHub is down).
3. **Bundled** — the cold-start fallback shipped inside the HemoScope install (only if the repo has never been reached and there is no cache).

Which version is fetched is set by `Configs.RemoteVersion` in the app's own `App.config`
— a branch name, tag, commit SHA, or a full URL. When unset, `main` is used. If the
pinned version is missing a file, that one file falls back to `main` and the app warns
that the pinned version was incomplete.

Look for `[ConfigSync]` lines in `HemoScope.log` to see which source, version and branch
were used for each layer. Worth knowing: a mistyped `Configs.RemoteVersion` fails quietly
by serving `main`, so check the `[ConfigSync] Summary` line rather than assuming.

`raw.githubusercontent.com` can serve a stale copy for a few minutes after a push, so a
bench started immediately afterwards may still see the previous version.

## Editing a file

You do **not** need this repo cloned to make changes.

- **From inside HemoScope** — open the Configuration window. The "Edit Location / Edit Computer / Edit Device on GitHub" buttons open the right file for your bench in the GitHub web editor, on the version the running config actually came from.
- **From any browser** — navigate to the file in this repo and click the pencil icon.
- **From a clone** — edit, commit, push.

Changes take effect on the next HemoScope launch on each bench.

## Safety notes

- Edits normally land directly on `main`, which every bench reads by default. Treat it
  like production config — small, reviewed changes only. To try a change on one bench
  first, push it to a branch and point that bench at it with `Configs.RemoteVersion`.
- **Keys are case-insensitive**, and a key defined twice in one file silently keeps the
  last value. Duplicates read as though the earlier one is active when it is not.
- A key nothing sets falls back to the default compiled into the app.
- **These files must be valid XML.** A comment containing `--` makes the whole file
  unparseable and the config layer fails to load.
- If you need to roll back, revert the commit; benches pick up the reverted file at
  next launch.
- To force a bench off the cloud entirely (e.g. for debugging or an emergency
  override), set `<add key="Configs.NoRemote" value="true" />` in the app's `App.config`
  on that machine — it must be the literal `true`. HemoScope then skips the cloud fetch
  and uses the bundled fallback shipped with the install, ignoring both this repo and
  the local cache. This is an app setting, not an environment variable.
