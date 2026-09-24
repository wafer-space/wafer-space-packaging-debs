# Independent review: `docs/requirements.md` vs. the owner's original instructions

- Date: 2026-09-24
- Reviewed: `docs/requirements.md` at commit `4bbae36`
- Method: I went through the original instructions one sentence at a time and
  traced each one to a section of the document. I then read the document
  looking for claims the original does not support, and checked the facts
  that were cheap to verify against primary sources.
- The findings are ordered by importance within each section.
  **[Q]** marks a finding that should become a clarifying question for the
  owner.

## Summary

The document is faithful overall. Every instruction in the original can be
traced to a section. The main problems are:

1. **Versioning (§7) is ambiguous in a way that matters.** It is unclear
   whether `git describe` counters refer to this packaging repository or to
   each packaged upstream tool. There is no open question about it. [Q]
2. **"Debian trixie" is probably a slip for "Debian testing".** Open
   question 3 asks about `bookworm` instead, so it misses the most likely
   intent. [Q]
3. **"Currently supported Ubuntu LTS" has been silently narrowed** to
   "standard support". Ubuntu's own metadata marks focal, bionic, xenial and
   trusty as supported (under ESM). [Q]
4. **Automatic updates (§9) only watch `flake.lock`.** Version changes made
   in `flake.nix` itself, such as the existing `magic` override, would be
   missed.
5. **Hosting.** Complete per-PR previews multiplied by the 1 GB Pages limit
   is a real conflict. Also, two custom domains need two Pages sites. Open
   question 5 is too vague. [Q]
6. **Practical blockers are missing from the questions.** Nothing covers
   DNS, the signing key, permission to create repositories in the org, or how
   to reach the `hetzner-ansible` session. After the questions the work is
   supposed to be autonomous, so these need answers up front. [Q]

---

## 1. Omissions, weakenings and changes of meaning

### 1.1 Automatic updates only trigger on `flake.lock` changes

- **Original:** "There should be tooling which automates the updating of the
  apt packages as the nix versions also change."
- **Doc §9:** "Tooling detects when the Nix environment changes (new
  `flake.lock` in the template)…"
- **Problem:** Nix versions also change through edits to `flake.nix` that
  leave `flake.lock` alone. The template already does this: the overlay sets
  `magic` to `version = "8.3.581"` in `flake.nix`. Its librelane input also
  follows a branch (`librelane/librelane/leo/gf180mcu`), so the result
  depends on the locked rev. The trigger must be the *evaluated* environment,
  not one file.
- **Fix:** "Tooling detects when the evaluated Nix environment changes (any
  change to the template's `flake.nix` or `flake.lock` that alters a package
  name, version, source hash, patches or build options) and produces a pull
  request…"

### 1.2 "Everything" has become "every tool"

- **Original:** "proper debian packages for everything that is used /
  installed by the current .nix environment"
- **Doc §1:** "every tool that the … Nix environment provides". Doc §2 says
  "provides". Doc §4.1 says "every tool in scope".
- **Problem:** The original's default is *everything*, and it says "used /
  installed", not "provides". Open question 1 is reasonable, but the
  document reads as though a narrower scope is already the baseline.
  "Used" can also cover runtime dependencies that are not in the shell's
  `PATH`, such as the yosys plugins, ABC and OpenROAD's libraries.
- **Fix:** In §1, say "every package that the Nix environment uses or
  installs". In §4.1 / open question 1, say explicitly that the default
  reading is "everything, including transitive runtime dependencies", and
  that the question is whether to narrow it.

### 1.3 Containers: "allowed / probably should" has become "must"

- **Original:** "The build are allow to (and probably should) happen in
  containers when running both locally and on github actions."
- **Doc §6:** "Builds run **inside containers** … both locally and in CI."
- **Assessment:** This is a strengthening, not a weakening. It is almost
  certainly what the owner wants, but it has become a hard requirement.
