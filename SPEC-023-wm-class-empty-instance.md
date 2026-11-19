# SPEC-023. Empty instance in `WM_CLASS`: BIMCraft window on the Remmina button

Status: **draft**, implementation required
Date: 2026-11-19
Project: BIMCraft (Rust, `apps/bimcraft`, `eframe`/`egui-winit` 0.36.2, `winit` 0.30.13)
Source: user request dated 2026-11-19: "for some reason the bimcraft window opens in the
zorinos favorites list on the remmina button. why? fix it"
Related specs: none; touches `packaging/linux` (`.deb` build) and the desktop shell integration.
Task type: window/desktop compatibility fix. The model, project file, commands and ABI do not change.

> **In short.** `egui-winit` 0.36.2 passes `ViewportBuilder::app_id` to
> `WindowAttributes::with_name(app_id, "")`. Through the shared `platform_specific.name` field this
> also reaches the X11 backend of `winit`: `WM_CLASS(STRING) = "", "io.github.ipedrax.bimcraft"` —
> the instance is empty. GNOME Shell 46, when looking up the application for a window, **first of
> all** matches the instance against the `StartupWMClass` table, and the Flatpak launcher for Remmina
> has an empty `StartupWMClass=`, so the table contains the key `""`. The BIMCraft window is bound to
> Remmina: its preview hangs on Remmina's button, and there is no window on its own button. The
> correct fix is one line in `egui-winit` (`with_name(app_id, app_id)`), then vendoring the crate and
> rebuilding BIMCraft. A local workaround via the Remmina launcher has already been made, but it
> only heals this one machine.

## 0. General working rules

1. Communication language — Russian; code, comments, strings and log messages — English.
   The document (`.md`) — Russian.
2. Tests (`cargo test`, `cargo nextest`, doctest) are **not run** in this task without an explicit
   request from the user (rule `~/.config/opencode/rust-rules.md`). Allowed: `cargo check`,
   `cargo build`, `cargo clippy` (without `--all-targets`), `cargo fmt --check`.
3. Commits are atomic: vendor and `[patch.crates-io]` separately, accompanying packaging changes
   separately, documentation separately. Every commit builds.
4. No new dependencies: the already used version `egui-winit` 0.36.2 from `Cargo.lock` is vendored.
5. There is still no `unsafe` in the workspace; lints and `deny` are not weakened.
6. The vendored crate keeps its licenses (`MIT OR Apache-2.0`) and is not rewritten beyond one
   line — otherwise the point of the "patch until upstream" is lost.

## 1. Symptom

On Zorin OS (GNOME Shell 46, X11) the BIMCraft window after launch:

- does not appear on its own icon in the panel: the BIMCraft icon has no running indicator, and the
  window preview on hover is empty;
- "lands" on the pinned **Remmina** button: it gets a window indicator, and the preview shows the
  window `Project · Floor Plan: Level 1 · BIMCraft`;
- clicking the Remmina button raises the BIMCraft window.

The installed Remmina meanwhile runs in the tray (Flatpak `org.remmina.Remmina` 1.4.44), its window
opens and closes normally — that is, the problem is not in Remmina itself but in how the Shell
matches windows to applications.

## 2. Diagnosis

### 2.1. Chain of cause

