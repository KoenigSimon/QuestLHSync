# QuestLHSync → Linux (CachyOS, Nvidia) migration plan

## Problem statement

QuestLHSync's PC side (SteamVR driver `driver_questlhsync.dll`, dashboard `QuestLHSync.exe`, installer) is Windows-only.
Goal: run it on **CachyOS (Arch-based) with an Nvidia GPU**, with SteamVR for Linux, the **Steam Frame streamed via
Steam Link**, and lighthouse trackers/controllers/base stations.

Constraints confirmed with the user:
- Headset parts (`frame/`, `magisk/`, `src/headset/`) stay **untouched**. The PC only has to receive their stream.
- Fork of `CreoleVR/QuestLHSync` (origin = `KoenigSimon/QuestLHSync`). It has to stay easy to update from upstream.
  **Minimal-touch strategy**: upstream files stay byte-identical. Linux code, shims and patches go under a new
  `linux/` folder. Some duplicated logic is acceptable.
- Linux dashboard: a **separate, compatible app** built with **Dear ImGui + SDL2 + OpenGL**. One binary handles both the
  SteamVR dashboard overlay and the desktop window.

## Analysis: what the PC side consists of, and what's Windows-specific

| Part | Files | Windows dependencies | Linux treatment |
|---|---|---|---|
| Solver core | `src/driver/sync.cpp/.h`, `mathx.h`, `json.h` | Almost none (`localtime_s` already has an `#ifdef`). `Join()` adds `"\\"` when the dir doesn't end in a separator | Compiles as is; the dir passed in always ends with `/` |
| Network link | `src/driver/net.cpp/.h` | Winsock (`SOCKET`, `closesocket`, `ioctlsocket`, `WSAStartup`), `select(0,…)`, `recvfrom(int*)`, `GetAdaptersAddresses`, QPC clock, `Sleep` | Compat shims + a small patch (getifaddrs, nfds, socklen_t) |
| Driver provider | `src/driver/driver_main.cpp` | `SHGetKnownFolderPath`, `CreateFileMappingW` shm, named mutex, `CreateProcessW`, `GetModuleHandleExW`, `Interlocked*`, `__declspec(dllexport)`, `"\\file"` path joins | Patch: XDG paths, POSIX shm, flock, posix_spawn, dladdr, visibility attribute |
| Pose hook | `driver_main.cpp` (`HookHost`, `UnhookHost`) + `third_party/minhook` | MinHook (inline x64 hooks); `UnhookHost` checks MinHook's `E9` → `FF25` relay byte layout | **POSIX port of MinHook's API** (`buffer.c`/`hook.c` rewritten, `trampoline.c`/`hde64.c` reused unchanged), with the same patch layout so `UnhookHost` works unchanged |
| Gravity levelling | `src/driver/gravity.cpp/.h` | HID through SetupAPI/`HidD_*`/overlapped `ReadFile`; `%LOCALAPPDATA%\openvr\openvrpaths.vrpath`; `"\\gravity.json"` | Patch: hidraw (`/sys/class/hidraw`, `HIDIOCGRAWUNIQ`, `poll`/`read`), `~/.config/openvr/openvrpaths.vrpath` |
| Base station averaging | `src/driver/relations.cpp/.h` | Hooks `driver_lighthouse.dll`: PE parsing, `.pdata` unwind table, MSVC x64 calling convention byte signature | Off on Linux at first (logged). Optional last chapter: ELF + `.eh_frame_hdr` finder, SysV signature |
| Dashboard app | `src/overlay/overlay_main.cpp` | GDI drawing, D3D11 texture, Win32 window, Toolhelp process scan, named mutex/shm | **New** `linux/dashboard/` (ImGui + SDL2 + OpenGL) with the same layout, reading the same `qlhs_status.h` |
| Installer | `src/installer/installer_main.cpp`, `install.py`, `release.py` | Win32 GUI, WinHTTP, `vrpathreg.exe`, `%LOCALAPPDATA%`, `vrserver.exe` | **New** `linux/install.sh`, `uninstall.sh` and `linux/release.py`. No auto-updater GUI |
| Test tools | `src/tools/replay.cpp`, `nettest.cpp`, `fake_lhsyncd.py` | `replay.cpp`: none. `nettest.cpp`: `Sleep`, windows.h | Built on Linux. They're the per-chapter regression tests |
| Shared IPC header | `src/common/qlhs_status.h` | Only the `L"Local\\…"` name macros | Struct used as is. Linux uses its own shm name `/QuestLHSync` |

### SteamVR Linux vs Windows: differences to handle

