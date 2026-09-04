# pnpm-issue-2026-09-03

this repository reproduces an issue with [pnpm 12](https://github.com/pnpm/pnpm/releases/tag/v12.0.0) where `pnpm install` fails with `ERR_PNPM_INVALID_PATCH` when a `patchedDependencies` patch file has CRLF line endings. the same patch file applies fine with pnpm 11.

```
Error: ERR_PNPM_INVALID_PATCH

  × installing dependencies
  ╰─▶ Applying patch ".../patches/is-odd@3.0.1.patch" failed: error parsing patches: error
      parsing patch: invalid char in unquoted filename
```

the patch file in this repo, [`patches/is-odd@3.0.1.patch`](patches/is-odd@3.0.1.patch), is committed with CRLF line endings on purpose (see [`.gitattributes`](.gitattributes)) so the failure reproduces on every platform. in the real world the file is LF in git and only becomes CRLF on a Windows checkout, because Git for Windows defaults to `core.autocrlf=true`. that is how this surfaced: a CI job on `windows-latest`-style runners started failing the moment the repo bumped from pnpm 11 to pnpm 12.

## reproduction steps

1. check out this repo

2. confirm the patch file has CRLF line endings (prints `8`, one per line):

   ```sh
   node -e "const s=require('fs').readFileSync('patches/is-odd@3.0.1.patch','latin1');console.log((s.match(/\r\n/g)||[]).length)"
   ```

3. install the dependencies with pnpm 12 using an empty store and note how it fails with `ERR_PNPM_INVALID_PATCH`:

   ```sh
   pnpm install --no-frozen-lockfile --store-dir .pnpm-store-12
   ```

   > expected behavior is that the install succeeds and `node_modules/is-odd/index.js` contains the line `* patched by pnpm-issue-2026-09-03`

4. install the same thing with pnpm 11 and note how it succeeds and the patch is applied:

   ```sh
   npx --yes pnpm@11.25.0 install --no-frozen-lockfile --store-dir .pnpm-store-11
   grep 'patched by' node_modules/is-odd/index.js
   ```

5. convert the patch file to LF, wipe `node_modules`, and repeat step 3. pnpm 12 now succeeds:

   ```sh
   perl -pi -e 's/\r\n/\n/' patches/is-odd@3.0.1.patch
   rm -rf node_modules
   pnpm install --no-frozen-lockfile --store-dir .pnpm-store-12
   grep 'patched by' node_modules/is-odd/index.js
   ```

6. convert the patch file back to CRLF, wipe `node_modules`, and repeat step 3 against the **same** store as step 5. pnpm 12 now succeeds even with the CRLF file, because the store already holds a patched copy and the patch is never re-parsed:

   ```sh
   perl -pi -e 's/\n/\r\n/' patches/is-odd@3.0.1.patch
   rm -rf node_modules
   pnpm install --no-frozen-lockfile --store-dir .pnpm-store-12
   grep 'patched by' node_modules/is-odd/index.js
   ```

   > this is why `--store-dir` pointing at an empty directory is used above: a warm store hides the bug, so the failure typically only shows up on fresh CI machines

the [`pnpm install` workflow](.github/workflows/test.yml) in this repo runs steps 1 to 3 (plus the step 4 success check) for pnpm 11.25.0 and 12.3.1 on ubuntu, windows, and macos. the pnpm 12 jobs fail on all three; the pnpm 11 jobs pass on all three.

## why this happens

- pnpm 12's Rust CLI parses patch files with the [`diffy`](https://github.com/bmwill/diffy) crate ([`crates/patching/src/apply.rs`](https://github.com/pnpm/pnpm/blob/v12.3.1/pnpm/crates/patching/src/apply.rs)). the pinned `diffy` 0.5.1 splits the `---` and `+++` header lines at `\n` only, so with CRLF endings the `\r` stays attached to the filename and the parser rejects it as an "invalid char in unquoted filename".
- pnpm 11's JavaScript patch parser tolerates the `\r`.
- pnpm 12 already normalizes CRLF to LF when *hashing* a patch file ([`crates/patching/src/hash.rs`](https://github.com/pnpm/pnpm/blob/v12.3.1/pnpm/crates/patching/src/hash.rs)), so an LF and a CRLF copy of the same patch hash identically. that is why step 6 above passes: the store lookup hits the entry produced from the LF copy and the CRLF file is never parsed.
- `diffy` fixed this in [0.5.2](https://github.com/bmwill/diffy/pull/87) (2026-08-31). as of pnpm 12.3.1 the `Cargo.lock` still pins 0.5.1.

## about this reproduction environment

```
  System:
    OS: macOS 26.6.2
    CPU: arm64
    Shell: zsh
  Binaries:
    Node: 24.20.0
    git: 2.50.1 (Apple Git-155)
    pnpm: 12.3.1 (fails), 11.25.0 and 11.4.0 (both pass)
```

the patch file was hand-written against `is-odd@3.0.1` and only adds one comment line, so it applies cleanly whenever it parses.