| # | What happens | Where |
|---|---|---|
| 1 | The application asks eframe for the identifier `io.github.ipedrax.bimcraft` | `apps/bimcraft/src/main.rs:38` (`const APP_ID`), `:222` (`.with_app_id(APP_ID)`) |
| 2 | `egui-winit` moves `app_id` into the window attributes as `with_name(app_id, "")`, that is **general = app_id, instance = empty** | `egui-winit-0.36.2/src/lib.rs:2152-2156` |
| 3 | `winit` stores the name in the field `platform_specific.name` shared by Wayland and X11 (`ApplicationName { general, instance }`) | `winit-0.30.13/src/platform/wayland.rs:111-118`, `src/platform_impl/linux/mod.rs:61-69` |
| 4 | The X11 backend of `winit` writes `WM_CLASS` from the pair `(name.instance, name.general)` | `winit-0.30.13/src/platform_impl/linux/x11/window.rs:386-405` |
| 5 | As a result the window has `WM_CLASS(STRING) = "", "io.github.ipedrax.bimcraft"` — the instance is empty (ICCCM violation: `res_name` must be non-empty) | `xprop -id <window> WM_CLASS` |
| 6 | GNOME Shell looks up the application for the window: **first** instance → `StartupWMClass` table, then class → table, then instance → `.desktop`, then class → `.desktop` | `gnome-shell-46/src/shell-window-tracker.c`, `get_app_from_window_wmclass()` |
| 7 | The `StartupWMClass` table is built from **all** `.desktop` files and accepts any non-`NULL` value, including an empty string | `gnome-shell-46/src/shell-app-system.c`, `scan_startup_wm_class_to_id()` |
| 8 | The Flatpak launcher for Remmina contains `StartupWMClass=` (empty) → the key `""` appears in the table → `org.remmina.Remmina.desktop` | `/var/lib/flatpak/exports/share/applications/org.remmina.Remmina.desktop` |
| 9 | The empty instance (step 5) hits the key `""` (step 8) → the Shell gives the window the Remmina application | — |

The key point: step 6 happens **before** the class check, so a correct
`StartupWMClass=io.github.ipedrax.bimcraft` in the BIMCraft launcher does not save the day — it is
never reached.

### 2.2. Why the empty instance is to blame

- If the instance is non-empty (`bimcraft`), step 6 misses, and at step 7 the window finds BIMCraft by
  class — that is, the bug only manifests with an empty instance.
- Verified by the reverse operation: the BIMCraft window was given
  `WM_CLASS = ("bimcraft", "io.github.ipedrax.bimcraft")` at runtime — the Shell immediately re-matched
  the window (the indicator appeared on the BIMCraft icon, the Remmina preview cleared).
- Verified that there are no other launchers with an empty `StartupWMClass` in the system: the
  conflict would have "moved" to another application.

### 2.3. Verified facts

```
$ xprop -id <BIMCraft window> WM_CLASS
WM_CLASS(STRING) = "", "io.github.ipedrax.bimcraft"

$ grep -n 'with_name' ~/.cargo/registry/src/*/egui-winit-0.36.2/src/lib.rs
2155:        window_attributes = window_attributes.with_name(app_id, "");

$ grep -n 'with_name' <winit>/src/platform_impl/linux/x11/window.rs
388:                let (instance, class) = if let Some(name) = window_attrs.platform_specific.name {
389:                    (name.instance, name.general)
...
402:                let class = format!("{instance}\0{class}\0");
```

The panel previews were checked by hovering the pointer over the icons: on Remmina — the BIMCraft
window, on BIMCraft — empty. After the runtime `WM_CLASS` fix — the opposite.

## 3. Local workaround that has already been applied

To keep the machine working right now, a user override of the Remmina launcher was made:

`~/.local/share/applications/org.remmina.Remmina.desktop`

- a copy of the Flatpak launcher in which the empty line `StartupWMClass=` is replaced with the real
  class of Remmina windows: `StartupWMClass=org.remmina.Remmina` (verified with `xprop` on the Remmina
  window);
- the system Flatpak file is untouched, and a Flatpak update will not overwrite the launcher;
- the empty key `""` is no longer in the Shell table, and BIMCraft is matched by class.

Limitations of the workaround:

- it heals only this machine and only the conflict with Remmina; on another machine with a launcher
  that has an empty `StartupWMClass`, the bug comes back;
- it does not eliminate the ICCCM violation in the application itself: the window's `res_name` is
  still empty, and this breaks window managers/dock panels in other environments (KDE, XFCE, tiling
  WMs);
- the workaround should be kept until a fixed BIMCraft build is released, and not removed right after
  the patch: it is harmless and protects against other applications with an empty instance.

