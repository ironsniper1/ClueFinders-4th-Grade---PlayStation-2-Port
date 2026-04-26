# ClueFinders 4th Grade - PlayStation 2 Port

A fan preservation project porting *The ClueFinders: 4th Grade Adventures*
(The Learning Company, 1999) from Windows PC to the Sony PlayStation 2.

This is a from-scratch reimplementation of the game's engine. No original
game code runs on the PS2 - only the game's data files (scripts, sprites,
audio, video) are reused. The engine is being rebuilt in C using the
homebrew PS2SDK, driven by the original game's MPS bytecode scripts.


## Latest milestone: complete reverse engineering

Every file format the game uses has been fully decoded. The Anchor
Offset Coordinates (AO coords / kUseAOCoords) - the long-standing
mystery of where buttons and characters appear on screen - have been
located inside ASEQ sprite files via Ghidra disassembly of the
original Win32 executable.

Result: positions are now known for every visible object in the game.

| Cast member | Cairo hub position |
|-------------|---------------------|
| Owen        | (19, 199) |
| Joni        | (46, 201) |
| Santiago    | (78, 181) |
| Leslie      | (115, 218) |
| LapTrap     | (147, 172) |
| Socrates    | (102, 390) |
| Dealer      | (504, 151) |

| Sign-In button | Position |
|----------------|----------|
| Background     | (0, 0)   |
| Exit           | (68, 316)  |
| New Player     | (200, 335) |
| Practice Mode  | (158, 426) |
| Up arrow       | (423, 213) |
| Down arrow     | (423, 249) |
| Start          | (347, 322) |

239 scene folders, 2792 sprites with AO coordinates extracted across
all 264 RSC resource files.


## Project overview

The original ClueFinders 4th Grade is a Windows point-and-click
educational adventure built on a proprietary engine by Knowledge
Adventure (The Learning Company). The engine has these distinctive
traits:

- **MPS bytecode scripting language** with 28 opcodes, big-endian
  binary format. All gameplay logic lives in 43 `.MPS` script files.
- **Expression language** embedded in pool entries with full
  arithmetic and comparison operators, including string operations.
- **Resource system** using repurposed Windows NE-format `.RSC` files
  as asset containers, holding RRGB palettes, ASEQ sprite animations,
  NFNT bitmap fonts, and WAVE audio.
- **Class-based engine runtime** with ~20 classes: RWorldPort,
  RScenePort, RSprite, RCharacter, RSound, RSmackerMovie, RQueue,
  RCompositeAction, RPButton, RSelList, RKbdInp, RLapTrap,
  RGraphicAnswer, RHotSpot, RAnimation, RBackPack.
- **Event-driven execution model:** scripts register event handlers
  (`movie.finished`, `character.mouseDown`, etc.) and the engine pumps
  events into them.
- **Adaptive difficulty** with telemetry on wrong guesses, per-activity
  auto-leveling thresholds, and multi-dataset random selection.
- **Smacker (.SMK) video** for cutscenes.
- **AO coords system**: scripts pass `kUseAOCoords` (=11111) as x/y to
  RPButton or RAnimation; the engine reads the actual position from a
  4-byte field inside the sprite's ASEQ data.


## Current state

**Overall project: ~70% complete**

| Phase | Area                     | Status        |
|-------|--------------------------|---------------|
| 1     | PS2 renderer             | ~95%          |
| 2     | MPS bytecode interpreter | **complete**  |
| 3     | Engine runtime classes   | **complete**  |
| 4a    | I/O - PS2 boot + run     | **complete**  |
| 4b    | I/O - reverse engineering| **complete**  |
| 4c    | I/O - draw real sprites  | 0%            |
| 4d    | I/O - input + audio      | 0%            |
| 5     | Smacker video playback   | 0%            |
| 6     | Memory/VRAM management   | 0%            |
| 7     | Polish and full testing  | 0%            |

**What runs today:**

- The interpreter boots in PCSX2 from `host:STARTUP.MPS`
- Steps through 237 instructions of STARTUP including all initialization
- Fires 3 movie.finished events to chain through Logo -> MVOP1 -> MVTitle
- Follows SCENE_LOAD to Signin.mps, runs MAIN_PROC + pInit
- Runs Signin's master init proc at inst 323, creating all UI objects
- 16 engine objects exist in PS2 memory with correct state
- Action queue cycles work end-to-end: SoundAction queued, played
  (logged), finished event fires, eSQueueDone runs
- Renderer draws engine state as colored rectangles
- All 264 RSC resource files extracted: 3116 sprites total
- All AO coordinates extracted into per-scene `aoc.json` files

**What doesn't run yet:**
- Drawing real sprites (rectangles only)
- Audio output (logged only)
- Controller input (no virtual cursor yet)
- Smacker cutscenes


## Reverse engineering complete