| Topic | Windows | Linux (SteamVR for Linux) |
|---|---|---|
| Driver binary | `bin/win64/driver_questlhsync.dll` | `bin/linux64/driver_questlhsync.so`, exporting `HmdDriverFactory` with default visibility |
| Registration | `bin/win64/vrpathreg.exe` | `<SteamVR>/bin/vrpathreg.sh` (wrapper that sets the library path) |
| openvrpaths | `%LOCALAPPDATA%\openvr\openvrpaths.vrpath` | `~/.config/openvr/openvrpaths.vrpath` |
| Config / logs | `<config>/steamvr.vrsettings`, `<Steam>/logs/vrserver.txt` | Same files under `~/.steam/steam/` (or `~/.local/share/Steam/`); config path read from openvrpaths |
| Process names | `vrserver.exe`, `vrmonitor.exe` | `vrserver`, `vrmonitor`, `vrcompositor` (`/proc/*/comm`) |
| Runtime environment | Native | vrserver may run inside the **Steam Linux Runtime (pressure-vessel, sniper)**. To be safe, build against old glibc (Zig `x86_64-linux-gnu.2.31`, like `frame/build.py`), link libstdc++/libgcc statically, and check `/dev/shm`, `/dev/hidraw*` and `$HOME` visibility |
| Overlay texture | `TextureType_DirectX` | `TextureType_OpenGL` (GL texture name), with a `SetOverlayRaw` fallback |
| Lighthouse USB access | Driver installs itself | **udev rules** for Valve VID `28de` (hidraw + usb, `uaccess`). On Arch/CachyOS the `steam` package ships `steam-devices`-style rules; check, and fall back to a QuestLHSync rule |
| Network | Windows firewall prompt | ufw/firewalld (if enabled) **drop the unicast UDP replies to the broadcast discovery** (conntrack doesn't link them). Either allow UDP from the LAN or set `host` |
| Streamers | Link, Air Link, VD, ALVR, Steam Link, CreoleCast | **Steam Link** (`vrlink`) and ALVR only. WiVRn/Monado bypass SteamVR (out of scope) |
| Clock | `QueryPerformanceCounter` | `clock_gettime(CLOCK_MONOTONIC)` (any monotonic clock works: pose times are relative to "now") |
| Data folder | `%LOCALAPPDATA%\QuestLHSync` | `$XDG_DATA_HOME/QuestLHSync` (`~/.local/share/QuestLHSync`) |
| Nvidia specifics | n/a | The stream needs no direct mode. The overlay's GL context on Nvidia works under X11/XWayland/Wayland (EGL); `SDL_VIDEODRIVER=x11` is the fallback switch |

## Fork / updateability strategy (applies to every chapter)

1. **Git layout**: `git remote add upstream https://github.com/CreoleVR/QuestLHSync.git`. Work on branch `linux`.
   Upstream updates arrive with `git fetch upstream && git merge upstream/master`. The port **only adds files**
   (`linux/**`, optionally `.github/workflows/linux.yml`), so merges never conflict.
2. **Build-time staging** (`linux/build.py`, Python like `frame/build.py`):
   1. Copy the needed upstream sources to `build/linux/src/` (git-ignored by the existing `build/` rule).
   2. Apply **mechanical rewrites** that survive upstream changes: backslash path literals (`"\\name"` → `"/name"`,
      the `Join()` separator) in a fixed list of files.
   3. Apply **small unified patches** from `linux/patches/<file>.patch` (`git apply --3way` / `patch -p1`). Each one
      touches only the OS-integration hunks.
   4. Compile with `-Ilinux/compat/include` (fake `windows.h`, `winsock2.h`, `ws2tcpip.h`, `iphlpapi.h`, `shlobj.h`,
      `setupapi.h`) and force-include `linux/compat/win32_compat.h`. These cover only the purely mechanical Win32 → POSIX
      mappings: types, `Sleep`, QPC, `GetTickCount64`, `localtime_s`, `Interlocked*`, `MemoryBarrier`, `YieldProcessor`,
      Winsock basics, `__declspec`.
   5. Linux-only implementations live in `linux/src/` (minhook_posix, hidraw, shm/process helpers), `linux/dashboard/`
      and `linux/tools/`.
3. **Drift detection**: a patch that no longer applies fails the build loudly with the file name.
   `linux/tools/refresh_patch.py <file>` regenerates a patch from an edited staged copy. `linux/UPSTREAM.md` records
   the upstream commit each patch was last refreshed against.
4. Upstream files stay unchanged, so the Windows build (`build.bat`) stays unaffected.

## Chapters

Each chapter builds on the previous ones and ends with a **test gate**. Don't start the next chapter until the gate
passes. Commit at the end of each chapter on the `linux` branch.

---

### Chapter 0: Preparation: fork workflow, Linux environment, baseline

**Goal:** a reproducible environment and confirmed facts about SteamVR on this CachyOS machine before any code.

Steps:
1. Git: add the `upstream` remote, create branch `linux`, add `linux/README.md` (stub) and `linux/UPSTREAM.md`
   (merge routine, base commit `d941d4e`, QuestLHSync 1.24).
2. CachyOS packages: `base-devel clang lld python git patch zig sdl2 (sdl2-compat) libglvnd mesa nvidia-utils
   steam`, plus `objdump`/`binutils` (chapter 8). Install SteamVR through Steam.
3. Baseline without QuestLHSync: SteamVR starts, the Steam Frame streams through Steam Link, and base stations,
   trackers and controllers show up. If lighthouse devices are missing, check the udev rules
   (`ls /usr/lib/udev/rules.d | grep -i steam`, `ls -l /dev/hidraw*`, `getfacl /dev/hidraw*` for `uaccess`).
4. Record the facts the later chapters rely on, in `linux/README.md` ("Environment notes"):
   - SteamVR install path, the `vrpathreg.sh` location, the `openvrpaths.vrpath` content, the `steamvr.vrsettings` path.
   - Whether vrserver runs inside pressure-vessel:
     `tr '\0' '\n' < /proc/$(pgrep -x vrserver)/environ | grep -Ei 'pressure|container|LD_LIBRARY_PATH'`,
     `ls -l /proc/$(pgrep -x vrserver)/root`.
   - The HMD's properties as SteamVR sees them (`vrserver.txt`: model, tracking system name, e.g. `vrlink`), and
     whether the Frame is detected through `QuestFamily` ("Steam Frame"/"deckard").
   - Whether SteamVR reads the lighthouse receivers through hidraw: `ls -l /proc/$(pgrep -x vrserver)/fd | grep hidraw`.
5. Headset side: install the Frame package as the README describes (on the Frame, no change). Firewall check:
   `sudo ufw status` / `firewall-cmd --state`.

**Test gate 0:**
- A Python one-liner from the PC broadcasts `QLHS?` to UDP 47281 and gets `QLHS 1 <serial> <port> Steam_Frame`
  back. If it doesn't, fix the firewall or note that `host` has to be set.
- `nc <frame-ip> 47280` prints the `H QuestLHSync …` hello and a `C <len> …` calibration line.
- SteamVR + Steam Link + lighthouse devices work without QuestLHSync.

---

### Chapter 1: Build system, compat layer, portable solver (`qlhs_replay`)

**Goal:** the staging/patch/compile pipeline works, and the solver core runs on Linux with the same results.

Steps:
1. `linux/build.py` with targets `replay | nettest | linktest | driver | dashboard | all`:
   - Staging, rewrite and patch pipeline as described above. Toolchain: `clang++`/`g++` by default, or `--zig` for a
     glibc 2.31 target (used for the driver if chapter 0 showed pressure-vessel).
   - Flags: `-std=c++17 -O2 -Wall -pthread -DNOMINMAX -D_CRT_SECURE_NO_WARNINGS`. Outputs go to `out/linux/`.
2. `linux/compat/include/` + `linux/compat/win32_compat.h`: only what the replay and solver need at this point. Grow
   it per chapter.
3. Build `qlhs_replay` from `src/tools/replay.cpp` + `src/driver/sync.cpp`, unchanged except the separator rewrite.

**Test gate 1:**
- `python3 linux/build.py replay` builds without errors.
- With a recording (`recordings/qlhs-*.txt` from a Windows session or a GitHub issue) and its calibration JSON:
  `qlhs_replay <log> --calib <json> --dir /tmp/qlhs/ --out /tmp/linux.txt`. The output matches the Windows
  `qlhs_replay.exe` output on the same inputs (identical, or floating-point-level differences only). If no recording
  exists yet, the gate is: the binary runs and prints its usage. Repeat the full gate after chapter 4, which can
  produce a Linux recording.
- `build/linux/src` shows no remaining `"\\` path literals in the rewritten files (grep check built into build.py).

---

### Chapter 2: Network link on Linux (HeadsetLink, discovery, TCP stream, clock)

**Goal:** the driver's network code talks to the real Steam Frame from Linux, with no SteamVR involved.

Steps:
1. Extend the compat shims: `SOCKET`/`INVALID_SOCKET`/`closesocket`/`ioctlsocket(FIONBIO)`, `WSADATA`/`WSAStartup`/
   `WSACleanup` (no-ops), `BOOL`/`u_long`, `Sleep`, `LARGE_INTEGER` + `QueryPerformanceCounter/Frequency` on
   `CLOCK_MONOTONIC` (ns, frequency 1e9). This keeps `QpcNow()` in `net.cpp` unchanged.
2. `linux/patches/net.cpp.patch` (3 small hunks):
   - `Broadcasts()`: `getifaddrs()`. Up, non-loopback IPv4, prefix from the netmask, same /31 and /32 exclusions.
   - Both `select(0, …)` calls → `select(s + 1, …)`.
   - `recvfrom(…, int *fl)` → `socklen_t`.
3. `nettest.cpp`: built through the shims (windows.h → compat).
4. New `linux/tools/linktest.cpp`: runs `HeadsetLink` + `Sync` against the real headset with no HMD poses.
   Arguments `--host`, `--dir`, `--secs`. Prints connection state, model/firmware, calibration result, camera
   frames/s, spots/s and RTT once a second.

**Test gate 2:**
- `qlhs_nettest` against `fake_lhsyncd.py` on localhost (needs a recording): it connects and the fit locks as on
  Windows.
- `linktest` against the **real Steam Frame** (worn, so the cameras run):
  - discovery finds it without `--host`, or with `--host` when the firewall blocks broadcasts (document which);
  - `camera calibration: steam_frame …` is accepted;
  - camera frames/s > 0 and bright spots/s > 0 while base stations are in view;
  - RTT is in the low-ms range, and it reconnects after toggling the Frame's Wi-Fi.

---

### Chapter 3: The driver loads in SteamVR (paths, logging, shared memory, settings), no hooks yet

**Goal:** `driver_questlhsync.so` loads into Linux vrserver, connects to the Frame, classifies devices and publishes
its status. No pose changes yet.

Steps:
1. `linux/patches/driver_main.cpp.patch`:
   - `DataDir()` → `$XDG_DATA_HOME/QuestLHSync` (fallback `~/.local/share/QuestLHSync`). `cfg.dir` gets a
     trailing `/` (for `Sync`'s `Join`).
   - Status shm: `shm_open("/QuestLHSync", O_CREAT|O_RDWR, 0600)` + `ftruncate` + `mmap`. Seqlock with
     `__atomic_fetch_add` + `__atomic_thread_fence` (or the shim's `InterlockedIncrement`/`MemoryBarrier`).
   - `HmdDriverFactory`: `extern "C" __attribute__((visibility("default")))`.
   - `LaunchOverlay()`: no-op that logs "dashboard: chapter 6" for now.
2. MinHook **stub** for this chapter (`linux/src/minhook_stub.c`): `MH_Initialize` returns `MH_OK`, `MH_CreateHook`
   returns an error. `Init` keeps going: the hook failure gets logged, while the link, worker and classification run.
3. `gravity.cpp.patch` (part 1): `openvrpaths.vrpath` from `$XDG_CONFIG_HOME/openvr`. `FindPath()` returns false
   (hidraw comes in chapter 5).
4. `relations.cpp.patch` (part 1): `Hook()` logs "base station averaging isn't available on Linux yet" once and
   returns true.
5. Build: `-fPIC -shared -fvisibility=hidden -static-libstdc++ -static-libgcc -Wl,--no-undefined`
   (`-Wl,-z,defs`). Output: `driver/questlhsync/bin/linux64/driver_questlhsync.so` (git-ignored like `bin/win64`).
   The existing `driver.vrdrivermanifest` and `default.vrsettings` work for both platforms.
6. `linux/install.py` (dev install, mirroring `install.py`): find `vrpathreg.sh` through openvrpaths, refuse while
   `vrserver` runs, `adddriver <repo>/driver/questlhsync`, set `"steamvr": {"activateMultipleDrivers": true}` in
   `steamvr.vrsettings`, and a `remove` subcommand.
7. New `linux/tools/qlhs_status.cpp`: dumps the shm status (state, HMD, headset, fps, stations, log ring) and sends
   commands (`--cmd pause|resume|record-on|record-off|recalibrate`). It's the dashboard stand-in until chapter 6.

**Test gate 3:**
- `vrserver.txt` shows the driver loaded, with no `dlopen` error (an unresolved symbol would show here).
- `~/.local/share/QuestLHSync/questlhsync.log` contains `driver starting`,
  `HMD n: Steam Frame via Steam Link` (or whatever the streamer is named, noted in chapter 0), `device n: lighthouse
  LHB-… -> moved` per base station and tracker, and `Steam Frame at … connected`.
- `qlhs_status` shows the states SEARCHING → CONNECTING → NO_STATIONS (no hook = no station poses yet). The camera
  fps are live. `--cmd record-on` logs `recording to recordings/…`.
- Quitting SteamVR exits cleanly (`QuestLHSync driver stopped`, no hang, no crash dump). Restarting works.
- Tracking is unchanged, so the driver is harmless when loaded.

---

### Chapter 4: Pose hook on Linux (POSIX MinHook port): alignment works

**Goal:** the core feature works: lighthouse devices are moved into the Frame's tracking space.

Steps:
1. `linux/src/minhook/`: the MinHook API (`MH_Initialize/Uninitialize/CreateHook/EnableHook/DisableHook/RemoveHook`)
   for x86-64 Linux:
   - Reuse `third_party/minhook/src/trampoline.c` and `hde/hde64.c` **unchanged**, compiled against the compat
     `windows.h` (they only need types).
   - New `buffer_posix.c`: trampoline/relay blocks within ±2 GB of the target. Free gaps come from `/proc/self/maps`,
     then `mmap(MAP_FIXED_NOREPLACE)`, RWX.
   - New `hook_posix.c`: patching through `mprotect` (page-aligned RW+X → write → restore R+X). There's no thread
     freezing on Linux. To mitigate, write the 5-byte `jmp` with one atomic 8-byte store when the bytes don't cross
     an 8-byte boundary, and pad otherwise (this residual risk gets documented).
   - Keep MinHook's exact x64 layout: target `E9 rel32` → relay `FF 25 00000000 <abs detour>`. That way `UnhookHost`'s
     "is it still ours" check in the unchanged driver code holds.
   - License: MinHook is BSD-2. Keep its header in the new files and add it to `linux/THIRD_PARTY_NOTICES.md`.
2. Replace the stub from chapter 3 with the port. `Relations` stays off.
3. Optional fallback if inline hooking proves unstable on Linux vrserver: patch `HookHost` to vtable patching (the
   `IVRServerDriverHost_00x` vtable slot 1, `mprotect` on its page). This is documented, not built by default.

**Test gate 4:**
- The log shows `hooked IVRServerDriverHost_006::TrackedDevicePoseUpdated` (and `_005` or "shares the hooked
  function").
- `qlhs_status`: the hooked/moved counters rise (if they're published; otherwise check via the log) and the station
  list fills. The state goes ACQUIRING → **LOCKED** within about 30 s of looking at both base stations. The median
  sighting error is similar to Windows (< 0.3°).
- In the headset: controllers and trackers line up with your hands (Steam Link passthrough or the virtual hands
  test). `--cmd pause` makes them drift back to SteamVR's own space, and `resume` realigns them.
- `--cmd recalibrate` resets and it locks again. `state.json`/`stations.json` appear under
  `headsets/<model>-<id>/`.
- 30+ minutes of play with no vrserver crash or stutter. A clean SteamVR shutdown logs `pose hook taken out`.
- Record a session and replay it with the chapter 1 Linux `qlhs_replay` (this closes chapter 1's gate if no Windows
  recording existed).

---

### Chapter 5: Gravity levelling via hidraw (IMU of controllers and trackers)

**Goal:** levelling by the lighthouse devices' accelerometers works with two base stations.

Steps:
1. `gravity.cpp.patch` (part 2), `FindPath()` and `ReadLoop()` reimplemented:
   - Enumerate `/sys/class/hidraw/hidraw*/device/uevent`: `HID_ID=…:000028DE:…` and `HID_UNIQ=<serial>`, or open
     the node and use `ioctl(HIDIOCGRAWUNIQ)`. Match the receiver serial (`Prop_ConnectedWirelessDongle_String`).
     The path goes back in the `std::wstring` the unchanged header asks for.
   - `open(O_RDONLY|O_NONBLOCK)`, `poll(200 ms)`, `read()`. hidraw delivers numbered reports with the report ID
     byte first, as Windows `ReadFile` does, so the existing 0x23/0x24 Watchman parsing stays as is. Each hidraw
     reader gets its own copy of every report, so nothing is taken from SteamVR. There's no
     `HidD_SetNumInputBuffers` equivalent (kernel queue); note it.
2. Permissions: if the chapter 0 udev check showed no read access to Valve hidraw nodes, add
   `linux/package/60-questlhsync-hidraw.rules` (`KERNEL=="hidraw*", ATTRS{idVendor}=="28de", TAG+="uaccess"`).
3. If chapter 0 showed that SteamVR on Linux talks to the receivers through libusb (no hidraw node while it runs),
   gravity reports "receiver not readable on Linux" once and stays off. Three base stations still level through
   the solver.

**Test gate 5:**
- The log shows the receivers found (one line per tracker or controller) and IMU samples arriving (the gravity
  status lines count samples and windows). No `no IMU calibration` errors, which confirms `config/lighthouse/<serial>/
  config.json` is found through the Linux openvrpaths.
- With two base stations, after a few minutes of moving the devices, the log shows the level applied, and
  `gravity.json` appears in the data folder and survives restarts.
- A tracker resting on the floor sits level within about 1 cm across the play space. Compare with `gravity: false`
  in the settings.
- SteamVR's own tracking of the devices isn't disturbed (no dropouts) while QuestLHSync reads.

---

### Chapter 6: Linux dashboard (ImGui + SDL2 + OpenGL) and launch from the driver

**Goal:** the dashboard page in SteamVR's dashboard and on the desktop, matching the Windows page's content and
buttons.

Steps:
1. Dependencies: vendor Dear ImGui at a pinned release in `linux/third_party/imgui/` (core + `imgui_impl_sdl2` +
   `imgui_impl_opengl3`, MIT). Use system SDL2 (`sdl2`/`sdl2-compat`) and libGL. Add OpenVR's
   `bin/linux64/libopenvr_api.so` from the **same SDK version as the headers (2.15.6)** to
   `linux/third_party/openvr/bin/linux64/`. Link with `-Wl,-rpath,'$ORIGIN'`.
2. `linux/dashboard/main.cpp` (binary `QuestLHSync`, next to the driver `.so`):
   - Status: `shm_open("/QuestLHSync")` read-only mapping, the same seqlock read as `ReadStatus`. Commands are written
     like `SendCmd` (`cmd`, then an atomic `cmd_seq++`).
   - UI: a fixed 1280×800 logical page rendered with ImGui into an **FBO texture**. Same structure as
     `overlay_main.cpp::Draw`: header + state pill (colour per state, with the same `StateText` wording, copied),
     the Headset / Base stations / Alignment cards, the 5-line log, and the Pause/Resume, Record and Force
     recalibrate buttons. Same colour palette (`col::*`). Font: a system TTF (Noto Sans / DejaVu via known paths),
     falling back to ImGui's default.
   - Desktop window: SDL2 window, resizable with the page's aspect ratio. The FBO is blitted scaled and mouse
     coordinates are mapped to page coordinates. Closing hides the window (as on Windows).
   - VR: `VR_Init(VRApplication_Overlay)` only once the driver reports an HMD (as on Windows), then
     `CreateDashboardOverlay("questlhsync.dashboard", …)`, width 2.4 m, mouse input, mouse scale 1280×800.
     `SetOverlayTexture` with `TextureType_OpenGL` (`ColorSpace_Gamma`), double-buffered textures. Setting
     `QLHS_OVERLAY_RAW=1` switches to `SetOverlayRaw` if the Nvidia GL interop misbehaves. Overlay mouse events go to
     ImGui (y flipped). The thumbnail icon is a port of `DrawIcon` (pure pixel code, copied).
   - Lifetime: single instance through `flock($XDG_RUNTIME_DIR/questlhsync-dashboard.lock)`. It exits when
     `vrserver` is gone (`/proc/*/comm` scan every second), on `VREvent_Quit` (acknowledged), or on state
     `QLHS_WIRED`.
   - `--preview out.png <locked|acquiring|still|frozen|searching|nohmd|stale|icon>`: offscreen render (hidden SDL
     window/EGL), using a copy of `Sample()`. This is for testing without VR.
3. `driver_main.cpp.patch` (part 2), `LaunchOverlay()`: the lock-file check replaces `OpenMutexW`. The binary is
   located next to the `.so` through `dladdr(&HmdDriverFactoryAnchor)`. Then `posix_spawn` + `setsid`, with a
   cleaned environment: SteamVR's `LD_LIBRARY_PATH`/`LD_PRELOAD` entries removed, `DISPLAY`/`WAYLAND_DISPLAY`/
   `XDG_RUNTIME_DIR` kept.

**Test gate 6:**
- `QuestLHSync --preview` PNGs for each sample state show the same information as Windows' `docs/dashboard.png`
  layout (visual review).
- Started by hand with SteamVR running, the desktop window shows live status. Pause, Record and Recalibrate change
  the driver's state (log lines appear, and `qlhs_status` agrees).
- Started automatically by the driver about 2 s after SteamVR starts (`dashboard app started` in the log), only one
  instance runs.
- In the headset (Steam Link), the QuestLHSync tab shows in the SteamVR dashboard with a crisp page and no blinking.
  The laser pointer's hover and clicks work.
- Quitting SteamVR, or killing vrserver, closes the app within about 1 s. Also check under Wayland and X11 sessions
  on Nvidia.

---

### Chapter 7: Packaging, installation, documentation

**Goal:** an end-user install path on Linux that mirrors the Windows release.

Steps:
1. `linux/release.py` → `out/release-<ver>/QuestLHSync-steamvr-linux-<ver>.tar.gz` containing:
   - `questlhsync/` with `driver.vrdrivermanifest`, `resources/settings/default.vrsettings`, and
     `bin/linux64/{driver_questlhsync.so, QuestLHSync, libopenvr_api.so}`;
   - `install.sh`, `uninstall.sh`, the optional udev rule, LICENSE, THIRD_PARTY_NOTICES.md and
     `linux/THIRD_PARTY_NOTICES.md` (ImGui, MinHook POSIX port, OpenVR).
   - The version comes from `QLHS_RELEASE` in `qlhs_status.h`. It refuses stale builds (same rule as `release.py`).
2. `install.sh` (no root, except the optional udev rule):
   - copy to `~/.local/share/QuestLHSync/questlhsync` (new files next to the old ones, then rename);
   - `vrpathreg.sh adddriver`, after removing other registered `questlhsync` copies (as the Windows installer does);
   - ensure `activateMultipleDrivers` (edit the JSON with python3 and keep a backup);
   - check that SteamVR is closed, check the udev access, and print a firewall hint when discovery fails (it runs the
     chapter 0 probe).

   `uninstall.sh` reverses this. The data folder is kept unless `--purge` is given.
3. `linux/README.md`: requirements (CachyOS/Arch packages, Steam Link + Frame, lighthouse udev), install,
   settings (identical keys), data paths, troubleshooting (firewall/`host`, udev, pressure-vessel, Nvidia/Wayland
   `SDL_VIDEODRIVER=x11`, `QLHS_OVERLAY_RAW`), building, and the upstream merge routine.

**Test gate 7:**
- On a clean state (after `uninstall.sh --purge`): extract the tarball → `install.sh` → start SteamVR. The full
  chapter 4–6 behaviour works with no repo checkout.
- `uninstall.sh` unregisters the driver (`vrpathreg.sh show` no longer lists it), and SteamVR starts normally.
- Reinstalling over an installed version (an update) works while SteamVR is closed.

---

### Chapter 8 (optional, advanced): base station averaging (Relations hook) on Linux

**Goal:** the `steadyStations` feature on Linux. SteamVR places base stations by an average instead of each new
relationship.

#### How the Windows finder works (what has to be reproduced)

The hook target is found from a log line rather than a symbol. When SteamVR moves a station, `vrserver.txt` shows
`Moving base %08X %.0fmm and %.1f deg because of relationship with %08X`. In a disassembler, that string leads to the
function that writes it: the relationship step, `(this, a, b, rel, variance)` with `a`/`b` the stations' serials and
`rel` 7 floats (qx qy qz qw tx ty tz, `a = rel * b`). `Relations::Find` repeats that path at runtime on the loaded
DLL, and every step has an ELF equivalent:

| Step | Windows (`relations.cpp`) | Linux equivalent |
|---|---|---|
| Find the module | `GetModuleHandleW(L"driver_lighthouse.dll")` | `dl_iterate_phdr` → `driver_lighthouse.so` base + `PT_LOAD` segments |
| 1. Find the string | Scan non-executable sections for `kMoving` | Scan segments without `PF_X`, if the Linux build has the same text |
| 2. Find the code that uses it | `lea r64, [rip+disp]` (`48/4C 8D modrm`) with `p + 7 + disp == str` | Same scan in `PF_X` segments (same instruction set). GCC/Clang may use another register or load the address another way |
| 3. Find the function start | Binary search the `.pdata` unwind table (`RUNTIME_FUNCTION`), following chained entries | `PT_GNU_EH_FRAME` / `.eh_frame_hdr`: sorted FDE table, last `initial_location <= ref` |
| 4. Check it's the expected code | `kArgs` in the first 0x40 bytes: `mov r13, r9` (rel), `mov r15d, r8d`, `mov r12d, edx` (serials), the Microsoft x64 argument registers being saved | A new byte pattern from the SysV prologue (`this`=rdi, `a`=esi, `b`=edx, `rel`=rcx, …), read from the Linux disassembly |
| 5. Hook it | MinHook; `Detour` swaps the `rel` pointer for the averaged pose | Chapter 4 MinHook port. This is an internal function, so there's no function table to patch instead |

#### Research procedure (do this before writing code)

Work on the installed `driver_lighthouse.so`
(`~/.steam/steam/steamapps/common/SteamVR/drivers/lighthouse/bin/linux64/`), in this order:

1. **String check:** `strings -a driver_lighthouse.so | grep 'Moving base'`. If it's missing, the feature stays off
   for good; document that and stop.
2. **Symbol check:** Linux libraries often keep more symbol names than Windows DLLs, unless built with
   `-fvisibility=hidden`. Run `nm -D --defined-only … | c++filt | grep -iE 'relation|base'`, and `nm` (full
   `.symtab`, if not stripped). If the relationship step has a visible name, resolve it with `dlsym` or the symbol
   table and skip the string, `lea` and unwind-table search. That's simpler and survives SteamVR updates better.
3. **Static analysis:** load the `.so` into Ghidra (or `objdump -d --no-show-raw-insn`). Follow the string's
   reference to the enclosing function and record:
   - its real start (check for a hot/cold split, see pitfalls);
   - the prologue's argument moves, which become the Linux byte pattern that replaces `kArgs`;
   - the **types** of all arguments and of the return value (integer, pointer, `float`/`double`, struct);
   - that `rel` still points to 7 floats in the order `Average` expects.
4. **Dynamic confirmation:** attach `gdb` to the running `vrserver`, break on the function, and compare the
   registers (serials in `esi`/`edx`, floats behind `rcx`) with the "Moving base" lines in `vrserver.txt`. This may
   need `sysctl kernel.yama.ptrace_scope=0`. If vrserver runs in pressure-vessel (chapter 0), attach from inside the
   container or use its PID as seen from the host.
5. Record the findings (SteamVR version, function offset, prologue bytes, argument types) in `linux/README.md`. Only
   then write the ELF finder.

#### Linux-specific pitfalls

- **Floating-point arguments:** on Windows the 5th argument (`variance`) is passed on the stack, so `Detour`'s
  "8 × 64-bit pass-through" forwards it bit for bit even if it's a `float`. Under SysV, `float`/`double` arguments
  travel in `xmm0`–`xmm7`, which `Average`'s maths overwrites before `g_orig` is called. If any argument (or the
  return value) is floating point, the Linux detour must declare the real types, at least for those arguments.
  Otherwise SteamVR receives garbage.
- **Struct return values:** a struct larger than 16 bytes is returned through a hidden pointer in `rdi`, which shifts
  every argument by one register (`this`=rsi, `a`=edx, …).
- **Hot/cold splitting:** GCC (and Clang with profile-guided optimisation) often moves rarely run code, like log calls,
  into a separate `<fn>.cold` fragment with its own FDE. The `lea` may then be in the cold fragment, and step 3 of the
  finder would return the fragment, not the function. Detect it (the fragment's code doesn't start with a normal
  prologue, and it's reached by a jump rather than a call) and follow that jump back to the main function, or anchor
  the search on a different reference.
- **Inlining:** the Linux compiler may have inlined the relationship step into its caller, so there's no separate
  function to hook. Then another target has to be found (repeat the research on the caller), or the feature stays off.
- **Code layout differs from Windows:** different compiler, flags and register allocation. Don't reuse anything from
  the Windows pattern except the method.

Steps:
1. Research on the installed `driver_lighthouse.so`, following the procedure above. Its outcome decides the finder:
   symbol lookup (step 2 found a name), the string search (string present, no symbol), or off for good (string
   missing, or the step is inlined with no usable caller).
2. `relations.cpp.patch` (part 2), `Find()`/`Hook()` for ELF in memory:
   - `dl_iterate_phdr` → `driver_lighthouse.so` base + `PT_LOAD` segments: the string in non-executable segments,
     the `lea` with rip+disp in executable segments.
   - Function start from `PT_GNU_EH_FRAME` (`.eh_frame_hdr`'s sorted FDE table: binary search for the last
     `initial_location <= ref`), with the `.cold` fragment check. Verify the Linux signature within the first 0x40
     bytes.
   - Hook through the chapter 4 MinHook port. The `Detour` signature (8 × 64-bit pass-through) is only valid under
     SysV if research shows no floating-point arguments, no floating-point return value, and no struct return.
     Otherwise give the Linux `Detour` the real prototype.
3. Keep the graceful fallback: unknown code → the existing "isn't one this version knows" log line, no hook.

**Test gate 8:**
- The log shows `base stations: SteamVR places them by the average of its measurements`, and `relations.json`
  appears and grows.
- `vrserver.txt` "Moving base" distances shrink over a session compared with `steadyStations: false`.
- A 30+ minute session with power-cycled trackers shows no crash.
- After a SteamVR update, the finder either still hooks or logs the fallback (never crashes).

---

### Chapter 9 (recommended): CI and the upstream-sync routine

**Goal:** know immediately when an upstream update breaks the Linux patches.

Steps:
1. `.github/workflows/linux.yml` (a new file, no upstream conflict): on push, on PR and weekly. Ubuntu runner:
   install clang, zig, SDL2 and GL dev packages. Run `linux/build.py all` (patch application + compile). Run
   `qlhs_replay` on a small checked-in fixture recording (`linux/tests/fixtures/`, if one can be shared) and
   `QuestLHSync --preview` under `xvfb-run`. A scheduled job tries `git merge upstream/master` on a temporary branch
   and reports failures.
2. `linux/tools/refresh_patch.py` and the `linux/UPSTREAM.md` routine:
   fetch → merge → build → if a patch fails, fix the staged copy → refresh the patch → commit
   "linux: rebase patches onto upstream <sha>".

**Test gate 9:**
- CI is green on the `linux` branch.
- A dry run: merge an older upstream commit (or a hand-edited hunk) to confirm a broken patch fails CI with a clear
  message.

## Notes and risks

- **Inline hooking without thread freezing** (chapter 4). The patch is written at driver Init, mostly before pose
  traffic, and with atomic stores, but the risk isn't zero. The vtable-patch fallback is documented.
- **Steam Linux Runtime**: if vrserver runs containerized, the driver must not depend on new host libraries. The
  dashboard inherits the container, but SDL2/GL are available there (pressure-vessel passes the Nvidia GL through).
  Chapter 0 decides whether the Zig glibc-2.31 build is needed.
- **hidraw access** depends on SteamVR using hidraw on Linux (chapter 0 checks). Otherwise gravity is off, and three
  base stations still level the frame.
- **Base station hook on Linux** (chapter 8) relies on reverse engineering Valve's closed `driver_lighthouse.so`. The
  Linux build may lack the log string, inline the function, split it into hot/cold parts, or take floating-point
  arguments that the pass-through `Detour` would corrupt. The research procedure in chapter 8 settles this before any
  code. Every SteamVR update can break it, so the finder must always fall back to "off" instead of guessing.
- **Firewall** is the most likely "Looking for the headset" cause on Linux. It's covered by the chapter 0 probe and
  `install.sh`.
- **The Steam Link HMD name on Linux** may differ from Windows. `anyHmd: true` is the escape hatch, and chapter 3
  checks it.
- The headset-side code and all upstream files stay unchanged, so the Windows build and behaviour stay unaffected.
- No time estimates are given, on purpose.
