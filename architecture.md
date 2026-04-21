# Architecture

Deep-dive into how the table patches INSIDE's camera pipeline. Read the README first for a high-level summary.

## Camera pipeline in INSIDE

Per-frame execution, mostly in `Assembly-CSharp-firstpass.dll`:

```
FixedUpdate (50 Hz)
  └─ CameraBlendProbe.FixedUpdate
       └─ CameraBlendProbe.UpdateCamera
            └─ CameraBlendProbe.UpdateWeightsPosition
                 └─ WRITES to probe fields: cameraPosition, aimPosition   ← we NOP these writes
  └─ CameraScript.FixedUpdate
       └─ CameraScript.FixedUpdateCamera
            └─ CameraScript.UpdateBlendPosition
                 └─ READS probe, lerps blend state into positionCamera / positionAim
            └─ CameraScript.SmoothMovement
                 └─ writes blendCameraPosition / blendAimPosition

Update (vsync Hz)
  └─ CameraScript.Update
       └─ CameraScript.UpdateCamera
            └─ CameraScript.MoveCamera
                 ├─ quaternion = Quaternion.LookRotation(blendAim − blendCam, blendUp)
                 ├─ apply handshake / shake offsets
                 ├─ this._transform.position = vector;        ← we NOP the set_position call
                 └─ this._transform.rotation = quaternion;    ← we NOP the set_rotation call

Render
  └─ Unity draws Camera.main
```

We hook at three levels:

1. **Probe-level write blockers** (native machine code NOPs inside `UpdateWeightsPosition`) - stops the game from regenerating probe state each frame.
2. **Transform write blockers** (JIT NOPs of the `call [set_rotation]` and `call [set_position]` inside `MoveCamera`) - stops the game from applying its computed final pose each frame.
3. **User-driven writes** (Mono reflection calls to `Transform.Rotate(float,float,float)` and `Transform.Translate(float,float,float)` from our timer tick) - our camera inputs are now the only source of transform changes.

## cam struct layout at the pointer-chain tail

The cam struct is reached via `INSIDE.exe + 0xF8D6B0 → [0] → [+0x6B8] → [+0x28] → [deref + 0x58] → deref`. The final pointer targets a 32-byte-plus chunk where the interesting floats live at:

| Offset | Name | Write pathway (in this table) | Observed in side-view |
|---|---|---|---|
| `+0xF0` | aim X / yaw component | mouse RMB, arrows, right stick (legacy - no visible effect after set_rotation NOP) | yaw-like rotation |
| `+0xF4` | aim Y / pitch component | mouse RMB, arrows (legacy - no effect) | pitch-like tilt |
| `+0xF8` | world X (depth) | `Translate(localX=?, 0, localZ=1)` → writes here through game's probe → final position | zoom in/out of scene |
| `+0xFC` | world Z (lateral) | same | horizontal pan |
| `+0x100` | world Y (vertical) | same | vertical fly |

Important: abarichello's original CT labels these as "X / Z / Y" but diagnostic tests show the coordinate role shifts after camera rotation, which is why we moved from direct world-axis writes to `Transform.Translate` with local-space deltas.

## The three machine code patches

All three are applied in the ENABLE script. All three are restored in DISABLE. All three are idempotent - re-running ENABLE on already-patched memory is safe.

### Patch 1, 2, 3: `CameraBlendProbe.UpdateWeightsPosition`

Three write sites at known offsets (see `NOP_SITES` table in the ENABLE Lua). We overwrite the exact original bytes (10, 10, 7 bytes) with `0x90` (NOP) each.

Role: prevents the probe from regenerating `cameraPosition` / `aimPosition` each fixed-update. Without this, the probe would push the camera back toward its "follow Boy" heuristic.

Restore: write the saved original bytes back on DISABLE.

### Patch 4: `call [set_rotation]` in `CameraScript.MoveCamera`

Located by:

1. `mono_compile_method(mono_class_findMethod(Transform, 'set_rotation'))` - returns the JIT entry of the stub that calls Unity's native `Transform::INTERNAL_set_rotation`.
2. Scan first 16 KB of `getAddress('CameraScript:MoveCamera')` for `0x48/0x49 0xB8-BF <imm64 matching set_rotation JIT>` (the `mov r64, imm64` that loads the thunk address into a register).
3. Check the 2 or 3 bytes immediately after the 10-byte `mov`:
   - `0xFF 0xD0..0xD7` - `call rax..rdi` (2 bytes) → NOP 2 × `0x90`
   - `0x41 0xFF 0xD0..0xD7` - REX.B + `call r8..r15` (3 bytes) → NOP 3 × `0x90`
