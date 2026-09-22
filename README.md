# florida-macos-arm64

Automated builds of [Frida](https://github.com/frida/frida) patched with the
[**Ylarod/Florida**](https://github.com/Ylarod/Florida) anti-detection patches,
targeting **macOS on Apple Silicon (`macos-arm64`)** and
**Android arm64 (`android-arm64`, i.e. `arm64-v8a`)**.

A GitHub Actions workflow runs daily. It pulls the **latest Frida release**,
applies the vendored Florida patches, builds each target, and publishes the
binaries as a single release tagged with the Frida version.

## Releases

Each release contains, gzip-compressed:

| Asset | Description |
| --- | --- |
| `florida-server-<ver>-macos-arm64.gz` | `frida-server` for macOS apps on the host |
| `florida-gadget-<ver>-macos-arm64.dylib.gz` | `frida-gadget` (macOS) |
| `florida-inject-<ver>-macos-arm64.gz` | `frida-inject` (macOS) |
| `florida-gumjs-<ver>-macos-arm64.a.gz` | `libfrida-gumjs-1.0.a` (macOS) |
| `florida-server-<ver>-android-arm64.gz` | `frida-server` to run **inside** an arm64 Android device/emulator |
| `florida-gadget-<ver>-android-arm64.so.gz` | `frida-gadget` (Android) |
| `florida-inject-<ver>-android-arm64.gz` | `frida-inject` (Android) |
| `florida-gumjs-<ver>-android-arm64.a.gz` | `libfrida-gumjs-1.0.a` (Android) |
| `SHA256SUMS-macos-arm64`, `SHA256SUMS-android-arm64` | checksums |

The **client** (`frida`, `frida-trace`, …) is *not* here — install it with
`pip install frida==<ver> frida-tools` on your host. Only the target-side
binaries need the Florida patches.

### macOS host + arm64 Android emulator

To instrument Android apps in an `arm64-v8a` emulator running on an
Apple-Silicon Mac, you need the **`android-arm64`** server (not the macOS one):

```sh
gunzip florida-server-<ver>-android-arm64.gz
adb root
adb push florida-server-<ver>-android-arm64 /data/local/tmp/fs
adb shell "chmod 755 /data/local/tmp/fs && /data/local/tmp/fs &"
# from the Mac host (client from pip):
frida -U -f com.example.app -l script.js
```

### macOS apps on the host

```sh
gunzip florida-server-<ver>-macos-arm64.gz
chmod +x florida-server-<ver>-macos-arm64
sudo ./florida-server-<ver>-macos-arm64
# then, from a client:
frida -H 127.0.0.1 -f /path/to/App.app/Contents/MacOS/App -l script.js
```

## Matching the client version to the server

Frida requires the **host client and the target server to be the same version**
— it refuses to connect on a mismatch (e.g. *"unable to communicate with the
remote frida-server; please ensure that major versions match"*). The releases
here always track the **latest Frida release**, so set your client to the tag
you downloaded. Vanilla and Florida builds of the same version are compatible,
so the client stays stock — only the version has to match.

1. **Find the server version.** It's the release tag, and it's in every asset
   name: `florida-server-17.18.0-android-arm64` → **17.18.0**.

2. **Check your current client:**

   ```sh
   frida --version
   ```

3. **Set the client to that exact version** (pip upgrades *or* downgrades):

   ```sh
   pip install -U "frida==17.18.0" frida-tools
   frida --version          # should now print 17.18.0
   ```

   - **pipx:** `pipx runpip frida-tools install "frida==17.18.0"`
   - **venv / conda:** activate the environment first so the change lands there.
   - **"externally-managed-environment" error** (Homebrew/system Python): use a
     venv, or `pipx`, or append `--break-system-packages`.

4. **Confirm the handshake:**

   ```sh
   frida-ps -U              # Android emulator/device over adb
   frida-ps -H 127.0.0.1    # local macOS server
   ```

When a newer Frida comes out, this repo publishes a new release; bump the client
to that new tag the same way (step 3) to keep them matched.

## What the Florida patches change

The patches rename Frida's telltale strings, symbols and thread names
(`gum-js-loop`, `gmain`, the `frida:rpc` marker, the agent entrypoint symbol,
etc.) and adjust the protocol handling, to make instrumentation harder to
fingerprint. See the upstream repo for details.

### macOS caveat

Florida's `anti-anti-frida.py` binary-rewriting step is gated, upstream, to the
`linux`/`android` branch of Frida's `embed-agent.py`, so it is **not applied to
the macOS agent**. All the cross-platform source patches (string/symbol/thread
renames, protocol changes) still apply and affect the macOS build; only that
extra post-link symbol rewrite is skipped. The Linux-specific patches (e.g.
`memfd`, `linux-host-session`) apply to the source tree but do not affect the
macOS binaries.

## Rebuilding / triggering

- **Automatic:** daily schedule; a new Frida release produces a new release here.
- **Manual:** run the *Build Florida (macOS arm64 + Android arm64)* workflow from the Actions tab
  (`workflow_dispatch`). A manual or push trigger will rebuild and replace an
  existing release for the same version; the scheduled run skips versions already
  released.

## Updating the patches

Patches are vendored under [`patches/`](patches/) from Ylarod/Florida. To move to
a newer patch set, copy the updated files from upstream over `patches/` and push;
the workflow re-runs on changes to `patches/**`.

## Code signing

The **macOS** binaries are **ad-hoc signed** (`codesign -s -`) — Frida's macOS
build requires a signing identity and CI has no Apple certificate. Ad-hoc-signed
`frida-server` is expected to be run as **root** (`sudo`), and hardened-runtime
apps will refuse injection unless SIP is disabled. If you need a Developer ID /
entitlement-signed build, re-sign the downloaded binaries locally, or fork and
set the `MACOS_CERTID` env in the workflow to your identity. The Android binaries
are ordinary ELF executables and are not code-signed.

## Intended use

For authorized security research, mobile/app pentesting and CTF use on systems
you own or are permitted to test.

## Credits & license

Frida © the Frida authors. The patches are from
[Ylarod/Florida](https://github.com/Ylarod/Florida). This repository packages and
builds that work and is licensed **GPL-3.0** (see [`LICENSE`](LICENSE)), matching
upstream Florida.
