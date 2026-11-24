# SPEC-006. BIMCraft Plugin Self-Update: User Consent, Update Channel, and Atomic Replacement

## 1. Context and Goal

Today BIMCraft plugins are installed as a folder: `plugin.json` and the module live in the
plugins folder, the application reads them at startup, and updating means manually replacing
the folder (the delivery installer places the new version alongside and moves the old one to
`<id>.backup-<date>`).

The goal of this spec is to let a plugin **check for and install its own updates** without
granting it network access, files, or disk writes. All network and disk work is done by the
BIMCraft host, and the plugin only asks: "check" and "update". The user grants a separate
permission for this and sees exactly what is being installed.

What must be preserved:

- the sandbox stays as it is: the module still imports only what is declared in the ABI
  and gets neither files nor sockets;
- no update happens without the user's explicit permission;
- the update is atomic: a failure at any step leaves the previous working version in place;
- the digest and signature are verified before new code reaches the plugins folder;
- the user's state — disabled flag and granted permissions — survives the update.

## 2. What Already Exists in the Code (a Foundation, Not a Rewrite)

- `crates/plugin/src/manifest.rs` — `Manifest` is parsed strictly
  (`deny_unknown_fields`), `PERMISSIONS = ["files"]`, an unknown permission causes load
  rejection. `Manifest::asks_for_files` is the template for permission checks.
- `crates/plugin/src/lib.rs` — plugins folder, `Plugins::load_from`, `state.json`
  (`disabled` and `permissions`), limits `MAX_DISABLED`, `MAX_GRANTS`, `MAX_STATE`,
  the `Log` journal, `set_permission`.
- `crates/plugin/src/runtime.rs` — the sandbox: `HOST = ["execute", "log", "info"]`,
  any other import is rejected when the module is compiled; limits `FUEL`,
  `MEMORY = 64 MB`, `MAX_COMMANDS`, `MAX_REPLY`, `MAX_MODULE = 16 MB`.
- `crates/ui-egui/src/plugins.rs` — the Manage → Plugins window: `permit_plugin`,
  `PluginView`, granting `files` by the user, re-reading the folder, the journal.
- `docs/areas/plugins.md` — the current description of the manifest, permissions, and
  sandbox; after implementation it must be extended with a section on updating.

### 2.3. The Application Updater (`crates/update`) — Reused

The project already has a separate crate for updating BIMCraft itself, and its design is
the model for plugins:

- `update::Net` — the network trait, `update::net::Ureq` — the implementation on `ureq` +
  `rustls` (`Cargo.toml`: `ureq 3`, `ring 0.17`), `net::hop_allowed` — redirect check;
- `update::sha256_file`, `sha256_of` — SHA-256 via `ring` with no new dependencies;
- `update::parse_release`, `is_newer`, `tag_version` — strict response parsing and
  version comparison (`semver`);
- `prefs::Prefs` (`update.json` next to `favorites.json`) — "check at startup" and
  "do not offer this version"; the file is written atomically via `.part` + `rename`;
- `testing::Fake` — the network as a table of responses for tests;
- the download is size-limited, hashed as it is read, and stored in a separate folder only
  after checks.

For plugins this means: no new network code is written, `crates/plugin` depends on
`bimcraft-update` and takes `Net`, `sha256_file`, and `hop_allowed`; the Ed25519 signature
is added through the same `ring`. The only difference is that the application replaces the
installer per platform, while the plugin replaces a folder inside the plugins folder.

## 3. Why Self-Update Is Impossible Without This Spec

A plugin has neither network, files, nor writes: the host imports exactly three functions,
and a module importing anything else does not load. So "the plugin downloaded a new version
and replaced itself" cannot be done by a manifest or by plugin code — only by changing the
host. This spec describes such a change.

## 4. Terms

- **Update channel** — an https URL of a JSON file that announces the available plugin
  version (the feed URL); set in the manifest of the installed version.
- **Feed** — the JSON of the update channel: id, ABI, version, archive link, digest,
  signature.
- **Update archive** — a zip with `plugin.json` and the module at its root, as in delivery.
- **`update` permission** — a new manifest permission that the user grants in
  Manage → Plugins and can revoke.
- **Atomic replacement** — installation via a temporary folder and renaming, so that the
  plugins folder never contains a "half-installed" version.

## 5. Requirements

### 5.1. Manifest

1. `"update"` is added to `PERMISSIONS`: a manifest without it does not get access to
   updating, and an attempt to call the update function returns a permission error.
