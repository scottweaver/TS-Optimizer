# Building Yuku From Source

## Install Zig And Bun
```shell
brew install zig
curl -fsSL https://bun.com/install | bash
```
## Build Zig
```shell
bun install
bun load-files
bun run build:npm
```


# Yuku Build Issue: Missing Parser Benchmark Files

## Symptom

Running `bun run build:npm` in the `yuku` repo fails during the Zig profiler compile step with errors like:

```
profiler/profile.zig:20:49: error: unable to open 'files/react.js': FileNotFound
profiler/profile.zig:20:49: error: unable to open 'files/three.js': FileNotFound
profiler/profile.zig:20:49: error: unable to open 'files/typescript.js': FileNotFound
error: 3 compilation errors
```

The `napi build --release` step (cross-compilation) succeeds, but the subsequent `zig build gen-estree-decoder` fails because the profiler executable can't be built.

## Root Cause

`profiler/profile.zig` embeds three benchmark fixtures at compile time via `@embedFile`:

```zig
const parser_bench_files = .{
    .{ .name = "typescript.js", .path = "files/typescript.js" },
    .{ .name = "three.js",      .path = "files/three.js" },
    .{ .name = "react.js",      .path = "files/react.js" },
};
```

These files live in `profiler/files/`, which is **not** committed to the repo. They are fetched on demand by `profiler/load-parser-bench-files.ts`, which git-clones `https://github.com/yuku-toolchain/parser-benchmark-files` into `profiler/files/`.

On a fresh checkout (or after deleting `profiler/files/`), the directory is absent and the Zig compile fails.

## Resolution

Run the `load-files` script once before building:

```bash
bun load-files
bun run build:npm
```

`bun load-files` is defined in `package.json` and does two things:

1. `bun profiler/load-parser-bench-files` — clones the parser benchmark fixtures into `profiler/files/`.
2. `bun test/parser/load test/parser/suite -b typescript` — clones the TypeScript parser test suite into `test/parser/suite/` (needed for tests, not the build).

Both clones are idempotent — the scripts skip if the target directory already exists — so running `bun load-files` repeatedly is safe.

## Why the Build Script Doesn't Auto-Run It

The `test` script chains them (`bun load-files && bun build:npm && …`), but `build:npm` on its own assumes the fixtures are already present. If you only ever invoke `bun test` or `bun play`, the dependency is transparent; calling `build:npm` directly on a fresh clone is the failure case.

## Notes

- The full benchmark repo is small, but the test suite clone (`test/parser/suite`) pulls ~100k files (~80 MiB) — first-time setup takes a minute or two.
- If fixtures are ever corrupted, delete `profiler/files/` and re-run `bun load-files` to re-clone.
