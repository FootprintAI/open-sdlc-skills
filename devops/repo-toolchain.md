---
name: "DevOps: Repo Toolchain"
description: Work out which toolchains and tools a repository needs from the repository itself (version files, manifests, CI), install exactly those pinned versions in user space on the box or runner the agent works on, prove them with a real build and test, and record a manifest. Use it when an agent starts on a repo whose build tools are missing or have drifted from what the repo pins.
category: DevOps
tags: [devops, toolchain, environment, bootstrap, agent-runtime, verification, supply-chain]
---

> **Conventions notice:** every issue you open carries one type label
> (`bug`, `feature-request`, `epic`, `ci`, `docs`, `security`, `chore`) —
> see `_shared/labels.md`.

Act as the **DevOps role** preparing the *agent's own build environment*
for one repository. The role owns **reproducibility**: the tools the
repo's build, test and lint need are present at the versions the repo
pins, installed without privileges, and proven by running them. It does
not own the repo's code, the deploy (`/devops-deploy`), the agent's
credentials or login, or the runtime's wider capabilities
(`/devops-containarium-env`).

**Why this exists.** An agent put on a fresh machine cannot build the
project: the compiler, the package manager and the linters are not there,
or are there at a different version than CI uses. Someone installs them by
hand once, nobody records how, and the next machine starts from zero. The
repo already says what it needs; this skill reads that, installs it
safely, and leaves evidence.

**Input**: `/devops-repo-toolchain` (default mode `check`). Modes:

- `check` — derive the requirements, compare them with what is installed,
  report the gap. Changes nothing
- `setup` — install what `check` found missing or different, then prove it
- `sync` — re-run `check`, and fix drift **only if the owner of this
  machine has said, in this session, that automatic changes are allowed**;
  otherwise report the drift and stop

Options: `--repo <path>` (the checkout to read; default the current
directory), `--issue N` (where the manifest is posted as a comment;
default none), `--prefix <dir>` (default `~/.local`).

**Principles**

- **The repo is the source of truth.** Versions come from files the repo
  already commits: a language version directive, a lockfile or engines
  field, a version file, a CI workflow's setup step, a pinned
  `go install pkg@version`. Never from memory and never from "latest"
- **Pinned means the repo's pin, not the newest.** A newer upstream
  release is not drift. A repo that changed its pin is. Where the repo
  pins nothing, say so, propose a version with the reason, and wait
- **User space only.** Install under the prefix (`~/.local`), put binaries
  on the PATH through `<prefix>/bin`, and do not use `sudo` or the system
  package manager. If something truly needs more, stop and hand the exact
  command to the human
- **Verify what you download.** Fetch from the project's official release
  location, check the published checksum before unpacking, and fail
  closed on a mismatch. Never pipe a download into a shell
- **Prove, don't assume.** A tool is present when `--version` printed in
  this session, and the toolchain works when the repo's own build and
  test commands ran. "Installed" without output is a claim
- **Leave a record.** Every install writes a manifest: tool, version,
  source URL, checksum, what pinned it. The next run reads it instead of
  guessing, and a human can audit it
- **Ask before widening.** Anything outside the prefix, any tool the repo
  does not pin, any change while the owner has not consented: ask first

**Steps**

1. **Resolve the target**

   The checkout (`--repo`, else the current directory) must be a git
   working tree; read `git rev-parse --show-toplevel`. Detect the
   platform with `uname -s` and `uname -m`; map it to each project's
   download naming (for example `linux` and `x86_64` → `linux-amd64`).
   Stop with a clear message on a platform you cannot map.

2. **Derive the requirements**

   Read, in this order, and note `file:line` for every pin you use:

   - version files: `.tool-versions`, `.nvmrc`, `.node-version`,
     `.python-version`, `rust-toolchain(.toml)`, `.go-version`
   - manifests: `go.mod` (`go`, `toolchain`), `package.json` (`engines`,
     `packageManager`), `pyproject.toml` (`requires-python`),
     `Cargo.toml` (`rust-version`)
   - the build entry points: `Makefile` targets and the README's build
     section, for the commands you will run in step 5
   - CI: workflow setup steps (`setup-go`, `setup-node`, `setup-python`
     inputs), pinned installs such as `go install <pkg>@<version>` or
     `npx <pkg>@<version>`, container images in a `Dockerfile`

   Produce a requirements table: tool, required version, source
   `file:line`. List separately anything the repo needs but does not pin.

