## playdead-INSIDE-FreeCam v1.1.0

Pitch tilt, no-roll yaw, and a handful of stability fixes for the FreeCam Cheat Engine table.

### Highlights

- **D-Pad Up / Down now tilts the camera (pitch)** instead of flying it up or down. Range is clamped to +/- 30 degrees from the FreeCam start orientation so the horizon stays controllable and the camera cannot accidentally flip. Implementation uses `UnityEngine.Transform.Rotate(single,single,single)` with the default `Space.Self` (rotates around the camera's local X axis). A manual accumulator tracks the cumulative tilt because reading `Transform.eulerAngles` through Mono can hang on Vector3 marshalling. Sign is negated so D-Pad UP looks UP (Unity convention is +X rotation = pitch DOWN).
- **Yaw no longer rolls the camera** when combined with pitch. All `Transform.Rotate` overloads were enumerated through `mono_class_enumMethods` + `mono_method_getSignature`, and the 4-arg form with exact signature `single,single,single,UnityEngine.Space` was located. D-Pad Left / Right now invokes that overload with `Space.World` (enum value 0) so yaw stays in the world-Y plane regardless of current pitch. The previous 3-arg overload defaulted to `Space.Self` which produced cumulative roll on combined input.
- **Stuck slow-mo auto-fix.** If a previous Cheat Engine session crashed or was closed without disabling Time Pause, INSIDE could be left stuck at `timeScale = 0.001`, which made the left stick feel like it had no effect on the Boy. Two fixes: (a) FreeCam ENABLE now reads `Time.timeScale` and forces it back to 1.0 if it is below 0.5, (b) Time Pause ENABLE refuses to save a sub-0.5 timeScale as the "previous" value and DISABLE refuses to restore one. The previous failure mode is gone.
- **CE window no longer steals focus on toggle.** Every ENABLE / DISABLE script now hides the Cheat Engine Lua engine window via `getLuaEngine():hide()` at the top, so subsequent `print()` calls do not bring it to the foreground. Console logs are still produced and viewable through View -> Lua Engine if you want them. No more alt-tab back to the game after every F1 / LB / RB press.
- **Left stick interference removed.** The tick loop no longer reads the left thumbstick at all (it had two `snorm()` calls on `ThumbLeftX/Y` that summed into nothing but could in some setups interfere with the game's own XInput query path). Boy's movement is now untouched on every tick.
- **Cleaner D-Pad code path.** Removed the separate `cam+0xF4` fallback for pitch (the `Transform.Rotate` path is reliable on this build) and the file-based dlog system added during development.

### Install

1. Install Cheat Engine 7.5 or newer from https://www.cheatengine.org
2. Download `INSIDE_Gamepad_FreeCam.CT` attached to this release
3. Launch INSIDE, start or load a save, reach an in-level screen (not main menu, not loading)
4. Open the `.CT` file in Cheat Engine. It auto-attaches to the INSIDE process and auto-activates the gamepad listener. If Cheat Engine asks whether to run Lua from the table, answer yes.
5. Press F1 or LB to enable FreeCam

See the README for the full controls layout and a technical deep dive.

### Known limitations

- Pitch is clamped to +/- 30 degrees from the FreeCam start orientation. Yaw is unrestricted (full 360 degrees). The clamp is a deliberate ergonomics choice for a 2.5D game; if you need wider tilt, raise the cap in the ENABLE script.
- All limitations from v1.0.0 still apply: camera-relative Boy controls (stick forward = run into the screen when camera is rotated 90 degrees) is still deferred and needs an AA trampoline hook into `GameInput.Core.UpdateCommand`. Aim-pan inputs from mouse RMB and arrow keys still write to the legacy `cam+0xF0/F4` fields but have no visible effect once `set_rotation` is NOPed. Game-scripted camera swaps (cinematic sequences, some puzzle rooms) may temporarily override the FreeCam; toggle FreeCam off for those sequences.

### Files

- `INSIDE_Gamepad_FreeCam.CT` - the table itself, loadable directly in Cheat Engine
- `README.md` - user guide, controls reference, troubleshooting
- `architecture.md` - technical deep dive on the NOP patches, Mono reflection, tick logic

### License

Public domain (Unlicense). Do whatever you want with it.