- **Fix:** Either keep it and record the choice as a decision in
  `docs/decisions/`, or write "Builds run inside containers (permitted and
  preferred by the owner)…".

### 1.4 The rule about the owner's approval of versioning exceptions is softened

- **Original:** "unless there is a very strong reason why that does not make
  sense or wouldn't be a good idea -- in that case confirm with me that a
  different approach should be used before using that."
- **Doc §7:** "only used where `git describe` counters are clearly
  unsuitable, and only after explicit confirmation from the project owner."
- **Problem:** The meaning is close, but "very strong reason" sets a higher
  bar than "clearly unsuitable". It also covers "wouldn't be a good idea",
  not only "unsuitable".
- **Fix:** "A different scheme is only used when there is a very strong
  reason that `git describe` counters do not make sense or would be a bad
  idea, and only after the project owner has confirmed the alternative
  before it is used."

### 1.5 SSH key / login rule

- **Original:** "Do not use an existing ssh key or login to these machines
  unless given permission by that claude session."
- **Doc §6:** "Never use an existing SSH key, and never log in to these
  machines, until that session has provided the access and given
  permission."
- **Problem:** "Or login" can also be read as "an existing login", meaning
  an existing user account. The document forbids existing keys but does not
  forbid existing accounts. Its "never use an existing SSH key" is also
  absolute, while in the original "unless given permission" covers both
  items.
- **Fix:** "Do not use any existing SSH key or existing user account, and do
  not log in to these machines at all, unless the `hetzner-ansible` session
  has given permission. The expected route is a dedicated user with a
  dedicated key created by that session."

### 1.6 "Checked" is interpreted, but the reproducibility check is not placed in CI

- **Original:** "The packages should pass all reproduciable build checks."
- **Doc §6:** This is described, but the document never says *where* the
  check runs or that failing it blocks a merge or publish.
- **Fix:** Add "The reproducibility check runs in CI for every changed
  package. A failure blocks merge and publish."

### 1.7 The "final state only" editing rule is not recorded

- **Original:** "Do *not* write things like "previously tim said A and is now
  saying B" -- write just the correct final state."
- **Doc:** §13 says answers are "written into the sections above". The rule
  itself is not recorded, so later editors of this document will not know
  it.
- **Fix:** Add to the preamble: "The document always states only the current
  requirements. It does not record how or when a requirement changed; git
  history does that."

### 1.8 "Proper" Debian packages is not defined

- **Original:** "proper debian packages"
- **Doc:** The word is repeated in §1 and never defined.
- **Fix:** Define it in §4.1, for example: "Built from a Debian source
  package (`debian/` directory, `.dsc`); sensible `Depends`/`Provides`;
  lintian-clean apart from documented overrides; source packages published
  alongside binaries." Whether to publish source packages could be a
  decision record rather than a question.

### 1.9 Minor tracing notes (no change of meaning, listed for completeness)

- "Put together a plan…" is covered by §12 "Plan first". Fine.
- "(IE packaging done by upstream debian maintainers)" is covered by §4.1.
- "verifying that tests pass before merging on github using a merge
  request" is covered by §12 "Merging". "Merge request" is read as a
  GitHub merge commit, which matches `GitHub.md` (merge commits only).
- "Rewrite all my instructions … have another sub-agent review it" is a
  one-off process step. It is satisfied by this review, and the document
  does not need to state it.

---

## 2. Claims not supported by the original

### 2.1 Reasonable clarifying interpretations (keep them, but consider recording them as decisions)