3. **Compare with what is installed**

   Read the manifest if one exists (`<prefix>/share/toolchain/manifest.md`
   or `~/toolchain-manifest.md`), then check each tool with
   `command -v` and `--version`. Classify each as `ok`, `missing`,
   `different` (installed version ≠ pinned) or `unpinned`. In `check`
   mode, report and stop here.

   In a non-interactive session some of these probes may be refused. Do
   not guess: list the directories you can read (`<prefix>/bin`, the
   manifest), and mark every tool you could not probe as `unknown`, with
   the reason. "Could not check" is a result; "probably missing" is not.

4. **Install (modes `setup` and `sync`)**

   For each `missing` or `different` tool, one at a time:

   ```bash
   # example shape — substitute the project's own release location
   tmp="<prefix>/share/toolchain/dl/<tool>-<version>"   # inside the prefix
   mkdir -p "$tmp"
   curl -fsSLo "$tmp/pkg.tar.gz" "<official release URL for the pinned version>"
   curl -fsSLo "$tmp/pkg.sha256" "<official checksum URL or checksum file>"
   # compare the digest to the published one; abort on any mismatch
   sha256sum "$tmp/pkg.tar.gz"
   tar -C "<prefix>/<tool>-<version>" -xzf "$tmp/pkg.tar.gz"
   ln -sfn "<prefix>/<tool>-<version>/bin/<tool>" "<prefix>/bin/<tool>"
   ```

   Keep each version in its own directory and move the symlink, so a
   failed install leaves the old version usable. Tools installed through
   the language's own tooling (`go install`, `npm install --prefix`) are
   installed with the toolchain you just proved, at the pinned version.
   In `sync` mode without the owner's stated consent, do not install:
   report each `different` tool and the exact change it would take.

   Keep every download and every language cache inside the prefix, so
   "nothing outside the prefix" is checkable: download into
   `<prefix>/share/toolchain/dl/` (keep the archives and the published
   checksum files there as the audit trail), and point the toolchains'
   own caches there too, for example `GOPATH`, `GOCACHE` and the package
   manager's cache directory, as environment variables on each command.
   Tell the human which variables and which `PATH` entry their own shell
   needs to see the same tools.

5. **Prove it**

   Run `--version` for every installed tool and record the output. Then
   run the repo's own build and test commands as found in step 2, in the
   checkout. Report pass or fail with the output tail. A failing test is
   a finding about the repo, not something to fix here: do not edit the
   repo's code or its pins. Report, without acting on it, anything the
   package manager flags (for example an audit warning) and anything the
   build leaves in the checkout; confirm it is ignored by the repo's
   `.gitignore` with `git status`.

6. **Write the manifest — the artifact**

   Write `<prefix>/share/toolchain/manifest.md` and, with the file-writing
   tool (not a copy command, which a restricted session may refuse), the
   same content to `~/toolchain-manifest.md`. With `--issue`, post the
   same content as one comment.

   ```markdown
   ## Toolchain manifest — <host-or-runner> — <date>

   Repo: <owner/name or path> at <short sha>; platform: linux/amd64

   | Tool | Version | Pinned by | Source | Checksum | State |
   |------|---------|-----------|--------|----------|-------|
   | go | 1.xx.y | go.mod:3 | <official URL> | sha256:… (verified) | ✅ |
   | node | 2x.y.z | .nvmrc:1 | <official URL> | sha256:… (verified) | ✅ |
   | <tool> | – | not pinned | – | – | ⚠️ proposed <v>: <reason> |

   Proofs: `go version` → …; `node --version` → …
   Build/test: `<command>` → pass | fail (<tail of output>)
   Not done: <anything skipped, and why>
   ```

**Guardrails**

- Never use `sudo`, a system package manager, or any write outside the
  prefix without the human's explicit say-so in this session
- Never pipe a download into a shell, and never install an unverified
  download
- Never install "latest" for something the repo pins, and never change
  the repo's pins to make a build pass
- Never edit the repo's code or run anything that publishes, deploys or
  pushes
- Never read, print or store credentials or tokens. If a private
  dependency needs one, report where the repo expects it and hand the
  step to the human
- Never report a tool or a build as working without the command output
  from this session; say what you could not run
- Unattended consent is the platform's decision, not this skill's. In
  `sync` mode, without a stated consent, report drift and change nothing
