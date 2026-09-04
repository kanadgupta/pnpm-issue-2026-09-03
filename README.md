# pnpm-issue-2026-09-03

this repository reproduces an issue (https://github.com/pnpm/pnpm/issues/14557) with [pnpm 12](https://github.com/pnpm/pnpm/releases/tag/v12.0.0) where `pnpm install` fails with `ERR_PNPM_INVALID_PATCH` when a `patchedDependencies` patch file has CRLF line endings. pnpm 11 applies the same patch without complaint.

```
Error: ERR_PNPM_INVALID_PATCH

  × installing dependencies
  ╰─▶ Applying patch ".../patches/is-odd@3.0.1.patch" failed: error parsing patches: error
      parsing patch: invalid char in unquoted filename
```

[`lf/`](lf) and [`crlf/`](crlf) are identical single-dependency projects. the only difference is the line endings of their patch file, which [`.gitattributes`](.gitattributes) stops git from normalizing:

| directory | patch file | pnpm 11.25.0 | pnpm 12.3.4 |
| --------- | ---------- | ------------ | ----------- |
| `lf/`     | LF         | applies      | applies     |
| `crlf/`   | CRLF       | applies      | **`ERR_PNPM_INVALID_PATCH`** |

## reproduction steps

1. check out this repo

2. install `crlf/` with pnpm 12 and note how it fails with `ERR_PNPM_INVALID_PATCH`:

   ```sh
   cd crlf && npx --yes pnpm@12.3.4 install
   ```

   > expected behavior is that the install succeeds and `node_modules/is-odd/index.js` contains the line `* patched by pnpm-issue-2026-09-03`, which is what happens for every other cell of the table above:
   >
   > ```sh
   > npx --yes pnpm@11.25.0 install                       # crlf/ with pnpm 11: applies
   > cd ../lf && npx --yes pnpm@12.3.4 install            # lf/ with pnpm 12: applies
   > ```

the [`pnpm install` workflow](.github/workflows/test.yml) runs one job per cell of the table above on ubuntu, windows, and macos (12 jobs). the only failures are the three `pnpm 12.3.4 … for crlf` jobs.

## why this happens

- pnpm 12 parses patch files with the [`diffy`](https://github.com/bmwill/diffy) crate ([`crates/patching/src/apply.rs`](https://github.com/pnpm/pnpm/blob/v12.3.4/pnpm/crates/patching/src/apply.rs)). `Cargo.lock` resolves it to 0.5.1, whose `parse_filename` splits the `---`/`+++` header lines at `\n` only, so with CRLF endings the `\r` stays attached to the filename and is rejected as an "invalid char in unquoted filename". pnpm 11's JavaScript parser tolerates the `\r`.
- `diffy` [0.5.2](https://github.com/bmwill/diffy/blob/0.5.2/CHANGELOG.md) (2026-08-31) strips the `\r`. `Cargo.toml` asks for `diffy = "0.5.0"`, a semver range that already allows 0.5.2, so only the lockfile needs updating.
- pnpm 12 normalizes CRLF to LF when *hashing* a patch ([`crates/patching/src/hash.rs`](https://github.com/pnpm/pnpm/blob/v12.3.4/pnpm/crates/patching/src/hash.rs)), so the LF and CRLF patches share a store key and a store warmed by `lf/` would satisfy `crlf/` without parsing it. that is why each project pins `storeDir` to its own directory.
- CRLF patch files are common in practice without anyone committing one: Git for Windows defaults to `core.autocrlf=true`, so an LF patch in git checks out as CRLF on Windows.

## about this reproduction environment

```
  System:
    OS: macOS 26.6.2
    CPU: arm64
    Shell: zsh
  Binaries:
    Node: 24.20.0
    git: 2.50.1 (Apple Git-155)
    pnpm: 12.3.4 (fails), 11.25.0 (passes), both via npx
```

the patch files were hand-written against `is-odd@3.0.1` and only add one comment line, so they apply cleanly whenever they parse. `crlf/patches/is-odd@3.0.1.patch` is byte-for-byte `lf/patches/is-odd@3.0.1.patch` with `\n` replaced by `\r\n`.