4. Save the original 2 or 3 bytes for DISABLE restore.

In the INSIDE builds we tested, Mono emits `mov r11, imm64 ; call r11` → our NOP is 3 bytes at offset `+0xA7D`.

Role: `MoveCamera` computes the LookRotation quaternion every frame regardless; our NOP just stops the final `set_rotation` call from applying it. Our `Transform.Rotate` from the tick timer becomes the only mutator of `camera.transform.rotation`.

Cross-session safety: if scan finds `0x90 0x90 0x90` at the expected position (leftover from a crashed or non-deactivated CE session), the scan marks the site as "already applied" and saves canonical `{0x41, 0xFF, 0xD3}` as the restore bytes. DISABLE then puts the game's rotation system back to default. (`0xD3` = r11; this is the register emitted by every Mono 2.0 build we have seen, so the canonical restore is almost always byte-perfect.)

### Patch 5: `call [set_position]` in `CameraScript.MoveCamera`

Identical pattern, one instruction earlier in the same method. Located by scanning for the `mov r64, imm64` of `set_position`'s JIT thunk and checking the following call.

Role: makes `Transform.Translate(x, y, z)` from our timer the only mutator of `camera.transform.position`. Combined with patch 4, this gives us both position and rotation fully under CE control.

## The timer tick

Created in ENABLE: `createTimer(getMainForm())` at 16 ms. Each tick:

1. Read gamepad via `getXBox360ControllerState(0)` (Cheat Engine's built-in XInput wrapper).
2. Resolve cam struct via the pointer chain. If nil (level transitioning): auto-release the `CameraBlendProbe` NOPs so Unity's streaming can position everything correctly, mark a `SETTLE_UNTIL` timestamp, return.
3. If just came back from nil: re-resolve `Camera.main.transform` (because a level change can invalidate the cached managed object pointer), re-apply the `CameraBlendProbe` NOPs, proceed.
4. Handle R / Y (track Boy): temporarily release the `CameraBlendProbe` NOPs so the game snaps camera to Boy; restore when released.
5. Read all movement inputs (WASD, triggers, sticks, D-Pad U/D) into an aggregated `(localX, localY, localZ)` vector. Read D-Pad L/R into `dYaw`.
6. If `dYaw != 0`: call `Transform.Rotate(0, dYaw, 0)` via `mono_invoke_method` (pcall-wrapped).
7. If any of `localX / Y / Z` is non-zero: call `Transform.Translate(localX, localY, localZ)` via `mono_invoke_method` (pcall-wrapped). Space.Self is the default for the three-float overload, so movement is relative to the camera's current rotation.

Only **two** `mono_invoke_method` calls per tick maximum, and only when the user actually inputs something. Mono GC pressure stays acceptable.

## Why Mono reflection for the camera writes (and not direct memory pokes)

Directly writing the four quaternion floats + three position floats to the cached managed Transform's backing store would require knowing the native `m_CachedPtr` offset and the native Transform layout. Those offsets are Unity-version-specific and not documented for 5.0.4. We tried; it's a 1-2-day reverse engineering detour.

Invoking `Transform.Rotate(x, y, z)` and `Transform.Translate(x, y, z)` through Mono is robust because Unity maintains API stability for these methods across decades of engine versions. The only catch was finding the right overload - `mono_class_findMethod('Transform', 'Rotate')` returns the first match by name, which tends to be `Rotate(Vector3)` and **Vector3 does not marshal correctly through Cheat Engine 7.5's `mono_invoke_method`**. Enumerating overloads and filtering by `mono_method_getSignature() == 'single,single,single'` picks the right entry.

## Independence from `Time.timeScale`

Because our camera writes go through Mono reflection into Unity's native transform code (not through the game's Update → probe → MoveCamera chain), they do not depend on `Time.deltaTime`. This is why we can set `timeScale = 0.001` for slow-mo pause and still have full camera control. If we were writing to the probe's `cameraPosition` float and relying on `MoveCamera` to pick it up, `timeScale = 0` would freeze everything.

## Gamepad listener entry (ID=12)

