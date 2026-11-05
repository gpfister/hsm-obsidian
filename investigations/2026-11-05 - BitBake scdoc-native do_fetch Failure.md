---
title: "Issue Assessment: BitBake scdoc-native do_fetch Failure in rpi-z2w-yocto-hsm"
tags:
  - investigation
  - yocto
  - bitbake
  - git
  - cache
  - kas
  - rpm
created: 2026-11-05
status: resolved
severity: high
investigation_id: b19e546e-c9d5-4366-b8df-e022139d411e
source_artifact: "[[Sessions/attachments/b19e546e-c9d5-4366-b8df-e022139d411e/issue-assessment.md|Sessions/attachments/b19e546e-c9d5-4366-b8df-e022139d411e/issue-assessment.md]]"
affected_components:
  - rpi-z2w-yocto-hsm
related_guides:
  - "[[Development/Yocto Build and RAUC Update Guide|Yocto Build and RAUC Update Guide]]"
related_architecture:
  - "[[Architecture/07 - Air-Gapped RPi HSM OS Architecture|Air-Gapped RPi HSM OS Architecture]]"
---

# Issue Assessment: BitBake `scdoc-native` `do_fetch` Failure in `rpi-z2w-yocto-hsm`

> [!NOTE]
> **Investigation Source**: [`issue-assessment.md`](file:///Users/gpfister/.gemini/antigravity/brain/b19e546e-c9d5-4366-b8df-e022139d411e/issue-assessment.md)  
> **Archived Artifact**: [[Sessions/attachments/b19e546e-c9d5-4366-b8df-e022139d411e/issue-assessment.md|Attachments]]  
> **Investigation Session**: `b19e546e-c9d5-4366-b8df-e022139d411e`  
> **Status**: Resolved / Purged & Prevented  
> **Severity**: High (Blocks RPM host tooling & complete OS image build)  
> **Affected Component**: [`rpi-z2w-yocto-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm)  
> **Architecture Linkages**: [[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]], [[Development/Yocto Build and RAUC Update Guide|Yocto Build and RAUC Update Guide]]

---

## 1. Summary & Observed Behavior

While executing `make build-dev` (or root `make yocto-build-dev`) in the [`rpi-z2w-yocto-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) repository using Kas and BitBake on Yocto 6.1 (*Blacksail*), the build halted during the `do_fetch` task of `scdoc-native-1.11.5-r0`:

```text
ERROR: scdoc-native-1.11.5-r0 do_fetch: Fetcher failure: Unable to find revision 24be30ef17d5ca0162f11b35258c8fa2413d556a in branch master even from upstream
ERROR: scdoc-native-1.11.5-r0 do_fetch: Bitbake Fetcher Error: FetchError('Unable to fetch URL from any source.', 'git://git.sr.ht/~sircmpwn/scdoc;protocol=https;branch=master;tag=1.11.5')
ERROR: Logfile of failure stored in: /repo/build/tmp/work/aarch64-linux/scdoc-native/1.11.5/temp/log.do_fetch.215667
ERROR: Task (virtual:native:/repo/build/../openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb:do_fetch) failed with exit code '1'
```

### Impact
`scdoc-native` is a man-page generator required by `rpm-native` (which BitBake relies on for package feed management, dependency resolution, and rootfs composition). This failure was a hard blocker for constructing both development (`hsm-image-dev`) and production (`hsm-image-prod`) OS images.

---

## 2. Failure Analysis & Resolution Architecture

```mermaid
flowchart TD
    subgraph BuildFailure["1. Build Failure Symptom"]
        A["make yocto-build-dev / make build-dev"] --> B["BitBake executes do_fetch for scdoc-native-1.11.5"]
        B --> C["FetchError: Unable to find revision 24be30e... in branch master"]
        C --> D["Build Halted: rpm & rootfs generation blocked"]
    end

    subgraph RootCause["2. Root Cause: Corrupted Local Git Cache"]
        E["Interrupted Initial Clone / Abrupt Container Kill"] --> F["Git leaves bare clone in downloads/git2/git.sr.ht.~sircmpwn.scdoc"]
        F --> G["HEAD contains 'ref: refs/heads/.invalid' (Git clone interim state)"]
        G --> H["BitBake runs: git branch --contains 24be30e... --list master"]
        H --> I["Git exits 128: 'fatal: failed to resolve HEAD as a valid ref'"]
        I -->|Interpreted by BitBake as missing ref| C
    end

    subgraph Resolution["3. Resolution & Automated Prevention"]
        J["Immediate Fix: Purge downloads/git2/git.sr.ht.~sircmpwn.scdoc"] --> K["Re-fetch succeeds cleanly"]
        L["Automated Guard in scripts/run-kas.sh"] --> M["Scan downloads/git2/*/HEAD for 'refs/heads/.invalid'"]
        M --> N["Auto-purge corrupted bare clone directories before container launch"]
        N --> O["Resilient build pipeline prevents future do_fetch halts"]
    end

    BuildFailure -.-> RootCause
    RootCause -.-> Resolution

    classDef fail fill:#ffebee,stroke:#c62828,stroke-width:2px;
    classDef cause fill:#fff3e0,stroke:#ef6c00,stroke-width:2px;
    classDef fix fill:#e8f5e9,stroke:#2e7d32,stroke-width:2px;

    class C,D fail;
    class E,F,G,H,I cause;
    class J,K,L,M,N,O fix;
```

---

## 3. Deep Root Cause Analysis

### 3.1 Upstream Repository & Commit Verification
The recipe [`openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb#L9-L10) specifies:
```bitbake
SRC_URI = "git://git.sr.ht/~sircmpwn/scdoc;protocol=https;branch=master;tag=${PV}"
SRCREV = "24be30ef17d5ca0162f11b35258c8fa2413d556a"
```

Direct inspection of upstream remote repository `https://git.sr.ht/~sircmpwn/scdoc` via `git ls-remote` proved that commit `24be30ef17d5ca0162f11b35258c8fa2413d556a` **does exist**, is the exact tip of `refs/heads/master`, and is tagged as `refs/tags/1.11.5`:
```text
24be30ef17d5ca0162f11b35258c8fa2413d556a  HEAD
24be30ef17d5ca0162f11b35258c8fa2413d556a  refs/heads/master
1ed7454466a55bd0d50ab547aec109c335b15096  refs/tags/1.11.5
24be30ef17d5ca0162f11b35258c8fa2413d556a  refs/tags/1.11.5^{}
```
Furthermore, the Yocto Project upstream source mirror at `https://downloads.yoctoproject.org/mirror/sources/git2_git.sr.ht.~sircmpwn.scdoc.tar.gz` returned HTTP 200 and contained the identical commit.

### 3.2 BitBake Git Fetcher Mechanism
BitBake's Git fetcher in [`bitbake/lib/bb/fetch/git.py`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/bitbake/lib/bb/fetch/git.py#L824-L835) verifies target revisions using the `_contains_ref` helper:

```python
def _contains_ref(self, ud, d, name, wd, tag=False):
    git_ref_name = 'refs/tags/%s' % ud.parm['tag'] if tag else ud.revision

    if ud.nobranch:
        cmd = ud.basecmd + ['log', '--pretty=oneline', '-n', '1', git_ref_name, "--"]
    else:
        cmd = ud.basecmd + ['branch', '--contains', git_ref_name, '--list', ud.branch]
    try:
        output = runfetchcmd(cmd, d, workdir=wd)
    except (bb.fetch.FetchError):
        return False
    return len(output.splitlines()) > 0
```

Because `branch=master` was defined in `SRC_URI`, BitBake invoked:
```bash
git branch --contains 24be30ef17d5ca0162f11b35258c8fa2413d556a --list master
```

### 3.3 Corrupted Local Download Cache (`refs/heads/.invalid`)
Inspection of the bare Git clone in [`downloads/git2/git.sr.ht.~sircmpwn.scdoc`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/downloads/git2/git.sr.ht.~sircmpwn.scdoc) revealed that its `HEAD` file contained:
```text
ref: refs/heads/.invalid
```

#### Git Internal Mechanism
In Git's clone implementation (`builtin/clone.c`), `ref: refs/heads/.invalid` is initialized as a temporary placeholder symref when creating a repository before the remote default branch symref is resolved and written.

When an initial clone or network fetch is interrupted (due to a transient network timeout, build container abortion, or SIGINT), Git leaves the bare clone with `HEAD` stuck pointing to `refs/heads/.invalid`.

#### Error Cascading
When BitBake subsequent invocations run `git branch --contains ...` inside this bare clone, Git fails immediately with fatal exit code `128`:
```text
fatal: failed to resolve HEAD as a valid ref
```
BitBake caught this `FetchError`, swallowed the underlying `failed to resolve HEAD` diagnostic, and produced the misleading high-level error:
`"Unable to find revision 24be30ef17d5ca0162f11b35258c8fa2413d556a in branch master even from upstream"`.

---

## 4. Verification & Validation

The corrupted download cache directory was removed:
```bash
rm -rf downloads/git2/git.sr.ht.~sircmpwn.scdoc
```

A standalone verification fetch was executed via Kas:
```bash
./scripts/run-kas.sh shell kas-dev.yml -c "bitbake scdoc-native -c fetch"
```

### Output:
```text
NOTE: Executing Tasks
NOTE: Tasks Summary: Attempted 2 tasks of which 0 didn't need to be rerun and all succeeded.
```
The repository clone re-initialized cleanly, the symref resolved to `refs/heads/master`, and the commit was validated without error.

---

## 5. Remediation & Automated Prevention Guard

### 5.1 Immediate Remediation
Purging the corrupt bare clone in `downloads/git2/git.sr.ht.~sircmpwn.scdoc` immediately resolved the build failure.

### 5.2 Automated Sanity Guard in Kas Runner
To prevent aborted fetches or network hiccups from permanently wedging future local or CI builds, an automated pre-flight sanity check should be integrated into [`rpi-z2w-yocto-hsm/scripts/run-kas.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/scripts/run-kas.sh#L23-L25):

```bash
# Clean up any interrupted bare clones left with invalid HEAD references
if [ -d "${REPO_DIR}/downloads/git2" ]; then
    corrupt_repos=$(find "${REPO_DIR}/downloads/git2" -name HEAD -exec grep -l "refs/heads/\.invalid" {} + 2>/dev/null | sed 's#/HEAD$##' || true)
    if [ -n "${corrupt_repos}" ]; then
        echo "==> [WARN] Detected corrupt Git bare clones with .invalid HEAD. Purging:"
        echo "${corrupt_repos}" | while IFS= read -r dir; do
            echo "    -> ${dir}"
            rm -rf "${dir}"
        done
    fi
fi
```

This ensures that any interrupted clone is automatically cleared before BitBake attempts to read from the local cache.

---

## 6. Related Resources & Links

- **Assessment Artifact**: [`issue-assessment.md`](file:///Users/gpfister/.gemini/antigravity/brain/b19e546e-c9d5-4366-b8df-e022139d411e/issue-assessment.md)
- **Vault Archive**: [[Sessions/attachments/b19e546e-c9d5-4366-b8df-e022139d411e/issue-assessment.md|Archived issue-assessment.md]]
- **Affected Recipe**: [`openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb)
- **Kas Execution Wrapper**: [`rpi-z2w-yocto-hsm/scripts/run-kas.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/scripts/run-kas.sh)
- **Yocto Architecture**: [[Architecture/07 - Air-Gapped RPi HSM OS Architecture|07 - Air-Gapped RPi HSM OS Architecture]]
- **Yocto Build Guide**: [[Development/Yocto Build and RAUC Update Guide|Yocto Build and RAUC Update Guide]]
