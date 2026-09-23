# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

NVIDIA Data Center GPU Manager (DCGM): a C++ daemon (`nv-hostengine`), client library (`libdcgm.so`), CLI (`dcgmi`), diagnostics suite (NVVS), and Python/C SDK for monitoring and managing data-center GPUs. This GitHub repo is a per-release snapshot of NVIDIA's internal code base (roughly one commit per release, e.g. "DCGM 4.6.1"), so large refactors are hard to merge upstream — keep changes focused.

## Building

All builds run inside a Docker build image; `build.sh`, `format-dcgm` and `validate_format.sh` re-exec themselves inside it via `intodocker.sh` (unless `DCGM_BUILD_INSIDE_DOCKER=1`). Building the image itself (`cd dcgmbuild && ./build.sh`) takes hours; override the image with `DCGM_DOCKER_IMAGE`.

```bash
./build.sh -r                 # release (RelWithDebInfo) build for amd64, runs unit tests
./build.sh -d                 # debug build
./build.sh -r --deb           # also produce .deb (--rpm, -p/--packages for tar.gz)
./build.sh -d -n              # skip build-time tests
./build.sh -c ...             # clean rebuild
./build.sh -a aarch64 ...     # cross-arch
./build.sh --address-san      # also --thread-san, --ub-san, --leak-san, --coverage
./build.sh -d -- -DCMAKE_VERBOSE_MAKEFILE=ON   # extra cmake args after --
```

- Build tree: `_out/build/<Linux-arch-buildtype>/`; install tree: `_out/<Linux-arch-buildtype>/`.
- `NPROC` controls parallelism; `DCGM_SKIP_PYTHON_LINTING=1` skips the pylint pass that runs after each build.
- `CMakePresets.json` (Ninja, CMake ≥ 4.0) has `Debug`/`RelWithDebInfo`/`Release`/`DEB`/`RPM`/`TGZ` presets for use from inside the container (e.g. via `./intodocker.sh -- bash`).

## Testing

**Unit tests (C++, Catch2)** are registered with CTest via `catch_discover_tests` in each `*/tests/CMakeLists.txt` and run automatically by `build.sh` (`ctest --output-on-failure --test-dir <build dir>`). To run a subset inside the container:

```bash
ctest --test-dir _out/build/<suffix> -R <regex> --output-on-failure
# or run a test binary directly with a Catch2 filter, e.g.
_out/build/<suffix>/dcgmlib/src/tests/dcgmlibtests "<test case name>"
```

Test binaries include `dcgmlibtests` (dcgmlib/src/tests), `coreModuleTests`, `healthtests`, `policyModuleTests`, `diagtests`, `dcgmitests`, `nvvscoretests`, etc. Legacy unit tests also live in `testing/Test*.cpp`.

**Integration tests (Python)** live in `testing/python3/tests/` and require real GPUs (or NVML injection). They ship in the `datacenter-gpu-manager-tests` package: extract it, `cd share/dcgm_tests`, `sudo ./run_tests.sh`. Run a subset with `main.py -f/--filter-tests <regex>` (matches `module.test_fn`); `-d <nvml id>` limits to one GPU. The full suite takes ~30+ minutes.

**NVML injection** (`nvml-injection/`): a fake `libnvml_injection.so` selected by `NVML_INJECTION_MODE`, fed from a YAML capture (`NVML_YAML_FILE`), used to test without specific hardware.

## Formatting / linting

- C/C++: `clang-format` (`.clang-format`; `sdk/` and `_out/` are ignored), `.clang-tidy`. Python: `autopep8` (`.pep8`), pylint (`testing/python3/pylintrc`).
- `./format-dcgm` formats the whole tree; `./validate_format.sh` checks staged files and is the pre-commit hook installed by `./install_git_hooks.sh`.
- Upstream contributions require DCO sign-off (`git commit -s`).

## Architecture