### File formats fully decoded

```
RSC container (NE format)
  +-- 0x800F directory
  +-- 0xFF01 ASEQ sprite blocks
  +-- 0xFF02 RRGB palette OR WAVE audio (1536-byte test)
  +-- 0xFF03 WAVE audio
```

**ASEQ sprite format:**

```
0x00  u16 version (=1)
0x02  u16 frame_count
0x04  u8  bpp (=8)
0x05  u8  reserved
0x06  u32 reserved
0x0A  u32 frame_offsets[frame_count]
???   AOC: 4 bytes (X 12-bit, Y 12-bit, Z 8-bit, packed)
???   Metadata records (8 bytes each: u32 value, i16 tag, u16 reserved)
???   Image marker: 00 00 04 00 W H
???   RLE pixel data
```

AOC location: `0x0A + 4 * frame_count`

AOC encoding:
```
byte 0   = X_low
byte 1   = Y_low
byte 2   = X_high (low nibble), Y_high (high nibble)
byte 3   = Z_default

X = byte0 + (byte2 & 0x0F) * 256
Y = byte1 + (byte2 & 0xF0) * 16
if X > 0x800: X = 0x800 - X     (negative)
if Y > 0x800: Y = 0x800 - Y
```

### How AO coords are resolved (from Ghidra)

```c
// FUN_0048dbf5 - sprite/animation constructor
//   called as FUN_0048dbf5(obj, image, 0x2b67, 0x2b67, scenePort)
//   when script passes kUseAOCoords (=0x2b67)
//   then calls FUN_0049245b which reads the AOC bytes from ASEQ data

void apply_aoc(int *obj, short *aseq_data) {
    int frame_count = aseq_data[1];
    byte *aoc = (byte *)(aseq_data + 5 + frame_count * 2);
    
    obj[X_FIELD] = aoc[0] + (aoc[2] & 0x0F) * 256;
    obj[Y_FIELD] = aoc[1] + (aoc[2] & 0xF0) * 16;
    obj[Z_FIELD] = aoc[3];
    
    if (obj[X_FIELD] > 0x800) obj[X_FIELD] = 0x800 - obj[X_FIELD];
    if (obj[Y_FIELD] > 0x800) obj[Y_FIELD] = 0x800 - obj[Y_FIELD];
}
```

### Tools added

- `tools/cf4_extract_all_v2.py` - extracts all RSC contents (sprites,
  palettes, fonts, WAV files), with classifier bug fixed (was
  misidentifying ASEQ as palettes/wavs in some files)
- `tools/extract_aoc.py` - reads .sprite files and writes per-scene
  `aoc.json` with X/Y/Z for each sprite ID
- `tools/ghidra_scripts/find_ao_coords.py` - Ghidra headless script
  that locates kUseAOCoords usages and decompiles surrounding functions
- `tools/ghidra_scripts/decompile_func.py` - Ghidra headless script
  for targeted decompilation by address


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
- **Crocodile click (OMA):** cycles through 4 canned lines

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
`walkInCAct` RCompositeAction containing 6 parallel children:

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
to the teacher panel.

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


## Engine runtime detail

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
                                 +-- engine_get_object_array
```

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


## How the PS2 boot works

```c
// src/main_mps.c
SifInitRpc(0);
padInit(0); padPortOpen(0, 0, pad_buf);
engine_init();

mps_context_t ctx;
mps_load(&ctx, "host:STARTUP.MPS");
run(&ctx, 5000);

// Fire 3 movie events to advance through intro
for (int i = 0; i < 3; i++) {
    mps_fire_event(&ctx, "movie", "finished");
    run(&ctx, 5000);
}

// Follow SCENE_LOAD chain
const char *next = mps_pending_script(&ctx);
mps_free(&ctx);
mps_load(&ctx, "host:Signin.mps");
mps_call_proc(&ctx, "pInit");
run(&ctx, 5000);
ctx.ip = 323;  // Master init proc
run(&ctx, 5000);

