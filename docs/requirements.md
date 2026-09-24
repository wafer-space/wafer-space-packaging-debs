# Requirements: apt.wafer.space Debian/Ubuntu packages

This document is the authoritative statement of what this repository must
deliver and how the work on it must be carried out. Design documents,
decision records and plans elsewhere in `docs/` must conform to it. When this
document changes, everything that depends on it is updated to match.

## 1. Goal

Publish an APT repository at **`https://apt.wafer.space`** (hosted on GitHub
Pages, in the style of `https://apt.fpgas.online`) containing proper Debian
packages of every tool that the
[wafer-space/gf180mcu-project-template](https://github.com/wafer-space/gf180mcu-project-template)
Nix environment provides. Each package is built from **exactly the same
upstream revision** as the Nix environment uses.

This lets users of Debian and Ubuntu get the same toolchain as Nix users
through their normal package manager.

## 2. Source of truth: the Nix environment

- The reference environment is the `devShells.default` output of the
  `flake.nix` / `flake.lock` in the `main` branch of
  `wafer-space/gf180mcu-project-template`.
- "Identical revision" means the same upstream source (same git commit or
  same release tarball hash), with the same patches applied, as the Nix
  derivation that ends up in that dev shell. This includes overrides made in
  the template's own `flake.nix` (for example the `magic` version override).
- The set of packages to produce is derived from that environment
  mechanically, not maintained by hand. When the environment gains, loses or
  changes a package, the package set follows.

Snapshot of the environment on 2026-09-24 (`nix-eda` 5.9.0, `librelane`
`3affff86`, `nixpkgs` `b2485d56`), for orientation only:

| Kind | Packages (Nix name-version) |
|---|---|
| EDA tools (librelane `includedTools`) | yosys-with-plugins 0.54, openroad 2025-10-28, opensta, netgen 1.5.305, magic-vlsi 8.3.581, klayout 0.30.4, iverilog s20250103-60-gdb82380ce, verilator 5.038, surelog 1.84-unstable-2024-12-06, tclFull, ruby 3.3.8 |
| Template extras | gtkwave 3.3.121, surfer 0.3.0, gnumake 4.4.1, gnugrep 3.11, gawk 5.3.2, coreutils 9.7 |
| Shell conveniences | git 2.49.0, zsh 5.9, delta 0.18.2, graphviz 12.2.1 |
| Python (3.12.10) | librelane, click 8.1.8, cloup 3.0.7, pyyaml 6.0.2, yamlcore 0.0.2, rich 14.0.0, requests 2.32.3, pcpp 1.30, ciel 2.3.1, tkinter, lxml 5.3.1, deprecated 1.2.18, libparse 0.56.0, psutil 7.0.0, rapidfuzz 3.13.0, semver 3.0.4, cocotb 2.0.0, docopt 0.6.2, pillow 11.2.1 (plus their transitive dependencies) |

The exact scope boundary (which of these, and which transitive dependencies,
become packages here) is defined in §4.

## 3. Target distributions and architectures

### 3.1 Distributions

Packages are built for every one of:

- **Every Ubuntu LTS release in standard support.** Today: 22.04 `jammy`,
  24.04 `noble`, 26.04 `resolute`.
- **The newest Ubuntu release**, whether LTS or interim. Today this is 26.04
  `resolute`. When 26.10 is released it is added automatically.
- **Debian stable.** Today: 13 `trixie`.
- **Debian `trixie`**, by name.
- **Debian `sid` / unstable.**

The list of releases is not hard-coded as a one-off: tooling detects new
releases and releases that leave support, and the release list is kept in
one place.

### 3.2 Architectures

- Built now, on GitHub-hosted runners: **`amd64`** and **`arm64`**.
- Planned for later, on self-hosted runners: **RISC-V 64-bit, RISC-V 32-bit,
  and 32-bit ARM (`armhf`, for Raspberry Pi)**. The design must make adding
  them a configuration change. **No self-hosted runners are used now.**

## 4. What gets packaged

### 4.1 Tools

- A Debian package exists for every tool in scope from the Nix environment
  (§2), at the Nix revision.
- **Reuse existing Debian packaging.** Where Debian (or Ubuntu) already
  packages a tool, start from the Debian maintainers' `debian/` directory
  and adapt it to the Nix revision and configuration. Only write packaging
  from scratch when none exists.
- **Never contact upstream Debian maintainers.** No bug reports, pull or
  merge requests, emails or any other communication with them or with
  Debian/Ubuntu infrastructure on their behalf.

### 4.2 PDKs

- PDK data is **not** packaged directly.
- Instead there is one small **"dummy" package per PDK revision that ciel
  exposes**. Each depends on the ciel package and, when installed, uses ciel
  to download that PDK revision into a **system-wide location**.
- The set of PDK dummy packages is generated from ciel's list of revisions
  and follows it automatically.

## 5. Fidelity to Nix

- **Configuration must be identical.** Every package is built with the same
  configure/build options, enabled features, plugins and patches as the Nix
  derivation. A difference caused by a different configuration choice is a
  bug and must never be shipped.
- **Bit-identical binaries are the ideal.** They will probably not be
  achievable (different compilers, libc, library versions, paths). For every
  package:
  - the reasons its binaries differ from Nix's are **written down**, and
  - an automated check compares the Debian build against the Nix build
    (configuration, version reporting, feature flags, and binary contents
    where meaningful) and fails on any unexplained difference.

## 6. Build requirements

- **Everything builds both locally and on GitHub Actions**, through the same
  entry point, so that a local build is a faithful reproduction of CI.
- Builds run **inside containers** (a clean container per distribution and
  architecture) both locally and in CI.
- **Reproducible builds.** Every package must pass the Debian reproducible
  builds checks (building twice under varied conditions, e.g. with
  `reprotest`, and getting identical `.deb`s; `SOURCE_DATE_EPOCH` honoured;
  no build paths, timestamps or host details leak into packages).
- **Rebuild only on real change.** A package is rebuilt only when something
  it depends on changed. Otherwise the previously built `.deb`s are reused.
  The inputs that decide this must be complete and explicit:
  - the upstream source (revision / tarball hash),
  - the packaging (`debian/` directory, patches, build scripts),
  - build dependencies (other packages built here, and the distribution's
    build environment / container image),
  - the build tooling in this repository itself.
- **Periodic full rebuild.** On a schedule, everything is rebuilt from
  scratch and compared against the reused artifacts, to catch hidden state
  that the dependency tracking missed. Weekly if the total cost is modest,
  monthly if it is resource-heavy; the chosen frequency and its reasoning
  are documented.
- **Build acceleration.** The following machines may be used to speed up
  builds: `big-storage.welland.mithis.com` (storage) and the Docker VMs on
  `harken.mithis.com` and `micky.mithis.com`.
  - Access is obtained **only** by asking the `hetzner-ansible` Claude
    session to create a dedicated user and a dedicated SSH key for this
    work on those machines.
  - Never use an existing SSH key, and never log in to these machines,
    until that session has provided the access and given permission.
  - GitHub Actions must still be able to build everything without them.

## 7. Versioning

- Package versions are derived from **`git describe` commit counters**, per
  the repository conventions (`v0.0` tag on the first commit).
- A different scheme is only used where `git describe` counters are clearly
  unsuitable, and only after explicit confirmation from the project owner.

## 8. Publishing

- **Production:** `https://apt.wafer.space`, a signed APT repository served
  by GitHub Pages, updated from `main`.
- **Pull-request previews:** every pull request publishes a complete,
  installable preview repository (all packages, not just the changed ones)
  at `preview.apt.wafer.space`, following the same pattern as
  `preview.wafer.space` does for web pages (one directory per PR, an index
  of active previews, cleaned up when the PR closes).
- **Hosting repositories:** the build logic lives in
  `wafer-space/wafer-space-packaging-debs`. Separate repositories are used to
  host `apt.wafer.space` and/or `preview.apt.wafer.space` if package sizes or
  GitHub Pages limits require it.
- The repository landing page explains how to add the repository (key,
  `sources.list` entry) for each supported distribution.

## 9. Automation and monitoring

- **Automatic updates.** Tooling detects when the Nix environment changes
  (new `flake.lock` in the template) and produces a pull request that moves
  the affected packages to the new revisions.
- **Daily verification.** A daily CI job checks that the revisions published
  in the APT repository match the revisions the Nix environment currently
  provides, and fails (visibly) on any mismatch.
- **Periodic full rebuild** as described in §6.

## 10. Documentation

- `README.md` carries a prominent **warning that the packages are not
  currently usable and are being generated with AI**.
- This setup will later be reused as a template for producing **conda** and
  **PyPI** packages of the same toolchain. Therefore:
  - how everything works is documented, and
  - every significant decision is recorded with its reasoning and the
    alternatives considered (decision records in `docs/decisions/`).

## 11. Repository setup

- All repositories created for this work are configured following
  `~/.claude/GitHub.md` (the `github-setup` skill): `v0.0` tag on the first
  commit, wiki/projects/discussions disabled, merge commits only, delete
  branches on merge, secret scanning and push protection, update-branch
  suggestions, default branch protection, tag format ruleset.
- License: Apache 2.0 for this repository's own content.

## 12. Way of working

- **Plan first.** Keep a list of everything that needs to be done (tools to
  package, infrastructure to build) and work through it **one tool at a
  time**.
- **Small, logical commits and small, logical pull requests.**
- **Sub-agents.** Sub-agents may be used. **At most two run at any one time.**
  Each works in its own git worktree and branch.
- **Independent review.** All work is reviewed by an independent agent
  (not the author) before it is opened as a pull request.
- **Merging.** A pull request is merged on GitHub with a merge commit, and
  only after its CI tests pass.
- **Restartability.** Process logs, the todo list, and status notes are
  committed to this repository as work progresses, so that someone with no
  prior context can pick up exactly where the work stopped.
- **Autonomy.** After the clarifying questions in this document have been
  answered, work proceeds independently without further input from the
  project owner, except where this document explicitly requires
  confirmation (§7).

## 13. Open questions

To be resolved with the project owner before implementation starts; this
section is removed once they are answered and the answers are written into
the sections above.

1. **Scope boundary (§4.1).** Which parts of the environment become
   packages: only the EDA tools and Python libraries, or also the general
   utilities (coreutils, make, grep, gawk, git, zsh, delta, graphviz, Tcl,
   Ruby, the Python interpreter itself) that the distributions already ship?
2. **Coexistence with distribution packages.** Should packages use the
   distribution's package names and replace the distribution versions (with
   an APT pin so ours win), or be namespaced and installed side by side
   (e.g. under `/opt/wafer-space`)?
3. **"Debian stable" vs `trixie`.** Debian stable *is* `trixie` today.
   Should `bookworm` (oldstable) also be built, and does "stable" mean
   "follow the `stable` alias automatically when `forky` releases"?
4. **PDK dummy packages.** Ciel currently exposes 83 gf180mcu, 116 sky130 and
   24 IHP revisions. Are dummy packages wanted for all families and all
   revisions? The template itself does not use ciel for its PDK; it clones
   `wafer-space/gf180mcu` at tag `1.4.4`. Should that PDK get a dummy
   package too?
5. **Hosting limits.** GitHub Pages sites are limited to about 1 GB. If the
   repository exceeds that, what is the preferred fallback?
6. **Tag format** for the new repositories: `vXX.ZZZ` or `vXX.YY.ZZZ`?
