# workflow-testing

A GitHub Actions harness for exercising the **perl build system across operating
systems, architectures and toolchains**. It builds perl on every combination in
a declarative matrix, records the outcome of each combination as a small JSON
artifact, and renders a per-OS results table in the run summary and on a status
page.

The setups are declared once, as a default input on
[`generate-matrix.yml`](.github/workflows/generate-matrix.yml).
[`ci.yml`](.github/workflows/ci.yml) calls it, then dispatches to three more
reusable workflows:

| Workflow | Purpose |
|:--|:--|
| [`generate-matrix.yml`](.github/workflows/generate-matrix.yml) | Works out what a commit asked for and prunes the setup table down to it. |
| [`run-vm.yml`](.github/workflows/run-vm.yml) | Runs a setup inside a `vmactions` VM. Every OS that `vmactions` publishes a VM for is wired up; see [VM families](#vm-families). |
| [`run-direct.yml`](.github/workflows/run-direct.yml) | Runs a setup directly on a GitHub-hosted runner: Linux, macOS, Windows. |
| [`results-summary.yml`](.github/workflows/results-summary.yml) | Collects every result artifact, renders the tables, emits `status.json`. |

`generate-matrix.yml`, `run-vm.yml` and `run-direct.yml` are all
`workflow_call`, so another repository can use them directly. See
[Using the reusable workflows from another repository](#using-the-reusable-workflows-from-another-repository).

---

## Contents

- [How a run works](#how-a-run-works)
- [Selecting what runs](#selecting-what-runs)
  - [`[runner:...]`](#runner)
  - [`[test:...]`](#test)
  - [`[feed:...]`](#feed)
  - [Manual runs](#manual-runs)
  - [How a selection is applied](#how-a-selection-is-applied)
  - [Edge cases](#edge-cases)
- [The setups table](#the-setups-table)
  - [Entry fields](#entry-fields)
  - [Recipes](#recipes)
  - [Adding an OS family](#adding-an-os-family)
- [Per-job mechanics](#per-job-mechanics)
- [Results and the status page](#results-and-the-status-page)
  - [Machine-readable feeds](#machine-readable-feeds)
- [Reference tables](#reference-tables)
- [Notes and caveats](#notes-and-caveats)

---

## How a run works

```
   push / pull_request / schedule / workflow_dispatch
                    |
                    v
        +---------------------------------------+
        |  setup  (generate-matrix.yml)        |
        |                                       |
        |  parse the commit tags                |  runner / test / feed
        |  prune the built-in setup table      |  drop setups that were not
        |                                       |  selected (jq)
        |                                       |
        |  runner    " freebsd … "              |  which families were asked for
        |  ready     " freebsd linux "          |  which of those have work left
        |  matrices  {"linux": [ … ]}          |  the surviving setups per family
        +---------------------------------------+
                    |                    |
               family gate          family gate
            if: contains(…)       if: contains(…)
                    |                    |
      freebsd ───────┘                    └──── linux ── macos ── windows
      openbsd                              (run-vm)   (run-direct)
      netbsd  …                                     │
      dragonflybsd                                  v
      solaris                            run  (one job per setup,
      omnios                              matrix = the pruned entries)
      (run-vm)                                     │
                                                  v
                            result-artifact-<setup>.json
                            result-artifact-<setup>.result
                                                  │
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

All three tags are plain text in the commit message. For a pull request, the
**PR title** is used, since the head commit's message is not available to
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

<a name="feed"></a>
### `[feed:...]`

Asks for machine-readable copies of the results, in addition to the tables:

```
Fix the ARM VM boot hang [runner:linux#arm] [feed:json,rss]
```

| Tag | Result |
|:--|:--|
| `[feed:json]` | `feed.json` only. |
| `[feed:rss]` | `feed.xml` only. |
| `[feed:json,rss]` | Both. |
| *(absent)* | Neither. Feeds are opt-in, so a default run uploads no extra artifact. |

Unknown format names are warned about and dropped, so a typo cannot invent a
filename. Unlike the other two tags this one does not change *which* setups run —
it only changes what gets published at the end. See
[Machine-readable feeds](#machine-readable-feeds) for the formats.

### Manual runs

`workflow_dispatch` exposes the same three selectors as inputs, and they take
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
      feed:
        default: ''
        type: string
```

Dispatching with `runner=openbsd#riscv,windows` and
`test_files=t/3000_jenny/*.t` behaves exactly like a commit carrying
`[runner:openbsd#riscv,windows] [test:t/3000_jenny/*.t]`.

### How a selection is applied

All of it happens in the one `setup` job inside
[`generate-matrix.yml`](.github/workflows/generate-matrix.yml), so a filtered run
never starts work it does not need. It has five outputs:

| Output | Example | Meaning |
|:--|:--|:--|
| `runner` | `" linux macos "` | The families the tag *asked for*. Reported as *not selected*. |
| `ready` | `" linux "` | Those families that still have at least one setup to run. |
| `matrices` | `{"linux":[…],"macos":[]}` | The surviving setups, per family. |
| `test_files` | `t/3000_jenny/*.t` | The test pattern, already filtered to glob characters. |
| `feed` | `json,rss` | Which feeds to emit, if any. |

The distinction between `runner` and `ready` is what makes the report honest:
`runner` remembers what was asked for even when nothing survives to run, so a
typo cannot quietly read as success.

1. **Family gate** — every family job tests `ready` with `contains()`:

   ```yaml
   freebsd:
     needs: setup
     if: "${{ contains(needs.setup.outputs.ready, ' freebsd ') }}"
   ```

   The padding is what makes this a whole-token test: without it, `contains()`
   would match any token that merely *contained* the family name, and a
   misspelled or future token could silently enable the wrong job. `' all '` is
   not special-cased either — `all` expands to the full list, so every gate
   below is written the same way.

2. **Matrix prune** — the same step reduces the built-in table to the entries
   that survive the selection, and emits them per family:

   ```yaml
   linux:
     needs: setup
     if: "${{ contains(needs.setup.outputs.ready, ' linux ') }}"
     uses: ./.github/workflows/run-direct.yml
     with:
       matrix: ${{ toJSON(fromJSON(needs.setup.outputs.matrices).linux) }}
   ```

   The reusable workflow's matrix is `include: ${{ fromJSON(inputs.matrix) }}`,
   so it starts exactly the jobs `setup` left behind. Nothing is filtered
   downstream, and a family with no surviving setups is dropped from `ready`, so
   its job is never even scheduled.

Because pruning happens before dispatch, a selection that matches no setup at all
(`[runner:dragonflybsd#arm]`, where DragonFly BSD only builds `x86_64`) simply
does not start that family, and the report shows `:x: no results reported`.

### Edge cases

| Tag | Result |
|:--|:--|
| *(absent)* | Everything runs. |
| `[runner:all]` | Everything runs. |
| `[runner:]` | Everything runs. |
| `[runner:plan9]` | Two warnings, and **everything runs** — an unknown family is treated as a typo, not as "run nothing". |
| `[runner:linux#arm,linux#arm]` | One `aarch64` filter; the family appears once. |
| `[runner:linux#sparc]` | Two warnings, and **nothing runs**. An unknown arch becomes a sentinel that matches no setup, so a typo fails closed instead of quietly running every Linux setup. |
| `[runner:dragonflybsd#arm]` | The family is selected, but no setup in its matrix is `aarch64`, so the job is never started and the report shows `:x: no results reported`. |

A tag is recognised wherever it appears in the commit message, not just on a
line of its own. A message that *writes about* tags is therefore read as though
it *meant* them:

```text
Docs: explain the [runner:linux] tag and the [feed:json] output
```

That run logs `ignoring unknown runner 'linux'` (if `linux` is not in the
caller's table) or silently enables the JSON feed, and then the commit itself
trips the very tags it was describing. The fallbacks are safe — an unknown
family still runs everything — but the warnings are real, so it is worth
rewording a commit that mentions a tag rather than trying to escape it. There
is no quoting that suppresses a tag, because the message is matched as plain
text.

---

## The setups table

Every setup is declared once, as the default value of the `table` input on
[`generate-matrix.yml`](.github/workflows/generate-matrix.yml), keyed by family.
There is no per-family matrix anywhere else: `setup` prunes this table and hands
each family its own slice.

```yaml
      table:
        default: >
          {"freebsd":[{"platform_name":"freebsd","os":"freebsd","arch":"x86_64",…}],
          "linux":[…],…}
```

The default is a JSON object in a YAML block scalar, one family per line, so a
change to one family shows up as a one-line diff. It is a *default*, not a
constant: any caller can override it wholesale via the `table` input.

A family job receives its slice and passes it straight through:

```yaml
  freebsd:
    name: FreeBSD
    needs: setup
    if: "${{ contains(needs.setup.outputs.ready, ' freebsd ') }}"
    uses: ./.github/workflows/run-vm.yml
    secrets: inherit
    with:
      os_version: '15.0'
      task: 'perl -V'
      matrix: ${{ toJSON(fromJSON(needs.setup.outputs.matrices).freebsd) }}
      test_files: "${{ needs.setup.outputs.test_files }}"
```

The run workflows are therefore family-agnostic: they take a JSON array of
setups and start one job per entry. They do not know what a `[runner:…]` tag
means, and a family that is absent from the table is absent from the run.

### Using the reusable workflows from another repository

There are two things you can borrow, and the difference matters.

**`generate-matrix.yml` — the whole selection layer.** Call it and you get
`[runner:linux#arm]` filtering, with **nothing to copy and no file to add to your
repo**. The table travels with the workflow, so the `uses:` ref pins the logic
and the data together:

```yaml
  # in another repository's workflow
  jobs:
    setup:
      uses: your-org/workflow-testing/.github/workflows/generate-matrix.yml@v1
      with:
        runner: 'linux#arm,windows'   # or leave empty to read the commit tag

    linux:
      needs: setup
      if: "${{ contains(needs.setup.outputs.ready, ' linux ') }}"
      uses: ./.github/workflows/run-direct.yml
      secrets: inherit
      with:
        matrix: ${{ toJSON(fromJSON(needs.setup.outputs.matrices).linux) }}
```

| Input | Default | Meaning |
|:--|:--|:--|
| `runner` | `''` | Overrides the `[runner:…]` tag. Empty means "use the tag". |
| `test_files` | `''` | Overrides the `[test:…]` tag. |
| `feed` | `''` | Overrides the `[feed:…]` tag. |
| `table` | the built-in table | Your own setups, as a JSON object. Pass it to run a completely different portfolio. |

Outputs are `runner`, `ready`, `matrices`, `test_files` and `feed`, as described
in [How a selection is applied](#how-a-selection-is-applied).

**`run-vm.yml` / `run-direct.yml` — the executors.** These are
`workflow_call`-only and deliberately dumb: they take a `matrix` of setups and
start one job per entry, understanding no tags and reading no table. Use these
when you just want a VM or a hosted runner and already know what to run:

```yaml
  perl-smoke:
    uses: your-org/workflow-testing/.github/workflows/run-vm.yml@main
    secrets: inherit
    with:
      os_version: '15.0'
      task: 'perl -V'
      matrix: >
        [
          {"platform_name":"freebsd","os":"freebsd","arch":"x86_64","toolchain":"gcc","prepare":"pkg install -y perl5"}
        ]
```

Whichever you pick, nothing in this repo has to be added to yours.

### A complete downstream example

[`examples/downstream-ci.yml`](examples/downstream-ci.yml) is a whole `ci.yml`
for a caller with its own portfolio: eleven families and 22 setups, two of the
families (`haiku` and `wasm`) absent from the built-in table. It is the shape
everything above adds up to, and it is linted and executed by the same checks as
the real workflows, so it cannot drift from them unnoticed.

The whole file comes down to four things.

**1. One `setup` job, and the caller's own table.** Everything about selection
lives in the reusable workflow; the table travels into it as a literal block
scalar:

```yaml
  setup:
    name: Generate Testing Matrix
    uses: your-org/workflow-testing/.github/workflows/generate-matrix.yml@v1
    with:
      runner: '${{ github.event.inputs.runner }}'   # empty means "use the commit tag"
      table: |
        {
          "omnios": [
            {"platform_name":"omnios","os":"omnios","arch":"x86_64","toolchain":"gcc",
             "configure":"-A'ccflags=-D__EXTENSIONS__ -fno-lto'"}
          ],
          "haiku": [
            {"name":"Intel","platform_name":"haiku","os":"haiku","arch":"x86_64","toolchain":"gcc"}
          ]
        }
```

The families you can tag are the keys of *your* table, not a list fixed inside
the reusable workflow. That is what makes `haiku` selectable above even though
no built-in table has it. One family per line, each with a trailing comma.

Write the table as a literal. The apostrophes in that `configure` are ordinary
characters in a YAML file, which is exactly why they are safe there — and
exactly why they must not be produced by an expression. Interpolating a table
into the middle of a YAML document puts untrusted text where the parser is
reading, and a single quote ends the file early. The reusable workflow applies
`toJSON()` to the values it receives for the same reason.

**2. One job per family, gated on `ready`.** The gate is the whole trick: a
family pruned to nothing is absent from `ready`, so its job never starts and no
reusable workflow is ever called with an empty matrix.

```yaml
  omnios:
    name: OmniOS
    needs: setup
    if: "${{ contains(needs.setup.outputs.ready, ' omnios ') }}"
    uses: your-org/workflow-testing/.github/workflows/run-vm.yml@v1
    secrets: inherit
    with:
      os_version: 'r151050'
      task: 'perl -V'
      matrix: ${{ toJSON(fromJSON(needs.setup.outputs.matrices).omnios) }}
      test_files: "${{ needs.setup.outputs.test_files }}"
```

A family uses `run-vm.yml` to boot an image (`os_version` required) or
`run-direct.yml` to take a hosted runner (no `os_version`; a wasm build, for
instance, is a Linux direct job). Nothing else about the two differs.

**3. A `results` job that depends on `setup` and every family.** It needs
`setup` in `needs:` even though it wants no result from it, because
`runner`, `test_files` and `feed` are read from `needs.setup.outputs`. A job
that reads `needs.X` without depending on `X` is a validation error. Depending
on the families is what lets the report say "not selected by runner filter"
rather than reading an unstarted family as a silent failure:

```yaml
  results:
    name: Results
    if: always()
    needs: [setup, dragonflybsd, freebsd, haiku, linux, macos,
            netbsd, omnios, openbsd, solaris, wasm, windows]
    uses: your-org/workflow-testing/.github/workflows/results-summary.yml@v1
    with:
      title: My Project CI Matrix
      runner: "${{ needs.setup.outputs.runner }}"
      test_files: "${{ needs.setup.outputs.test_files }}"
      feed: "${{ needs.setup.outputs.feed }}"
```

**4. Nothing else.** There is no tag parsing, no `jq` and no selection logic in
the caller's file, so there is no second copy of it to fall out of step.

The second option is the older design and still works, but it costs more than it
looks. Selection used to happen in a `select` job *inside* each reusable
workflow, so a caller passed only `select:` and got pruning for free — at the
price of one throwaway runner per family and nine `Family / Select setups` rows
in every run's job list. Moving selection into a single shared `setup` job
removed all of that, and is what makes the first option possible at all.

Beyond the table fields, two inputs matter to a caller:

| Input | Meaning |
|:--|:--|
| `checkout_submodules` | `''` (the default) fetches no submodules; `recursive` fetches nested ones too. Only meaningful when a setup's `prepare` or `task` needs the source tree. |
| `continue_on_error` | `run-vm` only. Report failed setups without failing the caller's run, so experimental legs stay visible while the caller reads green. Defaults off. `run-direct` has no equivalent: a failure there is a real failure. |

### Families this repo does not run by default

Several families have a VM step and a `case` arm in *Collect environment info*,
so a caller naming them gets a real run: Haiku, MidnightBSD and the Linux and
alternative-UNIX guests listed under [VM families](#vm-families). None of them is
in the built-in setup table or has a job in `ci.yml`, so none appears in this
repo's own runs. Adding the three pieces from *Adding an OS family* — a table
key, a job and an `order` entry — is what turns one on.

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
| `test_files` | optional | optional | Test pattern from the `[test:...]` tag. |

`matrix` is always the *already-pruned* slice of **this repo's** setup table when
called from `ci.yml`; a direct caller supplies whatever it likes. See
[How a selection is applied](#how-a-selection-is-applied) and
[Using the reusable workflows from another repository](#using-the-reusable-workflows-from-another-repository).

<a name="recipes"></a>
### Recipes

**Build NetBSD on x86_64 with clang** — the family already exists, so this is a
one-line addition to the `netbsd` list in the `table` input:

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
the family:

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

1. **`generate-matrix.yml`** — add a line to the `table` default. The table is a
   JSON object *keyed by family*, so the new family is one more `"name": [...]`
   member on a line of its own, sitting between the existing ones:

   ```
   "midnightbsd":[{"platform_name":"midnightbsd","os":"midnightbsd","arch":"x86_64","toolchain":"gcc","prepare":"sudo mport install perl gcc"}],
   ```

   Three things about that line are easy to get wrong: the family name is a
   **quoted key**, not a nested object; it ends with a **comma** like every
   family line but the last; and it is **one line**, because the whole table is
   a folded YAML scalar and a newline inside a family would be folded into a
   space and break the JSON.

   This is the family’s whole definition. `setup` prunes from this key, so a
   typo in the key name silently runs nothing — the key must match the gate in
   step 2 exactly.

2. **`ci.yml`** — add a job that calls the reusable workflow, gated like the
   others. Nothing in `generate-matrix.yml` has to change: the family list is
   derived from the table's own keys, so the entry from step 1 is already what
   makes the family selectable.

   ```yaml
     midnightbsd:
       name: MidnightBSD
       needs: setup
       if: "${{ contains(needs.setup.outputs.ready, ' midnightbsd ') }}"
       uses: ./.github/workflows/run-vm.yml
       secrets: inherit
       with:
         os_version: '4.0.4'
         task: 'perl -V'
         matrix: ${{ toJSON(fromJSON(needs.setup.outputs.matrices).midnightbsd) }}
         test_files: "${{ needs.setup.outputs.test_files }}"
   ```

   And add it to the `results` job's `needs:` list.

   If you would rather see the family ordered among the built-ins in the emitted
   `runner` list than appended after them, add the key to `KNOWN` in
   `generate-matrix.yml`. That is optional and affects nothing else. The order
   families appear in the report comes from `results-summary.yml` in step 4.

3. **`run-vm.yml`** — add a VM step for it, gated on the `os` value, and a `case`
   arm in the *Collect environment info* step so the report names it correctly:

   ```bash
     midnightbsd)  family='MidnightBSD' ;;
   ```

   The `os` value in the table entry is what selects the step
   (`if: matrix.os == 'midnightbsd'`); an entry with an `os` no step handles
   silently builds nothing. The arm quoted above is a real one, MidnightBSD
   being wired up; copy the shape for one that is not.

4. **`results-summary.yml`** — add the display name to `order` so it is not
   appended alphabetically at the end:

   ```bash
   order=("Linux" "macOS" "Windows" "FreeBSD" "OpenBSD" "NetBSD" "DragonFly BSD" "Solaris" "OmniOS" "MidnightBSD")
   ```

   Two things to know about this list. It also decides who is reported as *not
   selected*, so add the name only once the `ci.yml` job from step 2 exists,
   otherwise every full run grows a misleading row. And the name has to match the
   table key apart from punctuation and spacing, because that is all `selected()`
   normalises: "DragonFly BSD" and "Rocky Linux" are fine, "GNU/Hurd" for the
   `hurd` key is not.

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

<a name="feeds"></a>
### Machine-readable feeds

The status page is a page for humans. To consume the same results from a script,
a bot, or an RSS reader, ask for a feed with the
[`[feed:…]` tag](#feed) or the `feed` dispatch input.

`feed.json` is one object per run:

```json
{
  "schema": "workflow-testing/feed/1",
  "generated": "2026-09-29T05:42:11Z",
  "run": { "id": "1234", "url": "https://github.com/…/actions/runs/1234", "event": "push", "ref": "main", "sha": "…", "title": "…" },
  "selection": { "runner": " linux ", "test_files": null },
  "results": [
    { "family": "Linux", "setup": "ARM", "id": "1a2b3c4d", "arch": "aarch64",
      "toolchain": "gcc", "perl": null, "variant": null, "platform": "linux",
      "status": "👍🏽 passing", "ok": true }
  ]
}
```

Fields that do not apply are `null` rather than `"-"`, so a consumer can tell
"not applicable" from "empty string". `ok` is a tri-state: `true`/`false` where
a setup reported, `null` where it did not — either because the family was
filtered out (`status: "skipped"`) or because it was selected but silent
(`status: "no_results"`). The string `status` is always present and is the field
to branch on if the tri-state is inconvenient.

`feed.xml` is an RSS 2.0 channel over the same records, with a stable `guid` per
setup fingerprint. Both files are written by `results-summary` and uploaded as
the `result-feed-<run id>` artifact with a 30-day retention, alongside — not
instead of — the `status-files` artifact. `status.json` itself is unchanged by
any of this.

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

### VM families

`run-vm.yml` boots a `vmactions` VM for every OS that publishes one, so a setup
table entry can name any of these keys. Only the families in the
[table above](#families) are selected by this repo's own CI; the rest are wired up
and waiting for a caller to ask for them.

| Tag | Report name | Action | Images |
|:--|:--|:--|:--|
| `freebsd` | FreeBSD | `v1.5.2` | x86_64, aarch64, riscv64, powerpc64 |
| `openbsd` | OpenBSD | `v1.4.5` | x86_64, aarch64, riscv64, sparc64 |
| `netbsd` | NetBSD | `v1.4.5` | x86_64, aarch64, riscv64, sparc64 |
| `dragonflybsd` | DragonFly BSD | `v1.3.1` | x86_64 |
| `midnightbsd` | MidnightBSD | `v1.0.7` | x86_64 |
| `solaris` | Solaris | `v1.3.8` | x86_64 |
| `omnios` | OmniOS | `v1.3.4` | x86_64 |
| `haiku` | Haiku | `v1.1.5` | x86_64 |
| `almalinux` | AlmaLinux | `v1.0.1` | x86_64, aarch64, s390x, ppc64le |
| `alpine` | Alpine | `v1.0.2` | x86_64, aarch64, riscv64 |
| `blissos` | BlissOS | `v1.0.3` | x86_64 |
| `debian` | Debian | `v1.0.1` | x86_64, aarch64, riscv64, ppc64le |
| `ghostbsd` | GhostBSD | `v1.0.4` | x86_64 |
| `hardenedbsd` | HardenedBSD | `v1.0.1` | x86_64 |
| `hurd` | Hurd | `v1.0.2` | x86_64, i386 |
| `nextbsd` | NextBSD | `v1.0.2` | x86_64, aarch64 |
| `openeuler` | openEuler | `v1.0.3` | x86_64, aarch64, riscv64, loongarch64 |
| `openindiana` | OpenIndiana | `v1.1.7` | x86_64 |
| `opnsense` | OPNsense | `v1.0.1` | x86_64 |
| `plan9` | Plan 9 | `v1.0.2` | x86_64 |
| `reactos` | ReactOS | `v1.0.5` | i386 |
| `redox` | Redox | `v1.0.2` | x86_64 |
| `riscos` | RISC OS | `v1.0.2` | armv7 |
| `rockylinux` | Rocky Linux | `v1.0.2` | x86_64, aarch64, s390x, ppc64le |
| `tribblix` | Tribblix | `v1.0.6` | x86_64 |
| `ubuntu` | Ubuntu | `v1.0.2` | x86_64, aarch64, riscv64, s390x, ppc64le |

Two of these have no x86_64 image at all, so their steps take no `arch` and boot the
only image that exists: `reactos` is i386 and `riscos` is armv7. Asking for
`[runner:reactos#aarch64]` still boots i386, and the result records the arch that was
asked for, so the Images column above is the one to trust. `s390x` and `ppc64le` are
published by `almalinux`, `debian`, `rockylinux` and `ubuntu`, but `canon_arch` in
`generate-matrix.yml` does not model them yet.

Nothing in this repo's CI selects the families below `haiku`, so a green run here
says the wiring parses, not that those guests work. Expect to fix a `prepare` command
or two before a caller gets a clean first run on the less common ones.

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
- **Unselected architectures are skipped, not failed.** The `setup` job prunes
  the built-in table before dispatch, so a `[runner:linux#arm]` commit never
  starts the x86_64 Linux jobs. This costs one extra runner — `setup` itself,
  which now prunes the table instead of doing nothing but string matching.
- **Untrusted input is filtered twice.** Runner names are matched against a fixed list with `grep -qxF` (fixed-string, never a regex), and the test pattern is reduced to glob characters. Family and arch names are also canonicalised, so nothing from a commit message can reach a shell unfiltered.
