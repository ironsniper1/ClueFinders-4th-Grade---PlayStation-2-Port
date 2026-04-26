# ClueFinders 4th Grade - PlayStation 2 Port

A fan preservation project porting *The ClueFinders: 4th Grade Adventures*
(The Learning Company, 1999) from Windows PC to the Sony PlayStation 2.

This is a from-scratch reimplementation of the game's engine. No original
game code runs on the PS2 - only the game's data files (scripts, sprites,
audio, video) are reused. The engine is being rebuilt in C using the
homebrew PS2SDK, driven by the original game's MPS bytecode scripts.


## Latest milestone: full Sign-In screen with background

The complete Sign-In scene now renders on PlayStation 2 with the original
papyrus map background, all six teal-scarab buttons, the orange scroll
arrows, and the player-list panel - all positioned at the exact game
coordinates extracted from the original engine.

The background was a real fight. At 640x480, the papyrus image is 1.2 MB
of raw RGBA. Power-of-two padding takes it to 1024x512 (2 MB), and the
PS2 has only 4 MB of GS VRAM, of which ~2.2 MB is already spoken for
by the double-buffered 640x448 framebuffer pair. There simply isn't
room for a 2 MB background plus the button textures plus the
framebuffers.

Solution: split the background into four 320x240 quadrants (POT-padded
to 512x256 each) and store them in 16-bit PSMCT16 format. Each quadrant
is now 256 KB, the four total 1 MB, and they composite seamlessly at
draw time. The buttons stay 32-bit for crisp alpha edges.

The 16-bit path required teaching the renderer to honor a per-texture
PSM, allocate VRAM at the right pixel size, transfer with the right
format, bind the right texbuffer PSM at draw time, and crucially - set
the GS TEXA register so the 1-bit alpha in 16-bit textures expands to
0x80 (opaque) when set. Without TEXA configured, every BG pixel was
being alpha-tested out and the background drew completely invisible.


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

**Overall project: ~78% complete**

| Phase | Area                     | Status        |
|-------|--------------------------|---------------|
| 1     | PS2 renderer             | **complete**  |
| 2     | MPS bytecode interpreter | **complete**  |
| 3     | Engine runtime classes   | **complete**  |
| 4a    | I/O - PS2 boot + run     | **complete**  |
| 4b    | I/O - reverse engineering| **complete**  |
| 4c    | I/O - draw real sprites  | **complete**  |
| 4d    | I/O - text rendering     | next up       |
| 4e    | I/O - input + audio      | 0%            |
| 5     | Smacker video playback   | 0%            |
| 6     | Memory/VRAM management   | partial       |
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
- All 10 SIGNIN textures loaded into VRAM (4 BG quadrants + 6 buttons)
- Background renders as 4 quadrants composited from 16-bit PSMCT16 data
- AOC coordinate table looked up during object creation
- Render loop draws each visible object with its real sprite at its
  real game position (kUseAOCoords resolved via aoc.json data)
- All 264 RSC resource files extracted: 3116 sprites total
- All AO coordinates extracted into per-scene `aoc.json` files
- 239 scenes, 2792 sprites with positions ready

**What doesn't run yet:**
- Text rendering (the 6 affidavit text lines and "Player" input field)
- Audio output (logged only)
- Controller input (no virtual cursor yet)
- Smacker cutscenes


## Sign-In screen positions

| Button         | Position   | Game role |
|----------------|------------|-----------|
| Background     | (0, 0)     | Papyrus map background, four 512x256 16-bit quadrants |
| Up arrow       | (423, 213) | Scroll list up |
| Down arrow     | (423, 249) | Scroll list down |
| Exit           | (68, 316)  | "Exit Game" |
| Start          | (347, 322) | "Start Game" |
| New Player     | (200, 335) | "New Player Sign-In" |
| Practice Mode  | (158, 426) | "Practice Mode" |
| SignInList     | (51, 208)  | Player name list/input |


## Cairo hub character positions (CHUBP)

| Cast member | Position |
|-------------|----------|
| Owen        | (19, 199) |
| Joni        | (46, 201) |
| Santiago    | (78, 181) |
| Leslie      | (115, 218) |
| LapTrap     | (147, 172) |
| Socrates    | (102, 390) |
| Dealer      | (504, 151) |


## VRAM budget on PS2 GS

The GS has 4 MB of embedded VRAM. The Sign-In scene fills it like this:

```
Framebuffer 0   640x448 PSMCT32   ~1.10 MB
Framebuffer 1   640x448 PSMCT32   ~1.10 MB
BG quadrant TL  512x256 PSMCT16    256 KB
BG quadrant TR  512x256 PSMCT16    256 KB
BG quadrant BL  512x256 PSMCT16    256 KB
BG quadrant BR  512x256 PSMCT16    256 KB
6 button tex    various PSMCT32   ~196 KB
                                  ----------
                            Total ~3.4 MB
```

The 16-bit BG quadrants are the key. At 32-bit they'd consume 2 MB
total and we'd run out of VRAM before the BR quadrant could allocate.