| Doc § | Addition | Comment |
|---|---|---|
| §2 | The reference is `devShells.default` on the template's `main` branch | The original only says "the current .nix environment". This is a reasonable reading. |
| §2 | "same patches applied" | Follows from "identical configuration settings". Fine. |
| §2 | Package set is derived mechanically | Follows from "tooling which automates". Fine. |
| §3.1 | Tooling detects new and end-of-life releases; the list lives in one place | Follows from "currently supported" and "latest". Fine. |
| §3.2 | "The design must make adding them a configuration change" | Reasonable. |
| §4.1 | "(or Ubuntu)" packaging may be reused | This extends the original, which only says Debian. It is harmless, but the ban on contact should then clearly cover Ubuntu maintainers too. |
| §5 | "an automated check … fails on any unexplained difference" | A strong but reasonable reading of "documented and checked". |
| §6 | "through the same entry point"; "clean container per distribution and architecture" | Reasonable. |
| §6 | Rebuild inputs include the container image and the repository's own tooling | Reasonable. Note that this implies pinned base-image digests. Otherwise a daily upstream image refresh rebuilds everything. Say this in the design. |
| §6 | The full rebuild is "compared against the reused artifacts" | A good reading of "verify that there are no hidden state changes". |
| §6 | "GitHub Actions must still be able to build everything without them" | Follows from "build … on github actions". Fine. |
| §8 | Repository is "signed"; landing page with `sources.list` instructions | Not in the original, but apt.fpgas.online (the model site) does both. Fine. |
| §8 | Preview details (a directory per PR, an index, cleanup) | Verified to match preview.wafer.space (§4.4). Fine. |
| §9 | Updates arrive "as a pull request" | Consistent with the PR/review workflow. Fine. |
| §10 | Decision records in `docs/decisions/` with alternatives | A reasonable way to meet "why certain decisions where made". |
| §12 | Reviewer is "not the author" | This is what "independent" means. Fine. |

### 2.2 Genuine additions (not in the original; confirm them or label them as such)

- **§6 "`big-storage.welland.mithis.com` (storage)"**: the "(storage)" role
  is inferred from the hostname. The original only says "resources
  available at". **Fix:** drop "(storage)" or write "(role to be
  confirmed)".
- **§6 "the Docker VMs on harken and micky"**: the original says "the docker
  vm", singular, "found on harken.mithis.com and micky.mithis.com". This is
  harmless. Keep the plural.
- **§11 "License: Apache 2.0"**: the original never mentions a license. The
  repository already contains an Apache-2.0 `LICENSE` from the initial
  commit, so this records an existing fact rather than a requirement. It is
  fine to keep, but it could be flagged as "existing, not owner-mandated".
- **§11 the list of GitHub settings**: this summarises `~/.claude/GitHub.md`
  correctly. It leaves out the UI-only "Include Git LFS objects in archives"
  setting. **Fix:** say "all settings in `GitHub.md` (summary: …)" so the
  file stays authoritative.
- **§13 Q2 (coexistence / namespacing)**: the original does not raise this.
  It is a legitimate question because it follows from "identical revisions"
  combined with distro-provided packages. Keep it.

---

## 3. Ambiguities and suggested questions

### 3.1 Assessment of the existing open questions

| Q | Needed? | Comment / suggested rewrite |
|---|---|---|
| 1 Scope boundary | **Yes** | Reframe: "Your instruction says *everything* used or installed by the environment. Do you really want packages of coreutils, make, grep, gawk, git, zsh, Tcl, Ruby and a separate Python 3.12.10 interpreter at the Nix revisions, or only the EDA tools and the Python libraries? Note that jammy ships Python 3.10 and trixie ships 3.13, so matching Nix's Python library revisions probably means shipping our own interpreter as well." |
| 2 Coexistence | **Yes** | Merge it with Q1. The answer to Q1 largely decides Q2. |
| 3 Stable vs trixie | **Yes, but re-aim it** | See 3.2.B. The likely intent is stable/**testing**/unstable. Ask that first. `bookworm` is a secondary point. |
| 4 PDK dummies | **Partly** | The instruction already answers "all families, all revisions": "one dummy package per PDK revision that ceil tooling exposes". Keep only (a) a confirmation of scale (currently 223 packages, see §4.5) and (b) the real question: should the template's own PDK (`wafer-space/gf180mcu` tag `1.4.4`, cloned by the `Makefile`, not taken from ciel) get a dummy package too? |
| 5 Hosting limits | **Yes, sharpen it** | See 3.2.D. Also fix the fact: it is a hard "no larger than 1 GB", not "about 1 GB" (§4.3). |
| 6 Tag format | **Yes** | `GitHub.md` explicitly says "ask user for format preference". |