Separate entry with its own timer, runs at 50 ms. Always active. Edge-triggers on `GAMEPAD_LEFT_SHOULDER` / `GAMEPAD_RIGHT_SHOULDER` transitions and toggles the FreeCam and Time Pause entries via `getAddressList().getMemoryRecordByDescription(...).Active = not Active`.

Why separate: the FreeCam tick only runs while FreeCam is active, so it cannot detect a press that wants to *enable* FreeCam. The listener runs independently of FreeCam state, giving LB full parity with F1.

Auto-activation via top-level `<LuaScript>` in the `<CheatTable>` root, which runs on table load and sets `Active = true` on this entry.

## Auto-attach

`<TargetProcess>INSIDE</TargetProcess>` in the CT XML root. Cheat Engine 7.5+ interprets this as "attach to this process automatically when the table opens". Process name is matched without the `.exe` suffix on Windows - `INSIDE` is correct, `INSIDE.exe` can fail depending on CE version / locale.

## Runtime state (`_G` globals)

All state kept in the global Lua table (CE's Lua state persists across Reload Table). Cleanup happens in each entry's DISABLE:

| Global | Purpose |
|---|---|
| `INSIDE_CAM_NOP_SITES` | List of probe NOP sites with addr + original bytes |
| `INSIDE_CAM_RELEASED` | Whether probe NOPs are currently released (hold-to-track mode) |
| `INSIDE_CAM_SETTS` / `INSIDE_CAM_GETTS` | Cached `UnityEngine.Time.set_timeScale` / `get_timeScale` methods |
| `INSIDE_CAM_SNAP_TSWAS` | TimeScale saved during R-snap (restored on release) |
| `INSIDE_CAM_TIMER` | The tick timer |
| `INSIDE_CAM_CLASS` / `INSIDE_CAM_GETMAIN` | Cached `Camera` class + `get_main` method |
| `INSIDE_MAIN_TRANSFORM_CACHED` | Cached Camera.main.transform managed object pointer |
| `INSIDE_TR_GET_EULER` | Cached `Transform.get_eulerAngles` (legacy - was used for view-relative vectors when we did math ourselves before switching to Translate) |
| `INSIDE_ROTATE_3F` | Resolved address of `Transform.Rotate(single,single,single)` |
| `INSIDE_TRANSLATE_3F` | Resolved address of `Transform.Translate(single,single,single)` |
| `INSIDE_MOVECAM_NOP_ADDR` / `_ORIG` / `_SIZE` | set_rotation NOP bookkeeping |
| `INSIDE_MOVECAM_SETPOS_NOP_ADDR` / `_ORIG` / `_SIZE` | set_position NOP bookkeeping |
| `INSIDE_CAM_AUTO_RELEASED` / `_SETTLE_UNTIL` | Level-transition state machine |
| `INSIDE_LBGLOBAL_PREV` / `INSIDE_RBGLOBAL_PREV` | Edge detection for LB / RB in listener |
| `INSIDE_GPLISTEN_TIMER` | Listener timer handle |
| `INSIDE_PAUSE_PREV` / `INSIDE_PAUSE_SETTS` | Time Pause entry state |

## Why this design

Other approaches we considered and rejected:

- **BepInEx plugin with Harmony** - ideal for Unity 2018+, but Unity 5.0.4 Mono 2.0 is right at the edge of BepInEx 5.4.22's support and did not run reliably in our tests. Also increases install friction (drop winhttp.dll, launch once, install plugin).
- **Disable entire `CameraScript`** via `Behaviour.enabled = false` - tried this; destabilised Mono subsystem in ways that broke Time.timeScale among other things. We think Unity garbage-collected internal state we did not expect it to.
- **IL2CPP / Mono patching via MonoMod / RuntimeDetour** - same install-friction / Mono-stability tradeoffs as BepInEx.
- **Native Transform writes via `m_CachedPtr`** - viable but requires reverse engineering native Transform layout for Unity 5.0.4 x64 (position at `~+0x38`, rotation at `~+0x50`, confirmed by pointer scans but exact offsets not reliable). Would avoid the `mono_invoke_method` overhead entirely but is significantly more fragile.

The current two-NOP + two-Mono-invoke hybrid sits in the sweet spot: minimal invasive patching, maximum compatibility with Unity's own API, easy to reason about.

## Deferred work: camera-relative Boy controls

Biggest known limitation. When the camera has been rotated (say 90 degrees via D-Pad), pushing the left stick "up" makes the Boy run in his fixed world X axis direction, not in the direction the camera is now pointing. For exploration it would be more natural if stick forward meant "forward in the camera's view".

### Where to hook

Research (full notes live in comments of entry #2) found a single clean injection point:

`GameInput.Core.UpdateCommand()` in `firstpass/Assembly-CSharp-firstpass/GameInput.cs`, around line 246:

```csharp
cmd.mStick = this.controller.LeftStick.Value.ToVector2f();
```

This is the **only** write site of `mStick`. Everything downstream (Boy.UpdateLogic, BoyRunState, physics integration in CustomBody2D) reads from `mStick` via the frame-ring buffer. Rotating the Vector2 right after this assignment propagates to all consumers.

### Implementation sketch

1. Resolve `GameInput.Core:UpdateCommand` JIT address via `mono_compile_method`.
2. Scan method body for the `call ToVector2f` pattern. On Mono 2.0 x64 this is usually `mov r64, imm64 ; call r11`.
3. Identify the `cmd` pointer register used right after the call (typically `rcx` or `rdi`, needs disassembly review on target build).
4. Identify the `mStick` offset inside the `Cmd` class (byte offset of the `vector2f` field). Typically in the `0x10 - 0x30` range after Mono's object header.
5. `alloc(trampoline, 0x200, <nearby>)` a code cave. Inject `jmp trampoline` right after the call site. Trampoline:
   - Preserve caller-save registers (`rax`, `rcx`, `rdx`, `r8-r11`, `xmm0-xmm5`).
   - Read the current camera yaw from a global we would cache on each D-Pad rotation, e.g. `_G['INSIDE_CAM_YAW_RAD']`.
   - Check if Boy is in a state where rotation should be skipped (Swim, Ladder, Rope, Grab, ClimbDown). Read `ScriptGlobals.boy.mState` type tag via Mono or native pointer chain.
   - Rotate `[cmd + mStick_offset]` floats: `new_x = cos(yaw)*x + sin(yaw)*y`, `new_y = -sin(yaw)*x + cos(yaw)*y`.
   - Write back to `[cmd + mStick_offset + 0]` and `[cmd + mStick_offset + 4]`.
   - Restore registers, jump back.

### Why it's deferred

Complexity budget. The current v1.0.0 table is stable and does 90% of what a user wants. Adding an AA trampoline would add:

- 200-400 bytes of assembly code with register preservation.
- Mono runtime field-offset reads for `ScriptGlobals.boy` and `Boy.mState`.
- Per-state exception logic (swimming, ladders, etc.)
- Careful testing to make sure the trampoline never corrupts the Cmd ring buffer.

Expected effort: 4 to 5 hours of focused implementation plus testing. Clean path, no unknowns, just time.

### Caveats to know before implementing

1. **Swim / Ladder / Rope states use `stick.y` as world-up intent**. For example `Boy.cs:622` checks `this.input.stick.y > num` for "swimming toward surface". After rotation, stick-forward at 90-degree yaw would end up in `x` and swim-upward detection would break. Hence the per-state skip.
2. **`BoyInput.BaseUpdateInput` uses `Mathf.Sign(stick.x)` for run/stop hysteresis**. Rotation changes sign transitions; small behavior shift expected but tolerable.
3. **The GameInput ring buffer keeps 100 previous commands**. If you toggle the hack mid-play, the ring mixes rotated and raw frames. Not a concern for stick floats (they are interpolated anyway), but if you extend the hack to other fields later, be aware.
4. **Mono 2.0 on Unity 5.0.4 x64 is not Harmony-friendly**. This is why we use pure JIT patching via CE rather than any managed-code patching library.

### Key file paths in decompiled source

If you have a decompiled copy of `Assembly-CSharp-firstpass.dll`:

- Primary injection target: `GameInput.cs:246`
- Alternative (stick getter that's called many times per frame, more fragile): `DefaultBoyInput.cs:62`
- Base class derivation: `BoyInput.cs` (stick to loose/strict/run computation)
- Downstream consumer: `BoyRunState.cs:251-264` (`runLooseDir`, `GetPhysicsDeltaVelocity`, `AddVelocity`)
- State tag source: `ScriptGlobals.cs:93` (`public static Boy boy;`)
- Gutted cheat infrastructure (do NOT try to inject here): `CheatManager.cs` (all methods return false/none in retail)