2. An optional `update` field is added to `Manifest` (strict parsing is preserved):

   ```json
   "permissions": ["update"],
   "update": {
     "feed": "https://example.org/bimcraft-scripts/latest.json",
     "public_key": "ed25519:BASE64_32_BYTES"
   }
   ```

3. `update` is required if the `update` permission is declared, and vice versa: a
   permission without a channel causes load rejection with a reason, a channel without a
   permission is rejected.
4. `feed` — `https` only, no username/password in the URL, no fragment; length within
   `MAX_TEXT`. An empty field is allowed only for local development, and then the channel
   is considered undeclared.
5. `public_key` — the publisher's key in Ed25519 encoding (32 bytes). The key is pinned at
   the moment of first installation (TOFU) and stored in `state.json` next to the
   permission. Changing the key in a new manifest version requires the user's renewed
   consent.
6. The manifest of the version in the archive must have the same `id` and a supported
   `abi`.
7. A manifest with `update` requires a BIMCraft version not lower than the one that
   introduces this spec: older builds will reject the unknown permission. This is
   intentional: a plugin that can update itself must not load where updating is not
   verified.

### 5.2. User Permission

1. The `update` permission is granted **only by the user** in Manage → Plugins, as `files`
   is today; the manifest merely declares it.
2. The permission and automatic updating are **a single "Update plugin" toggle**. When
   enabled, it means both "this plugin may update itself" and "check for and install new
   versions yourself". Disabled by default.
3. Storage — in the existing `state.json`:
   `"permissions": {"<id>": ["update"]}` and an `update_confirmed` mark indicating that the
   channel and publisher key have already been confirmed; limits are `MAX_GRANTS`,
   `MAX_STATE`. Turning the toggle off removes the permission.
4. On enabling — a warning: "The plugin will be able to replace its code with a new version
   without further questions. Allow this only for plugins from a publisher you trust."
   Next to the plugin description, not in "fine print".
5. **The first installation after enabling always asks**: the channel host, key
   fingerprint, version, and size are shown. This is the pinning of the publisher's key
   (TOFU); after confirmation, further installations proceed automatically.
6. Automatic updating **stops and asks** in two cases:
   - the new version declares permissions the plugin does not have (for example, adds
     `files`) — a new right cannot be granted silently;
   - the publisher's `public_key` changes.
   The user sees exactly what is changing and can decline.
7. The toggle does not cancel signature, `id`, ABI, and SHA-256 verification; the
   installation is recorded in the journal and shown as a notification with "What's new".
8. Revoking the permission immediately forbids new checks and installations; the already
   installed version remains. If it is disabled during an ongoing update, the installation
   is cancelled before the folders are renamed: the current version remains, and the
   temporary folder is deleted.
9. For the cautious, there is an application setting "Ask before each plugin update
   installation" (disabled by default): it restores confirmation for every version without
   changing anything else.
10. The plugin row also offers "Check for update", "Update now", and "Roll back" (if a
    backup exists).

### 5.3. Sandbox Host API

Two imports are added (names are a proposal, open to discussion):

| Import | Signature | Purpose |
|---|---|---|
| `bimcraft.update_check` | `() -> i64` | check the channel and return information about the available version |
| `bimcraft.update_apply` | `() -> i64` | install the available version after user consent |

1. Both functions return a packed response in the ABI format (pointer in the upper 32
   bits, length in the lower ones), like `bimcraft.execute` and `bimcraft.info`.
2. The response is an envelope `{"ok": true, "result": …}` or
   `{"ok": false, "error": "…"}`.
3. `update_check` returns:

   ```json
   {"current": "0.2.1", "available": "0.3.0", "released": "2026-11-18",
    "size": 762000, "notes": "…", "host": "example.org",
    "key": "ed25519:AB12…"}
   ```

   The `available` field equals `current` if there is no update. If a check has already
   been done in this session, the response is taken from memory; otherwise one check is
   allowed, no more than once an hour (5.10).
4. `update_apply`:

   - when auto-update is off, opens a modal confirmation: the plugin, current and new
     versions, channel host, size, key fingerprint, "What's new";
   - if the user declines, returns `{"ok": false, "error": "the user declined"}`;
   - on success installs the version and returns
     `{"ok": true, "result": {"installed": "0.3.0", "restart": true}}`.