This is a glimpse of the full Phase 6 problem. Real scenes have many
more textures than Sign-In does, so an LRU-style streaming texture
cache will eventually be needed, mirroring the original engine's
CacheDLL/UncacheAllDLLs scene-scoped resource pattern.


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

### Custom .tex file format used by the PS2 port

```
0x00  u32 magic = 0x54583200  ('TX2\0')
0x04  u16 width   (texels)
0x06  u16 height  (texels)
0x08  u16 frame_count
0x0A  u16 flags
        bit 0:  legacy-padded flag (set by older padding tool)
        bit 4:  16-bit format - PSMCT16 (ABGR1555). Else 32-bit PSMCT32.
0x0C  u32 reserved
0x10+ pixel data, frame by frame, top-left to bottom-right
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


## PS2 renderer (complete)

The renderer (`src/renderer.c`) drives the PS2's Graphics Synthesizer
directly using `ps2sdk`'s draw/graph/dma/packet libraries.

**GS configuration that worked:**

```c
graph_initialize(fb_addr, 640, 448, GS_PSM_32, 0, 0);
graph_set_mode(GRAPH_MODE_INTERLACED, GRAPH_MODE_NTSC,
               GRAPH_MODE_FIELD,        // NOT FRAME
               GRAPH_ENABLE);
graph_set_screen(0, 0, 640, 448);

// TEXA: alpha expansion for PSMCT16 textures.
// TA0=0 (A bit=0 -> alpha 0), TA1=0x80 (A bit=1 -> opaque), AEM=0.
write_gs_register(GS_TEXA, 0x00 | (0 << 15) | ((u64)0x80 << 32));
```

The interlaced FIELD mode was an earlier breakthrough: in FRAME mode,
the GS renders to a buffer where each line is doubled across the two
interlaced fields, halving the effective Y resolution. FIELD mode
renders each visible line directly, giving us the full 0..447 Y range
the script expects.

The TEXA write was the breakthrough for 16-bit textures: without it,
the 1-bit alpha in PSMCT16 expands to 0 by default, every pixel fails
the alpha test, and the entire 16-bit texture renders invisible.

**Capabilities:**

- GS initialization, double-buffered 640x448 NTSC framebuffers
- Manual GS register setup (FRAME, ZBUF, XYOFFSET, SCISSOR, TEXA, etc.)
- Rectangle drawing with alpha
- Textured sprite drawing with alpha testing (ATST=6 GREATER, AREF=0)
- Multi-frame sprite animation
- PSMCT32 32-bit RGBA textures (sharp, used for buttons and characters)
- PSMCT16 16-bit RGBA5551 textures (compact, used for backgrounds)
- Per-texture PSM stored in `texture_t.psm`, honored at every step
  (load, vram allocate, transfer, texbuffer bind)
- Texture loading from host: filesystem at runtime
- Sprite-to-texture mapping by set_id

All 2598 textures pre-padded to POT dimensions via
`tools/cf4_pad_pot.py`. Background quadrants additionally split and
re-encoded to 16-bit by `tools/cf4_split_bg_16.py`.


## How the PS2 boot works

```c
// src/main_mps.c (Phase 4c version)
SifInitRpc(0);
padInit(0); padPortOpen(0, 0, pad_buf);
engine_init();
register_signin_aoc();    // load aoc.json positions

renderer_t r;
renderer_init(&r);
renderer_set_bg(&r, 30, 30, 50);

// Load sprites by set_id from host:
for (each tex in {306001..306004 BG quads, 30610..30615 buttons}) {
    int tid = renderer_load_tex(&r, "host:assets/textures_pot/signin/...");
    register_sprite(set_id, tid);
}

// Run the boot chain
mps_load(&ctx, "host:STARTUP.MPS");
run(&ctx, 5000);
for (3 movies) mps_fire_event("movie", "finished");
mps_load(&ctx, "host:Signin.mps");
mps_call_proc("pInit");
ctx.ip = 323;             // master init
run(&ctx, 5000);