### 3.2 New questions to add

**A. [Q] What does "`git describe` commit counters" apply to?** (Doc §7.
Critical, and not covered.)

- **Original:** "make sure that all package versions use `git-describe`
  commit counters for versioning"
- **Problem:** A `.deb` of yosys needs a version. The candidates are:
  - (a) yosys's own `git describe`, for example `0.54` or
    `0.54+12-gabcdef`;
  - (b) this packaging repository's `git describe`, which loses the upstream
    version entirely;
  - (c) both combined, for example `0.54-0.0.post123` or
    `0.54+wafer0.0.123~noble`.

  Some upstreams do not use git tags usefully: iverilog's
  `s20250103-60-gdb82380ce` already *is* a describe string, and surelog's is
  `1.84-unstable-2024-12-06`. Tarball-only sources have no describe at all.
  PDK dummy packages are identified by a ciel commit hash, and the
  instruction attaches no counter to them.
- **Also:** The same upstream version is built for 7+ suites and shares one
  pool, so versions need a per-suite suffix (for example `~jammy`) to avoid
  identical-name, different-content collisions. This conflicts with a pure
  counter scheme.
- **Suggested question:** "For packaged upstream tools, should the Debian
  version be `<upstream version>-<this repo's git-describe counter>~<suite>`
  (for example `0.54-0.0.post123~noble`)? Or did you mean something else by
  git-describe counters?" The rest of §7 needs rewriting to match the answer.

**B. [Q] "debian trixie": did you mean Debian *testing*?** (Doc §3.1, Q3.)

- **Original:** "debian stable / debian trixie / debian sid/unstable"
- **Problem:** Listing stable, trixie and sid reads like the classic
  stable/testing/unstable trio, written when trixie *was* testing (before
  August 2025). Today stable **is** trixie (13.7), so the item is
  redundant, and testing (`forky`) is missing (§4.2).
- **Suggested question:** "Should the Debian set be stable (trixie), testing
  (forky) and unstable (sid)? Should the suites follow the `stable`/`testing`
  aliases automatically when forky is released, or stay pinned by codename?
  Should bookworm (oldstable) be included?"

**C. [Q] "Currently supported Ubuntu LTS": standard support only, or ESM
too?** (Doc §3.1.)

- **Original:** "all currently supported ubuntu LTS versions"
- **Doc:** "Every Ubuntu LTS release in **standard** support". This
  narrowing is not in the original.
- **Problem:** Ubuntu's own `meta-release` file lists `trusty`, `xenial`,
  `bionic` and `focal` with `Supported: 1` (they are in ESM / Ubuntu Pro),
  alongside `jammy`, `noble` and `resolute` (§4.1). Any tooling that reads
  that flag would include them. Building on focal or older (GCC 9, Python
  3.8) would be very costly.
- **Suggested question:** "Does 'supported' mean standard support only
  (today jammy, noble, resolute), or also ESM releases (focal, bionic,
  xenial, trusty)?"

**D. [Q] Preview size versus "complete contents".** (Doc §8, Q5.)

- **Original:** "Pull requests should publish a "preview" repo with complete
  contents repos" and "a seperate repo might be needed … depending on the
  resulting package sizes."
- **Problem:** A published Pages site is hard-limited to 1 GB. The full
  repository covers 7 suites × 2 architectures × roughly 40+ packages
  (OpenROAD, KLayout, Verilator…), which may alone approach that limit. A
  *complete* copy per open PR is then impossible within one preview site.
  apt.fpgas.online also keeps every historical version, which compounds
  this.