5. **The URL is taken only from the installed manifest.** The plugin cannot pass an
   arbitrary address: this prevents SSRF and turning the permission into "internet
   access".
6. Both functions require the `update` permission; otherwise —
   `{"ok": false, "error": "the plugin is not allowed to update itself"}`.
7. The host performs the network request itself, in a separate thread, with a time limit;
   the plugin call waits for a response no longer than the limit. A network hang must not
   block the window longer than the agreed time (see 5.8).

### 5.4. Update Channel Feed

Format (schema 1):

```json
{
  "schema": 1,
  "plugin": {"id": "bimcraft-scripts", "abi": 1},
  "version": "0.3.0",
  "released": "2026-11-18",
  "notes": "What changed, short.",
  "module": {
    "url": "https://example.org/bimcraft-scripts/bimcraft-scripts-0.3.0.zip",
    "sha256": "…64 hex…",
    "size": 762000
  },
  "signature": "ed25519:BASE64"
}
```

1. The signature is computed over the canonical JSON form of all fields except
   `signature` (key order is fixed, nested objects as in the example), with the
   publisher's key.
2. It is verified before the archive is downloaded: an unsigned or incorrectly signed
   feed is rejected, with no network requests to the archive.
3. Only `https` is allowed for `module.url`; redirects — no more than 3, https only.
4. Limits: feed ≤ 64 KiB, archive ≤ 32 MiB (the module inside — ≤ 16 MiB), connection
   time 10 s, total time 30 s.
5. Loopback and link-local addresses are forbidden by default; an exception for local
   development is allowed only via a separate application setting and an explicit
   warning.
6. If the signature is not used (no `public_key`/`signature` in the feed), updating is
   allowed only with a separate "allow unsigned updates" setting, disabled by default,
   with a warning on every installation.
7. Redirects are checked by `bimcraft_update::net::hop_allowed`; the download goes through
   `bimcraft_update::Net` with a byte limit, like the application installer.

### 5.5. Checks Before Installation

The order is mandatory; any rejection stops the installation and does not change the
plugins folder:

1. the `update` permission is granted;
2. the feed is received, its size is within limits, the JSON is valid, the schema is
   supported;
3. the feed signature is valid and the key matches the pinned one;
4. `plugin.id` matches the `id` of the installed plugin;
5. `plugin.abi` is supported by the host;
6. the version differs from the current one; a downgrade is allowed only with separate
   user confirmation and a "this is a rollback" warning;
7. the archive is downloaded, and its size and SHA-256 match the feed;
8. the archive is unpacked into a temporary folder: `plugin.json` at the root, the module
   path inside the folder, no absolute paths, `..`, symlinks, or extra files outside what
   is declared;
9. the archive manifest passes the existing `Manifest::check`, its `id` matches, `abi` is
   supported, and `update.feed` and `public_key` are either the same or the key change is
   confirmed by the user;
10. the version in the archive manifest matches the feed version.

### 5.6. Atomic Replacement and Rollback

1. The temporary folder is created inside the plugins folder: `.update-<id>-<random>`.
2. The current folder is renamed to `<id>.backup-<date>-<time>` (not deleted); the
   temporary one is renamed to `<id>`.
3. `state.json` is not touched: the key is the `id`, so the disabled flag and permissions
   are preserved.
4. No more than N backups per plugin are kept (N = 3 by default, an application setting);
   the oldest are deleted.
5. A failure at any step after renaming: the current folder is restored to its place, the
   temporary one is deleted, and the reason is shown to the user.
6. No more than one update runs at a time: a lock file in the plugins folder; a second
   call waits or receives the error "an update is already in progress".
7. The update is not performed if the plugins folder is not writable; the reason is shown
   to the user.

### 5.7. Applying the New Version

1. Modules are loaded at startup: after installation, the new version is **not swapped in
   during the current session**. The host marks the plugin as "installed, restart
   required" and shows this in Manage → Plugins and in a notification.
2. The old version's commands keep working until restart; there is no mixing of versions
   in one session.
3. After restart the new version is loaded and the old commands are replaced by new ones;
   `state.json` and permissions already refer to the same `id`.
4. If the new version fails to load (broken module, manifest rejection), it goes into the
   "rejected" status with the reason in the journal; a backup remains alongside, and the
   user can restore it by renaming or with the "Roll back" button (see 5.9).

### 5.8. Limits and Errors

