# workflow-testing

A GitHub Actions harness for exercising the **perl build system across operating
systems, architectures and toolchains**. It builds perl on every combination in
a declarative matrix, records the outcome of each combination as a small JSON
artifact, and renders a per-OS results table in the run summary and on a status
page.

The matrix itself lives in [`ci.yml`](.github/workflows/ci.yml). Three reusable
workflows do the work:

| Workflow | Purpose |
|:--|:--|
| [`run-vm.yml`](.github/workflows/run-vm.yml) | Runs a setup inside a `vmactions` VM: FreeBSD, OpenBSD, NetBSD, DragonFly BSD, Solaris, OmniOS. |
| [`run-direct.yml`](.github/workflows/run-direct.yml) | Runs a setup directly on a GitHub-hosted runner: Linux, macOS, Windows. |
| [`results-summary.yml`](.github/workflows/results-summary.yml) | Collects every result artifact, renders the tables, emits `status.json`. |

---

## Contents

- [How a run works](#how-a-run-works)
- [Selecting what runs](#selecting-what-runs)
  - [`[runner:...]`](#runner)
  - [`[test:...]`](#test)
  - [Manual runs](#manual-runs)
  - [How a selection is applied](#how-a-selection-is-applied)
  - [Edge cases](#edge-cases)
- [The matrix](#the-matrix)
  - [Entry fields](#entry-fields)
  - [Recipes](#recipes)
  - [Adding an OS family](#adding-an-os-family)
- [Per-job mechanics](#per-job-mechanics)
- [Results and the status page](#results-and-the-status-page)
- [Reference tables](#reference-tables)
- [Notes and caveats](#notes-and-caveats)

---

## How a run works

```
   push / pull_request / schedule / workflow_dispatch
                    |
                    v
        +---------------------------+
        |  setup  (in ci.yml)       |
        |                           |
        |  runner    " freebsd … "  |  which families should run
        |  select    {"linux": …}   |  which archs, per family
        |  test_files                |  the test pattern
        +---------------------------+
             |                  |
        family gate          family gate
     if: contains(...)     if: contains(...)
             |                  |
   freebsd ──┘                  └── linux  ── macos ── windows
   openbsd                        (run-vm)    (run-direct)
   netbsd  ...                            |
   dragonflybsd                           v
   solaris                        +----------------+
   omnios                         |  select  job   |  prune matrix entries
   (run-vm)                       |  jq filter     |  whose arch was not
                                  +----------------+  selected
                                          |
                                          v
                                 run  (one job per setup)
                                          |
                                          v
                            result-artifact-<setup>.json
                            result-artifact-<setup>.result
                                          |
                                          v
                              results ──> results-summary
                                          │
                                          v
                            status.md + status.json
                                          │
                                          v
                        publish-status (main only) ──> gh-pages /status
```

Triggers: `pull_request`, `push`, a weekly `schedule` (Sunday 05:42 UTC) and
`workflow_dispatch`.

An untagged commit runs everything, so the default behaviour is unchanged from a
workflow with no filtering at all.

---

## Selecting what runs

Both tags are plain text in the commit message. For a pull request, the **PR
title** is used, since the head commit's message is not available to
`pull_request` events.

<a name="runner"></a>
### `[runner:...]`

Comma-separated list of OS families, each optionally narrowed to one or more
architectures with `#<arch>`:

```
[runner:freebsd,linux#arm]
      │      └── linux, but only the aarch64 setups
      └───────── only the freebsd setups (every arch)
```

- A bare family name (`freebsd`) runs **every setup** in that family.
- `family#arch` runs only the setups in that family built for that
  architecture.
- There is no shorthand for "this family, these two architectures" — repeat the
  family name instead: `[runner:linux#arm,linux#riscv]` selects both the ARM and
  the RISC-V Linux setups.
- `all` — or an absent tag — runs every family and every setup.
- Matching is case-insensitive and surrounding whitespace is ignored, so
  `[runner: FreeBSD , LINUX#ARM64 ]` works.

Arch aliases (all interchangeable):

| You may write | Canonical value | Report label |
|:--|:--|:--|
| `x86-64`, `x86_64`, `x64`, `intel`, `amd64` | `x86_64` | Intel |
| `arm`, `arm64`, `aarch64`, `a64` | `aarch64` | ARM |
| `riscv`, `riscv64`, `r64` | `riscv64` | RISC-V |

<a name="test"></a>
### `[test:...]`

A test file pattern, passed down to every selected setup as `$TEST_FILES`:

```
fix the isolation tests [runner:linux,macos] [test:t/3000_jenny/3500_isolate/*.t]
```

`$TEST_FILES` is honoured by ExtUtils::MakeMaker's `test_harness` target, which
is what a coverage build ultimately runs. The value is filtered down to glob
characters (`A-Za-z0-9_.,-/*`) before it reaches any shell, so a commit message
cannot inject commands into the run steps.

Note that the setups in this repository run `perl -V` as their task, so
`$TEST_FILES` is currently inert here — it becomes live as soon as a task runs
tests.

### Manual runs

`workflow_dispatch` exposes the same two selectors as inputs, and they take
precedence over the commit message:

```yaml
on:
  workflow_dispatch:
    inputs:
      runner:
        default: all
        type: string
      test_files:
        default: ''
        type: string
```

Dispatching with `runner=openbsd#riscv,windows` and
`test_files=t/3000_jenny/*.t` behaves exactly like a commit carrying
`[runner:openbsd#riscv,windows] [test:t/3000_jenny/*.t]`.

### How a selection is applied

Selection happens in two stages, so a filtered run never starts work it does not
need:

1. **Family gate** — `setup` emits a *space-padded* list of family names and
   every family job tests it with `contains()`:

   ```yaml
   freebsd:
     needs: setup
     if: "${{ contains(needs.setup.outputs.runner, ' freebsd ') }}"
   ```

   The padding is what makes this a whole-token test: without it, `contains()`
   would match any token that merely *contained* the family name, and a
   misspelled or future token could silently enable the wrong job. `' all '` is
   not special-cased either — `all` expands to the full list, so every gate
   below is written the same way.

2. **Matrix prune** — `setup` also emits a per-family arch filter
   (`{"freebsd":"", "linux":"aarch64", …}`), which the reusable workflow hands to
   a small `select` job. That job filters the JSON matrix with `jq` and the
   matrix job consumes the result:

   ```yaml
   select:
     outputs:
       include: '${{ steps.select.outputs.include }}'
       count:    '${{ steps.select.outputs.count }}'
   run:
     needs: select
     if: needs.select.outputs.count != '0'
     strategy:
       matrix:
         include: ${{ fromJSON(needs.select.outputs.include) }}
   ```

   When a selection matches no setup at all (`[runner:dragonflybsd#arm]`, where
   DragonFly BSD only builds `x86_64`), the matrix job is skipped rather than
   failing.

### Edge cases

| Tag | Result |
|:--|:--|
| *(absent)* | Everything runs. |
| `[runner:all]` | Everything runs. |
| `[runner:]` | Everything runs. |
| `[runner:plan9]` | Warning emitted, **everything runs** — an unknown family is treated as a typo, not as "run nothing". |
| `[runner:linux#arm,linux#arm]` | One `aarch64` filter; the family appears once. |
| `[runner:linux#sparc]` | Warning emitted, the arch filter is **dropped**, and every Linux setup runs. |
| `[runner:dragonflybsd#arm]` | The family is selected, but no setup in its matrix is `aarch64`, so the matrix job is skipped and the report shows `:x: no results reported`. |

---

## The matrix

Each family job passes a JSON array of setups to a reusable workflow. Every
entry describes one job.

```yaml
  freebsd:
    name: FreeBSD
    needs: setup
    if: "${{ contains(needs.setup.outputs.runner, ' freebsd ') }}"
    uses: ./.github/workflows/run-vm.yml
    secrets: inherit
    with:
      os_version: '15.0'
      task: 'perl -V'
      matrix: >
        [
          {"platform_name":"freebsd","os":"freebsd","arch":"x86_64","toolchain":"gcc","prepare":"pkg install -y perl5 lang/gcc"},
          {"platform_name":"freebsd","os":"freebsd","arch":"aarch64","toolchain":"clang","prepare":"pkg install -y perl5 llvm"}
        ]
      select: "${{ fromJSON(needs.setup.outputs.select).freebsd }}"
      test_files: "${{ needs.setup.outputs.test_files }}"
```

<a name="entry-fields"></a>
### Entry fields

| Field | Required | Meaning |
|:--|:--:|:--|
| `platform_name` | yes | Short platform tag. Prefixes the setup's filenames and lands in the result JSON as `platform`. |
| `os` | yes | `run-direct`: the GitHub runner label. `run-vm`: the OS family, which also selects the VM action step. |
| `arch` | yes | `x86_64`, `aarch64` or `riscv64`. Also selects the VM architecture. |
| `toolchain` | no | `gcc`, `clang`, `egcc`, `msvc`, … Used in the generated job name and in the default build command. |
| `name` | no | Overrides the generated job name *and* the report's Setup column. |
| `prepare` | no | Shell command run before the build — the VM's package install, or a host-side install. |
| `perl_version` | no | Full version such as `5.42.0`. Enables building perl from source, and the `perl-dist` cache. |
| `configure` | no | Extra `./Configure` flags, used only with `perl_version`. |
| `variant` | no | Free-form suffix, e.g. `debug`. Appears in the job name and the report. |

Reusable-workflow inputs:

| Input | `run-vm` | `run-direct` | Meaning |
|:--|:--:|:--:|:--|
| `matrix` | yes | yes | The JSON array of setups. |
| `os_version` | yes | – | Release passed to the `vmactions` action, e.g. `15.0`, `r151056`. |
| `host_os` | optional | – | Runner label hosting the VM. Defaults to `ubuntu-latest`. |
| `task` | optional | optional | Command to run. Defaults to `perl build.pl --compiler=<toolchain> coverage`. |
| `select` | optional | optional | Arch filter from the `[runner:...]` tag. |
| `test_files` | optional | optional | Test pattern from the `[test:...]` tag. |

<a name="recipes"></a>
### Recipes

**Build NetBSD on x86_64 with clang** — the family already exists, so this is a
one-line addition to the `netbsd` matrix:

```json
{"platform_name":"netbsd","os":"netbsd","arch":"x86_64","toolchain":"clang","prepare":"/usr/sbin/pkg_add perl"}
```

**Add a named, labelled setup**

```json
{"platform_name":"freebsd","os":"freebsd","arch":"x86_64","toolchain":"gcc","prepare":"pkg install -y perl5 lang/gcc","name":"smoke build"}
```

The generated job name (`x64-gcc`) is replaced by `smoke build` everywhere, and
the report's Setup column shows `smoke build` instead of `Intel`. A `name` is
also prefixed to the setup's filenames (`smoke-build-freebsd-x86_64-gcc-1a2b3c4d`),
with spaces and slashes replaced by hyphens so it is safe in an artifact name.

**Tag a build as a variant** — for two builds of the same toolchain that differ
in some other way:

```json
{"platform_name":"openbsd","os":"openbsd","arch":"x86_64","toolchain":"egcc","variant":"debug"}
```

`variant` disambiguates the job name (`x64-egcc-debug`) and the report row
(`Intel (debug)`).

**Build a specific perl from source and cache it**

```json
{"platform_name":"macos","os":"macos-15","arch":"aarch64","toolchain":"clang","perl_version":"5.40.0","configure":"-Accflags=-O0"}
```

The first run compiles perl and installs it into `./perl-dist`; later runs reuse
it. When `perl_version` is set, the default task's environment is prefixed with
`perl-dist/bin`, and the version actually produced is reported in the summary
(it may differ from the requested one).

**Run a one-off command instead of a build** — `task` applies to every setup in
the matrix:

```yaml
      task: 'perl build.pl --compiler=gcc -Dusethreads coverage'
```

**Filter a run from a commit**

```
Only chase the Windows regressions [runner:windows] [test:t/5005pod.t]
ARM only, and a single directory                     [runner:linux#arm,openbsd#arm] [test:t/porting/*.t]
All the BSDs, nothing else                           [runner:freebsd,openbsd,netbsd,dragonflybsd]
```

<a name="adding-os-family"></a>
### Adding an OS family

A family spans four places. For a new BSD, say:

1. **`ci.yml` `setup`** — add the name to `FAMILIES`:

   ```bash
   FAMILIES=(freebsd openbsd netbsd dragonflybsd solaris omnios linux macos windows midnightbsd)
   ```

   The order here only affects the order of the emitted `runner` list; the order
   families appear in the report comes from `results-summary.yml` in step 4.

2. **`ci.yml`** — add a job that calls the reusable workflow, gated like the
   others:

   ```yaml
     midnightbsd:
       name: MidnightBSD
       needs: setup
       if: "${{ contains(needs.setup.outputs.runner, ' midnightbsd ') }}"
       uses: ./.github/workflows/run-vm.yml
       secrets: inherit
       with:
         os_version: '4.0.4'
         task: 'perl -V'
         matrix: >
           [
             {"platform_name":"midnightbsd","os":"midnightbsd","arch":"x86_64","toolchain":"gcc","prepare":"sudo mport install perl gcc"}
           ]
         select: "${{ fromJSON(needs.setup.outputs.select).midnightbsd }}"
         test_files: "${{ needs.setup.outputs.test_files }}"
   ```

   And add it to the `results` job's `needs:` list.

3. **`run-vm.yml`** — add a VM step for it, gated on the `os` value, and a `case`
   arm in the *Collect environment info* step so the report names it correctly:

   ```bash
     midnightbsd)  family='MidnightBSD' ;;
   ```

   The `os` value in the matrix entry is what selects the step
   (`if: matrix.os == 'midnightbsd'`); an entry with an `os` no step handles
   silently builds nothing.

4. **`results-summary.yml`** — add the display name to `order` so it is not
   appended alphabetically at the end:

   ```bash
   order=("Linux" "macOS" "Windows" "FreeBSD" "OpenBSD" "NetBSD" "DragonFly BSD" "Solaris" "OmniOS" "MidnightBSD")
   ```

For a hosted runner, the equivalent steps are a `run-direct` job in `ci.yml` and
an `os` arm in *Collect environment info* in `run-direct.yml`.

---

## Per-job mechanics

### Job names

A setup's job name is generated from its entry:

```
<arch>-<toolchain>[-v<perl_version>][-<variant>]

x64-gcc
a64-clang
r64-egcc
x64-gcc-v5.42.0
x64-egcc-debug
```

where `<arch>` is the short form: `x64` for `x86_64`, `a64` for `aarch64`,
`r64` for `riscv64`. A `name` field replaces the whole thing.

### Fingerprint and filenames

Every setup gets a stable 8-character id: the SHA-256 of every field of its
matrix entry, truncated. Two setups that differ in *any* way — including only
`configure` or `prepare` — get different ids, so they never overwrite each
other's artifacts or collide as report rows.

The id is appended to the base filename:

```
[<name>-]<platform_name>-<arch>-<toolchain>[-v<perl_version>][-<variant>]-<id>

freebsd-x86_64-gcc-1a2b3c4d.json
smoke-build-freebsd-x86_64-gcc-1a2b3c4d.json
```

Each job uploads `result-artifact-<FNAME>` containing `<FNAME>.json` and
`<FNAME>.result` (the raw job status), with a one-day retention. Both are
uploaded with `if: always()`, so a failed setup still reports.

### Caching source-built perls

Setups with a `perl_version` build perl from source and install it into
`./perl-dist`, which is cached with a strict key:

```
perl-dist-<os>-<arch>-<toolchain>-<perl_version>-<configure flags>
```

with spaces and hyphens stripped from the flags. There are deliberately no
`restore-keys`: every component that changes the produced binary is in the key,
so a cache hit always means "exactly this perl".

Inside the VMs, perl is built serially on purpose. A parallel build trips over
gmake's jobserver tokens in the nested subdirectory makes — those are the
guest's make, not gmake — which aborts with `argument ... to option '-j' must be
a positive number` on the BSDs. Hosted runners have no such problem and build in
parallel.

### Windows specifics

`run-direct.yml` prepends Strawberry Perl to `PATH` in two places, because
git-bash re-injects its own MSYS2 perl ahead of anything the workflow adds. A
guard step then fails fast if `perl` reports anything other than `MSWin32`, which
turns a confusing downstream `Can't locate <core module>` into an obvious
message. `msvc` setups additionally use `TheMrMilchmann/setup-msvc-dev`.

---

## Results and the status page

### What each job reports

```json
{
  "status": "success",
  "platform": "ubuntu",
  "family": "Linux",
  "arch": "aarch64",
  "arch_short": "a64",
  "toolchain": "clang",
  "perl": "v5.42.0",
  "variant": "",
  "name": "",
  "id": "1a2b3c4d"
}
```

### The summary table

One table per OS family, in a fixed order, with unknown families appended
alphabetically:

```markdown
## Build and Test Summary

Test pattern: `t/3000_jenny/*.t`

### Linux
| Setup | Toolchain | Perl | Result |
|:---|:---|:---|:---|
| ARM | clang | 5.42.0 | :x: |
| Intel | gcc | 5.42.0 | :white_check_mark: |

### macOS
| Setup | Toolchain | Perl | Result |
|:---|:---|:---|:---|
| ARM | gcc | 5.40.0 | :white_check_mark: |
| my custom build | gcc | 5.42.0 | :white_check_mark: |

### Windows
| Setup | Toolchain | Perl | Result |
|:---|:---|:---|:---|
| all setups | - | - | :fast_forward: not selected by runner filter |
```

- The **Perl** column appears only when more than one distinct perl version was
  seen across the whole run; otherwise every row would repeat the same value.
- A family that was filtered out of the run shows a single
  `:fast_forward: not selected by runner filter` row, so a partial run still
  reads as a full picture. A family that *was* selected but reported nothing
  shows `:x: no results reported` instead.

### status.json and the status page

```json
{
  "1a2b3c4d": { "name": "Intel", "status": "👍🏽 passing" },
  "5e6f7a8b": { "name": "ARM",   "status": "👎🏽 failing" }
}
```

Keys are the setup fingerprints, which stay the same for as long as a matrix
entry does not change — so a page built from this file tracks configurations
rather than run numbers. Change any field of an entry and it becomes a new
configuration with a new key.

`results-summary` uploads `status.md` and `status.json` as the `status-files`
artifact, and `publish-status` deploys it to the `gh-pages` branch under
`/status` — **only** on pushes to `main`.

---

## Reference tables

### Families

| Tag | Report name | Runner | Architectures | Toolchains |
|:--|:--|:--|:--|:--|
| `freebsd` | FreeBSD | VM 15.0 | x86_64, aarch64 | gcc, clang |
| `openbsd` | OpenBSD | VM 7.8 | x86_64, aarch64, riscv64 | egcc, clang |
| `netbsd` | NetBSD | VM 10.1 | x86_64, aarch64 | gcc |
| `dragonflybsd` | DragonFly BSD | VM 6.4.0 | x86_64 | gcc |
| `solaris` | Solaris | VM 11.4 | x86_64 | gcc |
| `omnios` | OmniOS | VM r151056 | x86_64 | gcc |
| `linux` | Linux | hosted | x86_64, aarch64, riscv64 | gcc, clang |
| `macos` | macOS | hosted | x86_64, aarch64 | gcc, clang |
| `windows` | Windows | hosted | x86_64, aarch64 | gcc, clang, msvc |

### Hosted runners

| Arch | `linux` | `macos` | `windows` |
|:--|:--|:--|:--|
| x86_64 | `ubuntu-latest` | `macos-15-intel` | `windows-latest` |
| aarch64 | `ubuntu-24.04-arm` | `macos-15` | `windows-11-arm` |
| riscv64 | `ubuntu-24.04-riscv` | – | – |

The `os` field is the runner label, so adding a new hosted architecture is a
matter of pointing it at whichever label GitHub offers.

## Notes and caveats

- **A filtered run publishes a partial status page.** `status.json` only contains families that produced results, and `publish-status` overwrites the previous file (with `keep_files: true`, which preserves the site's *other* files, not the previous `status.json`). This is pre-existing behaviour, but it is much more visible now that filtered runs are easy to trigger — a filtered run on `main` will make unselected configurations disappear from the page.
- **The `select` job costs a runner.** Each family now starts one short job before its matrix job, which is what allows unselected architectures to be skipped rather than run and reported as failures.
- **Untrusted input is filtered twice.** Runner names are matched against a fixed list with `grep -qxF` (fixed-string, never a regex), and the test pattern is reduced to glob characters. Family and arch names are also canonicalised, so nothing from a commit message can reach a shell unfiltered.
