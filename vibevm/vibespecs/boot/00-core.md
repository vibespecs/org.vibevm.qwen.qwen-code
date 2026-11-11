# Qwen Code application package

This package declares how Vibe installs Qwen Code, the official **standalone**
upstream build, as a portable user application on five targets (Linux glibc
x86_64/aarch64, macOS x86_64/aarch64, Windows x86_64). It contains no binaries.
Each installer option downloads one archive identified by its exact URL and
SHA-256, unpacks it with its top-level directory (`qwen-code/`) stripped, and
declares one launcher, `qwen`, exposed as the public command `qwen`.

The standalone archives bundle their own Node.js, so an installation needs
nothing on PATH: verified by running `bin/qwen --version` under
`env -i PATH=/usr/bin:/bin`, which prints the version and exits 0.

**`vibe.toml` is a template manifest** (`[package] template = true`,
PROP-075): one file with `{version}` and `{sha256:…}` variables plus a
`[template]` declaration, rendered on the client at resolution time. The
digests come from each release's own `SHA256SUMS`
(`https://github.com/QwenLM/qwen-code/releases/download/v{version}/SHA256SUMS`),
which carries no sizes — so no step declares `size`, the same deliberate loss
the Node.js package records.

`[template.versions] kind = "list"`, not `kind = "json"`, for one measured
reason: `[template.versions].url` must carry **no query string** (the validator
refuses `?`), so the only reachable document is the GitHub releases API at its
default page size of 30; that repository releases four product lines from one
tag space plus nightlies, and a selector cannot filter. Measured 2026-11-11:
30 releases → **9** usable `vX.Y.Z` versions spanning 12 days, 1.1 MB per
resolution. A discovered domain would drop every version about a week after
release, including one pinned in a corporate drop. **Do not convert this to
`kind = "json"`** without re-measuring that page, and never to buy brevity.

`[package].version` is the **template revision** (`0.0.1`), not a Qwen Code
version, and it is deliberately **below** every produced version. Rendering a
template fetched from a per-package Git registry is not wired (PROP-075
`#P2-AS-BUILT` (8)), so such a client reads the revision as a release and would
install an unsubstituted `{version}` manifest; numbered below `0.25.1`, it
always loses to a concretely published version. Keep it that way: a revision
bump goes `0.0.1` → `0.0.2`, never to `1.0.0`.

Two tags therefore mean two different things, the arrangement
`org.vibevm.node/node` uses:

- `v0.0.1` on `main` — the template, which every directory-shaped registry
  resolves (`--registry DIR`, `localdir`, a `monorepo` working copy, a mirrored
  layout registry). This is the whole corporate path.
- `v0.27.2` on `release/0.27.2` — the same manifest rendered concretely by
  Vibe's own renderer (`crates/vibe-template`), which is what the public index
  carries and what a public `vibe install` resolves. Never hand-write it: take
  `<settings>/cache/templates/org.vibevm.qwen/qwen-code/rendered/v<version>/vibe.toml`
  after one install from a local registry.

The launcher runs upstream's own shim `bin/qwen` (`bin\qwen.cmd` on Windows),
not the bundled `node` directly, because the shim announces itself through
`QWEN_CODE_LAUNCHER_PATH` — the hint Qwen Code reads to stamp its own entry
point for subprocesses and to switch off its managed-npm self-update path.
Calling `node lib/cli-entry.js` works but loses both. The bundled `node` is
not exposed on PATH.

Three asset families of those releases are excluded on purpose and must stay
excluded: the `*-opentui-preview` archives (a preview of a different terminal
UI), `qwen-managed-agent-server-*.jar` (a different product), and the loose
`cli.js`.

Before changing the package, read `vibe.toml` and `README.md`. **Do not add a
generator**: the renderer is Vibe's own pure one. A new upstream release is one
`[[template.versions.item]]` row plus a revision bump plus a concrete tag —
and first a check that the release actually publishes all five archives and a
`SHA256SUMS` naming them, because a listed version that cannot render is a
refusal at install time.

Upstream is Apache-2.0; this package's own files are UPL-1.0. Both are stated
in `README.md` and must stay stated.
