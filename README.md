# ClueFinders 4th Grade - PlayStation 2 Port

A fan preservation project porting *The ClueFinders: 4th Grade Adventures*
(The Learning Company, 1999) from Windows PC to the Sony PlayStation 2.

This is a from-scratch reimplementation of the game's engine. No original
game code runs on the PS2 - only the game's data files (scripts, sprites,
audio, video) are reused. The engine is being rebuilt in C using the
homebrew PS2SDK, driven by the original game's MPS bytecode scripts.


## **Milestone: ClueFinders bytecode now running on PS2**

As of this build, the PS2 ELF:

- Boots in PCSX2
- Reads `STARTUP.MPS` from the host filesystem
- Parses 103 instructions, 365 operands, 96 pool entries
- Steps through the bytecode opcode-by-opcode on the EE
- Evaluates expressions like `"CMAPieceX"+i` for game property names
- Iterates FOR loops on real game data
- Creates engine objects (RWorldPort, RScenePort, RSmackerMovie)
- Fires events that advance the game state machine
- Chains via SCENE_LOAD to Signin.mps
- Parses a second 704-instruction script and runs its pInit

A complete trace prints to TTY (visible in PCSX2's console window).
This is the first time CF4's actual game logic has executed on
PlayStation hardware emulation.


## Project overview

The original ClueFinders 4th Grade is a Windows point-and-click
educational adventure built on a proprietary engine by Knowledge Adventure
(The Learning Company). The engine has several distinctive traits:

- **MPS bytecode scripting language** with 28 opcodes, big-endian binary
  format. All gameplay logic lives in 43 `.MPS` script files.
- **Expression language** embedded in pool entries with full arithmetic
  and comparison operators, including string operations.
- **Resource system** using repurposed Windows NE-format `.RSC` files as
  asset containers, holding RRGB palettes, ASEQ sprite animations, NFNT
  bitmap fonts, and WAVE audio.
- **Class-based engine runtime** with ~20 classes: RWorldPort, RScenePort,
  RSprite, RCharacter, RSound, RSmackerMovie, RQueue, RCompositeAction,
  RPButton, RSelList, RKbdInp, RLapTrap, RGraphicAnswer, RHotSpot,
  RAnimation, RBackPack.
- **Event-driven execution model:** scripts register event handlers
  (`movie.finished`, `character.mouseDown`, etc.) and the engine pumps
  events into them.
- **Adaptive difficulty** with telemetry on wrong guesses, per-activity
  auto-leveling thresholds, and multi-dataset random selection.
- **Smacker (.SMK) video** for cutscenes.


## Current state

**Overall project: ~60% complete**

| Phase | Area                     | Status                  |
|-------|--------------------------|-------------------------|
| 1     | PS2 renderer             | ~95%                    |
| 2     | MPS bytecode interpreter | **complete**            |
| 3     | Engine runtime classes   | **complete**            |
| 4     | I/O (input/audio/files)  | ~30% (PS2 boot working) |
| 5     | Smacker video playback   | 0%                      |
| 6     | Memory/VRAM management   | 0%                      |
| 7     | Polish and full testing  | 0%                      |

**What runs today on PS2:**

- The interpreter boots in PCSX2 from `host:STARTUP.MPS`
- Steps through 237 instructions of STARTUP including:
  - 5 CacheDLL calls (logged)
  - 60+ property declarations (`port.addIntProperty(...)`)
  - 3 nested FOR loops with expression-built property names
  - The instantiation of RWorldPort
  - Replacement of RWorldPort with RScenePort
  - The pLogoMovie SCENE_JUMP and proc-skip-after-return logic
- Fires `movie.finished` 3 times to advance through the intro cinematics
- SCENE_LOAD chains to Signin.mps
- Parses Signin.mps and runs MAIN_PROC + pInit
  - Creates `gCurLocation = "SIGN-IN"`, sQueue (RQueue), all the
    dialog box / button / cursor constants the script expects
- Final engine state correctly contains the `port` (RScenePort) and
  `sQueue` (RQueue) objects, just like on host

**What runs today on host (no graphics):**

- Full engine runtime hosting all classes:
  RQueue, RCompositeAction, RCharacter, RPButton, RAnimation, RSelList,
  RHotSpot (with real bounds and hit-testing), RLapTrap, RBackPack
  (with inventory), RGraphicAnswer, plus generic-class fallback for
  RScenePort, RWorldPort, RKbdInp, RText, RSmackerMovie.
- Six action types (CharacterSpeech, CharacterAnim, Property, Sound,
  Anim, Movie) all dispatched into the engine's queue tick loop.
- All 43 scripts load and run cleanly.
- End-to-end opening sequence, full character roster, hotspot
  hit-testing, inventory state.

**What doesn't run yet:** anything visible. The PS2 build runs the
interpreter but doesn't draw. The renderer code is in the binary but
not yet wired to engine state. That's the next big task: bridge
visible engine objects (RCharacter, RPButton, RAnimation) into the GS
draw loop.


## How the PS2 boot works

```c
// src/main_mps.c
SifInitRpc(0);
engine_init();

mps_context_t ctx;
mps_load(&ctx, "host:STARTUP.MPS");   // PCSX2 maps host: to a folder

while (mps_running(&ctx) && n < 5000) {
    mps_step(&ctx);
    engine_tick(&ctx);
    n++;
}

// Fire movie.finished 3 times (player would normally click)
mps_fire_event(&ctx, "movie", "finished");
mps_fire_event(&ctx, "movie", "finished");
mps_fire_event(&ctx, "movie", "finished");

// Follow SCENE_LOAD chains
const char *next = mps_pending_script(&ctx);
mps_free(&ctx);
mps_load(&ctx, "host:Signin.mps");
mps_call_proc(&ctx, "pInit");
// ...
```

To build:

```sh
docker run -it --rm -v ~/cf4_ps2:/project ps2dev/ps2dev:latest sh
cd /project
make
```

To run: copy `cf4.elf` and the `.MPS` files to a folder PCSX2 maps as
`host:`, then System -> Run ELF. Output appears in the PCSX2 console.


## 43-script survey results

Every `.MPS` file in the game loads, parses, and executes its
MAIN_PROC + pInit + pStart protocol with the engine attached.

```
Total: 43/43 scripts pass.
```


## Verified engine mechanics

### Character click escalation

Characters respond differently based on state or click counter:

- **Socrates click:** context-sensitive help based on scene state
- **Character click (CHUB):** first click has Owen speak, subsequent
  clicks cycle
- **Geezar click (CMA):** 4-tier dialogue with level-tiered taunts
- **Crocodile click (OMA):** cycles through 4 canned lines (mod-4 counter)

### Scroll/clue progression (CHUB)

The Cairo world has a 5-clue story arc where each clue triggers
different character reactions with increasing ensemble size:

| Clue # | Characters who react | Narrative beat |
|--------|---------------------|----------------|
| 1 | Owen, Joni | "Oh cool, a clue!" |
| 2 | Owen, Leslie, Joni, LapTrap | Group reaction |
| 3 | Santiago (first line!), Leslie, LapTrap, Joni | Leader emerges |
| 4 | Santiago, Joni | "We're close!" beat |
| 5 | Owen, Santiago, Leslie, Joni | Full-cast reveal |

### Cairo hub walk-in choreography

When the player enters the hub on a return visit, the game queues a
`walkInCAct` RCompositeAction containing 6 parallel children. The
interpreter and engine assemble this exactly as designed:

```
walkInCAct (RCompositeAction, 6 children):
  joniWalkInQAct (real queue, 3 actions):
    PropertyAction: closedBackpack.visible = 0
    CharacterAnimAction: joni walks in
    PropertyAction: closedBackpack.visible = 1
  walkInCAct_anon0: Santiago anim
  walkInCAct_anon1: Owen anim
  walkInCAct_anon2: Leslie anim
  walkInCAct_anon3: Socrates anim
  walkInCAct_anon4: LapTrap anim
```

### "Limburger" QA cheat

Typing "Limburger" as the player name on the sign-in screen shortcuts
to the teacher panel. Discovered through bytecode tracing.

### Multi-outcome collision (OMA)

Catapult rocks can hit 4 distinct targets each with its own handler:
`pRockHitWater`, `pRockHitStatue` (spawns RAnimation with
`kRockCracksUpID[activeRock]` - different crack anim per rock type!),
`pRockHitLocation`, `pRockStraightUp`.

### Object userData and string operators

Strings like `"FIRE_GEM#3"` encode multi-field data in a single
property, parsed via:

- `@` (find): `"#" @ ud` - returns 1-based position of `#` in ud
- `|` (substring): `ud | 1 | p` - substring from position 1, length p


## Engine runtime detail (Phase 3 - complete)

### Module layout

```
mps_interp.c (interpreter)       engine.c (runtime)
    +-- mps_load                 +-- engine_init
    +-- mps_step                 +-- engine_create_queue
    +-- mps_call_proc            +-- engine_create_character
    +-- mps_fire_event           +-- engine_create_button
    +-- mps_pending_script       +-- engine_create_hotspot
                                 +-- engine_create_backpack
                                 +-- engine_queue_add_*
                                 +-- engine_set_property
                                 +-- engine_tick
                                 +-- engine_hotspot_hit_test
                                 +-- engine_backpack_add_item
```

The same module runs on host (where actions print to stdout) and on
PS2 (where actions print to TTY and will eventually drive the
renderer/audio).

### Implemented action types

| Action | Args | Status |
|--------|------|--------|
| CharacterSpeechAction | who, speech_id | Logged |
| CharacterAnimAction | who, anim_id | Logged |
| PropertyAction | obj, prop, value | Applied to engine state |
| SoundAction | sfx_id | Logged |
| AnimAction | anim_id, z, frame_start, hold_last | Logged |
| MovieAction | movie_name, sound_id | Logged |

### Implemented object types (specialized)

| Object | State | Constructor |
|--------|-------|-------------|
| RQueue | actions[] + playing index + finished flag | () |
| RCompositeAction | child names + started flag | () |
| RCharacter | set_id, z, movable, frame, visible | (set_id, z) |
| RPButton | x, y, set_id, click_sound, enabled | (x, y, set_id) |
| RAnimation | set_id, z, touchy, frame | (anim_id [, z]) |
| RSelList | x, y, w, h, list[16], selected | (x, y, w, h) |
| RHotSpot | x1, y1, x2, y2, z + hit-test | (x1, y1, x2, y2, z) |
| RLapTrap | set_id | (set_id) |
| RBackPack | x, y, z, inventory[12] | (x, y, z) |
| RGraphicAnswer | x, y, z, set_id, frame, draggable | (x, y, z, set_id, frame) |

### Universal property setter

Method calls of the form `obj.propname(value)` route to
`engine_set_property` if `obj` is a known engine object and `propname`
is one of: `visible`, `enabled`, `movable`, `touchy`, `frame`, `x`,
`y`, `z`, `clickSoundID`.


## MPS bytecode interpreter (Phase 2 - complete)

### File format (big-endian, version 1)

```
u8   version
u32  inst_count
N *  { u8 opcode, u16 operand_idx }
u32  op_table_count
N *  int16                            (operand table, -1 = separator)
u32  pool_count                       (includes 3 pre-allocated)
N *  pool_entry                       (from entry 3)
```

All 28 opcodes handled. Full expression evaluator with arithmetic,
comparison (int + string), find/substring, and parens.


## Reverse engineering work

Documentation in `docs/`:

- **`docs/mps_opcodes.md`** - bytecode format spec and opcode table
- **`docs/engine_classes.md`** - engine classes, built-in commands, events
- **`docs/file_formats.md`** - RSC/RRGB/ASEQ/NFNT/WAVE specs
- **`docs/disassembly/`** - all 43 MPS scripts disassembled

### Characters catalogued

**Main cast (6):** Joni, Santiago, Owen, Leslie, Socrates, LapTrap
**Villain:** Geezar
**NPCs (13):** Dealer, Boat Girl, Coffee Guy, Seller, Mason, Exportress,
Purrina, Fixnuppan, Art Mouse, Bilko, Rocko, Sphinxo, Thoth, crocodile


## PS2 renderer (Phase 1)

The renderer (`src/renderer.c`) drives the PS2's Graphics Synthesizer
directly using `ps2sdk`'s draw/graph/dma/packet libraries. Currently
linked into the ELF but not driven from engine state - that's the next
Phase 4 task.

**Capabilities:**

- GS initialization, double-buffered 640x448 NTSC framebuffers
- Manual GS register setup (FRAME, ZBUF, XYOFFSET, SCISSOR, etc.)
- Rectangle drawing with alpha
- Textured sprite drawing with alpha testing (ATST=6 GREATER, AREF=0)
- Multi-frame sprite animation
- PSMCT32 32-bit RGBA texture format

All 2598 textures pre-padded to POT dimensions via
`tools/cf4_pad_pot.py`.


## Project layout

```
cf4_ps2/
|-- Makefile               PS2 build (EE GCC + ps2sdk)
|-- README.md
|
|-- src/
|   |-- main.c             Original sprite-viewer test
|   |-- main_mps.c         Phase 4 PS2 boot test (current main)
|   |-- renderer.c/.h      PS2 GS renderer
|   |-- mps_interp.c/.h    MPS bytecode interpreter
|   |-- engine.c/.h        Engine runtime classes
|   |-- embedded_sprite.h  Test sprite (Joni)
|   `-- embedded_anim.h    Test sprite (Santiago, animated)
|
|-- assets/
|   |-- textures/          Raw .tex files
|   |-- textures_pot/      POT-padded textures (2598)
|   |-- sprites/           PNG sprites
|   |-- audio/             WAVs
|   |-- fonts/             NFNT data
|   |-- palettes/          RRGB JSONs
|   |-- scripts/           43 .MPS files
|   `-- video/             SMK cutscenes
|
|-- tools/
|   |-- cf4_pad_pot.py
|   |-- tex_to_embedded_h.py
|   |-- mps_disasm.py
|   |-- mps_test.c         Single-script trace tool
|   |-- mps_chain_test.c   Multi-script chain test (host)
|   |-- script_survey.c    All-43-scripts batch test
|   `-- survey_proc.c      Per-procedure breakdown
|
`-- docs/
    |-- mps_opcodes.md
    |-- engine_classes.md
    |-- file_formats.md
    `-- disassembly/       All 43 scripts disassembled
```


## What's left to do

### Phase 4 continuation (~30% done, 70% to go)

- **Bridge engine state into the renderer:** for each visible
  RCharacter / RPButton / RAnimation / RHotSpot, look up its sprite
  by set_id and draw at (x,y) with current frame. z-order the draw
  list. This is the moment ClueFinders becomes visible.
- **CacheDLL implementation:** parse `.rsc` resource files to extract
  ASEQ sprite animations, RRGB palettes, NFNT bitmap fonts, WAVE audio.
  Currently CacheDLL is a logged stub.
- **DS2 input via libpad:** convert button presses to a virtual mouse
  cursor. Pump engine_hotspot_hit_test for click handling.
- **SPU2 audio via audsrv:** SoundAction actually plays.
- **USB / CD / DVD asset loading on real hardware** (currently uses
  PCSX2's host:/ which only works in emulation).
- **Memory card save/load** for player profiles.

### Phase 5 (Smacker video) - 0%

Port libsmacker/ffmpeg to PS2, OR pre-convert SMK -> MPEG-1.
MovieAction would actually play through `RSmackerMovie.start`.

### Phase 6 (Memory/VRAM) - 0%

32MB RAM + 4MB VRAM budget; LRU texture cache; scene-scoped resource
sets fit the CacheDLL/UncacheAllDLLs pattern.

### Phase 7 (Polish) - 0%

Full playthrough, matching original feel, 60Hz perf target.


## How to pick up the work

**PS2 side:**

```sh
docker run -it --rm -v ~/cf4_ps2:/project ps2dev/ps2dev:latest sh
cd /project
make
# Copy cf4.elf and assets/scripts/*.MPS to PCSX2 host folder
# In PCSX2: System -> Run ELF
# Watch output in PCSX2 console
```

**Host (interpreter + engine, no graphics):**

```sh
gcc -I src tools/mps_chain_test.c src/mps_interp.c src/engine.c -o tools/mps_chain_test
./tools/mps_chain_test
```

Full 43-script survey:

```sh
gcc -I src tools/script_survey.c src/mps_interp.c src/engine.c -o tools/script_survey
tools/script_survey 2>/dev/null | grep -E "inst="
```


## License and credit

This is a fan preservation project. ClueFinders and all its assets are
copyright The Learning Company / Knowledge Adventure. The port code
(renderer, tools, interpreter, engine) is hobbyist work. Engine
understanding comes from Ghidra decompilation of the original
executable.

Not for commercial distribution. The goal is to make this great
educational game playable on hardware it was never released for.
