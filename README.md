# playdead-INSIDE-FreeCam

A Cheat Engine table that adds a **true 360° free camera** to Playdead's **INSIDE** (2016, Unity 5.0.4, x64 Mono build). Camera position and orientation are fully user-controlled, **view-relative** movement is implemented through Unity's native `Transform.Translate` and `Transform.Rotate` APIs, and game camera scripts are neutralised via targeted JIT NOP-patches so your rotation and position are never overwritten by the game. Works with keyboard + mouse and Xbox-compatible gamepads.

![INSIDE](https://upload.wikimedia.org/wikipedia/en/thumb/5/58/INSIDE_Cover_Art.jpg/220px-INSIDE_Cover_Art.jpg)

---

## Table of contents

- [What this does](#what-this-does)
- [Requirements](#requirements)
- [Installation](#installation)
- [Controls](#controls)
- [Quick reference card](#quick-reference-card)
- [Technical overview](#technical-overview)
- [How it works (deep dive)](#how-it-works-deep-dive)
  - [1. cam struct NOP patches](#1-cam-struct-nop-patches-camerablendprobeupdateweightsposition)
  - [2. JIT NOP of `set_rotation` for 360° rotation](#2-jit-nop-of-set_rotation-for-360-rotation)
  - [3. JIT NOP of `set_position` for view-relative Translate](#3-jit-nop-of-set_position-for-view-relative-translate)
  - [4. Gamepad global listener entry](#4-gamepad-global-listener-entry-lb--freecam-rb--slow-mo)
  - [5. Auto-attach and auto-activate](#5-auto-attach-and-auto-activate)
- [Known limitations](#known-limitations)
- [Roadmap / deferred work](#roadmap--deferred-work)
- [Extending / modifying the table](#extending--modifying-the-table)
- [Troubleshooting](#troubleshooting)
- [Credits](#credits)
- [License](#license)

---

## What this does

INSIDE ships with a hard-coded orbit camera that tracks the Boy along a fixed axis. This table gives you a genuine flying camera:

- **Real 360° yaw rotation** - look behind the scene, spin around the player, view locations from any angle
- **View-relative movement** - WASD / left-stick-analog / triggers always move the camera along its own facing direction, regardless of how you have rotated it
- **Slow-motion pause** - `` `~ `` or RB freezes the Boy at `timeScale = 0.001` while the camera stays fully responsive (possible because our camera writes bypass Unity's Update/deltaTime pipeline)
- **Gamepad parity with keyboard** - `LB` is identical to `F1`, `RB` is identical to `` `~ ``, everything else mapped sensibly on an Xbox-layout controller
- **Crash-resistant** - auto-releases NOPs on level transitions (so Unity streaming works), re-resolves stale `Transform` pointers, pcall-wraps every Mono invoke
- **Cross-session resilient** - if Cheat Engine is closed without deactivating the table first (leaving NOPs in game memory), the next run of the table detects the leftover NOP pattern and continues working instead of bailing out

No Boy controls are touched - the left thumbstick, jump, grab, etc. all work exactly as the game intends.

## Requirements

| | |
|---|---|
| Game | **INSIDE** (Steam / GOG), Playdead, 2016 |
| Game binary | `INSIDE.exe`, x64, built with Unity 5.0.4 + Mono 2.0 |
| Cheat Engine | **7.5 or newer** |
| OS | Windows 10 / 11 |
| Input | Keyboard + mouse or any Xbox-compatible controller (XInput) |

Tested on Steam's retail release. The table makes no changes to game files - all patching is in-memory via Cheat Engine.

## Installation

1. Install [Cheat Engine 7.5+](https://www.cheatengine.org/downloads.php) (official release, not a fork).
2. Download `INSIDE_Gamepad_FreeCam.CT` from the [latest release](https://github.com/sashaok123/playdead-INSIDE-FreeCam/releases/latest).
3. Launch INSIDE. Wait for the main menu, then start or load a game and **reach an actual in-level screen** (not main menu, not loading - Unity must have JIT-compiled `CameraBlendProbe` and `CameraScript`).
4. Double-click `INSIDE_Gamepad_FreeCam.CT`. Cheat Engine will open with the table loaded and auto-attach to the running `INSIDE` process (no need to pick the process manually).
5. On first open, Cheat Engine will warn about running a Lua script from a table - click **Yes / Run**. This is required to auto-activate the gamepad listener entry.
6. The status bar should show `[Gamepad Listener] running (LB=FreeCam, RB=Pause)` in the Lua console. You are ready.

If Cheat Engine does not auto-attach, open `Process List` (the top-left computer icon), pick `INSIDE.exe`, then proceed.

## Controls

### Keyboard + mouse

| Input | Action |
|---|---|
| **F1** | Toggle FreeCam on / off |
| **`` `~ ``** (tilde) | Toggle slow-mo pause (`timeScale = 0.001`) |
| **RMB + mouse** | Aim-pan while held (legacy 2D aim-point move; largely redundant once rotation is available, kept as a quick-look tool) |
| **Arrow keys** | Aim-pan (legacy) |
| **W / S** | Move camera forward / back, view-relative |
| **A / D** | Strafe left / right, view-relative |
| **Q / E** | Fly down / up, world-vertical |
| **HOLD R** | Track Boy - releases NOP patches so the game updates camera position toward the Boy; release to re-freeze |
| **Shift** | x3 speed modifier |
| **Ctrl** | x5 speed modifier |
| **Shift + Ctrl** | x15 speed (for crossing entire levels quickly) |

### Gamepad (Xbox / XInput)

| Input | Action |
|---|---|
| **Left Stick** | Boy movement - NOT touched by FreeCam |
| **Right Stick X** | Strafe left / right (view-relative) |
| **Right Stick Y** | Fly down / up (world-vertical) |
| **D-Pad L / R** | 360° yaw rotation (horizontal only, no head tilt) |
| **D-Pad U / D** | Fly up / down (alt) |
| **LT / RT** | Move back / forward (view-relative) |
| **LB** | Toggle FreeCam on / off (same as F1) |
| **RB** | Toggle slow-mo pause (same as `` `~ ``) |
| **HOLD Y** | Track Boy (same as HOLD R on keyboard) |
| **A / B / X** | Unused / game-controlled (typically jump, grab, interact) |

The **Gamepad Listener** is a background entry kept enabled at all times. It polls the controller at 50 ms intervals and toggles the FreeCam / Time Pause entries on LB / RB edge events. Works even when FreeCam itself is off - so LB genuinely flips the camera on and off, not just turns it off.

## Quick reference card

```
  ┌─────────────── KEYBOARD ───────────────┐   ┌─────────────── GAMEPAD ───────────────┐
  │  F1        toggle FreeCam               │   │  LB        toggle FreeCam             │
  │  ` (tilde) toggle slow-mo pause         │   │  RB        toggle slow-mo pause       │
  │                                         │   │                                       │
  │  W / S     forward / back   (view-rel)  │   │  RStick X  strafe L / R   (view-rel)  │
  │  A / D     strafe  L / R    (view-rel)  │   │  RStick Y  fly down / up  (world Y)   │
  │  Q / E     fly down / up    (world Y)   │   │  DPad L/R  yaw rotation 360°          │
  │  RMB+mouse aim-pan (2D legacy)          │   │  DPad U/D  fly up / down  (alt)       │
  │                                         │   │  LT / RT   back / forward (view-rel)  │
  │  Shift     x3 speed                     │   │  Y (hold)  track Boy                  │
  │  Ctrl      x5 speed                     │   │  LStick    Boy (untouched)            │
  │  R (hold)  track Boy                    │   └───────────────────────────────────────┘
  └─────────────────────────────────────────┘
```

## Technical overview

INSIDE's camera is driven by a chain of Unity MonoBehaviours in `Assembly-CSharp-firstpass.dll`:

```
CameraBlendProbe.UpdateWeightsPosition()   // per-fixed-update, writes cameraPosition / aimPosition in the probe struct
        ↓
CameraScript.FixedUpdateCamera()           // reads probe, lerps blend state
        ↓
CameraScript.MoveCamera()                  // per-frame, writes Camera.main.transform.position + .rotation
        ↓
Unity renders the frame
```

To take control of the camera we need to:

1. **Stop the probe from overwriting the position fields** we want to hold steady (`cam+0xF8 / 0xFC / 0x100`). → NOP three write instructions inside `CameraBlendProbe.UpdateWeightsPosition`.
2. **Stop `CameraScript.MoveCamera` from overwriting `transform.rotation`** after we have rotated via `Transform.Rotate`. → NOP the specific `call [set_rotation]` instruction inside the JIT-compiled body.
3. **Stop `CameraScript.MoveCamera` from overwriting `transform.position`** after we have translated via `Transform.Translate`. → NOP the specific `call [set_position]` instruction in the same method.
4. **Drive the camera** via Mono reflection, calling Unity's own `Transform.Rotate(x, y, z)` and `Transform.Translate(x, y, z)` with three primitive `single` arguments (note: Vector3-argument overloads do NOT marshal correctly through Cheat Engine's Lua - we had to enumerate all overloads and pick the 3-float one).

This hybrid approach - write blockers in native machine code plus movement through the managed Unity API - gives us camera control that is completely independent of the game's Time.deltaTime, Unity's update cycle, and the game's rotation / position invariants.

## How it works (deep dive)

### 1. cam struct NOP patches (`CameraBlendProbe.UpdateWeightsPosition`)

Three instructions inside this method write to the probe's position / aim fields. They sit at symbol offsets:

| Offset from method start | Original bytes (10 / 10 / 7) | What it writes |
|---|---|---|
| `+0x429` | `89 48 08 48 8D 86 F0 00 00 00` | zoom / depth + `lea rax,[rsi+F0]` follow-up |
| `+0x41F` | `48 89 08 48 63 8D 18 FF FF FF` | Y / Z |
| `+0x5F0` | `48 89 08 48 63 4D 88` | yaw / pitch components |

We overwrite them with `0x90` NOPs on ENABLE, restore the original bytes on DISABLE. Detection is idempotent - if the scan sees `0x90 0x90 ...` already in place (left over from a previous Cheat Engine session that was closed without deactivation) it still accepts the site and records a canonical restore pattern, so DISABLE remains safe.

These addresses were identified from [abarichello/inside-noclip](https://github.com/abarichello/inside-noclip) and have been stable across every Steam build of INSIDE we have tested.

### 2. JIT NOP of `set_rotation` for 360° rotation

The key problem with simply writing to `transform.rotation` or `Transform.eulerAngles` from Cheat Engine is that `CameraScript.MoveCamera()` overwrites it every frame by calling `LookRotation(blendAim - blendCamera, blendUp)` and assigning the result. We could not find a clean way to stop that short of disabling the entire `CameraScript`, which breaks other things (it also sets position, applies handshake, etc.).

Solution: surgically NOP **only** the single `set_rotation` call inside `MoveCamera`, leaving the rest of the method to keep running.

The JIT output for the line

```csharp
this._transform.rotation = quaternion;
```

compiles (on Mono 2.0 / x64) to:

```
mov  r11, <absolute address of Transform.set_rotation JIT thunk>   ; 49 BB XX XX XX XX XX XX XX XX   (10 bytes)
call r11                                                           ; 41 FF D3                         (3 bytes, REX.B + call r11)
```

On ENABLE the table:

1. Calls `mono_compile_method(set_rotation MonoMethod*)` to obtain the JIT address of `Transform.set_rotation`.
2. Scans the first 16 KB of `CameraScript:MoveCamera` JIT body for the pattern `49 B8-BF <imm64 matching set_rotation>` followed by `41 FF D0-D7` (or the 2-byte form `FF D0-D7` for rax-rdi).
3. Writes 3 × `0x90` over the `call r11` (or 2 × `0x90` for the non-REX form), keeping the `mov r11, imm64` in place (harmless, just loads the address into a register nobody reads).

After this patch, `MoveCamera` computes the quaternion, *almost* writes it, and then slides past the NOPs without doing the store. Our `Transform.Rotate(0, dYaw, 0)` calls from the tick timer are then the only source of rotation changes on `Camera.main.transform`, giving us true 360° yaw.

Cross-session safety: if the scan finds `0x90 0x90 0x90` where `41 FF D3` should be (previous session's leftover), it registers the address with canonical restore bytes `{0x41, 0xFF, 0xD3}` so DISABLE puts the camera system back. This means no "game camera forever frozen after CE crash" class of bug.

### 3. JIT NOP of `set_position` for view-relative Translate

Identical pattern, one instruction earlier in the same method (`this._transform.position = vector;`). With this NOP applied, `Transform.Translate(localX, localY, localZ)` - called once per tick with the aggregated delta from keyboard, right stick, triggers, and D-Pad U/D - becomes the only writer of camera position.

Because `Translate` with three float args defaults to `Space.Self`, movement is automatically view-relative. `Translate(0, 1, 0)` pushes the camera along its own local +Y regardless of yaw. Since we never let the camera roll or pitch (D-Pad is yaw-only, no tilt), local Y equals world Y, so vertical input feels natural.

### 4. Gamepad global listener entry (LB → FreeCam, RB → slow-mo)

The main FreeCam `tick()` only runs while the FreeCam entry is active, so any gamepad key handled there can only *deactivate* the camera, not activate it. To give LB full on/off parity with F1, we introduced a separate always-active entry that:

- Runs its own `createTimer` at 50 ms
- Reads `getXBox360ControllerState(0)` each tick (this is Cheat Engine's built-in XInput wrapper - no Mono, no game interaction, no crash risk)
- On LB edge (rising transition) calls `getAddressList().getMemoryRecordByDescription('ENABLE FreeCam (toggle: F1 / LB)').Active = not Active`
- Same for RB and the Time Pause entry

The `<CheatTable>` root has a `<LuaScript>` block that activates this listener on CT load, so the user never has to tick the checkbox. On `TargetProcess=INSIDE` auto-attach, the whole pipeline is hands-free from opening the `.CT` file.

### 5. Auto-attach and auto-activate

```xml
<CheatTable CheatEngineTableVersion="45">
  <TargetProcess>INSIDE</TargetProcess>
  <LuaScript>
    -- Activates the Gamepad Listener entry on table load
  </LuaScript>
  ...
</CheatTable>
```

`TargetProcess` tells Cheat Engine to attach to `INSIDE` (no `.exe` suffix) when the table opens. If the process is not running, attach is deferred - as soon as INSIDE starts, Cheat Engine will pick it up.

The top-level `<LuaScript>` runs once at table load, finds the Gamepad Listener by description, and sets `Active = true`. Cheat Engine prompts the user once per session to confirm Lua execution - after that, everything is automatic.

## Known limitations

- **Camera-relative Boy controls** (stick forward → Boy runs into the screen when camera is rotated 90°) is **not** implemented. Designed and researched (see [`docs/research-notes.md`](docs/research-notes.md)) but requires an AA trampoline hook into `GameInput.Core.UpdateCommand` that is several hours of careful low-level work. Deferred.
- **Rotation is yaw-only** by design - D-Pad up/down moves the camera vertically instead of pitching. This prevents the horizon from tilting and makes the camera much easier to control in a 2.5D game. If you want pitch back, see [Extending](#extending--modifying-the-table).
- **Aim-pan (RMB + mouse, arrows, legacy)** still writes to `cam+0xF0 / 0xF4` but those fields no longer affect the rendered camera because `set_rotation` is NOPed - kept mostly for legacy compat.
- **Game scripts that call `GameObject.SetActive(Camera)` or swap cameras** (cinematic sequences, some puzzle rooms) may temporarily override the FreeCam. This is expected - the table is designed for exploration, not cutscene replacement. Toggle FreeCam off if the scripted sequence misbehaves.
- **Very fast level transitions** may leave the cached `Transform` pointer stale for one frame; the table detects this and re-resolves automatically, but you may see a one-frame teleport in rare cases.

## Roadmap / deferred work

Tracked in [`docs/research-notes.md`](docs/research-notes.md). Highlights:

- **Camera-relative Boy movement** (AA trampoline in `GameInput.Core.UpdateCommand`, ~4–5 hours)
- Save/load camera rotation (quaternion) in bookmarks
- FOV control (`Camera.main.fieldOfView` via Mono)
- Record/replay camera path for flythrough videos
- Per-state exceptions for the Boy-input rotation (skip in Swim / Ladder / Rope / Grab / ClimbDown states)

## Extending / modifying the table

All logic lives in Lua inside the `<AssemblerScript>` CDATA blocks of four entries:

| ID | Description | Purpose |
|---|---|---|
| 2 | `ENABLE FreeCam (toggle: F1 / LB)` | main script - NOP patches, tick timer, movement logic, rotation |
| 3 | `--- Camera state (read-only) ---` group | live read-only view of yaw / pitch / X / Y / Z for debugging |
| 9 | `Toggle Time Pause (toggle: `~ / RB)` | `Time.timeScale = 0.001` toggle via Mono |
| 12 | `Gamepad toggle listener: LB=FreeCam, RB=Pause (keep enabled)` | background 50 ms polling timer |

Everything is self-contained in the single `.CT` file - no external Lua modules, no side files, no DLL injection.

**Common mods:**

- **Restore D-Pad pitch**: in entry #2 tick, find the D-Pad yaw block and add `dPitch` handling before the `Rotate` call (there's a commented-out version in git history).
- **Change speeds**: adjust the `MOUSE_SENS`, `PAN`, `ZOOM`, `GP_ROT`, `GP_PAN`, `GP_ZOOM`, `GP_DPAD_ROT`, `KEY_ROT` constants near the top of the ENABLE block.
- **Different toggle keys**: edit the XML `<Hotkey><Key>N</Key></Hotkey>` VK-code inside entries #2 and #9. [VK code table](https://learn.microsoft.com/en-us/windows/win32/inputdev/virtual-key-codes).
- **Different gamepad bindings**: in entry #12 the listener checks `gp.GAMEPAD_LEFT_SHOULDER` / `GAMEPAD_RIGHT_SHOULDER` - swap those for any member of `getXBox360ControllerState()` (`GAMEPAD_A`, `GAMEPAD_Y`, `GAMEPAD_DPAD_*`, etc.).

## Troubleshooting

### `Library Injection failed` / `cannot resolve CameraBlendProbe:UpdateWeightsPosition+429`

You tried to enable the table before Unity had a chance to JIT-compile the camera classes. **Start or load a save, reach an actual in-level screen**, then try again. Main menu and loading screens are too early.

### Camera doesn't rotate, D-Pad only slides in 2D

The `set_rotation` NOP did not apply. Possible causes:
- JIT body layout changed (new game update). Check the ROTATION ANALYSIS section of the Lua console - if you see `RESULT: 0 matches to set_rotation`, the scan pattern needs updating. Please open an issue with the console output.
- You reloaded the table without closing CE cleanly before; the re-use path should handle this, but if you see repeated `[REUSE]` messages and rotation still doesn't work, restart both Cheat Engine and INSIDE.

### Slow-mo toggles but nothing happens to the game

`UnityEngine.Time.set_timeScale` could not be found. Reach an in-level screen first - in pure main menu Unity sometimes has not loaded the engine bindings yet.

### LB / RB do nothing

Make sure the `Gamepad toggle listener` entry has its checkbox ✓. On first open of the CT, Cheat Engine asks once whether to run the embedded Lua - you must accept. After that the listener auto-activates each session.

Also check in `GAMEPAD DIAGNOSTIC` (older builds of this table) or by testing another XInput tool that your controller actually enumerates as `XInput slot 0`. The listener only polls slot 0.

### Cheat Engine crashes INSIDE on enable

Make sure INSIDE has been running for a few seconds before you enable. The very first JIT pass for any class can stall Mono for ~100 ms, and stacking our `mono_compile_method` calls on top of that can occasionally race. Wait, retry.

### "Game camera frozen" after Cheat Engine crashed

Close and re-open Cheat Engine, then open the CT and **activate then deactivate** the FreeCam entry once. The DISABLE block restores the original `set_rotation` / `set_position` call bytes even if the previous session left them NOPed. As of v1.0 this is handled automatically.

## Credits

- **abarichello** - original [inside-noclip](https://github.com/abarichello/inside-noclip) Cheat Engine table, which pinpointed the three `CameraBlendProbe` NOP sites this project builds on.
- **Dark Byte** and the Cheat Engine community - for `mono_invoke_method`, `mono_compile_method`, and the Mono Lua bindings that make JIT-aware hooking from Lua tractable.
- **Playdead** - for INSIDE, a remarkable game.

## License

[The Unlicense](LICENSE) - this is free and unencumbered software released into the public domain. Do whatever you want.