## 4. Goals

1. Fix the `WM_CLASS` of the BIMCraft window: the instance is non-empty, the class is
   `io.github.ipedrax.bimcraft`.
2. Make the fix reproducible: the patch lives in the repository, not in the local cargo registry.
3. Prepare an upstream patch for `emilk/egui` so that after updating dependencies the vendoring can
   be removed.
4. Do not change behavior on Wayland and macOS.

## 5. Non-goals

- Do not rewrite the window/eframe and do not move away from egui.
- Do not change the model, project file, commands and ABI.
- Do not add dependencies and do not patch `winit` (its API is already correct there: the parameters
  are named `general` and `instance`, the mistake is on the caller's side).
- Do not patch the local `~/.cargo/registry/src` (not reproducible, overwritten on update)
  and do not rely on `StartupWMClass` in the launcher as a "fix".
- Do not change the rule "the project contains no desktop data": the launcher and `StartupWMClass`
  remain the packaging's domain.

## 6. Key decision and alternatives

**Fix `egui-winit`: instance = the same `app_id`.**

| Option | Pros | Cons / why not |
|---|---|---|
| `with_name(app_id, app_id)` in `egui-winit` (**chosen**) | 1 line; fixes all eframe applications; breaks nothing on Wayland (see §7.2) | requires vendoring or waiting for a release |
| `StartupWMClass` in the BIMCraft launcher | zero code | does not help: the Shell checks the instance before the class (step 6) |
| Override of the Remmina launcher (done) | instant, no build | this machine only; the bug stays in the application |
| `RESOURCE_NAME=bimcraft` in the launcher's `Exec=` | no code | sets **instance**, not class; fragile, depends on env on every launch, does not work when launched not via the launcher |
| Patch `winit` | "deeper" | `winit` follows ICCCM correctly; the caller is to blame |
| Own X11 code in BIMCraft (`xcb`/`x11rb`) after window creation | no forking of a dependency | +dependency, +`unsafe`/FFI, "window already shown" race; unnecessary when one line above is fixed |

## 7. Exact patch

### 7.1. `egui-winit` 0.36.2, `src/lib.rs`

```diff
@@ -2152,7 +2152,7 @@
     #[cfg(all(feature = "wayland", target_os = "linux"))]
     if let Some(app_id) = _app_id {
         use winit::platform::wayland::WindowAttributesExtWayland as _;
-        window_attributes = window_attributes.with_name(app_id, "");
+        window_attributes = window_attributes.with_name(app_id.as_str(), app_id.as_str());
     }
```

`with_name` of `WindowAttributesExtWayland` takes `(general, instance)`; `instance` also goes into
X11 (`winit` writes `WM_CLASS` from `(instance, general)`), which is why an empty string in the second
argument yields `WM_CLASS = ("", "<app_id>")`.

A note on the branch name: the call sits under `cfg(feature = "wayland")`, but this is a
**compile-time** feature (it is enabled by default), so the code also executes in an X11 session, and
the value reaches the shared `platform_specific.name` field. That is exactly why the bug reproduces
on X11 even though the line is "about Wayland".

### 7.2. Why Wayland does not break

The Wayland backend of `winit` takes only `name.general`:
`src/platform_impl/linux/wayland/window/mod.rs:128` — `attributes.platform_specific.name.map(|name| name.general)`.
The second argument (`instance`) is not used on Wayland, so `app_id.as_str()` in the second argument
changes nothing: `app_id` stays the same.

### 7.3. Upstream PR (emilk/egui)

- File: `crates/egui-winit/src/lib.rs`, the one line from §7.1.
- Title: `egui-winit: set a non-empty WM_CLASS instance on X11`.
- Body (English): on Linux X11 the window gets `WM_CLASS(STRING) = "", "<app_id>"` because
  `ViewportBuilder::app_id` is passed as `general` while `instance` is empty; `winit` uses the same
  field on X11, where `WM_CLASS` is built from it. An empty `res_name` violates ICCCM and breaks
  window-to-application matching (example: GNOME Shell attaches the window to a foreign launcher with
  an empty `StartupWMClass`). Patch: pass `app_id` as the instance as well; on Wayland the behavior
  does not change (only `general` is read there).
- Check for the reviewer: build the eframe example on X11, `xprop WM_CLASS` — both fields equal
  `app_id`; the window lands on its own icon in GNOME.
- After a release with the patch: update `egui`/`eframe` in BIMCraft, remove `vendor/` and
  `[patch.crates-io]`.

## 8. Integration into the repository

While there is no release with the patch, we use local vendoring:

```
vendor/egui-winit/            # a copy of egui-winit-0.36.2 from the cargo registry
```

Steps:

1. Copy `egui-winit-0.36.2` into `vendor/egui-winit` (without `.cargo-ok`, without `target/`).
   Do **not** change the version in its `Cargo.toml` — `[patch.crates-io]` requires the version to
   match `Cargo.lock`.
2. Apply the fix from §7.1.
3. In the workspace root `Cargo.toml` add:

   ```toml
   [patch.crates-io]
   egui-winit = { path = "vendor/egui-winit" }
   ```

4. Make sure the dependency graph picked up the path:

   ```
   cargo tree -i egui-winit
   # egui-winit v0.36.2 (/…/vendor/egui-winit)
   ```

5. `Cargo.lock` changes minimally: the version and source (`registry`) of `egui-winit` are replaced
   with the path; the other packages are untouched.

Open question: there are two copies of the tree at the root — `BIMCraft-019` (which
`scripts/release.sh`, `scripts/debug.sh`, `scripts/bimcraft-*.sh` look at by default) and `BIMCraft-022`
(active work, branch `main`). The patch and the vendoring must be added to the copy from which the
installed package is built; when in doubt — to both, with separate commits.

## 9. Build and installation

### 9.1. Build

```
cargo build --release -p bimcraft
```

Tests are not run (§0.2). Verifying the result:

```
target/release/bimcraft --version
```

### 9.2. Installation via script (sudo)

The binary in `/usr/bin/bimcraft` is owned by root and came from the package `bimcraft 0.12.0-1`, so
installation is done by a separate script that the user runs themselves:

- back up the current binary next to it (`/usr/bin/bimcraft.bak-<timestamp>`);
- atomic replacement: `install` into a temporary file in the same directory, then `mv -f` (replacing
  a running file via `cp` yields `ETXTBSY`);
- self-check: `bimcraft --version`, permissions `0755`, size;
- print `VERIFICATION`, non-zero exit code on mismatch.

Before replacement it is better to close BIMCraft so as not to lose unsaved changes: the old process
will keep running on the old inode while the version in the system is already new.

## 10. Verification

1. Launch the fixed BIMCraft on X11.
2. `xprop -id <window> WM_CLASS` → `WM_CLASS(STRING) = "io.github.ipedrax.bimcraft", "io.github.ipedrax.bimcraft"`.
3. Hover the pointer over the BIMCraft icon in the panel → a preview with the BIMCraft window, the icon has an indicator.
4. Hover over the Remmina icon → the preview is empty (or only has Remmina windows), no BIMCraft window there.
5. Clicking the Remmina icon does not raise the BIMCraft window; clicking the BIMCraft icon does.
6. Restart BIMCraft and repeat steps 2–5 (to verify it is not a one-time effect).
7. Open the Remmina window (`flatpak run org.remmina.Remmina`) → the preview on its icon shows the
   Remmina window; close the window, Remmina stays in the tray.
8. Wayland session (if any): the window launches, `app_id` is the same, no regression.

## 11. Acceptance criteria

1. `WM_CLASS` of the BIMCraft window: the instance is non-empty, the class equals `APP_ID`.
2. The BIMCraft window is always on its own icon; it is not on the Remmina icon.
3. Remmina keeps being matched to its own icon.
4. The build is reproducible from the repository (vendoring + patch), without edits to `~/.cargo/registry`.
5. `cargo build --release -p bimcraft` passes; `cargo fmt --check` and `cargo clippy` (without
   `--all-targets`) produce no new warnings.
6. Wayland and macOS are unaffected.

## 12. Risks and open questions

| Risk | Mitigation |
|---|---|
| The vendored copy drifts from an eframe update | exactly one patch; on updating `egui`, check whether it was fixed upstream and remove the vendoring |
| `[patch.crates-io]` forgotten after the release | an item in the DoD and in the release checklist of `packaging/linux` |
| Fix made in one copy of the tree (019/022) and lost in the other | apply to the copy from which the package is built; when in doubt — to both |
| `instance = app_id` does not match the binary name (`bimcraft`) | this is acceptable: GTK applications with `GtkApplication` do the same; if desired, a short name could be discussed upstream, but then egui-winit would need to know the program name |
| The Remmina launcher workaround masks the bug on this machine | the §10 verification is done before removing the workaround; the workaround is removed only after installing the fixed build |

Questions for discussion:

1. Apply the patch to `BIMCraft-019`, `BIMCraft-022`, or both?
2. Do we keep the workaround with `StartupWMClass=org.remmina.Remmina` in the system after installing the build?
3. Do we send a PR to `emilk/egui` from the project account, or limit ourselves to local vendoring?

## 13. Order of execution and DoD

1. Vendoring `egui-winit` 0.36.2 + the §7.1 fix + `[patch.crates-io]`.
2. `cargo build --release -p bimcraft`.
3. Installation script (§9.2) — prepare, the user runs it.
4. Verification §10.
5. Upstream patch and PR (§7.3) — per the decision in §12.

- [ ] `WM_CLASS` with a non-empty instance.
- [ ] The BIMCraft window on its own icon, not on the Remmina icon.
- [ ] Remmina on its own icon.
- [ ] Build from the repository without edits to the cargo registry.
- [ ] `cargo fmt --check` and `cargo clippy` without new warnings.
- [ ] Upstream patch prepared (or a PR opened).

## 14. What not to do

- Do not patch `~/.cargo/registry/src/.../egui-winit-0.36.2/src/lib.rs` — not reproducible.
- Do not "fix" this by changing `StartupWMClass` in the BIMCraft launcher (does not help because of the order of checks).
- Do not add `RESOURCE_NAME` to `Exec=` as a permanent solution.
- Do not rewrite `winit` and do not pull `xcb`/`x11rb` into BIMCraft for the sake of one line.
- Do not change `APP_ID`: the identifier matches the `.desktop` name, the icon and `X-Flatpak`.

## 15. Appendix: how to diagnose again

```
# windows and their classes
for id in $(xprop -root _NET_CLIENT_LIST | sed 's/.*# //; s/,//g'); do
  echo "$id $(xprop -id "$id" WM_CLASS 2>/dev/null | tail -1)"
done

# who is on which button: hover the pointer over the icon and look at the preview
# (in Zorin/GNOME the preview shows the windows of exactly the application the window is bound to)

# launchers with an empty StartupWMClass
for d in /usr/share/applications ~/.local/share/applications \
         /var/lib/flatpak/exports/share/applications; do
  grep -l '^StartupWMClass=\s*$' "$d"/*.desktop 2>/dev/null
done
```

Source references used in the diagnosis:

- `gnome-shell` 46: `src/shell-window-tracker.c` (`get_app_from_window_wmclass`),
  `src/shell-app-system.c` (`scan_startup_wm_class_to_id`);
- `egui-winit` 0.36.2: `src/lib.rs:2152-2156`;
- `winit` 0.30.13: `src/platform/wayland.rs:111-118`,
  `src/platform_impl/linux/mod.rs:61-69`,
  `src/platform_impl/linux/x11/window.rs:386-405`,
  `src/platform_impl/linux/wayland/window/mod.rs:128`.
