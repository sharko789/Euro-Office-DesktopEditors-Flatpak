# Euro-Office Desktop Editors — Flatpak

> **Fair warning: this was almost entirely vibe-coded.** An AI assistant read the working
> ONLYOFFICE recipe, peeked inside the Euro-Office tarball, and wrote this. It builds and
> launches on **x86_64**, but treat every detail as "works on my machine — verify before
> trusting." Where something is a guess rather than a checked fact, it's called out.
>
> **This is a proof of concept / source of inspiration, not a finished product.** It's meant
> to show that a Flatpak for Desktop Editors is doable and to give the Euro-Office team (or
> anyone) something concrete to start from. It is **not** intended to be maintained or
> published as-is — expect rough edges, and don't treat it as a commitment.

A Flatpak for **Euro-Office Desktop Editors** (a fork of ONLYOFFICE Desktop Editors),
adapted from the working
[org.onlyoffice.desktopeditors](https://github.com/flathub/org.onlyoffice.desktopeditors) recipe.
The Euro-Office tarball is a *prebuilt* app, so there's nothing to compile — the manifest
just unpacks it and fixes up the paths so it runs inside a Flatpak.

## Files

| File | What it is |
|------|------------|
| `com.github.Euro-Office.DesktopEditors.json` | the flatpak-builder manifest (the actual recipe) |
| `com.github.Euro-Office.DesktopEditors.metainfo.xml` | AppStream metadata (name, description, screenshots, licence) |
| `flathub.json` | tells Flathub which arches to build (x86_64, aarch64) |

**App-id:** `com.github.Euro-Office.DesktopEditors` — Euro-Office has no domain yet, so a
GitHub-based reverse-DNS id is used.

## What the recipe does

1. **Unpacks the release tarball.** The app lives at `opt/euro-office/euro-office/`
   (ONLYOFFICE uses `opt/onlyoffice/desktopeditors/`), so the paths differ.
   `strip-components: 0` keeps the top-level `opt/` + `usr/` where the commands expect them.
2. **Moves the app to `/app/opt/euro-office/`** and copies the bundled launcher
   `usr/bin/euro-office-euro-office` to `/app/bin/eurooffice`.
3. **`sed`-rewrites the launcher.** That shell wrapper hard-codes
   `APP_PATH=/opt/euro-office/euro-office`; inside a Flatpak it becomes
   `/app/opt/euro-office/euro-office`. (The `DesktopEditors` binary itself has no
   hard-coded paths — only the wrapper does.)
4. **Renames the `.desktop` entry + icons** to the app-id and points `Exec=` at `eurooffice`.
5. **Copies `ar` + `libbfd`** from the SDK into the app (the converter's `x2t` needs them;
   carried over from the ONLYOFFICE recipe).

### Why `io.qt.qtwebengine.BaseApp` and not the ONLYOFFICE Electron base?

The app is **CEF + Qt6** and bundles its own Chromium/CEF *and* its own Qt6, so it doesn't
need the base app's Qt at all — it only uses the base app as a "kitchen sink" of system
libraries. We use **`io.qt.qtwebengine.BaseApp` 6.11** (on `org.kde.Platform` 6.11) because:

- It's the natural base for a Qt6 app.
- **It ships `libgssapi_krb5.so.2` (krb5 / GSS-API).** That's a transitive dependency of the
  app's network stack that the `org.freedesktop.Platform` runtime does *not* provide. The
  first build against the freedesktop/Electron base built fine but **failed at launch** with
  `error while loading shared libraries: libgssapi_krb5.so.2`. The QtWebEngine base builds
  krb5 itself, so the error is gone.

Everything else the binary needs (`libX11`, `libgtk-3`, `libglib-2.0`, `libcups`,
`libdbus-1`, `libnotify`, …) comes from the KDE platform + base app too.
(Highest glibc symbol the app needs is **GLIBC_2.34** — well within the 6.11 runtime.)

## Building

Assumes `flatpak`, `flatpak-builder`, and the Flathub remote are already set up.

1. Install the runtime + SDK + base app the recipe needs (one-time):
   ```sh
   flatpak install --user flathub \
     org.kde.Platform//6.11 org.kde.Sdk//6.11 io.qt.qtwebengine.BaseApp//6.11
   ```
2. Build + install for your user:
   ```sh
   flatpak-builder --force-clean --install --user --repo=repo build \
     com.github.Euro-Office.DesktopEditors.json
   ```
3. Run it:
   ```sh
   flatpak run com.github.Euro-Office.DesktopEditors
   ```

The **x86_64** `sha256` is filled in. The **aarch64** one is still a placeholder
(`REPLACE_WITH_AARCH64_SHA256`) — fill it in before building aarch64 or submitting:
```sh
curl -sL -o aarch64.tar.xz \
  https://github.com/Euro-Office/DesktopEditors/releases/download/v9.3.5-2/euro-office-euro-office-9.3.5-2-aarch64.tar.xz
sha256sum aarch64.tar.xz   # paste the hex into the aarch64 "sha256" in the manifest
```

### If something breaks
- **`error while loading shared libraries: <lib>`** at launch → a lib the base app doesn't
  provide. Note the name and add a module for it (or pick a base app that has it).
- **`cp …/libbfd-*.so: No such file`** during the build → `org.kde.Sdk` lays out multiarch
  libs differently than the command expects; adjust that glob/path.
- **AppStream lint warnings** → run `appstreamcli validate` on the `.metainfo.xml` and fix.

## Publishing to Flathub

Flathub apps live in `github.com/flathub/<app-id>` and are built by Flathub's CI. Roughly:

1. Put the three files in a Git repo (ideally owned by the Euro-Office org).
2. Open a PR against `github.com/flathub/com.github.Euro-Office.DesktopEditors` (Flathub's bot
   creates the repo on merge). If you can't open it, ping a Flathub dev to seed it.
3. Keep CI green for **both arches** + the appstream lint.
4. A Flathub admin reviews (licence = `AGPL-3.0-only` ✔, permissions, security) and merges.

**Likely review feedback:**
- The recipe uses `--filesystem=host` (copied from ONLYOFFICE). Reviewers usually want the
  **file portal** instead (`--filesystem=xdg-documents` etc.). Start with portals and widen
  only if the app actually breaks.
- The appdata **icon is a placeholder** (a screenshot). Flathub wants a real, square icon.
- Fill in the **aarch64 sha256** (see above).

## Known gaps (the vibe-coded fine print)
- Only **x86_64** has been built and launched here. **aarch64 has never been built.**
- `--filesystem=host` is broad and will probably be narrowed for Flathub.
- The icon is a placeholder.
- None of this has been independently audited — it was generated by reading the tarball and
  the ONLYOFFICE recipe, then iterated against a local build until it launched.
