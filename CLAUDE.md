# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

WhiteSur is a macOS BigSur-like GTK theme. It's a pure Bash + SCSS project — no package manager, no build system beyond `sassc`. Output is a set of installable theme directories under `~/.themes` (or `$THEME_DIR`).

## Common commands

```bash
# Regenerate all CSS from SCSS sources (Light + Dark variants for gtk-3.0, gtk-4.0, gnome-shell, cinnamon)
./parse-sass.sh

# Generate only specific color variants
COLOR_VARIANTS='-Light' ./parse-sass.sh

# Install the theme (see install.sh -h for the full flag list — many variants)
./install.sh                          # default install
./install.sh -t all -c Dark -l        # all accents, dark only, libadwaita
./install.sh -r                       # uninstall

# Apply post-install tweaks (Firefox, Dash-to-Dock, Flatpak, Snap, GDM, etc.)
./tweaks.sh -h

# Cut a release tarball
./make-release.sh
```

There are no tests, no linter, and no build step beyond `parse-sass.sh`. Do **not** run a build after editing unless verifying SCSS compiles.

`sassc` is the only hard dependency for regeneration; `parse-sass.sh` will try to install it via the system package manager if missing.

## Architecture

The repo has three layers:

### 1. SCSS sources → CSS (`src/sass/` + `src/main/`)

`src/main/<target>/<target><Variant>.scss` are thin entry points (one per desktop target × Light/Dark). They `@import` the shared partials in `src/sass/`:

- `_variables.scss` — knobs (sizes, radii, accents)
- `_colors.scss`, `_colors-palette.scss` — color tokens
- `_gtk-base.scss`, `_theme-options.scss` — base rules and option toggles
- `src/sass/{gtk,gnome-shell,cinnamon}/` — target-specific partials

`parse-sass.sh` does something important to know about: it **copies** `_gtk-base.scss` and `_theme-options.scss` to `*-temp.scss` siblings before compiling. The `-temp.scss` files are throwaway working copies that the installer/tweaks scripts mutate at install time to bake user-selected options (accent, opacity, nautilus style, etc.) into the SCSS before recompiling. **Edit the non-temp files**; the `-temp` versions are regenerated.

Targets compiled: `gtk-3.0`, `gtk-4.0`, `gnome-shell`, `cinnamon` — each in `-Light` and `-Dark`. `gtk-2.0`, `metacity-1`, `xfwm4`, and the `assets/` trees ship as-is.

### 2. Installer libraries (`libs/`)

The installer is split into sourced bash libraries — `install.sh`, `tweaks.sh`, and `make-release.sh` are thin entry points that source these:

- `libs/lib-core.sh` — shared variables (`THEME_NAME`, `THEME_DIR`, all `*_VARIANTS` arrays), `helpify` arg-parsing helpers, color/log helpers. **Most defaults live here.**
- `libs/lib-install.sh` — the actual install/uninstall logic, per-variant theme assembly, GDM/GTK4/libadwaita patching, dialog/interactive mode.
- `libs/lib-flatpak.sh` — Flatpak override management.

When adding a new option (a new accent, a new size, a new tweak), the pattern is:

1. Add the variant to the relevant `*_VARIANTS` array in `lib-core.sh`.
2. Add the corresponding SCSS branch in `src/sass/_theme-options.scss` (the file the installer mutates).
3. Wire the CLI flag in `install.sh` (`usage()` + the arg parser) and the install behavior in `lib-install.sh`.
4. Update `parse-sass.sh` if a new compilation target is involved.

### 3. Output (`release/`)

`make-release.sh` builds tarballs into `release/`. Not used during normal development.

## Conventions specific to this repo

- `install.sh` warns at the top: keep scripts working-directory-independent (use `${REPO_DIR}/...`, not relative paths), and never `cd` outside a subshell.
- Don't edit `src/sass/_gtk-base-temp.scss` or `_theme-options-temp.scss` — they're regenerated from the non-temp versions by `parse-sass.sh` and rewritten by the installer.
- The `helpify` function in `lib-core.sh` is what formats `install.sh -h`; matching its column layout matters when adding flags.
