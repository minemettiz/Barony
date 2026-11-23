# Barony Low-Latency Engine Fix (Zero-Lag Mouselook & True Tick Interpolation)

A lightweight, **100% multiplayer-compatible** engine fix for [Barony](https://github.com/TurningWheel/Barony) (GPLv3) that eliminates camera/mouselook input lag and frame-pacing stutter on 60 Hz, 120 Hz, 144 Hz, and 240 Hz displays — without altering the 50 Hz gameplay simulation or network protocol.

---

## Why Barony Feels Laggy / Stuttery by Default

Even when rendering at 60–240 FPS (`fpsLimit`), vanilla Barony runs a fixed **50 Hz** simulation tick (`TICKS_PER_SECOND = 50`, every 20 ms) and stacks multiple latency sources on top of camera movement:

1. **50 Hz Mouse Sampling:** `SDL_MOUSEMOTION` deltas are accumulated and only applied once every 20 ms tick (`actPlayer` -> `handlePlayerCameraUpdate`), adding up to 20 ms of aim latency.
2. **Velocity Chase Filter + 60 Hz Subclock (`TimerExperiments`):** Instead of interpolating between consecutive simulation ticks, vanilla sets `velocity = 30.0 * (target - current)` and integrates it on a separate **60 Hz** subclock (`dt = 1/60 s`), then lerps again between `previousState` and `currentState`. At `lerpFactor = 30`, the camera only closes 50% of the remaining gap per 1/60 s step — causing **~40–65 ms (3–4 frames)** of elastic trailing lag on both mouselook and movement.
3. **1 ms Coarse Loop Clock:** `handleEvents()` and `TimerExperiments::Clock` use integer-millisecond `SDL_GetTicks()`, introducing frame-pacing jitter at 60/120/144+ Hz.

---

## What This Fix Changes

All gameplay physics, timings, and network packets (`PMOV`, etc.) stay **strictly at 50 Hz** so multiplayer (Steam lobbies, LAN, Direct-IP) remains 100% desync-free with vanilla hosts and clients.

1. **Zero-Lag Native-Hz Mouselook (`/zerolag_mouselook 1`, default ON)**
   - Applies raw `SDL_MOUSEMOTION` (`yaw` / `pitch`) **every render frame** (60 / 120 / 144 / 240 Hz) directly to the camera before rendering, completely bypassing the 50 Hz wait and velocity chase filter.
   - Automatically respects sensitivity, inverted mouse, `EFF_CONFUSED`, immobile/paralyzed states, Quick Turn, Pause/Menus, controller input, and built-in Mouse Smoothing (`smoothmouse`).
2. **True Fixed-Timestep Tick Interpolation (`/trueticklerp 1`, default ON)**
   - Replaces the 60 Hz subclock + `lerpFactor = 30` chase filter with exact linear interpolation between `prevTick` and `currTick` using sub-millisecond `tickAlpha = time_diff / 20.0 ms` (with automatic teleport/floor-change snapping so portals never streak across the map).
   - Cuts interpolation lag from ~40–65 ms down to a crisp, consistent 0–20 ms and stays smooth on high-refresh-rate monitors.
3. **High-Precision Microsecond Loop Timer (`/hires_timer 1`, default ON)**
   - Uses `SDL_GetPerformanceCounter()` instead of 1 ms `SDL_GetTicks()` in `handleEvents()` for smooth frame pacing and exact `tickAlpha`.
4. **Steamworks SDK 1.5x & 1.6x Build Compatibility**
   - Uses C++17 SFINAE shims so the repo compiles cleanly against both older Steamworks SDKs (`IsSteamRunningOnSteamDeck` / `RequestCurrentStats`) and Steamworks SDK 1.61+ (`IsRunningOnSteamHardware` / auto-synced stats).

---

## In-Game Console Commands (Live A/B Testing)

Open the in-game chat/console (`Enter`) and toggle any feature live to compare against vanilla:

| Command | Default | Effect |
| --- | --- | --- |
| `/zerolag_mouselook 1` / `0` | `1` | Toggle native render-rate mouselook |
| `/trueticklerp 1` / `0` | `1` | Toggle true fixed-timestep tick interpolation vs. vanilla 60 Hz chase lerp |
| `/hires_timer 1` / `0` | `1` | Toggle microsecond `SDL_GetPerformanceCounter()` loop timing |

> **Tip:** For lowest latency, make sure **Mouse Smoothing** and **VSync** are disabled in Barony's in-game Settings, and keep **Frame Interpolation** enabled so movement & entities interpolate smoothly at your monitor's refresh rate.

---

## Files Modified (Engine Fix Only)

- `src/game.hpp` — `TimerExperiments` engine-fix state & target helpers
- `src/game.cpp` — Zero-lag mouselook, true tick interpolation, microsecond timer, and CVars
- `src/actplayer.cpp` — Camera target setters & Quick-Turn query helper
- `src/steam_shared.cpp`, `src/init.cpp`, `src/input.cpp`, `src/ui/MainMenu.cpp` — Optional Steamworks SDK 1.5x/1.6x compile compatibility

---

## Building from Source

Follow Barony's standard CMake build instructions in `INSTALL.md` (C++17). To build with Steam multiplayer and original FMOD audio:

- Pass `-DSTEAMWORKS=ON` (with Valve Steamworks SDK + classic `nativefiledialog`) and `-DFMOD_ENABLED=ON` (with FMOD Core API 2.02.14).
- Or build open-source standalone (Direct-IP/LAN) with `-DFMOD_ENABLED=OFF -DSTEAMWORKS=OFF`.

---

## License

This patch is distributed under the **GNU General Public License v3.0 (GPLv3)**, matching upstream [TurningWheel/Barony](https://github.com/TurningWheel/Barony). All original game assets and copyrights belong to Turning Wheel LLC.
