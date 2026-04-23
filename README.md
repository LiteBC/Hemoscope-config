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

Layers merge in order Location -> Computer -> Hemoscope -> user.config, with
later layers overriding earlier ones. Only `appSettings` `<add key="..." value="..."/>`
entries are read.

## How HemoScope picks up changes

For each of the three files, on every app launch HemoScope tries, in order:

1. **Cloud** — fetch the latest from this repo (`raw.githubusercontent.com/LiteBC/Hemoscope-config/main`). On success, the local cache is refreshed.
2. **Cache** — the last successful copy under `%LOCALAPPDATA%\HemoScope\ConfigCache\` (used when offline or GitHub is down).
3. **Bundled** — the cold-start fallback shipped inside the HemoScope install (only if the repo has never been reached and there is no cache).

Look for `[ConfigSync]` lines in `HemoScope.log` to see which source and which version
were used for each layer.

## Editing a file

You do **not** need this repo cloned to make changes.

- **From inside HemoScope** — open the Configuration window. The "Edit Location / Edit Computer / Edit Device on GitHub" buttons open the right file for your bench in the GitHub web editor.
- **From any browser** — navigate to the file in this repo and click the pencil icon.
- **From a clone** — edit, commit, push to `main`.

Changes take effect on the next HemoScope launch on each bench.

## Safety notes

- All edits land directly on `main`; there is no staging branch. Treat this repo like
  production config — small, reviewed changes only.
- Keys are case-sensitive. A typo in a key name silently shadows nothing and the
  default value will be used.
- If you need to roll back, revert the commit on `main`; benches will pick up the
  reverted file at next launch.
- To force a bench off the cloud entirely (e.g. for debugging or an emergency
  override), set the env var `HEMOSCOPE_CONFIG_NOREMOTE=1` on that machine.
  HemoScope will then skip the cloud fetch and use the bundled fallback shipped
  with the install — ignoring both this repo and the local cache.
