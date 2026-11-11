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

## Why vibe cares about Qwen Code's system prompt, and what a future version would change

This section exists because the question will be asked again, by whoever
packages the next Qwen Code release — person or agent — and because the answer
is a *measurement* that has to be re-taken rather than a decision that can be
looked up. It is kept byte-identical in
[`PROP-076#SYS-QWEN`](https://github.com/vibevm/vibevm) and in this package's
boot snippet.

### (a) Why any of this exists

A project's standing instructions are the same bytes in every session. A
harness that learns them by **reading files with tools** pays for them in that
session's context every time, *after* the session has already started. Put the
same bytes in the harness's own instruction slot **before** the first request
and the provider caches them once: the instructions sit in the prompt's
cache-stable prefix, every later session in that project reads them from the
cache, and the agent's own tool budget goes on the task instead.

A harness with a slot that is **additive**, **file-based**, **per-launch** and
**not inherited by subagents** gets that for the price of one flag. Claude Code
and Codex each have one. A harness whose only subagent-safe slot **replaces**
its own prompt does not: delivering there means putting the harness's own
prompt in front of ours, which is only safe if that prompt is carried as
versioned data with provenance and proved against the installed program per
version. That is what `org.vibevm.opencode/opencode` does, and why it carries
`prompt-bases/<version>/` at all.

**Qwen Code is in the `launch` tier, through a third slot that replaces
exactly one paragraph.** `QWEN_SYSTEM_IDENTITY_MD` names a file whose
contents replace the base prompt's **first paragraph** — its single identity
sentence — and nothing else; Qwen Code's own tests pin both halves of that
(`packages/core/src/core/prompts.test.ts:614-628` and `:849-860`), the
resolver is `resolveCoreIdentityOverride` at
`packages/core/src/core/prompts.ts:119-153`, consumed at `:545` inside
`getCoreSystemPrompt`, and a subagent never enters that path
(`packages/core/src/agents/runtime/agent-core.ts:370-405`). So vibe writes
Qwen Code's **own** identity sentence, a blank line and the lane into one
file and names it in that variable — and the harness goes on building every
other byte of its prompt itself, live, for the session it is actually
starting.

**Why that is clean, and why it needs no vendored text.** The whole base
prompt is a function of the session, which is what closes the slot below.
The identity *sentence* is a function of two things — the interaction mode
and whether an output style is active — so the launcher samples it **at
launch** from the installed package's own `getCoreSystemPrompt`, the way
upstream's own test samples it (`getCoreSystemPrompt().split('\n\n', 1)[0]`).
Nothing of anyone's prompt is stored, so **this package carries no
`prompt-bases/` directory and never will**: there is nothing to vendor, no
digest of someone else's prose, and nothing to go stale at an upgrade. That
is the one real advantage Qwen Code has over OpenCode here, and the shipped
carrier is what it buys.

**The other two slots fail on opposite axes, and both are still closed:**

- `--append-system-prompt` is additive, cache-safe and **not** inherited by an
  ordinary subagent — four properties out of four — and it takes the
  instructions as **text**. A real lane is 200–300 KB; a single argument is
  capped at 128 KiB on Linux and a whole command line at about 32,767
  characters on Windows. Measured on 0.27.2: a 202,349-byte lane is `E2BIG`
  («Argument list too long», errno 7) at `execve`, before the program starts.
  So no lane byte goes there, on any platform — that is a rule, not a
  threshold.
- `QWEN_SYSTEM_MD` names a **file**, is read only on the main session's own
  prompt path, and **replaces** the base prompt. Composing Qwen Code's own base
  with the lane after it was measured to work: the request begins with Qwen
  Code's own prompt byte-exact, carries the lane exactly once at byte 13,858,
  and continues byte-identically to a plain run; live it carried the whole
  202,349-byte lane with the tail marker quoted byte-equal and no file read.
  **What closes it is that the prompt it replaces is a function of the
  session's shape rather than of the version**: under identical probe flags the
  headless base is **13,856** bytes and the interactive one **18,451**,
  differing by whole sections, and a probe with no terminal is *always* the
  headless one, because `--prompt-interactive` refuses piped input. A launcher
  cannot know which copy a given launch needs without allocating a pty and
  starting a second session, and handing over the wrong copy would silently
  swap the harness's own instructions — strictly worse than delivering no lane.
  A stored copy per verified version does not help either: it would be stale
  along that axis on the **first** launch, not at the next upgrade.

The context-file layer (`QWEN.md`, `AGENTS.md`) is additive, byte-stable, and
its `@spec://` tokens survive verbatim — and it reaches **every** subagent, so
it is out on worker exclusion.

`read` remains the floor under all of it, and a project may still declare it:
the router block arrives through `AGENTS.md`, which Qwen Code reads itself,
and the agent opens the boot files with its own tools. A launch that cannot
use the carrier — an unverified Qwen Code version, `--system-prompt`,
`--safe-mode`, `--bare`, an active `QWEN_SYSTEM_MD`, a foreign
`QWEN_SYSTEM_IDENTITY_MD`, or a session shape that cannot be classified —
soft-launches into exactly that tier behind one loud warning.

### (b) Where to look next time

**For a fresh OpenCode version** the runbook is that package's own README,
section *Adding prompt bases for a new OpenCode version*, with
`prompt-bases/2.2.7/PROVENANCE.toml` as the worked example: the seven base
files at their upstream paths, the root `LICENSE` beside them, the 35 assembly
fingerprints each with a `decides` line, and then
`cargo run -p xtask -- opencode-bases --version <new>` in the vibevm
repository, which **refuses** rather than guessing when a file that decides the
composition changed.

**For a fresh Qwen Code version** nothing needs vendoring — see above — and
there is exactly one thing to re-verify, because the shipped carrier is
gated on one version at a time by design: a release that replaced more than
the base prompt's first paragraph, or stopped reading
`QWEN_SYSTEM_IDENTITY_MD`, would swap the harness's own instructions or drop
the lane **silently**.

**How to re-verify, end to end** (in the vibevm repository, offline, no model
called and nothing reachable but `127.0.0.1`):

1. `campaigns/sys-prompt-2026-11/live/make-sample.sh <dir> 202349` — the
   sample project with a real-size lane.
2. `VIBE=<built vibe> campaigns/sys-prompt-2026-11/live/qwen-identity-prove.sh
   <dir> <out>` and the same again with `--headless`. Three real sessions per
   shape, each in its own network namespace (`bwrap --unshare-net`) against a
   loopback provider that records request bodies, each driving one subagent
   spawn; the carrier legs are launched by `vibe agent qwen` itself. It must
   print `VERDICT: PASS`: one contiguous insertion, the inserted bytes equal
   to `AGENT-LANE.md`, the lane exactly once, two launches byte-identical,
   and every subagent request byte-identical to its plain counterpart with
   zero lane canaries.
3. `campaigns/sys-prompt-2026-11/live/qwen-identity-live-check.sh <tiny dir>
   <out>` on a local model — the two conditions a byte diff cannot make: the
   delivered tail marker quoted byte-equal with no file read, and a rule that
   exists only in Qwen Code's own base prompt still obeyed.
4. Then raise the one constant,
   `QWEN_VERIFIED_VERSION` in `crates/vibe-workspace/src/system_prompt/
   qwen.rs`, and the measured-version constant beside it.

The per-launch half needs no maintenance: vibe's own helper samples the
sentence from the installed package every time, refuses a package whose first
paragraph is not a single short sentence or whose prompt no longer varies
that sentence by mode and style, and proves the insertion before the launch.

The question to re-ask on each new version for the **other** two slots is the
one that closed them: **does the base prompt still depend on the session's
shape?** The places that decide it:

| Where | What it decides |
| --- | --- |
| `QWEN_SYSTEM_MD` | the file-based slot: a path whose contents **replace** the base prompt |
| `QWEN_WRITE_SYSTEM_MD` | the dump: the installed program writing its own base prompt out, which is what makes a probe possible at all |
| `QWEN_SYSTEM_IDENTITY_MD` | **the shipped carrier**: a path to a file that replaces the base prompt's FIRST PARAGRAPH and nothing else. `resolveCoreIdentityOverride` is where it is read, and the two tests above are what pin its scope |
| `getCoreSystemPrompt` | the base's assembly, where `isSystemMdActive` and the identity override are consumed, and the first paragraph vibe samples — a subagent never enters this path, which is what makes both slots subagent-safe |
| `renderSubagentSystemPrompt` | whether a child enters that path at all. It did not on 0.27.2; if it ever does, worker exclusion closes this slot too |
| the interaction-mode branch those two read | the 13,856-vs-18,451 split, i.e. the whole blocker |
| `--append-system-prompt` | the additive slot. **Watch for a file variant of it**: that single change would carry a real lane and delete this whole problem for this harness |

Two switches would open the **other** two slots: a **file** form of
`--append-system-prompt`, or a documented way to ask the program for the base
prompt of *the session it is about to start*. Neither is needed for what
ships, and both are worth re-checking on every release.

Costs to keep stated rather than discovered later: `QWEN_SYSTEM_MD` disables
the user's `general.outputStyle`, which Qwen Code says itself; and the
shipped carrier cannot see a style set in `general.outputStyle` at all, since
reading another program's settings precedence is not a launcher's business —
so such a session receives Qwen Code's **unstyled** identity sentence, one
clause of one sentence, while the style's own section still lands (which
upstream's own test pins). `--output-style <name>` on the launch's own
command line **is** read.

**Measured 2026-11-11 on Qwen Code 0.27.2**, offline from the committed tree
and through `vibe agent qwen` itself: interactive and headless both
`VERDICT: PASS` — one contiguous insertion of 208,787 raw bytes at raw offset
303 and 306 respectively, the inserted bytes `AGENT-LANE.md` byte for byte,
two consecutive launches byte-identical throughout, and the general-purpose
subagent, the security classifier and the memory extractor each
byte-identical to their plain counterparts with zero lane canaries. Live on a
local 27B model with the tiny sample: the tail marker quoted byte-equal, no
file read with a tool, every canary reported, the base-only
`<qwen:user-prompt-submit-context>` rule stated, and its own first sentence
quoted back as «You are Qwen Code, a non-interactive CLI agent developed by
Alibaba Group, specializing in software engineering tasks.» Record:
`PROP-076#SYS-QWEN`, format record `SP-FMT-116`.

## License

This package — its manifest and its documents — is UPL-1.0; see `LICENSE.md`.

Qwen Code itself is **not**: it is distributed under the **Apache License
2.0**, whose text every installation keeps in the payload root (`LICENSE`,
and `lib/LICENSE` for the bundled dependencies), together with the Node.js
runtime bundled beside it under its own MIT-based license
(`node/LICENSE`). `[package].license` in the manifest names the license of
what an installation puts on the machine, `Apache-2.0`.
