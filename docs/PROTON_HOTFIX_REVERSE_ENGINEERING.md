# Proton Hotfix: reverse engineering of the installed runtime

**Scope.** The copy installed at `X:\SteamLibrary\steamapps\common\Proton Hotfix` on the Windows host, inspected
file-by-file and executed where the Windows host cannot (the Linux binaries were run under WSL). No Proton source was
downloaded, copied, or rebuilt, and nothing in the installation was modified.

**Status of the evidence.** Everything in this page is either read from the installed files or produced by a command
that is quoted with its result. Where a fact could not be established from the installation, the page says so instead
of guessing.

## 1. Build identity

| Fact | Value | Source |
| --- | --- | --- |
| Steam build id | `1785167113` | `version` |
| Build name | `hotfix-20260727-d4` | `version` |
| Wine version | `wine-11.0` | `files/bin/wine --version` under WSL |
| Prefix format | `11.0-100` | `CURRENT_PREFIX_VERSION` in `proton` |
| Tool manifest | `require_tool_appid 4183110`, `compatmanager_layer_name "proton"`, `use_sessions 1` | `toolmanifest.vdf` |
| Installed size | about 1.3 GiB | `du -sh` under WSL |

The build id is a Unix timestamp (2026-07-27), matching the `hotfix-20260727` name. The tree carries no Git metadata,
no `build/` directory and no source tarball, so **the exact upstream Proton commit is not identifiable from the
installation alone**. What can be stated from the files is the Wine base (`11.0`), the prefix format (`11.0-100`), the
DXVK and VKD3D-Proton commits below, and that the build is a Steam-distributed *Hotfix* build. Mapping it to a tag in
`ValveSoftware/Proton` would require either Valve's build metadata or a binary comparison against candidate tags; the
installation does not carry that mapping, so this page does not invent one.

## 2. Directory inventory

```
Proton Hotfix/
  version                     build id and name
  toolmanifest.vdf            Steam tool entry (appid 4183110, layer "proton")
  proton                      Python 3 launcher, ~93 KB
  filelock.py                 prefix lock helper
  user_settings.sample.py     template for PROTON_LOG and friends
  steampipe_fixups.py/.json   SteamPipe installation fixups
  LICENSE                     BSD-3-Clause (Valve)
  LICENSE.OFL, PATENTS.AV1    font and AV1 patent notices
  files/
    bin/                      wine, wineserver, msidb   (Linux ELF)
    lib/
      i386-linux-gnu/         host-side 32-bit libraries (GStreamer, dav1d, ...)
      x86_64-linux-gnu/       host-side 64-bit libraries (GStreamer, dav1d, ...)
      i386-w64-mingw32/       PE side 32-bit
      x86_64-w64-mingw32/     PE side 64-bit
      wine/
        i386-unix/  i386-windows/
        x86_64-unix/ x86_64-windows/
        dxvk/{i386,x86_64}-windows/      DXVK DLLs + version
        vkd3d-proton/{i386,x86_64}-windows/
        vkd3d/                            vkd3d (the older, non-Proton fork)
        nvapi/                            NVAPI shim
      vkd3d/                  host-side vkd3d libraries
    share/
      default_pfx/            seed prefix (system.reg, user.reg, drive_c)
      wine/  fonts/  openxr/  xalia/  espeak-ng-data/  media/
```

`files/bin/wine` and `wineserver` are x86-64 Linux ELF binaries. Their embedded build path is
`/builds/proton/proton/build-dir/dst-wine-x86_64`, and the binary reports `wine-11.0`. The 32-bit and 64-bit Windows
DLL trees are both present, so the installation supports WoW64-style 32-bit applications as well as 64-bit ones.

## 3. Component versions

| Component | Version | Source file |
| --- | --- | --- |
| Wine | `wine-11.0` | `files/bin/wine --version` |
| DXVK | `v3.0.2-5-g1a5919b7` (`1a5919b7edd111887648d1e8bf0c32733e2e00d3`) | `files/lib/wine/dxvk/version` |
| VKD3D-Proton | `vkd3d-1.1-5448-g767f5f02` (`767f5f0222af3341317e752e37ed8d23a0fedbcf`) | `files/lib/wine/vkd3d-proton/version` |
| VKD3D (upstream) | `vkd3d-2.0-246-g8b0b7f4c` (`8b0b7f4ccef27873b0fa464f6623443150c15e70`) | `files/lib/vkd3d/version` |
| GStreamer | 1.22.5 series (`libgst*.so.0.2205.0`) | `files/lib/x86_64-linux-gnu` |
| dav1d | 6.9.0 (`libdav1d.so.6.9.0`) | `files/lib/x86_64-linux-gnu` |