// Render loop
for (;;) {
    renderer_begin_frame(&r);

    // Background quadrants first
    renderer_draw_sprite(&r, sprite_tex_for(306001), 0,   0);
    renderer_draw_sprite(&r, sprite_tex_for(306002), 320, 0);
    renderer_draw_sprite(&r, sprite_tex_for(306003), 0,   240);
    renderer_draw_sprite(&r, sprite_tex_for(306004), 320, 240);

    // Then engine objects on top
    for (each visible engine_object) {
        if (kind == BUTTON || kind == ANIMATION) {
            int tid = sprite_tex_for(o->set_id);
            if (tid >= 0) renderer_draw_sprite(&r, tid, o->x, o->y);
        } else if (kind == SELLIST) {
            renderer_draw_rect(&r, o->x, o->y, o->w, o->h, ...);
        }
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

### 43-script survey results

Every `.MPS` file in the game loads, parses, and executes its
MAIN_PROC + pInit + pStart protocol with the engine attached.

```
Total: 43/43 scripts pass.
```


## Engine runtime detail

### Module layout

```
mps_interp.c (interpreter)       engine.c (runtime)
    +-- mps_load                 +-- engine_init
    +-- mps_step                 +-- engine_create_queue
    +-- mps_call_proc            +-- engine_create_character
    +-- mps_fire_event           +-- engine_create_button   <- AOC resolution
    +-- mps_pending_script       +-- engine_create_animation<- AOC resolution
                                 +-- engine_create_hotspot
                                 +-- engine_create_backpack
                                 +-- engine_aoc_register
                                 +-- engine_aoc_get_x/y/z
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

### Implemented object types

| Object | State | Constructor |
|--------|-------|-------------|
| RQueue | actions[] + playing index + finished flag | () |
| RCompositeAction | child names + started flag | () |
| RCharacter | set_id, z, movable, frame, visible | (set_id, z) |
| RPButton | x, y, set_id, click_sound, enabled | (x, y, set_id), AOC-aware |
| RAnimation | set_id, z, touchy, frame | (anim_id [, z]), AOC-aware |
| RSelList | x, y, w, h, list[16], selected | (x, y, w, h) |
| RHotSpot | x1, y1, x2, y2, z + hit-test | (x1, y1, x2, y2, z) |
| RLapTrap | set_id | (set_id) |
| RBackPack | x, y, z, inventory[12] | (x, y, z) |
| RGraphicAnswer | x, y, z, set_id, frame, draggable | (x, y, z, set_id, frame) |


## Project layout

```
cf4_ps2/
|-- Makefile                    PS2 build (EE GCC + ps2sdk)
|-- README.md
|
|-- src/
|   |-- main.c                  Sprite-viewer test (legacy)
|   |-- main_mps.c              Phase 4c PS2 boot + real sprite rendering
|   |-- renderer.c/.h           PS2 GS renderer (PSMCT32 + PSMCT16, TEXA)
|   |-- mps_interp.c/.h         MPS bytecode interpreter
|   |-- engine.c/.h             Engine runtime classes (with AOC table)
|   |-- aoc_signin.c            Auto-generated SIGNIN AOC registrations
|   |-- embedded_sprite.h       Test sprite (Joni)
|   `-- embedded_anim.h         Test sprite (Santiago, animated)
|
|-- assets/
|   |-- textures/               Raw .tex files
|   |-- textures_pot/           POT-padded textures (used by PS2)
|   |   `-- signin/             4 BG quads (16-bit) + 6 buttons (32-bit)
|   |-- sprites/                Per-scene .sprite + .png + aoc.json
|   |   `-- (239 scene folders, 2792 sprites with positions)
|   |-- audio/                  WAVs
|   |-- fonts/                  NFNT data
|   |-- palettes/               RRGB JSONs
|   |-- scripts/                43 .MPS files
|   `-- video/                  SMK cutscenes
|
|-- tools/
|   |-- cf4_extract_all_v2.py   Extracts all RSC contents
|   |-- extract_aoc.py          Per-scene aoc.json builder
|   |-- cf4_png2tex.py
|   |-- cf4_pad_pot.py
|   |-- cf4_split_bg_16.py      Background -> 4 quadrants, 16-bit PSMCT16
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

### Phase 4d (text rendering) - next up

The Signin scene creates 6 RText objects with the affidavit text and
needs the SignInList to render player names. Plan:

- Implement NFNT bitmap font rendering (glyph atlas in VRAM)
- Wire RText (string, color_id, x, y, touchy) to engine + renderer
- Render RText strings at their (x, y) coords with the right palette color
- Render SignInList contents (placeholder "Player" text + name entries)

### Phase 4e (input + audio) - 0%

- DS2 input via libpad - virtual mouse cursor for click handling,
  hit-testing against engine_hotspot_hit_test
- SPU2 audio via audsrv - SoundAction actually plays
- Background music streaming
- USB / CD / DVD asset loading for real hardware
- Memory card save/load for player profiles

### Phase 5 (Smacker video) - 0%

Port libsmacker/ffmpeg to PS2, OR pre-convert SMK -> MPEG-1.

### Phase 6 (Memory/VRAM) - partial

The 16-bit BG quadrants and per-texture PSM laid the groundwork.
Still need: LRU texture cache, scene-scoped CacheDLL/UncacheAllDLLs
pattern, eviction on scene change, optional 8-bit indexed textures
with shared palettes for further compression.

### Phase 7 (Polish) - 0%

- Fix BG quadrant seams (color quantization mismatch at TL/TR boundary)
- Full playthrough, matching original feel, 60Hz perf target


## How to pick up the work

**PS2 side:**

```sh
docker run -it --rm -v ~/cf4_ps2:/project ps2dev/ps2dev:latest sh
cd /project
make
# Copy cf4.elf, assets/textures_pot/signin/*.tex,
# and assets/scripts/*.MPS to PCSX2 host folder
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

**Re-split a background image into 16-bit quadrants:**

```sh
python3 tools/cf4_split_bg_16.py \
    assets/sprites/<scene>/<bg>.png \
    assets/textures_pot/<scene>/<bg>
# Produces 4 .tex files: _TL, _TR, _BL, _BR
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
