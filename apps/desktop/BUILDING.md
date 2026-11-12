# Development

Run the dev server without env validation or auth:

```bash
SKIP_ENV_VALIDATION=1 bun run dev
```

This skips environment variable validation and the sign-in screen. Desktop chat also falls back to local-only session bootstrap in this mode, so you can test chat/streaming without the cloud API as long as you have local model credentials configured.

# Release

When building for release, make sure `node-pty` is built for the correct architecture with `bun run install:deps`, then run `bun run release`.

# Linux (AppImage) local build

From `apps/desktop`:

```bash
bun run clean:dev
bun run compile:app
bun run package -- --publish never --config electron-builder.ts
```

Expected outputs in `apps/desktop/release/`:

- `*.AppImage`
- `*-linux.yml` (Linux auto-update manifest)

# Windows (NSIS x64)

Windows is not part of the release pipeline (`release-desktop.yml` builds macOS + Linux only). To produce an installer, run the opt-in workflow:

```bash
gh workflow run build-desktop-windows.yml -R <owner>/superset --ref <branch>
```

It builds on `windows-latest` and uploads `Superset-<version>-x64.exe` plus the updater manifests as artifacts. The build is unsigned — SmartScreen will warn on first launch.

Local equivalent, from `apps/desktop` on a Windows x64 machine (needs VS Build Tools with the MSVC toolset, Python for node-gyp, and Bun):

```bash
bun install --frozen --ignore-scripts
bun run install:deps
bun run clean:dev
bun run generate:icons
bun run compile:app
bun run package -- --publish never --config electron-builder.ts --win --x64
```

Two constraints:

- The build must run **on** Windows. `electron.vite.config.ts` bakes the host `process.platform` into the renderer bundle, and `npmRebuild` compiles `better-sqlite3` / `node-pty` / `native-keymap` against the Electron ABI with MSVC. A macOS/Linux cross-build would ship host-platform values and darwin/linux `.node` binaries.
- `node-pty`'s `binding.gyp` requests Spectre-mitigated libraries on Windows; without the `Microsoft.VisualStudio.Component.*.Spectre` VS component the rebuild fails with MSB8040.

Known limitation: terminals do not work on Windows yet. `pty-daemon` / `terminal-host` communicate over Unix domain sockets (`~/.superset/terminal-host.sock`), which Windows does not support — those paths need named pipes.

# Linux auto-update verification (local)

From `apps/desktop` after packaging:

```bash
ls -la release/*.AppImage
ls -la release/*-linux.yml
```

If both files exist, packaging produced the Linux artifact + updater metadata that `electron-updater` expects.