Both DXVK and VKD3D-Proton are **D3D-to-Vulkan translation layers**. They translate Direct3D 8/9/10/11 (DXVK) and
Direct3D 12 (VKD3D-Proton) into Vulkan. They are not a general graphics stack and they do not translate Vulkan calls.

## 4. What the launcher script does

`proton` is a Python 3 program, not a shell wrapper. The parts that matter for an application launched through it:

1. **Session environment.** It requires `STEAM_COMPAT_CLIENT_INSTALL_PATH` and `STEAM_COMPAT_DATA_PATH`, and reads
   `STEAM_COMPAT_INSTALL_PATH`, `STEAM_COMPAT_LIBRARY_PATHS`, `STEAM_COMPAT_APP_ID`, `STEAM_COMPAT_CONFIG` and
   `STEAM_COMPAT_MEDIA_PATH` when present. Missing client/library variables produce an explicit error rather than a
   best-effort guess.
2. **Prefix lifecycle.** `CURRENT_PREFIX_VERSION="11.0-100"`; the prefix lives under the app's compatibility data
   directory (`pfx/`), is created from `files/share/default_pfx` when missing, is upgraded in place when the recorded
   version differs, and is protected by `FileLock` (`pfx.lock`). A prefix is per app id, not global.
3. **Environment construction.** It builds `WINEPREFIX`, `WINESERVER`, `WINEDLLPATH`, `LD_LIBRARY_PATH`, the
   `STEAM_COMPAT_*` passthrough, and a large set of `PROTON_*` switches. `WINEESYNC`/`WINEFSYNC` are set unless
   `PROTON_NO_ESYNC`/`PROTON_NO_FSYNC` is present; `PROTON_CPU_TOPOLOGY` is mapped to `WINE_CPU_TOPOLOGY`;
   `PROTON_USE_WINED3D`, `PROTON_NO_D3D11`, `PROTON_DXVK_D3D8`, `PROTON_DISABLE_NVAPI`, `PROTON_FORCE_NVAPI`,
   `PROTON_HIDE_*_GPU`, `PROTON_SET_GAME_DRIVE` and others are read as feature switches.
4. **Logging.** `PROTON_LOG=1` (or a comma-separated channel list) writes a log next to the game or into
   `PROTON_LOG_DIR`; `PROTON_LOG` also feeds `WINEDEBUG`. DXVK and VKD3D-Proton have their own log variables.
5. **Media.** When `STEAM_COMPAT_MEDIA_PATH` is present it points the media converter at Steam's transcoded audio and
   video; `PROTON_MEDIACONV_NO_VIDEO` disables the video half. `files/lib/x86_64-linux-gnu` ships GStreamer and dav1d
   for that path.
6. **Runtime assumptions.** The script is written to run inside the Steam Linux Runtime (pressure-vessel) container.
   It uses the container's `/usr` and the `STEAM_COMPAT_*` variables that Steam sets; running it directly from a
   desktop shell without those variables is not a supported mode and fails early by design.

## 5. Mechanisms that are relevant to EmblemWeave

### 5.1 What EmblemWeave would actually use

EmblemWeave's engine is a **native Vulkan 1.3 renderer**. It does not issue Direct3D calls. Consequently:

* **DXVK and VKD3D-Proton are not on EmblemWeave's path at all.** They translate APIs the engine never calls. Copying
  them in, or depending on them, would add code and licensing surface with no rendering benefit.
* **Wine's value is the Win32 host surface**: process creation, file and path mapping, window creation, XAudio2,
  Media Foundation, registry, threading primitives, and DLL loading. That is what the Windows build needs on Linux.
* **Vulkan is passed through.** Wine loads the host's `libvulkan.so.1` and hands the application a normal Vulkan
  instance and surface. The Linux driver (Mesa RADV, NVIDIA, Intel) does the rendering; Proton does not emulate a GPU
  for Vulkan applications.

### 5.2 Patterns worth taking, not code

| Mechanism | What EmblemWeave should do |
| --- | --- |
| Prefix-per-app isolation | Keep caches, shader packs and logs under one app-owned directory (`$XDG_CACHE_HOME/EmblemWeave`, `$XDG_CONFIG_HOME/EmblemWeave`), never in the install tree. |
| Explicit environment contract | A Linux launch path should require and validate its variables up front (game data root, cache root, Vulkan loader), and print the missing one by name. |
| Versioned data directory | The prefix has a version and an upgrade path. EmblemWeave's config and shader caches should carry a format version and migrate, not silently reuse an incompatible cache. |
| Feature switches with defaults | `PROTON_*` switches are explicit and documented. EmblemWeave's Linux diagnostics should expose the equivalent (loader override, software device, cache reset) without hidden magic. |
| Log location policy | `PROTON_LOG`/`PROTON_LOG_DIR` put logs where the user asked. The launcher's `platform.rs` already centralises this; the engine should follow the same root. |
| Lock around a shared prefix | `pfx.lock` prevents two sessions from mutating the same prefix. A native port needs the same guard around shader-cache and save directories. |

