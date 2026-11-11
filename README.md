# Qwen Code application package

This package installs [Qwen Code](https://github.com/QwenLM/qwen-code) — the
open-source AI coding agent that lives in your terminal — as a portable Vibe
user application. It contains no binaries: each installation downloads one
pinned upstream archive and verifies it against the SHA-256 digest published in
that release's own `SHA256SUMS`.

The archives are the **standalone** builds, which carry their own Node.js
runtime. Nothing outside the payload is needed:

```text
$ env -i PATH=/usr/bin:/bin HOME=$HOME ~/.vibe/opt/bin/qwen --version
0.27.2
```

```text
vibe install -g org.vibevm.qwen/qwen-code
qwen --version
qwen
```

The command appears in `opt/bin` of the Vibe settings directory
(`~/.vibe/opt/bin` by default) as a PATH proxy; put that directory on PATH
(`vibe self doctor --fix`), or open a shell with `vibe shell`.

| Archive | From | Digest from |
| --- | --- | --- |
| `qwen-code-<target>.tar.gz` / `.zip` (Qwen Code, its bundled dependencies and a bundled Node.js 22) | the upstream [Qwen Code releases](https://github.com/QwenLM/qwen-code/releases) | that release's `SHA256SUMS` |

## One template, sixteen releases — and one concrete tag

`vibe.toml` on `main` is a **template manifest** (`[package] template = true`,
[PROP-075](https://github.com/vibevm/vibevm)): it is written once with
`{version}` and `{sha256:…}` variables plus a `[template]` declaration naming
where the versions and the digests come from, and Vibe renders it on the
client, at resolution time, for whichever version is asked for. One file covers
the whole version domain, and no digest is ever transcribed by hand.

`[package].version` (`0.0.1`) is the **template revision**, not a Qwen Code
version. It is numbered below every produced version on purpose — see
*Publishing* below.

### Why the versions are listed and not discovered

`[template.versions] kind = "list"`, one row per version, and **not**
`kind = "json"` over the GitHub releases API. That is a measured decision, not
a preference:

- `[template.versions].url` must be a credential-free HTTPS URL with **no query
  string** — the manifest validator refuses `?`. So the only reachable document
  is `https://api.github.com/repos/QwenLM/qwen-code/releases`, with GitHub's
  default page size of 30 and no way to ask for more.
- That repository releases **four product lines from one tag space** —
  `vX.Y.Z` (Qwen Code), `sdk-typescript-v…`, `desktop-v…`, `cua-driver-rs-v…` —
  plus nightlies, and a JSON Pointer selector cannot filter.
- Measured on 2026-11-11: the default page held 30 releases, of which **9** were
  usable `vX.Y.Z` versions, spanning **12 days**, in 1.1 MB of JSON per
  resolution.

A discovered domain would therefore *drop* each version about a week after its
release — including a version pinned in a corporate drop. A listed domain keeps
every listed version installable for good. The price is one row per upstream
release; the digests still come from upstream, so a release costs one line and
no checksum transcription.

### Excluded on purpose

Three kinds of asset in those releases are **not** part of this package:

- `qwen-code-<target>-opentui-preview.tar.gz` / `.zip` — a preview build of a
  different terminal UI, not the release build.
- `qwen-managed-agent-server-<version>.jar` (and its `.sha256`) — a different
  product (a JVM server), not the CLI.
- `cli.js` — a loose script, not a self-contained installation.

The other three release lines of that repository (the TypeScript SDK, the
desktop app and the CUA driver) are separate products and are not packaged
here.

## Targets

| Installer option | `--platform` / `--arch` | Upstream asset |
| --- | --- | --- |
| `linux-x64` | `linux` / `x86_64` | `qwen-code-linux-x64.tar.gz` |
| `linux-arm64` | `linux` / `aarch64` | `qwen-code-linux-arm64.tar.gz` |
| `macos-x64` | `macos` / `x86_64` | `qwen-code-darwin-x64.tar.gz` |
| `macos-arm64` | `macos` / `aarch64` | `qwen-code-darwin-arm64.tar.gz` |
| `windows-x64` | `windows` / `x86_64` | `qwen-code-win-x64.zip` |

Vibe picks the option of the host's platform and architecture and asks when it
cannot decide. Upstream publishes no Windows ARM64 and no musl build, so there
is no row for either; a musl system is detected as `linux` and would get the
glibc archive, which the bundled Node.js does not run — that combination is not
supported by this package because upstream does not build it.

Every archive unpacks from a single top-level directory `qwen-code/`
(`payload_root = "qwen-code"`) and contains **no symbolic or special entries**,
so an ordinary `archive` step places it — no recipe, no system `tar`.

## How the launcher runs

Each option declares one launcher, `qwen`, and exposes it as the public command
`qwen`. The launcher runs **upstream's own shim**, `bin/qwen` (`bin\qwen.cmd` on
Windows), and not the bundled `node` directly. That is deliberate: the shim
selects the payload's own Node.js *and* announces itself through
`QWEN_CODE_LAUNCHER_PATH`, which Qwen Code reads to learn which entry point
reaches this build. With that hint it stamps the shim as its own entry for
subprocesses and disables its managed-npm self-update path; without it, it
would stamp `lib/cli-entry.js` — a `#!/usr/bin/env node` script on a machine
that may have no `node` — and turn the npm update path back on. The shim is two
lines of `sh` and costs nothing.

The bundled `node` is **not** put on PATH: it belongs to this application, not
to the machine. A managed Node.js for general use is
`vibe install -g org.vibevm.node/node`.

## Closed networks

The same package, unedited, installs where `github.com` cannot be reached. The
administrator mirrors it into the corporate registry and the archives land in
the machine's `[artifacts] packages` drop; every archive is still verified
against the digest from upstream's own `SHA256SUMS`.

Because the manifest is a template, the closed network also has to reach the
one document it declares per version — that release's `SHA256SUMS`. The mirror
does that once, on the connected side, and carries the result into the
registry as a **template snapshot** (PROP-075 `#MIRROR-TEMPLATE-SNAPSHOT`); the
client then renders locally from the snapshot with the same code as outside and
makes no request at all. `payloads = true` makes the same run fill the drop.

1. On a connected machine, with a `mirror.toml`:

   ```toml
   schema = 1

   [select]
   packages = ["org.vibevm.qwen/qwen-code"]

   [[template]]
   package = "org.vibevm.qwen/qwen-code"
   versions = ["=0.27.2"]
   payloads = true

   [payloads]
   into = "/srv/vibe-drop"
   max_total = "2 GiB"
   ```

   ```text
   vibe registry mirror vibespecs --into <corporate monorepo url> --config mirror.toml
   ```

   That one run copies the template package, snapshots the digest document of
   the selected version, and downloads all five archives into the drop at

   ```text
   <drop>/org.vibevm.qwen/qwen-code/0.27.2/qwen-code-linux-x64.tar.gz
   <drop>/org.vibevm.qwen/qwen-code/0.27.2/qwen-code-win-x64.zip
   …
   ```

   `vibe artifacts where org.vibevm.qwen/qwen-code` prints, for every installer
   option, the exact file name, digest and destination path, and whether a
   verified copy is already there.

2. A site profile names the registry, the drop and the source policy:

   ```toml
   schema = 1
   name = "example-site"

   [[registry]]
   name = "corp"
   type = "monorepo"
   url = "ssh://git.example.internal/vibe/registry.git"

   [source_policy]
   exclude_hosting = ["github", "gitverse"]

   [artifacts]
   packages = "/srv/vibe-drop"
   ```

   applied with `vibe self apply site.toml`; then
   `vibe install -g org.vibevm.qwen/qwen-code` reads the package and the
   snapshot from `corp` and the archive from the drop, with no connection to
   any public host.

**Without a mirror snapshot a cold machine cannot render the template
offline.** A template's digest document is a real input: a fresh home with the
package in a `localdir` registry and the archive in the drop still refuses,
naming the document and the cache
(`spec://org.vibevm.core/vibevm/common/PROP-075#TEMPLATE-CACHE`). Either mirror
the template (above), or let the machine render the version once while it has
access — after that `<settings>/cache/templates/org.vibevm.qwen/qwen-code/`
holds the immutable digests and `--offline` resolves from them with no request.
A concretely published version (see below) needs no document at all.

With a proxy instead of a drop, the archives and the digest document are
downloaded through the `[proxy]` the site profile declares (`mode = "px"` with
a PAC file is supported).

## Publishing

The registry resolves a per-package repository by its **tag list**, and
rendering a template fetched that way is the one part of PROP-075
`#TEMPLATE-DISCOVERY` that is not wired yet (`#P2-AS-BUILT` (8)): such a client
reads the template's own revision as if it were a release and installs a
manifest whose `{version}` was never substituted. Hence the arrangement, which
is the one `org.vibevm.node/node` uses:

- **`main`, tag `v0.0.1`** — the template. It is what a directory-shaped
  registry resolves: `--registry DIR`, a `localdir` source, a `monorepo`
  working copy, a mirrored layout registry. That covers the whole corporate
  path, where every version in the snapshot is installable from one manifest.
- **tag `v0.27.2`** — the same manifest **rendered concretely** for that one
  version, by Vibe's own renderer (`crates/vibe-template`), on the branch
  `release/0.27.2`. This is the tag the public index carries and the one a
  public `vibe install -g org.vibevm.qwen/qwen-code` resolves.
- The revision is numbered **below** every produced version (`0.0.1` <
  `0.25.1`) so that even a client that ignores the index's `template` flag
  prefers a real release over the template.

A concrete tag is produced, never hand-written:

```text
vibe install -g org.vibevm.qwen/qwen-code --registry <dir holding this package>
cp <settings>/cache/templates/org.vibevm.qwen/qwen-code/rendered/v<version>/vibe.toml .
```

## Maintaining

A new upstream release needs **one row**:

```toml
[[template.versions.item]]
version = "0.28.0"
```

Then, in order:

1. Check the release really carries the five archives and a `SHA256SUMS` naming
   them — `gh release view v0.28.0 -R QwenLM/qwen-code --json assets`.
2. `vibe app versions org.vibevm.qwen/qwen-code --registry .` — the new version
   must appear and render.
3. Bump `[package].version` (the template revision) and tag it.
4. Render the concrete manifest for the new version, commit it on
   `release/0.28.0`, tag `v0.28.0`, and reindex.

What has to be checked when upstream changes *shape* rather than merely
publishing a release:

- a new target (a new archive in `SHA256SUMS`) — a new `[[installer_options]]`
  row, `template_optional = true` until every listed version has it;
- a target upstream stops building — the row becomes `template_optional`
  rather than disappearing, so older versions keep installing;
- `payload_root`, the shim path (`bin/qwen`, `bin\qwen.cmd`) or the asset
  naming changing — the template names all three;
- a release whose archive set is incomplete — leave it out of the list rather
  than letting it render half a manifest.

Upstream publishes digests but **no sizes**, so no step declares `size`: the
step panel falls back to the server's `Content-Length` and a mirror's byte
total is reported as a lower bound.

## License

This package — its manifest and its documents — is UPL-1.0; see `LICENSE.md`.

Qwen Code itself is **not**: it is distributed under the **Apache License
2.0**, whose text every installation keeps in the payload root (`LICENSE`,
and `lib/LICENSE` for the bundled dependencies), together with the Node.js
runtime bundled beside it under its own MIT-based license
(`node/LICENSE`). `[package].license` in the manifest names the license of
what an installation puts on the machine, `Apache-2.0`.