- **Suggested question:** "If a complete per-PR copy does not fit, is it
  acceptable for a preview to contain only the changed packages plus a
  pointer to the production repository for the rest (for example a combined
  `Packages` index whose unchanged entries point at production)? What
  should happen if that is not possible: GitHub Releases as the `.deb`
  store, an external host (big-storage?), or a CDN?"

**E. [Q] Retention of old package versions.** (Not covered.) Should
production keep every published version (as apt.fpgas.online does) or only
the current one per suite? This directly affects size (D).

**F. [Q] Owner-only actions needed before autonomous work.** (Not covered.
§12 promises autonomy after the questions.)

- DNS: who creates the `apt.wafer.space` and `preview.apt.wafer.space`
  CNAMEs?
- Permission to create new repositories in the `wafer-space` org (for
  example `wafer-space/apt.wafer.space`, `wafer-space/preview.apt.wafer.space`)
  and to configure Pages, deploy keys and secrets on them.
- The APT signing key: who generates it, what identity/email it uses, and
  where the private key lives (a repository or org secret).
- **Suggested question:** "Please confirm who does each of these, or grant
  permission for me to do them."

**G. [Q] How do I reach the `hetzner-ansible` Claude session?** (Doc §6.)
The original requires asking it for a user and key, but gives no channel
for doing so. Ask for the mechanism, and confirm that GitHub-only builds
proceed until access exists.

**H. (Lower priority) Interim Ubuntu releases once superseded.**

- **Original:** "the latest version of ubuntu"
- **Problem:** When 27.04 is released, 26.10 is still supported for about 3
  months but is no longer "latest".
- **Suggested wording, no question needed:** "Only the newest release is
  built. A superseded interim release stops receiving new builds, but its
  published packages stay until it reaches end of life." Record this as a
  decision.

**I. (Future, no question now) 32-bit targets.**

- **Original:** "RISC-V 32bit and 64bit, and arm32 (for rpi hardware)"
- **Problem:** No Debian or Ubuntu release has a `riscv32` architecture
  (§4.2). `armhf` (the doc's interpretation) needs ARMv7 or later, so it
  excludes Raspberry Pi 1/Zero, which are ARMv6 and are served by Raspberry
  Pi OS's own armhf build or by Debian `armel`.
- **Fix:** In §3.2, write "32-bit ARM for Raspberry Pi (exact Debian
  architecture, `armhf` or `armel`, decided when added)". Add a note that
  RISC-V 32-bit has no Debian/Ubuntu port today, so this item needs
  re-evaluation when it is scheduled.

**J. (Low) Contact with non-Debian upstreams.** The ban covers "these
people", meaning the Debian maintainers. The document correctly keeps that
scope. It could say explicitly that it does not cover upstream projects
(YosysHQ, OpenROAD…), or ask the owner whether upstream should also not be
contacted. Suggest asking this in the same batch, because "autonomous" plus
AI-generated bug reports to upstreams could be unwelcome.

### 3.3 A subtle point to state explicitly (no question needed)

"Reuse Debian packaging" and "identical configuration to Nix" pull in
opposite directions. Debian often unvendors libraries and enables or
disables features differently. §4.1 already says "adapt it to the Nix
revision and configuration". Add: "Where Debian's packaging choices differ
from Nix's configuration, Nix's configuration wins." This follows from §5.

---

## 4. Factual checks

### 4.1 Ubuntu releases: correct, with one caveat

Source: `https://changelogs.ubuntu.com/meta-release`, fetched 2026-09-24:

```
Dist: focal     Version: 20.04.5 LTS  Supported: 1
Dist: jammy     Version: 22.04.5 LTS  Supported: 1
Dist: noble     Version: 24.04.5 LTS  Supported: 1
Dist: questing  Version: 25.10        Supported: 0
Dist: resolute  Version: 26.04.1 LTS  Date: Thu, 23 April 2026  Supported: 1
```

