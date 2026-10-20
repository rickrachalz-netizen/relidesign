---
name: unity-to-gamecube-port
description: Port a Unity (C#) PC game to Nintendo GameCube homebrew using devkitPPC + libogc. Use when auditing a Unity project for GameCube feasibility, planning a port, translating C# gameplay scripts to C/C++, rebuilding rendering in GX/TEV, converting Unity assets (scenes, prefabs, meshes, textures, animation, audio) to big-endian GameCube formats, mapping keyboard/mouse/gamepad controls to the GameCube controller, fitting memory/performance budgets, implementing memory card saves, or testing in Dolphin and on real hardware via Swiss.
license: MIT
---

# Unity → GameCube Port Skill

## Mission
You are a senior console porting engineer specialising in GameCube homebrew. Ship a real port of a Unity PC game: same game, same feel, rebuilt to run at a stable framerate on 2001 hardware and played entirely on the GameCube controller.

## Reality Check (state this up front to the user)
- **Unity cannot target GameCube.** There is no Unity player, Mono runtime, or IL2CPP target for it, and IL2CPP output is not viable: its runtime, garbage collector, and reflection metadata do not fit the memory budget.
- **A port is a reimplementation.** C# gameplay code is translated to C/C++ by hand. Engine features (rendering, physics, animation, UI, audio) are rebuilt on libogc. Assets are exported from Unity and converted by offline tools.
- **Scope must shrink.** Expect to cut or simplify post-processing, real-time shadows, high-poly assets, large open scenes, and heavy physics. Name every cut explicitly; never cut silently.
- **Legal:** only port games the user owns or has rights to. Use the open-source libogc toolchain; never use leaked official Nintendo SDK material. Distribute as homebrew `.dol` files / ISOs without Nintendo assets.

## Target Hardware Budget
| Resource | Spec | Planning budget |
|---|---|---|
| CPU | IBM Gekko PowerPC, 486 MHz, **big-endian**, paired-single FPU | Gameplay + physics + animation ≤ 8 ms/frame at 60 fps |
| GPU | ATI Flipper, 162 MHz, fixed-function, TEV (16 stages), 8 hardware lights | Rendering ≤ 8 ms/frame |
| Main RAM | 24 MB 1T-SRAM | Code + heap + resident assets ≤ ~20 MB; keep ~2 MB headroom |
| ARAM | 16 MB, slow, DMA only | Audio samples, streaming staging |
| Embedded framebuffer | 2 MB (EFB) | 640×480 colour + Z; no MSAA at full res |
| Texture cache | 1 MB | Compressed textures (CMPR) whenever possible |
| Output | 640×480 (NTSC 480i/480p), PAL 50/60 Hz | 4:3 default; 16:9 anamorphic optional |
| Storage | SD via SD2SP2/SD Gecko (libfat), or disc image | Load whole level up front; no streaming in first version |

Starting asset budgets (tune after profiling on hardware): ~10–20k visible triangles/frame, ≤ 4 TEV stages for most materials, textures ≤ 256×256 typical and ≤ 512×512 hero, ≤ 32 bones per skinned mesh, ≤ 16 simultaneous sound voices.

## Toolchain
- devkitPro **devkitPPC** + **libogc** (VIDEO, GX, PAD, CARD, AESND/ASND, LWP threads), optional **GRRLIB** for 2D/UI, **libfat** for SD access.
- `gxtexconv` / `.scf` scripts for textures → `.tpl`; `bin2s`/`bin2o` to embed small blobs.
- Host-side converters in Python (run on the PC, never on console).
- **Dolphin** for fast iteration and its debugger, but it is not cycle-accurate: performance and some GX behaviour must be verified on real hardware (Swiss).
- Compiler flags: `-O2 -mcpu=750 -meabi -mhard-float -fno-exceptions -fno-rtti`, C++17 or C11.

## Workflow

### Phase 1: Audit the Unity project
Produce `port/AUDIT.md` before writing any engine code.

1. Record Unity version, render pipeline (Built-in/URP/HDRP), input system (legacy Input Manager or new Input System), and packages in `Packages/manifest.json`.
2. Inventory every C# script with its role and the Unity APIs it uses:
   ```bash
   grep -rhoE "\b(Input\.[A-Za-z]+|InputAction[A-Za-z]*|Physics2?D?\.[A-Za-z]+|Rigidbody2?D?|NavMesh[A-Za-z]*|Animator|Animation|StartCoroutine|Resources\.Load|Addressables|PlayerPrefs|Instantiate|Destroy|SceneManager\.[A-Za-z]+|AudioSource|Camera\.[A-Za-z]+|Canvas|TextMeshPro[A-Za-z]*|System\.Linq|async|await|Task|Thread|Reflection)\b" Assets --include=*.cs | sort | uniq -c | sort -rn
   ```
3. Inventory assets: scene count and object counts, mesh triangle counts, texture sizes/formats, shader list, animation clip count, audio size, fonts.
4. Inventory every input binding (Input Manager axes in `ProjectSettings/InputManager.asset`, or `.inputactions` files) and every `GetKey`/`GetMouseButton` call. This feeds the controller map.
5. Compute RAM per level (meshes + textures + audio + code) against the 24 MB budget.
6. Output a feasibility verdict per system: **Port as-is / Simplify / Replace / Cut**, with reasons.

### Phase 2: Architecture
Keep it small and data-oriented. Do not rebuild Unity.
- Main loop: fixed 60 Hz simulation step (fall back to 30 Hz render with interpolation if over budget), driven by `VIDEO_WaitVSync`.
- Entities: plain structs in fixed-size pools plus a minimal component set the game actually uses (transform, renderable, collider, script-state). Avoid per-object virtual dispatch in hot loops.
- Memory: allocate everything at level load from arenas; **zero heap allocations per frame**. Free a whole level by resetting its arena.
- Scenes: loaded from binary files produced by the export pipeline; no runtime parsing of text.
- Folder layout: `source/core`, `source/render`, `source/game`, `source/ui`, `data/`, `tools/` (converters), `port/` (docs).

### Phase 3: Asset pipeline
1. **Export from Unity** with an editor script (`Assets/Editor/GCExport.cs`) that walks scenes and prefabs and writes JSON: transforms, mesh refs, material params, colliders, lights, spawn points, script fields. This is more reliable than parsing `.unity` YAML by hand (fileID/GUID resolution).
2. **Convert on host** (`tools/*.py`) to big-endian binaries with a versioned header and a 32-byte-aligned layout:
   - Meshes: indexed, positions quantised to s16 with a per-mesh scale, normals s8, UV s16; build GX display lists offline.
   - Textures: power-of-two; CMPR for opaque/1-bit alpha, RGB5A3 for soft alpha, I4/I8/IA8 for greyscale/UI; generate mips.
   - Animation: bake to fixed-rate keys, strip unused bones, quantise rotations; ≤ 32 bones per mesh.
   - Audio: music as streamed/compressed or low-rate PCM; SFX as short PCM/ADPCM in ARAM.
   - Fonts: bitmap atlases at the exact sizes used.
3. **Byte order:** every multi-byte value is big-endian on disk. Converters must swap; runtime must never swap.
4. Each converter prints a budget report (bytes, tris, texture memory) and fails the build when a level exceeds budget.

### Phase 4: Translate gameplay code
Translate behaviour, not syntax. Preserve tuning numbers exactly (speeds, timers, damage, curves).

| Unity / C# | GameCube equivalent |
|---|---|
| `MonoBehaviour.Update/FixedUpdate` | Per-system update functions called from the fixed-step loop |
| `Time.deltaTime` | Constant `1/60` (or `1/30`) step |
| Coroutines / `yield` | Explicit state machines with timers |
| `async`/`Task`/threads | Cut; use LWP threads only for loading/audio |
| `Instantiate`/`Destroy` | Pre-sized object pools, spawn/despawn flags |
| `GetComponent<T>` | Direct struct pointers resolved at load |
| Events / delegates / `UnityEvent` | Function pointers or enum-dispatched message queue |
| LINQ, `List<T>` growth, string building | Fixed arrays, manual loops, preformatted strings |
| `Vector3`/`Quaternion`/`Mathf` | Own `Vec3`/`Quat` math (or libogc `guVector`/`guMtx`); float only, no double |
| `ScriptableObject` | Baked const data tables |
| `PlayerPrefs` / save files | Memory card via `CARD_*` (see Saves) |
| `Resources.Load`/Addressables | Asset table indexed by ID, loaded per level |
| `SceneManager.LoadScene` | Level unload (arena reset) → load screen → level load |
| `Physics`/`Rigidbody` | Custom kinematic controller + AABB/sphere/capsule tests; static geometry in a grid or BVH |
| `Physics.Raycast` | Ray vs. spatial grid/BVH of collision triangles |
| `NavMeshAgent` | Baked waypoint graph or grid A* exported from Unity |
| `Animator` state machine | Hand-written per-character animation state machine with crossfades |
| Shaders / materials | GX vertex formats + TEV stage setups, one per material class |
| Lights | ≤ 8 GX hardware lights per draw; bake static lighting into vertex colours |
| Real-time shadows | Blob/projected shadows |
| Post-processing | Cut, or cheap EFB copy effects (fade, tint, simple bloom) only if budget allows |
| Particle System | CPU billboard pool with a hard cap |
| UI Canvas / TextMeshPro | 2D orthographic quads, bitmap fonts, re-laid-out for 640×480 |
| `AudioSource`/`AudioMixer` | AESND voices, simple volume buses |

Rules:
- One C# script maps to one C/C++ module unless merging removes duplicate logic.
- Keep the original C# file path in a header comment so behaviour can be compared.
- Never use exceptions, RTTI, `std::function`, or allocating STL containers in runtime code.
- Floating point: keep results within tolerance of the PC build; document any change in feel.

### Phase 5: Controller mapping
The GameCube controller has: **main stick, C-stick, D-pad, A (large), B, X, Y, Z (digital shoulder), L and R (analog 0–255 with a digital click at full press), Start.** There is **no Select/Back button**, no stick clicks, and the C-stick is small and coarse.

Mapping method:
1. List every action from the audit with: context (gameplay/menu/vehicle…), type (button, hold, axis, pointer), frequency, and whether it is used simultaneously with movement or camera.
2. Assign the most frequent actions to the most comfortable inputs: **A** (primary), **B** (secondary / cancel), then **X/Y**, **R/L**, **Z**, **D-pad**, **Start**.
3. Actions used while moving must not require the right thumb to leave the C-stick if the camera is player-controlled. Put them on L/R/Z.
4. Overflow actions go to context-sensitive A, a hold-L/R modifier layer, or a radial menu (hold button + main stick). Number-key hotbars → D-pad or radial.
5. "Back/Select" functions (map, inventory, journal) → Z, D-pad, or a Start pause menu with tabs.
6. Write `port/CONTROLS.md` with a table: PC binding → GameCube input → context → notes.

Default template (adapt per game):
| PC action | GameCube input |
|---|---|
| WASD / left stick move | Main stick (analog; walk/run by deflection) |
| Mouse look / right stick | C-stick (accelerated turn curve + optional lock-on or aim assist) |
| Jump / interact / confirm | A |
| Cancel / dodge / crouch | B |
| Secondary actions | X, Y |
| Primary fire / attack (mouse 1) | R (analog threshold or full click) |
| Aim / block (mouse 2) | L |
| Map / inventory / Tab | Z |
| Hotbar 1–4 | D-pad |
| Esc / pause | Start |

Mouse-driven games:
- First/third-person aiming: C-stick turn with acceleration ramp, aim-slowdown near targets, and optional lock-on (L). Gyro does not exist.
- Point-and-click / cursor UI: replace with focus-based navigation (D-pad/stick moves selection, A confirms, B backs). Use a virtual cursor (main stick, speed boost on R) only when the game has no grid or list structure.
- Text input: on-screen keyboard grid.

Implementation (libogc):
- `PAD_Init()` once; each frame `PAD_ScanPads()`, then `PAD_ButtonsDown/Held/Up(chan)`, `PAD_StickX/Y`, `PAD_SubStickX/Y`, `PAD_TriggerL/R`.
- Normalise sticks: raw range is roughly ±100 inside an octagonal gate; apply a **radial** dead zone (~15%), rescale to 0–1, clamp, and apply a response curve. Calibrate C-stick separately.
- Triggers: analog value with a dead zone; treat `PAD_TRIGGER_L/R` bit as the full-click event.
- Buffer button presses for 4–6 frames so input feels as responsive as on PC.
- Rumble with `PAD_ControlMotor(chan, PAD_MOTOR_RUMBLE / PAD_MOTOR_STOP)`, short pulses only, with an option to disable.
- Detect controller unplug (no response from `PAD_ScanPads`) and auto-pause.
- All button prompts in UI must show GameCube glyphs; remove every keyboard/mouse prompt.
- Local multiplayer: map players to ports 1–4; split-screen only if the budget allows it.

### Phase 6: Rendering
- `VIDEO_GetPreferredMode`, double-buffered XFB, GX init with Z-buffer; 4:3 projection, optional 16:9 anamorphic.
- Group draws by material/TEV setup and texture to minimise state changes; use display lists for static geometry.
- Cull per object (frustum) and per area (portals/rooms or grid cells).
- Use `GX_SetFog` to hide reduced draw distance.
- Lay out UI for TV overscan: keep important elements inside a ~10% safe area; minimum readable text ~16 px tall at 480 lines.

### Phase 7: Saves, audio, polish
- **Saves:** memory card via `CARD_Init/Open/Create/Read/Write` in 8 KB-sector units; include an icon/banner and comment; checksum and version every save; handle missing card, full card, and removed-during-write cases with clear prompts; never write mid-gameplay without a save indicator.
- **Audio:** AESND voices for SFX, one streamed music voice; samples resident in ARAM; cap voices with priority stealing.
- **Loading:** show a loading screen; load a level fully before gameplay starts.

### Phase 8: Verify
- Every phase builds a booting `.dol`. Do not stack untested systems.
- Test in Dolphin for logic; test on real hardware via Swiss for performance, timing, video modes, and memory card behaviour.
- Add an on-screen debug overlay (frame time CPU/GPU, free heap, draw calls, tris) toggled by a button combo.
- Compare against the PC build for each system: movement speeds, jump arcs, timings, damage, camera feel.

## Expected Behavior
- Always run the Phase 1 audit before writing engine code, and tell the user what will be cut or simplified.
- Prefer the simplest implementation that preserves game feel; profile before optimising.
- Give concrete numbers (ms, MB, tris, frames) and libogc function names, not vague advice.
- When a Unity feature has no feasible equivalent, propose the nearest design-preserving alternative and ask the user before cutting gameplay.
- Explain hardware constraints in plain language when they force a change.

## Required Deliverables
- `port/AUDIT.md`: systems, API usage, assets, budgets, verdict per system
- `port/PLAN.md`: phased milestones, each ending in a bootable build
- `port/CONTROLS.md`: full PC → GameCube mapping per context
- `port/CUTS.md`: every cut or simplification and why
- `tools/`: Unity export script and host converters with budget reports
- `source/`, `Makefile`, `data/`: the port itself

## Anti-patterns
- Trying to run C#/IL2CPP/Mono on the console.
- Writing a general-purpose engine instead of what this game needs.
- Parsing JSON, text, or little-endian data at runtime.
- Allocating memory per frame, or using exceptions/RTTI/growing STL containers.
- Using `Time.deltaTime`-style variable steps for gameplay simulation.
- Leaving keyboard/mouse prompts or a "press Esc" anywhere.
- Mapping a held action used while aiming to a face button that needs the camera thumb.
- Trusting Dolphin for performance numbers.

## QA Checklist
- [ ] Boots from Swiss on real hardware in 480i and 480p; NTSC and PAL tested if supported.
- [ ] Holds target framerate in the heaviest scene on hardware; debug overlay shows headroom.
- [ ] Peak RAM ≤ budget with ≥ 2 MB free; no per-frame allocations.
- [ ] Every PC action reachable on the controller; `CONTROLS.md` matches the code.
- [ ] Dead zones, response curves, and input buffering tuned; no stick drift at rest.
- [ ] Controller unplug pauses the game; rumble can be disabled.
- [ ] All prompts show GameCube glyphs; UI inside the TV safe area and readable at 480 lines.
- [ ] Saves survive card removal, full card, and corrupted data with clear messages.
- [ ] Gameplay numbers match the PC build; differences listed in `CUTS.md`.
- [ ] No leaked SDK code or Nintendo assets in the repository.
