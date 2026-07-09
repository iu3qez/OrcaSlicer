# CLAUDE.md

OrcaSlicer — open-source C++17 3D slicer. wxWidgets GUI, CMake build system.

## Build Commands

Two-phase build: dependencies in `deps/` (Boost, wxWidgets, CGAL, TBB, OpenVDB, CURL, …) are
built once as a separate CMake project (`deps/deps-<platform>.cmake`), then the root
`CMakeLists.txt` links against them. The deps phase is the slow one (tens of minutes / hours);
a first-time build must build deps before the app can compile.

```bash
# macOS
cmake --build build/arm64 --config RelWithDebInfo --target all --

# Linux
cmake --build build --config RelWithDebInfo --target all --

# Windows (replace %build_type% with Debug/Release/RelWithDebInfo)
cmake --build . --config %build_type% --target ALL_BUILD -- -m
```

Convenience scripts wrap the two phases: `./build_linux.sh -u` (system deps, once) then
`./build_linux.sh -dsi` (deps + slicer + AppImage); add `-t` to build tests. `./build_linux.sh
-g -istrlL` reproduces the GitHub Actions Linux runner inside a Docker/Podman Ubuntu 24.04
container. macOS/Windows use `build_release_macos.sh` / `build_release_vs2022.bat`.

## CI (GitHub Actions)

Fully automated in `.github/workflows/`. Chain: `build_all.yml` (orchestrator, per-platform
matrix) → `build_check_cache.yml` (deps cache keyed by hashing the deps files) →
`build_deps.yml` (only on cache miss) → `build_orca.yml` (builds the app + AppImage/exe/dmg) →
`unit_tests.yml` (runs `ctest`). Triggers: push to `main`/`release/*`, every PR to those
branches (path-filtered on `src/**`, `deps/**`, `**/CMakeLists.txt`, `tests/**`, …), a nightly
schedule, and manual `workflow_dispatch`. Platforms: Linux amd64 + aarch64, Windows x64/arm64,
macOS. **On forks, Actions is disabled by default** and there are no self-hosted runners, so CI
won't run on fork PRs until enabled in the repo's Actions tab, and the deps cache starts empty
(first build is slow).

## Testing

Catch2 framework. Tests in `tests/` directory.

```bash
cd build && ctest --output-on-failure           # all tests
ctest --test-dir ./tests/libslic3r              # individual suite
ctest --test-dir ./tests/fff_print
```

## Code Style

- C++17, selective C++20. PascalCase classes, snake_case functions/variables
- `#pragma once` for headers. Smart pointers and RAII preferred
- Parallelization via TBB — be mindful of shared state

## Key Entry Points

- App startup: `src/OrcaSlicer.cpp`
- Slicing pipeline: `src/libslic3r/Print.cpp`
- All print/printer/material settings: `src/libslic3r/PrintConfig.cpp`
- GUI: `src/slic3r/GUI/`
- Core algorithms: `src/libslic3r/` (GCode/, Fill/, Support/, Geometry/, Format/, Arachne/)
- Printer profiles: `resources/profiles/[manufacturer].json`

## Critical Constraints

- **Backward compatibility required** for .3mf project files and printer profiles
- **Cross-platform** — all changes must work on Windows, macOS, and Linux
- Profile/format changes need version migration handling
- Dependencies built separately in `deps/build/`, then linked to main app

## Code review focus areas

- Changes must not cause regressions in existing functionality, defaults, profiles, or project compatibility.
- Features gated by options must not affect existing behavior when those options are disabled.
- Changes should follow the existing code style and architecture. Architectural changes should be justified in code comments and the PR description.
- Add helper functions or utilities only when existing code cannot reasonably be reused. Avoid duplication.
- Keep code concise and clear. Manually simplify AI generated bloated codes before review.
- Include targeted tests or documented verification for behavior changes, especially in slicing logic, profiles, formats, and GUI defaults.
