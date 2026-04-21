## playdead-INSIDE-FreeCam v1.0.0

First public release of the FreeCam Cheat Engine table for Playdead's INSIDE (Unity 5.0.4, Mono, x64).

### Highlights

- **True 360 degree yaw rotation** via `UnityEngine.Transform.Rotate(single, single, single)` through Mono reflection, combined with a targeted JIT NOP of the `call [set_rotation]` instruction in `CameraScript.MoveCamera`. The camera can look in any direction; no game script overrides the rotation any more.
- **View-relative movement** via `UnityEngine.Transform.Translate(single, single, single)`, paired with a JIT NOP of `call [set_position]` in the same method. WASD, left stick analogs (right stick really, left is reserved for Boy), triggers, and D-Pad U/D all move the camera relative to its current facing direction.
- **Slow-mo pause independent of FreeCam state**: `timeScale = 0.001` freezes the Boy visually while the camera pipeline stays fully responsive. The two toggles (FreeCam and Pause) work independently and can be combined in any order.
- **Full gamepad parity with keyboard**: LB toggles FreeCam (same as F1), RB toggles slow-mo (same as `` `~ ``). A dedicated always-active background entry polls the Xbox controller at 50 ms and triggers the toggle events, so LB works even when FreeCam is off.
- **Auto-attach to INSIDE process** when the CT is opened in Cheat Engine, and the gamepad listener entry is auto-activated via a top-level `<LuaScript>` block in the CT XML. Zero clicks to get set up.
- **Cross-session resilience**: if Cheat Engine is killed without deactivating the table first (leaving NOPs in INSIDE's memory), the next session of the CT detects the leftover NOP pattern at the expected offsets and continues working, with canonical restore bytes registered for a clean DISABLE. The previous "camera forever broken after CE crash" failure mode is handled.
- **Level transition handling**: on level load the cam struct pointer chain breaks transiently; the tick timer auto-releases the `CameraBlendProbe` NOPs so Unity's streaming pipeline can position the new level's camera, then re-resolves `Camera.main.transform` (the managed object pointer can change across levels) and re-applies the NOPs at the new position.
- **No Boy controls touched**: the left thumbstick, jump, grab, and every other Boy interaction work exactly as the game intends. Only the camera is controlled by this table.

### Install

1. Install Cheat Engine 7.5 or newer from https://www.cheatengine.org
2. Download `INSIDE_Gamepad_FreeCam.CT` attached to this release
3. Launch INSIDE, start or load a save, reach an in-level screen (not main menu, not loading)
4. Open the `.CT` file in Cheat Engine. It auto-attaches to the INSIDE process and auto-activates the gamepad listener. If Cheat Engine asks whether to run Lua from the table, answer yes.
5. Press F1 or LB to enable FreeCam

See the README for the full controls layout and a technical deep dive.

### Known limitations

- Camera-relative Boy controls (stick forward = run into the screen when camera is rotated 90 degrees) are not in this release. Research and plan are in `docs/research-notes.md` and the source code comments, the implementation is deferred for a future version because it requires an AA trampoline hook into `GameInput.Core.UpdateCommand` that is several hours of careful low-level work.
- Rotation is yaw-only by design (D-Pad up/down moves vertically instead of pitching). This keeps the horizon level and is easier to control in a 2.5D game.
- Aim-pan inputs from mouse RMB and arrow keys still write to the legacy `cam+0xF0/F4` fields but have no visible effect once `set_rotation` is NOPed. Kept only for compatibility with earlier versions of the table.

### Files

- `INSIDE_Gamepad_FreeCam.CT` - the table itself, loadable directly in Cheat Engine
- `README.md` - user guide, controls reference, troubleshooting
- `docs/architecture.md` - technical deep dive on the NOP patches, Mono reflection, tick logic

### License

Public domain (Unlicense). Do whatever you want with it.
