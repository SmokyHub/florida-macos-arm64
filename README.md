# florida-macos-arm64

Automated builds of [Frida](https://github.com/frida/frida) patched with the
[**Ylarod/Florida**](https://github.com/Ylarod/Florida) anti-detection patches,
targeting **macOS on Apple Silicon (`macos-arm64`)**.

A GitHub Actions workflow runs daily. It pulls the **latest Frida release**,
applies the vendored Florida patches, builds on an Apple-Silicon runner, and
publishes the binaries as a release tagged with the Frida version.

## Releases

Each release contains, gzip-compressed, for `macos-arm64`:

| Asset | Description |
| --- | --- |
| `florida-server-<ver>-macos-arm64.gz` | patched `frida-server` |
| `florida-inject-<ver>-macos-arm64.gz` | patched `frida-inject` |
| `florida-gadget-<ver>-macos-arm64.dylib.gz` | patched `frida-gadget` |
| `florida-gumjs-<ver>-macos-arm64.a.gz` | patched `libfrida-gumjs-1.0.a` |
| `SHA256SUMS` | checksums for the above |

Download, verify, then:

```sh
gunzip florida-server-<ver>-macos-arm64.gz
chmod +x florida-server-<ver>-macos-arm64
```

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
- **Manual:** run the *Build Florida (macOS arm64)* workflow from the Actions tab
  (`workflow_dispatch`). A manual or push trigger will rebuild and replace an
  existing release for the same version; the scheduled run skips versions already
  released.

## Updating the patches

Patches are vendored under [`patches/`](patches/) from Ylarod/Florida. To move to
a newer patch set, copy the updated files from upstream over `patches/` and push;
the workflow re-runs on changes to `patches/**`.

## Intended use

For authorized security research, mobile/app pentesting and CTF use on systems
you own or are permitted to test.

## Credits & license

Frida © the Frida authors. The patches are from
[Ylarod/Florida](https://github.com/Ylarod/Florida). This repository packages and
builds that work and is licensed **GPL-3.0** (see [`LICENSE`](LICENSE)), matching
upstream Florida.
