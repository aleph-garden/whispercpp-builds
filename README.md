# whispercpp-builds

Prebuilt [pywhispercpp](https://pypi.org/project/pywhispercpp/) wheels for
the [Aleph](https://github.com/aleph-garden/aleph) voice assistant.

Upstream pywhispercpp publishes CPU-only wheels on PyPI. Aleph needs GPU
backends (Vulkan on ROCm/AMD hosts, CUDA on NVIDIA, Metal on Apple Silicon)
which require a from-source build with `CMAKE_ARGS`. This repo runs that
from-source build in CI, one job per (platform x backend), and publishes
the resulting wheels as GitHub release assets. `internal/binprep/whispercpp`
in the Aleph repo fetches them on first use, falling back to an in-process
source build if the wheel is unavailable.

## Release scheme

Tag format: `v<upstream>-aleph<n>`, e.g. `v1.3.0-aleph1`.

`<upstream>` is the pywhispercpp PyPI version built; `<n>` is incremented
when the build config (CMake args, CUDA version, Vulkan SDK version)
changes for the same upstream.

> The workflow's `PYWHISPERCPP_VERSION` env var is the source of truth for
> the upstream version; bump it together with the tag. Verify the latest
> stable version on [PyPI](https://pypi.org/project/pywhispercpp/) before
> tagging.

Each release attaches one wheel per row, plus `SHA256SUMS`:

| Asset filename | Platform | GPU backend |
|---|---|---|
| `pywhispercpp-<v>+vulkan-cp312-cp312-linux_x86_64.whl` | Linux x86_64 | Vulkan (ROCm/AMD/Mesa) |
| `pywhispercpp-<v>+cuda12-cp312-cp312-linux_x86_64.whl` | Linux x86_64 | CUDA 12 |
| `pywhispercpp-<v>+metal-cp312-cp312-macosx_11_0_arm64.whl` | macOS Apple Silicon | Metal |
| `pywhispercpp-<v>+vulkan-cp312-cp312-win_amd64.whl` | Windows x86_64 | Vulkan |

## Local-version segment trick

PEP 440 reserves `+<segment>` as the "local version" — pip accepts it on
install (`pip install pywhispercpp==1.3.0+vulkan`) so we can ship multiple
backends from a single release tag.

The default wheel name from `pip wheel pywhispercpp==1.3.0` is
`pywhispercpp-1.3.0-cp312-cp312-<plat>.whl`. The workflow unpacks it, edits
the `Version:` line in `*.dist-info/METADATA` to `1.3.0+<backend>`, then
runs `wheel pack` which writes a new wheel whose filename reflects the new
version automatically.

### Alternative (if local-version repack proves brittle)

Publish wheels with the **standard** filename but in **separate releases**
per backend — tag `v1.3.0-vulkan-aleph1`, `v1.3.0-cuda12-aleph1`, etc. The
Aleph consumer then maps `(GOOS, GOARCH, gpuBackend)` to the right tag
instead of the right filename within one tag. Trade-off:

- separate tags = no METADATA editing, but more release noise and more
  releases to update when bumping the upstream version.
- single tag with `+<backend>` = one release per upstream version, but
  depends on the unpack/repack step staying reliable across pip/wheel
  versions.

We start with the single-tag approach. If CI flakes on the repack, switch.

## Build profile

Every backend is built via the upstream sdist with backend selection
through `CMAKE_ARGS` only:

```
CMAKE_ARGS="-DGGML_VULKAN=on"   pip wheel pywhispercpp==<v> --no-deps --no-binary=pywhispercpp -w dist/
CMAKE_ARGS="-DGGML_CUDA=on"     pip wheel pywhispercpp==<v> --no-deps --no-binary=pywhispercpp -w dist/
CMAKE_ARGS="-DGGML_METAL=on"    pip wheel pywhispercpp==<v> --no-deps --no-binary=pywhispercpp -w dist/
```

The `--no-binary=pywhispercpp` flag forces a from-source build so the
CMake invocation actually runs.

## Updating the pinned pywhispercpp version

1. Edit `PYWHISPERCPP_VERSION` in `.github/workflows/build.yml`.
2. Push a new tag `v<upstream>-aleph<n>`. The workflow runs on tag push,
   builds the matrix, attaches wheels + `SHA256SUMS` to the release.
3. Update `pinned.go` in `aleph/packages/aleph/server/internal/binprep/whispercpp/`
   with the new `PinnedVersion`, new wheel filenames, and per-target
   SHA256 values from `SHA256SUMS`.

## Why a separate repo

Keeps the Aleph repo clean of release artifacts (a single GPU wheel is
~30 MB) and lets us rebuild pywhispercpp without touching Aleph history.
Aleph depends on this repo only at runtime (HTTP fetch), not at build
time.

## License

Built wheels are derivative works of pywhispercpp + whisper.cpp, both
MIT-licensed. The build configuration in this repo is also MIT (see
`LICENSE`).