| What | Value |
|---|---|
| feed | ≤ 64 KiB |
| update archive | ≤ 32 MiB |
| module | ≤ 16 MiB (existing `MAX_MODULE`) |
| connection time | 10 s |
| total check/download time | 30 s |
| redirect hops | 3, https only |
| concurrent updates | 1 |

Errors are in human-readable text: "the channel is not responding", "the signature did not
match", "the archive contains a different plugin", "this version is not newer than the
installed one", "no permission to update", "not enough space". The plugin receives them in
the envelope, the user — in the window and the journal.

### 5.9. Journal, History, and Rollback

1. Every event is written to the existing plugin journal: check, rejection, installation,
   rollback, with versions, host, and digest.
2. An update history is added to `state.json` (no more than 50 entries):
   `{"id", "from", "to", "at", "sha256", "host", "by": "auto"|"manual"}`.
3. In Manage → Plugins the plugin has "Check for update", "Update now", "Roll back"
   (if a backup exists), with the version and date shown.
4. A rollback is the same atomic replacement, but with a backup selection; the backup's
   signature is not re-verified (it was already installed), but the digest is shown.

### 5.10. When Updates Are Checked

1. **Once at BIMCraft startup**, in the background and without blocking the window: the
   host iterates over plugins with the `update` permission.
2. There is **no periodic polling by timer**: an update is applied only after a restart
   anyway, and the network is not needed during operation. A long session will see the new
   version at the next startup — this is intentional.
3. The "Check for update" button works at any time, no more than once a minute.
4. A plugin's `update_check` call: if a check has already been done in this session — the
   response comes from memory, without network; if not — one check, but no more than once
   an hour per plugin. Without permission — an error.
5. After installation there are no new checks in this session: the next one is at the next
   startup or via the button.
6. No network or the channel is not responding: no modal windows, one line in the journal;
   retries until a manual check or the next startup.
7. Checks of one plugin do not run in parallel; the overall pass is a single task,
   sequentially over plugins.
8. The polling interval is not a manifest setting: a publisher cannot make the application
   access the network more often than at startup and via the button.

## 6. User Scenarios

1. **The user has not granted permission.** The plugin calls `update_check` → "not allowed"
   error; in the plugin window — "Allow update" with a warning.
2. **Permission granted, an update exists.** The plugin calls `update_check`, receives the
   version, then calls `update_apply`; the host shows a confirmation; after consent it
   downloads, verifies, installs, and requests a restart.
3. **The user declined.** No installation, the `the user declined` error, the version does
   not change.
4. **The signature did not match.** No installation, "the signature did not match" in the
   journal, the old version keeps working.
5. **The "Update plugin" toggle is on.** At BIMCraft startup the host checks the channel;
   the first installation asks and pins the channel and key, then installation proceeds
   without a question, with a notification and a history record. The signature, id, and ABI
   are verified just as strictly; new permissions or a key change trigger the question
   again.
6. **The network is unavailable.** The check returns an error, updating does not start, the
   retry is via the button or the plugin's next call.
7. **Rollback.** The user selects "Roll back"; the folders are swapped atomically, and the
   previous version works after a restart.

## 7. Changes in the BIMCraft Code

| File | What changes |
|---|---|
| `crates/plugin/src/manifest.rs` | `PERMISSIONS` + `"update"`; the `UpdateFeed { feed, public_key }` struct; checks in `Manifest::check`; the `asks_for_update` method |
| `crates/plugin/src/lib.rs` | reading/writing `state.json`: the pinned key, the update history; `set_permission` with unchanged semantics |
| `crates/plugin/src/runtime.rs` | `HOST` + `update_check`, `update_apply`; passing the identifier, folder, permissions, and `Arc<dyn Net>` into `HostState`; limits and timeouts |
| new module `crates/plugin/src/update.rs` | feed, Ed25519 signature (`ring`), download via `bimcraft_update::Net`, unpacking, checks, atomic replacement, rollback, lock |
| `crates/plugin/Cargo.toml` | `bimcraft-update.workspace = true`; `zip` (or the minimal ZIP reader moved from `delivery/backend/tools`) with `flate2`, already present in the workspace; no new network dependencies |
| `crates/update/src/lib.rs` | `sha256_of`/`sha256_file` and `Net` become `pub` for plugins if not already; `REPO`, `LATEST_URL` are not used — the plugin channel is its own |
| `crates/ui-egui/src/plugins.rs` | the `update` permission, check/update/rollback buttons, modal confirmation, "restart required" status, notifications |
| `docs/areas/plugins.md` | the "Plugin update" section: permission, channel, signature, rollback |
| `crates/plugin/tests/` | tests from section 9 |

