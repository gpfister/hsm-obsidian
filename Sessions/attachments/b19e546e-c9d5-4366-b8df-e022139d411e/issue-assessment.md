# Issue Assessment: BitBake `scdoc-native` do_fetch Failure in `rpi-z2w-yocto-hsm`

## 1. Summary & Observed Behavior

While executing `make build-dev` (or `make yocto-build-dev`) in the [`rpi-z2w-yocto-hsm`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm) repository using Kas and BitBake on Yocto 6.1 (*Blacksail*), the build halted during the `do_fetch` task of `scdoc-native-1.11.5-r0`:

```text
ERROR: scdoc-native-1.11.5-r0 do_fetch: Fetcher failure: Unable to find revision 24be30ef17d5ca0162f11b35258c8fa2413d556a in branch master even from upstream
ERROR: scdoc-native-1.11.5-r0 do_fetch: Bitbake Fetcher Error: FetchError('Unable to fetch URL from any source.', 'git://git.sr.ht/~sircmpwn/scdoc;protocol=https;branch=master;tag=1.11.5')
ERROR: Logfile of failure stored in: /repo/build/tmp/work/aarch64-linux/scdoc-native/1.11.5/temp/log.do_fetch.215667
ERROR: Task (virtual:native:/repo/build/../openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb:do_fetch) failed with exit code '1'
```

`scdoc-native` is required by `rpm` (used for package management and rootfs construction), making this build failure a hard blocker for OS image generation.

---

## 2. Root Cause Analysis

### 2.1 Investigation Findings & Evidence

1. **Upstream Repository & Commit Validity**:
   - Recipe [`openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/openembedded-core/meta/recipes-extended/scdoc/scdoc_1.11.5.bb#L9-L10) specifies:
     ```bitbake
     SRC_URI = "git://git.sr.ht/~sircmpwn/scdoc;protocol=https;branch=master;tag=${PV}"
     SRCREV = "24be30ef17d5ca0162f11b35258c8fa2413d556a"
     ```
   - Inspection of upstream remote repository `https://git.sr.ht/~sircmpwn/scdoc` via `git ls-remote` confirmed that commit `24be30ef17d5ca0162f11b35258c8fa2413d556a` **does exist**, is the exact tip of `refs/heads/master`, and is tagged as `refs/tags/1.11.5`:
     ```text
     24be30ef17d5ca0162f11b35258c8fa2413d556a  HEAD
     24be30ef17d5ca0162f11b35258c8fa2413d556a  refs/heads/master
     1ed7454466a55bd0d50ab547aec109c335b15096  refs/tags/1.11.5
     24be30ef17d5ca0162f11b35258c8fa2413d556a  refs/tags/1.11.5^{}
     ```
   - Likewise, the Yocto upstream mirror at `https://downloads.yoctoproject.org/mirror/sources/git2_git.sr.ht.~sircmpwn.scdoc.tar.gz` is present (HTTP 200) and contains the commit.

2. **BitBake Reference Validation Failure**:
   - BitBake's Git fetcher in [`bitbake/lib/bb/fetch/git.py`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/bitbake/lib/bb/fetch/git.py#L824-L835) verifies revisions using `_contains_ref`:
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
   - Because `branch=master` is defined in `SRC_URI`, BitBake executes:
     ```bash
     git branch --contains 24be30ef17d5ca0162f11b35258c8fa2413d556a --list master
     ```

3. **Corrupted Local Download Cache**:
   - Examination of the local bare Git clone located at [`downloads/git2/git.sr.ht.~sircmpwn.scdoc`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/downloads/git2/git.sr.ht.~sircmpwn.scdoc) revealed that its `HEAD` file contained:
     ```text
     ref: refs/heads/.invalid
     ```
   - `refs/heads/.invalid` is Git's internal temporary placeholder written during repository creation (`builtin/clone.c`) before the remote default branch symref is resolved and written.
   - When an initial clone or network fetch is interrupted (due to a network timeout, container kill, or process abort), Git leaves the bare clone with `HEAD` stuck pointing to `refs/heads/.invalid`.
   - Any subsequent call to `git branch --contains ...` against this repository fails immediately with fatal exit code 128:
     ```text
     fatal: failed to resolve HEAD as a valid ref
     ```
   - BitBake caught this error, interpreted it as ref absence, and raised the deceptive message `"Unable to find revision ... in branch master even from upstream"`.

---

## 2.2 Verification of Fix

The subagent purged the corrupted download cache directory `downloads/git2/git.sr.ht.~sircmpwn.scdoc` and ran a test fetch:
```bash
./scripts/run-kas.sh shell kas-dev.yml -c "bitbake scdoc-native -c fetch"
```
The task succeeded completely (`Attempted 2 tasks ... all succeeded`).

---

## 3. Proposed Solutions

### Solution 1: Download Cache Cleanup & Fresh Fetch (Recommended & Already Applied)
Clearing the corrupted bare clone allowed BitBake to successfully re-fetch the repository.

### Solution 2: Automated Ref Sanity Guard in Kas Runner
Add a defensive cleanup guard into [`rpi-z2w-yocto-hsm/scripts/run-kas.sh`](file:///Users/gpfister/Development/Projects/gpfister/nrf52840-hsm/rpi-z2w-yocto-hsm/scripts/run-kas.sh) before invoking Docker/Kas. The guard scans `downloads/git2/*/HEAD` for `ref: refs/heads/.invalid` and purges any corrupt directories automatically:
```bash
find "${REPO_DIR}/downloads/git2" -name HEAD -exec grep -q "refs/heads/.invalid" {} + 2>/dev/null && \
  find "${REPO_DIR}/downloads/git2" -name HEAD -exec grep -l "refs/heads/.invalid" {} + | sed 's#/HEAD##' | xargs rm -rf
```

---

## 4. Next Steps

Now that the corrupted cache has been cleared and `scdoc-native` fetches cleanly, do you want to:
1. Re-run `make build-dev` (or have the agent invoke `@coding` to apply the sanity guard script and re-trigger the build)?
