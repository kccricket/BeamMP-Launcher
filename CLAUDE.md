# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

BeamMP-Launcher is the standalone executable that mediates between the BeamNG.drive game process and
BeamMP's multiplayer backend/servers: it downloads and installs the BeamMP mod into the game's mods
folder, launches the game, self-updates, and then acts as a local proxy relaying messages between the
game (over a local TCP socket) and remote multiplayer servers (over TCP/UDP).

License: AGPL-3.0-or-later. New source files should keep the existing SPDX header block (see any
`src/*.cpp` file for the exact text).

## Build

This is a CMake + vcpkg C++20 project. The only git submodule is `vcpkg/` itself, vendored and pinned to
a specific commit so the toolchain version is a single source of truth (no `VCPKG_ROOT` env var, no
separate vcpkg install). The previously-vendored `evpp` submodule was unused — nothing in
`src/`/`include/` referenced it — and has been removed. Dependencies are declared once, in `vcpkg.json`
(manifest mode), and CI/local builds both consume them the same way via `CMakePresets.json`, whose
`CMAKE_TOOLCHAIN_FILE` points at `${sourceDir}/vcpkg/scripts/buildsystems/vcpkg.cmake`. That toolchain
file auto-bootstraps the `vcpkg` executable (downloads/builds it) on first configure if it isn't present
yet — no manual bootstrap step needed.

```sh
git submodule update --init   # only needed once, for vcpkg/

# Linux
cmake --preset linux
cmake --build --preset linux        # -> build/BeamMP-Launcher

# Windows (from a Developer Command Prompt so cl.exe is on PATH)
cmake --preset windows
cmake --build --preset windows      # -> build/BeamMP-Launcher.exe
```

Both presets default to `CMAKE_BUILD_TYPE=Release`; override on the configure line for a Debug build
(e.g. `cmake --preset linux -DCMAKE_BUILD_TYPE=Debug`) — command-line `-D` flags take precedence over a
preset's own `cacheVariables`, and it picks up vcpkg's debug library variants automatically since vcpkg
always builds both debug and release variants of every dependency regardless of build type.

Both presets use Ninja and put output under `build/` uniformly. Dependencies (see `vcpkg.json`):
cpp-httplib (with the `openssl` feature), nlohmann-json, zlib, openssl, curl. `discord-rpc` was
previously pulled on Windows but was likewise unused and has been removed. Windows builds use the
`x64-windows-static` triplet with a statically-linked MSVC runtime (the top of `CMakeLists.txt` forces
`/MD` -> `/MT`). Windows links `ws2_32`; Linux/macOS link OpenSSL/zlib/curl directly. There is no
MinGW-specific structured exception handling path (`__try/__except` is Windows-MSVC only; Linux uses a
`try/catch (...)` fallback in `CoreNetwork()`).

CI (`.github/workflows/cmake-linux.yml`, `cmake-windows.yml`) builds Release on Ubuntu and Windows,
checking out the `vcpkg/` submodule and using `lukka/run-vcpkg` (with `doNotUpdateVcpkg: true`, so it
respects the pinned submodule commit instead of fetching its own copy) purely for its GitHub Actions
binary-cache wiring — it caches vcpkg's built package binaries (e.g. compiled OpenSSL/curl) between runs
so they aren't rebuilt from source every time. Both jobs then configure/build via the same
`linux`/`windows` CMake presets used locally. There is no test suite and no lint step in CI —
`.clang-format` (WebKit-based) is the only style config, no enforcement is wired in.

A stray `bin/` directory from an old local CMake configure may still exist in some checkouts — it's a
build artifact, not source; the canonical build output directory is now `build/` (gitignored).

## Architecture

Everything is global mutable state and free functions declared in headers under `include/` and defined
in matching files under `src/` — there is essentially no OOP/class structure, no dependency injection,
and most cross-file communication happens through `extern` globals (e.g. `Options options` in
`main.cpp`, `Terminate`/`TCPTerminate`/`ping`/`CoreSocket` etc. in `Network/Core.cpp`). When editing,
follow this style rather than introducing new abstractions.

### Startup sequence (`src/main.cpp`)

1. `InitLog()` / `ConfigInit()` / `InitOptions()` — logging, config file, CLI args (`Options.cpp`,
   `Config.cpp`, `Logger.cpp`).