`trusty`, `xenial` and `bionic` are also `Supported: 1`, which reflects ESM.
From `https://changelogs.ubuntu.com/meta-release-development`:

```
Dist: stonking  Version: 26.10  Date: Thu, 15 October 2026  Supported: 0
```

- "26.04 `resolute`" is correct. It is the newest release, since 25.10
  `questing` is out of support.
- "jammy, noble, resolute" is correct for *standard* support. Jammy's
  standard support end (April 2027) comes from Ubuntu's published lifecycle,
  not from this file. The "Supported" flag in the file includes ESM
  releases; see 3.2.C.
- 26.10 is codenamed **`stonking`** and is due **2026-10-15**, three weeks
  from now. Tooling must handle this almost immediately. It is worth naming
  in §3.1.

### 4.2 Debian: correct

Sources:

- `https://deb.debian.org/debian/dists/stable/Release`:
  `Suite: stable`, `Version: 13.7`, `Codename: trixie`
- `…/dists/oldstable/Release`: `Version: 12.15`, `Codename: bookworm`
- `…/dists/testing/Release`: `Codename: forky`

"Debian stable. Today: 13 `trixie`" is correct.

Architectures, from the `Architectures:` lines of the Release files:

- trixie: `all amd64 arm64 armel armhf i386 ppc64el riscv64 s390x`
- sid: `all amd64 arm64 armhf i386 loong64 ppc64el riscv64 s390x`
- Ubuntu noble (ports.ubuntu.com): `amd64 arm64 armhf i386 ppc64el riscv64
  s390x`

No `riscv32` exists in any of them. `armel` is gone from sid.

### 4.3 GitHub Pages limits: partly inaccurate in Q5

Source: <https://docs.github.com/en/pages/getting-started-with-github-pages/github-pages-limits>

> "GitHub Pages source repositories have a recommended limit of 1 GB."
> "Published GitHub Pages sites may be no larger than 1 GB."
> "GitHub Pages deployments will timeout if they take longer than 10 minutes."
> "GitHub Pages sites have a *soft* bandwidth limit of 100 GB per month."

- **Fix Q5:** Replace "about 1 GB" with "a hard 1 GB limit per published
  site, a 10-minute deployment timeout, and a soft 100 GB/month bandwidth
  limit". Large `.deb`s downloaded by many users make the bandwidth limit
  relevant too.
- **Not verified from docs, but well known:** each Pages site has a single
  custom domain (one `CNAME`). So `apt.wafer.space` and
  `preview.apt.wafer.space` need **two Pages sites, meaning at least two
  repositories, whatever the package sizes**. This is how
  preview.wafer.space is already a separate repository. §8's "Separate
  repositories are used … if package sizes … require it" should say that the
  preview needs its own repository in any case.

### 4.4 preview.wafer.space pattern: correct

Source: the `wafer-space/preview.wafer.space` README (via `gh api`):

> "Each PR gets its own directory (`pr-1/`, `pr-2/`, etc.)"
> "`index.html` - Auto-generated listing of all active previews"
> "PR directories are automatically removed when PRs are closed or merged"
> "Uses SSH deploy keys for secure access from the main repository's GitHub Actions"

§8's description matches. The deploy-key mechanism is a useful precedent
for the design.

### 4.5 ciel: correct