The network code lives in a separate module and does not enter the sandbox: `runtime` only
calls it from the host.

## 8. Compatibility

1. `abi` stays `1`: existing modules do not change, and the new import is added only for
   plugins that declare it. A module without the update import loads on both old and new
   builds.
2. A manifest with `"permissions": ["update"]` requires a new BIMCraft build: the old one
   will reject the unknown permission — this is expected and must be documented in the
   plugin documentation.
3. The `update` field in the manifest is optional; old plugins do not have it and cannot
   update themselves.
4. Extending the feed language (schema 2) is allowed only while preserving the ability to
   read schema 1.

## 9. Acceptance Criteria and Tests

Mandatory tests (before the code, as is customary in the project):

1. manifest: `update` without an `update` block — rejected; a block without the
   permission — rejected; an `http` feed — rejected; a key of the wrong length —
   rejected;
2. permission: without a grant, `update_check`/`update_apply` return an error; after a
   grant they work; after revocation — an error again;
3. feed: bad signature, foreign `id`, unsupported `abi`, size over the limit, a redirect
   to `http` — rejected before the archive is downloaded;
4. archive: digest mismatch, size mismatch, missing `plugin.json`, a module path escaping
   the folder, a symlink — rejected;
5. version: the same version — rejected; a downgrade without consent — rejected; a
   downgrade with consent — installation with a warning;
6. replacement: a successful installation leaves a backup, `state.json` is unchanged,
   the status is "restart required"; a failure mid-replacement restores the previous
   folder;
7. rollback restores the previous version and does not touch permissions;
8. UI: granting/revoking the permission, showing the channel and key, confirmation,
   notification;
9. regression: a plugin without updating behaves as before; `files` works independently;
   commands and undo are unchanged;
10. the network in tests is `bimcraft_update::testing::Fake` (a table of responses),
    with no real requests; temporary folders use `tempfile`, as in `crates/plugin/tests`;
11. startup: the check runs once at startup, there is no timer; a manual check works at any
    time; a repeated `update_check` in a session responds from memory;
12. the combined toggle: off — `update_check`/`update_apply` return an error; on — the
    first installation asks, the next proceeds without a question; a new version with an
    additional permission (`files`) or a new key stops the automatic flow and asks.

## 10. Threats and Protection Boundaries

- **A plugin with the update right can become anything.** That is why the permission is
  granted by a human, is disabled by default, and every update is confirmed by default.
  This is a deliberate expansion of trust, not a "hidden feature".
- **Channel or CDN substitution** is covered by the feed signature and the archive digest;
  the key is pinned at installation, and a key change requires consent.
- **A malicious feed** cannot lead the host to an arbitrary address: the URL comes from the
  installed manifest, not from the plugin call.
- **A client-side attack** (memory dump, interception) remains possible, as it does today:
  the sandbox limits accidents and mistakes, not a determined adversary.
- **Rollback and backups** are not protection against an attacker but a means of recovery
  after a publisher's mistake.

## 11. Open Questions for the Maintainer

1. Should the feed signature be mandatory for plugins from the "store" and optional for
   local ones?
2. Is a host allowlist needed in the application settings (for corporate mirrors)?
3. How many backups should be kept, and should they be deleted automatically by age?
4. Should "silent" auto-update without a notification be allowed (currently — no)?
5. Is a separate "update installed but failed to load" status needed, with a rollback
   button right in the window?
6. Is background polling needed for sessions that last for days — or are startup and the
   button enough (currently: enough)?
6. Should a `public_key` change be treated as an event requiring the permission to be
   granted again (currently — yes)?

## 12. Alternatives

- **An external updater outside BIMCraft.** Rejected: it requires file system rights and
  bypasses the sandbox; the user does not see what is happening, and the application does
  not control the version.
- **Grant the plugin full network and file access.** Rejected: this gives the plugin more
  than it needs for updating and breaks the "commands are the only API" model.
- **Updating only manually with the installer.** Remains as a fallback path and always
  works; this spec does not remove it.

## 13. Summary

A minimal change that provides self-update without losing the sandbox: a new `update`
permission, an update channel in the manifest, two host functions, signature and digest
verification, atomic replacement with a backup, and a mandatory restart. The user
explicitly allows updating, sees the source and version, and can revoke the permission or
roll back.
