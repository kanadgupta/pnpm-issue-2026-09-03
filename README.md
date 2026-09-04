# pnpm-issue-2026-09-03

this repository reproduces an issue with [pnpm 12](https://github.com/pnpm/pnpm/releases/tag/v12.0.0) where `pnpm install` fails with `ERR_PNPM_INVALID_PATCH` when a `patchedDependencies` patch file has CRLF line endings. pnpm 11 applies the same patch without complaint.

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

the [`pnpm install` workflow](.github/workflows/test.yml) runs the same thing for pnpm 11.25.0 and 12.3.4 on ubuntu, windows, and macos. the pnpm 12 jobs fail on all three at the `crlf/` step; the pnpm 11 jobs pass on all three.

## why this happens

- pnpm 12's Rust CLI parses patch files with the [`diffy`](https://github.com/bmwill/diffy) crate ([`crates/patching/src/apply.rs`](https://github.com/pnpm/pnpm/blob/v12.3.4/pnpm/crates/patching/src/apply.rs)). the pinned `diffy` 0.5.1 splits the `---` and `+++` header lines at `\n` only, so with CRLF endings the `\r` stays attached to the filename and the parser rejects it as an "invalid char in unquoted filename". `diffy` fixed this in [0.5.2](https://github.com/bmwill/diffy/pull/87) (2026-08-31); as of pnpm 12.3.4 the `Cargo.lock` still pins 0.5.1.
- pnpm 11's JavaScript patch parser tolerates the `\r`.
- pnpm 12 already normalizes CRLF to LF when *hashing* a patch file ([`crates/patching/src/hash.rs`](https://github.com/pnpm/pnpm/blob/v12.3.4/pnpm/crates/patching/src/hash.rs)), so the LF and CRLF patches here produce the same lockfile hash and the same store key. a store that already holds a copy patched from the LF file therefore satisfies the CRLF install without the CRLF patch ever being parsed. that is why each project pins `storeDir` to its own directory in `pnpm-workspace.yaml`, and why in the real world this tends to show up only on fresh CI machines.
- in the real world nobody commits a CRLF patch: the file is LF in git and becomes CRLF on a Windows checkout because Git for Windows defaults to `core.autocrlf=true`. that is how this surfaced: a Windows CI job started failing the moment its repo bumped from pnpm 11 to pnpm 12, with no change to the patch file.

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
