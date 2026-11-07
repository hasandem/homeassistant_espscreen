# A Mac build server's first ESP-IDF install crashes in a forked Python process

## The symptom
During a remote ESP32-P4 build, ESPHome's ESP-IDF installer exits with `returncode=-11` before compiling the plugin.
The screen keeps its previous firmware. Retrying after a successful command-line compile completes the remote build.

## What we know
Measured on 2026-11-07, with ESPHome 2026.9.0, ESP-IDF 5.5.5 and the native Mac build server's bundled Python 3.14:

- The crash report records `EXC_BAD_ACCESS` and `SIGSEGV` in the forked child. Its stack includes
  `_os_log_preferences_refresh`, `NEFlowDirectorDestroy`, `nw_path_release_globals`,
  `nw_settings_child_has_forked`, the pthread child handlers and Python's subprocess fork/exec.
- A compile of the same frozen build profile with the same Python interpreter succeeds from the terminal. It installs
  the framework and toolchains into ESPHome's shared cache. No firmware is uploaded by that compile.
- Retrying the original remote build then succeeds and the screen reconnects. No application, ESPHome or framework
  source was changed. This establishes a usable retry path, but does not establish the crash's cause.
- Direct helper calls and small subprocess probes from the terminal do not reproduce the crash.

## Ruled out
A compiler error in the waste plugin does not explain this failure: the first attempt stops before compilation, and
the same plugin compiles in both the terminal build and the successful remote retry.

## Candidates
- A framework or toolchain first-install path that only runs with a cold cache.
- A difference in inherited macOS networking state between the native build server's Python worker and a terminal.
- A Python or macOS fork-child failure reached through that networking state.

## The deciding experiment
Use isolated temporary ESPHome caches to compare the same installer helper and frozen profile in the native worker
and in the terminal. Keep the interpreter, framework version and inputs identical. Record whether the helper starts,
its exit status and any child crash stack before warming either cache. Then repeat with warm caches. Do not clear the
working build server's shared cache or upload firmware for this experiment.

## Pointers
`docs/BUILDS.md` (On a build server), `screen_manager/app/build_server.py`, and ESPHome 2026.9.0's
`esphome/espidf/get_tool_downloads.py`. The macOS crash report and frozen profile are private local diagnostics;
profiles, keys and full logs do not belong in this repository.