### 5.3 What must not be taken

* **Wine or Proton source code.** Wine is LGPL-2.1-or-later; Proton's own glue is BSD-3-Clause; DXVK is zlib;
  VKD3D-Proton is MIT; upstream VKD3D is LGPL-2.1-or-later. Any reuse needs the component's own licence review and
  notices. This workstream copies none of it.
* **The installed binaries as a redistributable.** Proton is distributed by Valve through Steam under its own terms
  and includes third-party components with their own licences. EmblemWeave must not bundle the installed tree; the
  supported route is "use the Proton that Steam installed".
* **Proprietary drivers or NVAPI.** The NVAPI shim is NVIDIA-specific and not redistributable as part of an
  application. EmblemWeave's engine already has a vendor-neutral DLSS replacement and must not depend on NVAPI.
* **Pressure-vessel.** It is Steam Runtime infrastructure, not an application library. Do not embed it.

## 6. Licensing summary

| Component | Licence (upstream) | Redistribution note |
| --- | --- | --- |
| Proton glue / scripts | BSD-3-Clause (Valve) | Keep the notice if any part is reused. |
| Wine | LGPL-2.1-or-later | Dynamic linking and notice obligations; do not statically absorb into a proprietary binary. |
| DXVK | zlib | Permissive with notice; irrelevant to EmblemWeave's Vulkan path. |
| VKD3D-Proton | MIT | Permissive with notice; irrelevant to EmblemWeave's Vulkan path. |
| VKD3D (upstream) | LGPL-2.1-or-later | Same as Wine. |
| GStreamer / dav1d | LGPL-2.1-or-later / BSD-2-Clause | Used by Proton's media path; EmblemWeave already has libvpx/dav1d options. |

The installation itself contains only `LICENSE` (BSD-3-Clause), `LICENSE.OFL` and `PATENTS.AV1` at the top level; the
per-component licence texts live in the upstream projects, not in the Steam depot. That is one more reason not to
redistribute the tree.

## 7. Execution evidence

The Windows host cannot execute the Linux ELF binaries. Under WSL2 Ubuntu 26.04 the installed Wine does run:

```
$ ./files/bin/wine --version
wine-11.0
```

A first `wine cmd /c echo` in a fresh, isolated prefix created the prefix (`drive_c`, `system.reg`, `user.reg`,
`dosdevices`) but did not complete: `wineboot.exe --init` and `winedevice.exe` stayed alive past a 180-second timeout
and the command was killed. This is consistent with the script's own requirement to run inside the Steam Linux
Runtime and with a WSL environment that has no Steam client, no pressure-vessel container and no GPU driver beyond
Mesa's software lavapipe. **It is not evidence that Proton is broken, and it is not evidence that EmblemWeave runs
under it.** The correct conclusion is: Wine itself starts and reports its version; a full Proton session was not
established in this environment, so Proton-assisted EmblemWeave remains untested.

The Vulkan side of the same environment was probed separately and does work:

```
vulkan loader: libvulkan.so.1 opened
loader api version: 1.4.341
vulkan devices: 1
  [0] llvmpipe (LLVM 21.1.8, 256 bits) | cpu | api 1.4.335 ...
```

## 8. Conclusions

1. The installed runtime is **Wine 11.0 plus DXVK 3.0.2 and VKD3D-Proton** in a Steam-managed Hotfix build
   (`hotfix-20260727-d4`, build 1785167113, prefix 11.0-100).
2. Its graphics translation layers are **irrelevant to a Vulkan application**. EmblemWeave would use Wine's Win32
   surface and Vulkan passthrough, nothing else.
3. The exact upstream Proton commit **cannot be determined from the installation**; the identifiable facts are the
   Wine base, prefix format and component commits listed above.
4. Proton is usable only as an **external compatibility environment** launched by Steam. It is not a library to link
   into EmblemWeave, and this workstream does not fork or rebuild it.
5. A Proton-assisted EmblemWeave run is **plausible but unverified**: the Windows launcher and engine are ordinary
   Win32/Vulkan programs, but no full session was established here, and a launcher that starts is not a game that
   plays.