- The repository is `fossi-foundation/ciel` ("Open-source PDK version
  manager", from `gh api repos/fossi-foundation/ciel`). The original's
  "ceil" is a typo that the document silently fixes. Good.
- The template's `flake.lock` pins ciel rev `9620877b…`, whose
  `pyproject.toml` says `version = "2.3.1"`. This matches the §2 table.
  PyPI's latest is `3.0.0`, which is irrelevant because the Nix revision is
  authoritative, but it shows that the pinned ciel will drift from upstream.
- ciel 2.3.1's default data source is
  `static-web:https://fossi-foundation.github.io/ciel-releases`
  (`ciel/source.py`). Counting `.versions` in
  `https://fossi-foundation.github.io/ciel-releases/<pdk>/manifest.json`
  gives **gf180mcu 83, sky130 116, ihp-sg13g2 24**. Q4's numbers are
  correct.
- Revisions are identified by commit hash (for example
  `1689ac3f2dc763876eaf967227c7dfe831b031ae`). This is relevant to
  versioning; see 3.2.A.

### 4.6 Nix snapshot table (§2): spot-checked, correct

Source: the template's `flake.nix`/`flake.lock` on `main` (`6c384ae`), plus
`nix eval` of `legacyPackages.x86_64-linux`:

```
yosys-0.54 openroad-2025-10-28 netgen-1.5.305 magic-vlsi-8.3.581 klayout-0.30.4
iverilog-s20250103-60-gdb82380ce verilator-5.038 surelog-1.84-unstable-2024-12-06
gtkwave-3.3.121 surfer-0.3.0 git-2.49.0 python3-3.12.10 ruby-3.3.8 graphviz-12.2.1
python3.12-cocotb-2.0.0 python3.12-ciel-2.3.1 delta-0.18.2 zsh-5.9
```

- `flake.lock` confirms nix-eda `ref 5.9.0`, librelane rev `3affff86…`
  (following branch `leo/gf180mcu`) and nixpkgs rev `b2485d56…` (`nixos-25.05`).
- The `magic` override (`version = "8.3.581"`) is in `flake.nix`, as §2
  says.
- The template's `Makefile` has `PDK_TAG ?= 1.4.4` and clones
  `https://github.com/wafer-space/gf180mcu.git`. This confirms Q4's
  statement.
- I did not check: tclFull, the gnumake/gnugrep/gawk/coreutils versions,
  and the individual Python library versions other than cocotb and ciel.

### 4.7 apt.fpgas.online: the §8 additions are consistent with it

Source: `https://apt.fpgas.online/`, fetched 2026-09-24. It is a signed
repository ("`curl -fsSL https://fpgas-online.github.io/apt/pubkey.gpg`";
`[signed-by=…]`) with per-suite `$(lsb_release -cs)`, a landing page with
install snippet, git-describe style versions (`v0.0.post557`) and *all
historical versions retained*.

---

## 5. Minor wording

| Doc § | Current | Suggested |
|---|---|---|
| §3.1 | "Debian `trixie`, by name." | Rewrite once 3.2.B is answered. As it stands it lists the same suite twice without explaining why. |
| §3.1 | "When 26.10 is released it is added automatically." | "26.10 `stonking` (due 2026-10-15) must be picked up automatically." |
| §4.1 | "…or with Debian/Ubuntu infrastructure on their behalf." | Unclear. Write: "…no bug reports, merge requests, emails or other contact, by any channel (Debian BTS, Salsa, Launchpad, mailing lists)." |
| §4.2 | "PDK data is **not** packaged directly." | The original says "not packaged automatically". Write: "PDK data itself is never packaged. Only dummy installer packages are." |
| §4.2 | (no location given) | Say that the system-wide location and the uninstall behaviour are design decisions to record (for example `/usr/share/pdk` or `/opt/pdk`, `PDK_ROOT`). |
| §6 | "`reprotest`" as the check | The original says "all reproducible build checks". Add "and any additional checks used by reproduce.debian.net / `diffoscope`" so it does not read as "one tool is enough". |
| §6 | Rebuild inputs | Add "container base images pinned by digest", or daily image refreshes will defeat "rebuild only on real change". |
| §10 | "decision records in `docs/decisions/`" | Also say where the process log and todo list live (for example `docs/status/`), so §12 restartability has a fixed location. |
| §12 | "reviewed by an independent agent" | The original says "agents". Either wording is fine. Optionally add "at least one". |
| §13 Q5 | "about 1 GB" | "a hard 1 GB limit" (§4.3). |
