# BeamMP-Launcher

The launcher is the way we communitcate to outside the game, it does a few automated actions such as but not limited to: downloading the mod, launching the game, and create a connection to a server.

## [Getting started](https://docs.beammp.com/game/getting-started/)

## Building

Prerequisites:

- CMake >= 3.21
- [Ninja](https://ninja-build.org/)

[vcpkg](https://github.com/microsoft/vcpkg) is vendored as a git submodule, so no separate vcpkg install
or `VCPKG_ROOT` setup is needed — the first `cmake --preset` configure bootstraps it and installs
dependencies (declared in `vcpkg.json`) automatically.

```sh
git clone --recursive <this-repo-url>
cd BeamMP-Launcher

# Linux
cmake --preset linux
cmake --build --preset linux
./build/BeamMP-Launcher

# Windows (run from a Developer Command Prompt/PowerShell so cl.exe is on PATH)
cmake --preset windows
cmake --build --preset windows
.\build\BeamMP-Launcher.exe
```

Already have a clone without submodules? Run `git submodule update --init` first.

The presets default to a Release build. For a Debug build, override `CMAKE_BUILD_TYPE` on the configure
command, e.g. `cmake --preset linux -DCMAKE_BUILD_TYPE=Debug` (command-line `-D` flags take precedence
over a preset's own settings). This also picks up vcpkg's debug library variants automatically.

## License

BeamMP Launcher, a launcher for the BeamMP mod for BeamNG.drive
Copyright (C) 2024 BeamMP Ltd., BeamMP team and contributors.

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU Affero General Public License as published
by the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU Affero General Public License for more details.

You should have received a copy of the GNU Affero General Public License
along with this program.  If not, see <https://www.gnu.org/licenses/>.
