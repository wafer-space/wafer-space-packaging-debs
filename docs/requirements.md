# Requirements: apt.wafer.space Debian/Ubuntu packages

This document is the authoritative statement of what this repository must
deliver and how the work on it must be carried out. Design documents,
decision records and plans elsewhere in `docs/` must conform to it.

It always states only the **current** requirements. It does not record how
or when a requirement changed; git history does that. When a requirement
changes, this document is edited to the new final state and everything that
depends on it is updated to match.

## 1. Goal

Publish a signed APT repository at **`https://apt.wafer.space`**, hosted on
GitHub Pages in the style of `https://apt.fpgas.online`. It contains proper
Debian packages (§4.1) of everything that the
[wafer-space/gf180mcu-project-template](https://github.com/wafer-space/gf180mcu-project-template)
Nix environment uses or installs, within the scope defined in §4. Each
package is built from **exactly the same upstream revision** as the Nix
environment uses.

This lets Debian and Ubuntu users get the same toolchain as Nix users
through their normal package manager.

## 2. Source of truth: the Nix environment

- The reference environment is the `devShells.default` output of the
  `flake.nix` / `flake.lock` on the `main` branch of
  `wafer-space/gf180mcu-project-template`.
- "Identical revision" means the same upstream source (the same git commit,
  or a release tarball with the same hash), with the same patches applied,
  as the Nix derivation in that dev shell. Overrides made in the template's
  own `flake.nix` count too, for example the `magic` version override.
- The set of packages to produce, and their revisions, is derived
  mechanically by **evaluating** the Nix environment. It is not maintained by
  hand. When the evaluated environment gains, loses or changes a package
  (name, version, source hash, patches or build options), the package set
  follows.

Snapshot of the environment on 2026-09-24 (`nix-eda` 5.9.0, `librelane`
`3affff86`, `nixpkgs` `b2485d56`). It is here for orientation only:

| Kind | Packages (Nix name-version) |
|---|---|
| EDA tools (librelane `includedTools`) | yosys-with-plugins 0.54, openroad 2025-10-28, opensta, netgen 1.5.305, magic-vlsi 8.3.581, klayout 0.30.4, iverilog s20250103-60-gdb82380ce, verilator 5.038, surelog 1.84-unstable-2024-12-06, tclFull, ruby 3.3.8 |
| Template extras | gtkwave 3.3.121, surfer 0.3.0, gnumake 4.4.1, gnugrep 3.11, gawk 5.3.2, coreutils 9.7 |
| Shell conveniences | git 2.49.0, zsh 5.9, delta 0.18.2, graphviz 12.2.1 |
| Python (3.12.10) | librelane, click 8.1.8, cloup 3.0.7, pyyaml 6.0.2, yamlcore 0.0.2, rich 14.0.0, requests 2.32.3, pcpp 1.30, ciel 2.3.1, tkinter, lxml 5.3.1, deprecated 1.2.18, libparse 0.56.0, psutil 7.0.0, rapidfuzz 3.13.0, semver 3.0.4, cocotb 2.0.0, docopt 0.6.2, pillow 11.2.1, plus their transitive Python dependencies |

## 3. Target distributions and architectures

### 3.1 Distributions

Packages are built for all of the following:

- **Every Ubuntu LTS release in standard support.** Today: 22.04 `jammy`,
  24.04 `noble` and 26.04 `resolute`. Releases that are only covered by
  Extended Security Maintenance (ESM / Ubuntu Pro) are excluded.
- **The newest Ubuntu release**, whether LTS or interim. Today this is 26.04
  `resolute`. 26.10 `stonking` (due 2026-10-15) must be picked up
  automatically. Only the newest interim release gets new builds. When an
  interim release is superseded, it stops receiving new builds, and its
  published packages remain until it reaches end of life.
- **Debian stable**: today 13 `trixie`.
- **Debian testing**: today `forky`.
- **Debian unstable**: `sid`.

The Debian suites follow the `stable` / `testing` / `unstable` roles. When
`forky` becomes stable, the set moves with it automatically. The list of
releases is kept in one place. Tooling detects new releases and releases
that leave support, and updates that list through a pull request.

### 3.2 Architectures

- **Now**, on GitHub-hosted runners: **`amd64`** and **`arm64`**.
- **Later**, on self-hosted runners: RISC-V 64-bit, RISC-V 32-bit, and
  32-bit ARM for Raspberry Pi. The exact Debian architecture (`armhf` or
  `armel`) is decided when that target is added. No Debian or Ubuntu release
  has a RISC-V 32-bit port today, so that target is re-evaluated when it is
  scheduled.
- The design must make adding an architecture a configuration change.
- **No self-hosted runners are used now.**

## 4. What gets packaged

### 4.1 Tools

**Scope.** Every package that is directly part of the dev shell is packaged
at its Nix revision. This covers the EDA tools, the general utilities, the
Python interpreter and all the Python packages, including transitive Python
dependencies.

Lower-level C/C++ libraries (for example Qt, Boost, Tcl libraries, zlib,
glibc) come from the distribution. The exception is a library that a tool
needs at a specific version or configuration the distribution cannot
provide, for example or-tools for OpenROAD. Such a library is packaged here
at its Nix revision too. Every difference caused by using a distribution
library instead of the Nix one is documented (§5).

**Coexistence with distribution packages.** Packages are **namespaced**:

- package names carry a `ws-` prefix (for example `ws-yosys`);
- everything installs under `/opt/wafer-space`;
- an activation script and/or wrappers put the tools on `PATH`;
- the private Python interpreter at the Nix version carries the Python
  packages, so they never clash with the system `python3`.

They install side by side with the distribution's own packages of the same
tools and never conflict with them. Replacing the distribution's packages
(same names, installing into `/usr`) is a planned later step, taken once
the packages have proven to work correctly. The naming and layout must
allow that migration later, for example through transitional packages.

**Proper Debian packages.** Every package is:

- built from a Debian source package (`debian/` directory, `.dsc`) with
  `dpkg-buildpackage`;
- given correct `Depends` / `Provides` / `Conflicts` metadata;
- lintian-clean, apart from overrides that are documented;
- published with its source package alongside the binaries.

**Reuse existing packaging.** Where Debian or Ubuntu already packages a
tool, start from their `debian/` directory and adapt it to the Nix revision
and configuration. Where Debian's packaging choices differ from Nix's
configuration (for example unvendored libraries, or features enabled or
disabled), **Nix's configuration wins**. Packaging is written from scratch
only where none exists.

### 4.2 PDKs

- PDK data itself is never packaged. Only small "dummy" installer packages
  are.
- There is **one dummy package per PDK revision that ciel
  (`fossi-foundation/ciel`) exposes**, across all families ciel supports.
  Today that is 83 `gf180mcu`, 116 `sky130` and 24 `ihp-sg13g2` revisions.
- There is also one dummy package per tag of the **wafer-space PDK**
  (`wafer-space/gf180mcu`, which the template clones at tag `1.4.4`). Ciel
  does not expose this PDK.
- Each dummy package depends on the packaged ciel, or on git for the
  wafer-space PDK. On install it downloads that PDK revision into a
  **system-wide location**. The location, the environment it sets (for
  example `PDK_ROOT`) and what happens on removal are design decisions that
  get recorded.
- The set of PDK dummy packages is generated from ciel's list of revisions
  and the wafer-space PDK's tags, and follows them automatically.

## 5. Fidelity to Nix

- **Configuration must be identical.** Every package is built with the same
  configure and build options, enabled features, plugins and patches as the
  Nix derivation. A difference caused by a different configuration choice
  is a bug and is never shipped.
- **Bit-identical binaries are the ideal.** They will probably not be
  achievable, because of different compilers, libc, library versions and
  paths. For every package:
  - the reasons its binaries differ from Nix's are **written down**; and
  - an automated check compares the Debian build against the Nix build and
    fails on any difference that is not explained. The comparison covers
    configuration, reported versions, feature flags, and binary contents
    where that is meaningful.

## 6. Build requirements

- **Everything builds both locally and on GitHub Actions**, through the same
  entry point, so a local build faithfully reproduces CI.
- Builds run **inside containers**, with a clean container per distribution
  and architecture, both locally and in CI. The owner permits and prefers
  this, and the reasoning is recorded as a decision.
- **Reproducible builds.**
  - Every package must pass the Debian reproducible-builds checks. It is
    built twice under varied conditions (for example with `reprotest`) and
    both builds must produce identical `.deb`s. Differences are analysed
    with `diffoscope`.
  - `SOURCE_DATE_EPOCH` is honoured, and no build paths, timestamps or host
    details leak into packages.
  - The check runs in CI for every package that is (re)built. A failure
    blocks both merge and publish.
- **Rebuild only on real change.** A package is rebuilt only when one of its
  inputs changed. Otherwise the previously built `.deb`s are reused. The
  inputs are complete and explicit:
  - the upstream source (revision or tarball hash);
  - the packaging (`debian/` directory, patches, build scripts);
  - build dependencies: other packages built here, and the distribution
    build environment. Container base images are pinned by digest, so
    image refreshes are deliberate input changes;
  - the build tooling in this repository.
- **Periodic full rebuild.** On a schedule, everything is rebuilt from
  scratch and compared against the reused artifacts. This catches hidden
  state that the dependency tracking missed. The schedule is weekly if the
  total cost is modest and monthly if it is resource-heavy. The chosen
  frequency and its reasoning are documented.
- **Build acceleration.** These machines may be used to speed up builds:
  `big-storage.welland.mithis.com`, and the Docker VMs on
  `harken.mithis.com` and `micky.mithis.com`.
  - Access is requested from the `ansible-main` Claude session (the session
    that manages these hosts with Ansible). It is asked to create a
    **dedicated user with a dedicated SSH key** for this work on those
    machines.
  - Do not use any existing SSH key or existing user account, and do not
    log in to these machines at all, unless that session has given
    permission.
  - GitHub Actions must still be able to build everything without these
    machines.

## 7. Versioning

- Versions are based on this repository's **`git describe` commit
  counter**. The repositories carry a `v0.0` tag on their first commit, and
  release tags use the `vXX.ZZZ` format.
- A package's Debian version is
  `<upstream version>-<N>~<suite>`. For example, the Nix yosys `0.54`,
  built for `noble`, whose inputs last changed at describe counter `0.123`,
  becomes something like `0.54-0.123~noble`. Where:
  - `<upstream version>` is the version of the Nix derivation, normalised
    only as far as Debian version syntax requires;
  - `<N>` is the `git describe` counter of **the commit that last changed
    any of that package's inputs** (§6). A package whose inputs did not
    change keeps its version and its previous build;
  - `~<suite>` keeps the same upstream version distinct across suites.
- The exact normalisation rules are a design decision to record. A
  versioning scheme that is not based on `git describe` counters is only
  used when there is a very strong reason that these counters do not make
  sense or would be a bad idea. The project owner must confirm the
  alternative before it is used.

## 8. Publishing and hosting

- **Hosting organisation.** The published APT repositories live in the
  GitHub organisation **`wafer-space-packages`**. Its user site
  (`wafer-space-packages.github.io`) carries the custom domain
  `apt.wafer.space` and serves the landing page. Each suite is its own
  project repository with Pages enabled, served at
  `https://apt.wafer.space/<suite>/`. This keeps each site within GitHub
  Pages' limits: a published site may be at most 1 GB, a deployment times
  out after 10 minutes, and there is a soft bandwidth limit of 100 GB per
  month.
- **Retention.** Production keeps the latest version of each package per
  suite.
- **Production** is updated from `main` of this repository.
- **Pull-request previews.** Every pull request publishes a **complete,
  installable** preview repository, containing all packages and not just
  the changed ones, at `preview.apt.wafer.space`. It follows the pattern
  `preview.wafer.space` uses for web pages: one location per PR, an index
  of active previews, and cleanup when the PR closes. How previews stay
  within the Pages limits is a design decision to record.
- **Build logic** lives in `wafer-space/wafer-space-packaging-debs`.
- The landing page explains how to add the repository (signing key and
  `sources.list` / deb822 entry) for each supported distribution.
- **Signing.** A dedicated GPG key signs the repository. Its private part
  exists only as a GitHub Actions secret. The public key is committed and
  published.

## 9. Automation and monitoring

- **Automatic updates.** Tooling detects when the evaluated Nix environment
  of the template changes (§2). It then opens a pull request that moves the
  affected packages to the new revisions and configuration.
- **Daily verification.** A daily CI job checks that the revisions published
  in the APT repository match what the Nix environment currently provides,
  and fails visibly on any mismatch.
- **Release tracking.** Tooling tracks new and end-of-life distribution
  releases (§3.1).
- **Periodic full rebuild**, as described in §6.

## 10. Documentation

- `README.md` carries a prominent **warning that the packages are not
  currently usable and are being generated with AI**.
- This setup will later be reused as a template for producing **conda** and
  **PyPI** packages of the same toolchain. Therefore:
  - how everything works is documented; and
  - every significant decision is recorded with its reasoning and the
    alternatives considered, as decision records in `docs/decisions/`.
- Process state lives in `docs/status/`: the todo list, the per-tool
  progress table and the process log (§12).

## 11. Repository setup

- Every repository created for this work is configured following **all**
  the settings in `~/.claude/GitHub.md` (the `github-setup` skill). The
  file is authoritative. Summary:
  - a `v0.0` tag on the first commit;
  - wiki, projects and discussions disabled;
  - merge commits only;
  - branches deleted on merge;
  - secret scanning and push protection;
  - update-branch suggestions;
  - default-branch protection;
  - a tag ruleset using the `vXX.ZZZ` format;
  - Git LFS objects included in archives. This one is UI-only and is left
    as a tracked TODO for the owner.
- This repository's own content is licensed Apache 2.0.

## 12. Way of working

- **Plan first.** Keep a list of everything that needs to be done: the
  tools to package and the infrastructure to build. Work through it **one
  tool at a time**.
- **Small, logical commits and small, logical pull requests.**
- **Sub-agents.** Sub-agents may be used, with **at most two running at any
  one time**. Each works in its own git worktree and branch.
- **Independent review.** At least one independent agent, which is not the
  author, reviews all work before it is opened as a pull request.
- **Merging.** Pull requests are merged on GitHub with a merge commit, and
  only after their CI tests pass.
- **Restartability.** Process logs, the todo list and status notes are
  committed to this repository (`docs/status/`) as work progresses. Someone
  with no prior context can then pick up exactly where the work stopped.
- **Autonomy.** Work proceeds independently, without further input from the
  project owner. The only exception is where this document explicitly
  requires confirmation (§7).
- **Boundaries: do not disturb anyone.**
  - Actions affect only `wafer-space/wafer-space-packaging-debs` and
    repositories in the `wafer-space-packages` organisation.
  - Nothing is ever sent to upstream projects, to Debian or Ubuntu
    maintainers, or to anyone else. That means no bug reports, pull or
    merge requests, emails or other contact, by any channel (GitHub, Debian
    BTS, Salsa, Launchpad, mailing lists).
  - The only permitted outside contact is with the owner's own Claude
    sessions:
    - `ansible-main`, for machine access (§6);
    - the `ns1` session, for DNS records: `apt.wafer.space` and the preview
      hostname.
- **Authorised actions.** Within those boundaries, the agent may:
  - create public repositories;
  - configure Pages, secrets and deploy keys;
  - generate the APT signing key;
  - merge its own reviewed pull requests once CI passes.