2. `InitLauncher()` (`Startup.cpp`) — renames/relaunches itself if invoked under a different exe name
   (`CheckName`), applies Wine/Proton registry patches on Windows (`LinuxPatch`), and self-updates
   (`CheckForUpdates`): fetches a hash + version from `backend.beammp.com`, verifies the hash is a
   well-formed sha256 before trusting it, downloads a replacement binary, and (Windows only) verifies
   both an Authenticode signature (`VerifySignature`) and a hardcoded certificate thumbprint/pubkey hash
   (`CheckThumbprint`) before swapping itself and relaunching. **Linux has no auto-update** — it only
   warns the user to update manually.
3. `LegitimacyCheck()` / `CheckLocalKey()` (`Security/BeamNG.cpp`) — validates the local BeamNG.drive
   installation/version before proceeding.
4. `HTTP::StartProxy()` (`Network/Http.cpp`) — starts a local HTTP proxy the game uses for in-game
   browser/resource requests.
5. `PreGame(GetGameDir())` (`Startup.cpp`) — locates the game's user data folder (`GetGamePath()` in
   `GameStart.cpp`, which parses `startup.ini` / `BeamNG.Drive.ini` for a custom user path), cleans out
   the multiplayer mods folder (`CheckMP`), and downloads/installs the mod zip (`beammp.zip` on Linux —
   lowercase because the Linux game build can't handle uppercase mod filenames — vs `BeamMP.zip` on
   Windows) after checking its hash against the backend, mirroring the same sha256 hash validation
   pattern as the launcher self-update.
6. `InitGame(GetGameDir())` (`GameStart.cpp`) — spawns the actual BeamNG.drive process
   (`CreateProcessW` on Windows / `posix_spawn` on Linux) in a detached thread and waits on it.
7. `CoreNetwork()` (`Network/Core.cpp`) — runs forever, accepting a TCP connection from the game on
   `127.0.0.1:<options.port>` (default 4444) and dispatching game <-> launcher protocol messages.

### In-game <-> launcher protocol (`Network/Core.cpp`)

The game connects to the launcher over localhost TCP and sends single-letter-coded commands, handled
in `Parse()`'s switch statement (`GameHandler` -> `Parse`). Notable codes: `C` starts a connection to a
multiplayer server (`StartSync` -> spawns `TCPGameServer` thread), `I` fetches server info for the
server browser, `O` opens a URL in the system browser (only if it matches `IsAllowedLink()`'s allowlist
regex for beammp.com/beammp.gg/github.com/BeamMP/discord.gg/patreon.com/BeamMP — do not weaken this
check), `N` handles login, `Q`/`QS`/`QG` quit/reset/exit, `W` answers a "confirm mods" prompt tied to
`SecurityWarning()`, `Z`/`P`/`U`/`M` report version/proxy port/ping/mod status back to the game.
Responses are framed with `Utils::PrependHeader` and written back on the same socket.

### Game <-> multiplayer-server networking (`Network/`)

- `Core.cpp` — the launcher-facing protocol described above, plus the accept loop in `CoreNetwork()`.
- `GlobalHandler.cpp`, `VehicleData.cpp`, `VehicleEvent.cpp`, `Resources.cpp` — per-server TCP/UDP
  session handling: resource sync (mod/map downloads from the server the client is joining), vehicle
  spawn/state packets, and other in-session traffic once `TCPGameServer`/`UDPClientMain` are running.
- `DNS.cpp` — resolves server hostnames (`GetAddr`).
- `Http.h`/`Http.cpp` — thin wrapper over cpp-httplib/libcurl for backend REST calls (`HTTP::Get/Post/
  Download`) and the local proxy server used by the in-game browser.

### Security (`Security/`)

- `BeamNG.cpp` — `LegitimacyCheck`, `CheckLocalKey`, `CheckVer`: validates the game install and its
  version before allowing multiplayer.
- `Login.cpp` — backend authentication flow invoked from the `N` protocol command.
- `include/hashpp.h` — vendored hashing library used for the sha256 checks throughout (launcher update,
  mod zip, thumbprint comparisons).

### Platform split

Almost every file branches on `_WIN32` vs `__linux__` (occasionally `__APPLE__`, though macOS is not
built in CI) for: process spawning, game path discovery, self-relaunch (`execv` vs `ShellExecuteW`),
console handling, and the Windows-only code-signing verification in `CheckForUpdates`. When touching
platform-specific logic, update both branches or explicitly confirm the change is platform-specific.

### Vendored/third-party code (do not "clean up")

`include/rapidjson/`, `include/zip_file.h`, `include/hashpp.h`, `include/vdf_parser.hpp` are vendored
dependencies, not project code — leave their style/structure alone.