// Open renderer and draw engine state
renderer_t r;
renderer_init(&r);
for (;;) {
    renderer_begin_frame(&r);
    engine_object_t *objs = engine_get_object_array();
    for (int i = 0; i < 64; i++) {
        if (objs[i].kind == OBJ_NONE) continue;
        renderer_draw_rect(&r, x, y, w, h, r, g, b, alpha);
    }
    renderer_end_frame(&r);
}
```


## MPS bytecode interpreter

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

- `docs/mps_opcodes.md` - bytecode format spec and opcode table
- `docs/engine_classes.md` - engine classes, built-in commands, events
- `docs/file_formats.md` - RSC/RRGB/ASEQ/NFNT/WAVE specs (full)
- `docs/disassembly/` - all 43 MPS scripts disassembled

### Characters catalogued

**Main cast (6):** Joni, Santiago, Owen, Leslie, Socrates, LapTrap
**Villain:** Geezar
**NPCs (13):** Dealer, Boat Girl, Coffee Guy, Seller, Mason, Exportress,
Purrina, Fixnuppan, Art Mouse, Bilko, Rocko, Sphinxo, Thoth, crocodile


## PS2 renderer

The renderer (`src/renderer.c`) drives the PS2's Graphics Synthesizer
directly using `ps2sdk`'s draw/graph/dma/packet libraries. Now driven
from engine state.

**Capabilities:**

- GS initialization, double-buffered 640x448 NTSC framebuffers
- Manual GS register setup (FRAME, ZBUF, XYOFFSET, SCISSOR, etc.)
- Rectangle drawing with alpha (used for the scene graph display)
- Textured sprite drawing with alpha testing (ATST=6 GREATER, AREF=0)
- Multi-frame sprite animation
- PSMCT32 32-bit RGBA texture format

All 2598 textures pre-padded to POT dimensions via
`tools/cf4_pad_pot.py`.


## Project layout

```
cf4_ps2/
|-- Makefile                    PS2 build (EE GCC + ps2sdk)
|-- README.md
|
|-- src/
|   |-- main.c                  Sprite-viewer test (legacy)
|   |-- main_mps.c              Phase 4 PS2 boot + visible scene graph
|   |-- renderer.c/.h           PS2 GS renderer
|   |-- mps_interp.c/.h         MPS bytecode interpreter
|   |-- engine.c/.h             Engine runtime classes
|   |-- embedded_sprite.h       Test sprite (Joni)
|   `-- embedded_anim.h         Test sprite (Santiago, animated)
|
|-- assets/
|   |-- textures/               Raw .tex files
|   |-- textures_pot/           POT-padded textures (2598)
|   |-- sprites/                Per-scene .sprite + .png + aoc.json (239 scenes)
|   |-- audio/                  WAVs
|   |-- fonts/                  NFNT data
|   |-- palettes/               RRGB JSONs
|   |-- scripts/                43 .MPS files
|   `-- video/                  SMK cutscenes
|
|-- tools/
|   |-- cf4_extract_all_v2.py   Extracts all RSC contents
|   |-- extract_aoc.py          Per-scene aoc.json builder
|   |-- cf4_pad_pot.py
|   |-- tex_to_embedded_h.py
|   |-- mps_disasm.py
|   |-- mps_test.c              Single-script trace tool
|   |-- mps_chain_test.c        Multi-script chain test (host)
|   |-- script_survey.c         All-43-scripts batch test
|   |-- survey_proc.c           Per-procedure breakdown
|   `-- ghidra_scripts/
|       |-- find_ao_coords.py
|       `-- decompile_func.py
|
`-- docs/
    |-- mps_opcodes.md
    |-- engine_classes.md
    |-- file_formats.md
    `-- disassembly/            All 43 scripts disassembled
```


## What's left to do

### Phase 4c (real sprite drawing) - 0%

- Convert SIGNIN PNGs to PS2 .tex format
- Embed or load on demand
- Update main_mps.c to look up sprite by `set_id`, fetch AO coords from
  aoc.json, draw at correct position
- Result: actual button graphics on PS2 instead of colored rectangles

### Phase 4d (input + audio) - 0%

- DS2 input via libpad - virtual mouse cursor for click handling
- SPU2 audio via audsrv - SoundAction actually plays
- USB / CD / DVD asset loading for real hardware
- Memory card save/load for player profiles

### Phase 5 (Smacker video) - 0%

Port libsmacker/ffmpeg to PS2, OR pre-convert SMK -> MPEG-1.

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
```

**Host (interpreter + engine, no graphics):**

```sh
gcc -I src tools/mps_chain_test.c src/mps_interp.c src/engine.c -o tools/mps_chain_test
./tools/mps_chain_test
```

**Re-extract assets (fresh start from disc):**

```sh
python3 tools/cf4_extract_all_v2.py "/path/to/cf4-disc/RSC/" assets/sprites/
python3 tools/extract_aoc.py
```

**Ghidra reverse engineering (re-run scripts):**

```sh
GHIDRA=/path/to/ghidra_11.x
"$GHIDRA/support/analyzeHeadless" ~/cf4_ps2/ghidra_proj cf4 \
    -process 4THADV32.EXE \
    -postScript find_ao_coords.py \
    -scriptPath ~/cf4_ps2/tools/ghidra_scripts \
    -noanalysis
```


## License and credit

This is a fan preservation project. ClueFinders and all its assets are
copyright The Learning Company / Knowledge Adventure. The port code
(renderer, tools, interpreter, engine) is hobbyist work. Engine
understanding comes from Ghidra decompilation of the original
executable.

Not for commercial distribution. The goal is to make this great
educational game playable on hardware it was never released for.