**Process model.** `nv-hostengine` (`hostengine/`) is the daemon; it links the core engine from `dcgmlib/src/`. Clients (`dcgmi`, Python bindings, user programs) use the public C API in `dcgmlib/dcgm_agent.h` (implemented in `dcgmlib/src/DcgmApi.cpp`) and talk to the host engine over TCP (default port 5555) or a Unix socket (`common/transport/`), or run the engine embedded in-process. Exported symbols for each shared library are controlled by `*.linux_def` version scripts — new public API functions must be added there.

**Core engine (`dcgmlib/src/`).** `DcgmHostEngineHandler` is the central dispatcher; `DcgmCacheManager` polls NVML and caches field values per watch; `DcgmGroupManager`/`DcgmFieldGroup` manage entity and field groups; MIG, GPM, IMEX, topology, etc. have their own managers. Field IDs are defined in `dcgmlib/dcgm_fields.h`.

**Modules (`modules/`).** Features are loadable modules subclassing `DcgmModule` (`modules/DcgmModule.h`) and implementing `ProcessMessage()`. Each is built as `libdcgmmodule<name>.so.4` and `dlopen`ed lazily by `DcgmHostEngineHandler` (IDs in `dcgmModuleId_t` in `dcgm_structs.h`): core, nvswitch, vgpu, introspect, health, policy, config, diag, profiling (closed source), sysmon, mndiag. Client API calls build a `dcgm_module_command_header_t` message (per-module structs in `modules/<name>/dcgm_<name>_structs.h`) and send it via `dcgmModuleSendBlockingFixedRequest`; modules call back into the engine through `DcgmCoreProxy` (`modules/common`). Adding a feature usually means: public API + versioned struct → module message struct → module handler → dcgmi command → Python bindings → tests.

**Versioned structs.** Every public/module struct carries a `version` built with `MAKE_DCGM_VERSION(type, n)` (sizeof | version<<24). Changing a struct's layout requires a new version (`_vN` typedef + `dcgm..._versionN` define) and keeping old versions working; `DcgmModule::CheckVersion` validates incoming messages. `testing/TestVersioning.cpp` and `dcgmlib/src/tests/DcgmVersionTests.cpp` check these.

**Diagnostics.** The diag module launches the separate `nvvs` binary (`nvvs/src`, entry `NvvsMain.cpp`), which loads test plugins from `nvvs/plugin_src/` (diagnostic/gpuburn, memory, memtest, pcie, targetedpower, targetedstress, contextcreate, nvbandwidth, nccl_tests, software). Plugins are built per CUDA major version (11/12/13). Multi-node diag lives in `modules/mndiag` + `testing/mndiag`.

**Other components.** `dcgmi/` — CLI, one class per subcommand (`Diag`, `Health`, `Policy`, ...). `common/` — shared utilities (logging, threads, child processes, transport, serialization). `cublas_proxy/`, `dcgmproftester/` — CUDA load generators. `sdk_samples/` — C and Python API examples. `testing/python3/` — Python bindings (`dcgm_agent.py`, `dcgm_structs.py`, `dcgm_fields.py`, `pydcgm.py`) plus the test framework.

**Python bindings must stay in sync with the C API.** Any change to C structs, fields, or API functions needs matching changes in `testing/python3/dcgm_structs.py` / `dcgm_fields.py` / `dcgm_agent.py`; upstream treats C-only changes as incomplete.

## Coding conventions (from docs/coding_best_practices.md)

- No exceptions: use `Initialize()` methods and `dcgmReturn_t` return codes plus logging; catch third-party exceptions at the source.
- C++-style casts only; avoid `const_cast`; prefer `static_cast` over `reinterpret_cast`.
- `#include <...>` for headers in other directories, `"..."` for the same directory.
- No `using namespace` in `.cpp` files; no `using` of any kind in headers.
- Single-argument constructors are `explicit`.
- Naming: camelCase locals, `m_` members (private by default), `c_` constants, `g_` globals.
- One topic per change; keep functions small and unit-testable.
