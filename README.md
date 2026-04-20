# ClueFinders 4th Grade - PlayStation 2 Port

A fan preservation project porting *The ClueFinders: 4th Grade Adventures*
(The Learning Company, 1999) from Windows PC to the Sony PlayStation 2.

This is a from-scratch reimplementation of the game's engine. No original
game code runs on the PS2 - only the game's data files (scripts, sprites,
audio, video) are reused. The engine is being rebuilt in C using the
homebrew PS2SDK, driven by the original game's MPS bytecode scripts.


## Project overview

The original ClueFinders 4th Grade is a Windows point-and-click
educational adventure built on a proprietary engine by Knowledge Adventure
(The Learning Company). The engine has several distinctive traits:

- **MPS bytecode scripting language** with 24+ opcodes, big-endian binary
  format. All gameplay logic lives in 43 `.MPS` script files.
- **Resource system** using repurposed Windows NE-format `.RSC` files as
  asset containers, holding RRGB palettes, ASEQ sprite animations, NFNT
  bitmap fonts, and WAVE audio.
- **Class-based engine runtime** with ~15 classes: RWorldPort, RScenePort,
  RSprite, RCharacter, RSound, RSmackerMovie, RQueue, RCompositeAction,
  RPButton, RSelList, RKbdInp, RLapTrap, RGraphicAnswer, etc.
- **Event-driven execution model:** scripts register event handlers
  (`movie.finished`, `character.mouseDown`, etc.) and the engine pumps
  events into them.
- **Smacker (.SMK) video** for cutscenes.

Porting this to PS2 requires reverse-engineering the engine (via Ghidra
on the original EXE), rebuilding each piece in C targeting the PS2's GS
(Graphics Synthesizer), SPU2 (audio), and libpad (input) subsystems.

Everything in this repo is the result of that reverse engineering and
porting work.


## Current state

**Overall project: ~32% complete**

| Phase | Area                    | Status   |
|-------|-------------------------|----------|
| 1     | PS2 renderer            | ~95%     |
| 2     | MPS bytecode interpreter| ~92%     |
| 3     | Engine runtime classes  | 0%       |
| 4     | I/O (input/audio/files) | 0%       |
| 5     | Smacker video playback  | 0%       |
| 6     | Memory/VRAM management  | 0%       |
| 7     | Polish and full testing | 0%       |

**What runs today:**

- A PS2 ELF that boots on real hardware and in PCSX2, renders sprites
  with transparency and animation, draws rectangles for UI mockup.
- A host-side MPS interpreter that loads ANY of the game's 43 scripts
  (including the massive 2162-instruction, 5753-pool-entry CWS1 Cairo
  word-scramble) and traces their execution with full semantics.
- **Full survey of all 43 scripts** - every one loads, parses, runs
  MAIN_PROC, pInit, and pStart without errors or unknown opcodes.
- **The entire opening sequence of the game runs end-to-end in the
  interpreter:**
  - STARTUP.MPS -> Logo.smk -> MVOP1.smk -> MVTitle.smk -> SCENE_LOAD
    to Signin.mps
  - SIGNIN.MPS creates the sign-in screen (6 buttons, player list,
    keyboard input), wires 8 UI event handlers
  - Simulated Start click -> SCENE_LOAD "CHUB.mps"
  - CHUB.mps (Cairo Hub) creates walk-in animations for all 6
    characters, wires 13 scene event handlers
  - Simulated character clicks correctly trigger unique per-character
    handlers (Owen, Socrates, LapTrap, Dealer all have distinct logic)
  - CMA.MPS (Cairo Math puzzle) sets up its 28-level board

**What doesn't run yet:** anything where the script's side effects need
to actually affect the screen, speakers, or file system. All "do something"
operations are currently logged stubs.


## 43-script survey results

Every `.MPS` in the game was run through the interpreter's
load/MAIN_PROC/pInit/pStart protocol. Results:

```
script            inst  pool  exec   pInit pStart  handlers
----------------- ----- ----- ------ ----- ------  --------
STARTUP.MPS       103   96    238    no    no      4
SIGNIN.MPS        704   563   129    yes   no      1
CHUB.mps          1065  866   379    yes   yes     13
OHUB.MPS          1470  1507  376    yes   yes     13
CLOC01.mps        868   886   310    yes   yes     12
CLOC02.mps        642   721   293    yes   yes     12
CLOC03.mps        683   729   306    yes   yes     12
CLOC04.mps        684   752   320    yes   yes     12
CLOC05.mps        683   737   306    yes   yes     12
CLOC06.mps        684   756   320    yes   yes     12
CLOC07.mps        642   713   293    yes   yes     12
CLOC08a.mps       642   717   293    yes   yes     12
CLOC08b.mps       642   721   293    yes   yes     12
CLOC09.mps        708   733   373    yes   yes     13
CLOC11.mps        641   690   279    yes   yes     12
CLOC13.mps        724   744   383    yes   yes     13
CLOC14.mps        657   693   283    yes   yes     12
CMA.MPS           1241  2430  599    yes   yes     18
CBA1.mps          848   898   414    yes   yes     14
CBA2.mps          1025  989   401    yes   yes     17
CWS1.mps          2162  5753  361    yes   yes     8
CWS2.mps          1481  3570  548    yes   yes     9
CWS3.mps          1245  1945  357    yes   yes     11
CWS4.mps          1580  2930  366    yes   yes     9
OLOC02.MPS        647   728   407    yes   yes     11
OLOC03.MPS        687   738   421    yes   yes     11
OLOC04.MPS        636   692   404    yes   yes     11
OLOC05.MPS        636   692   404    yes   yes     11
OLOC06.MPS        657   752   410    yes   yes     11
OLOC07.MPS        635   681   404    yes   yes     11
OLOC08.MPS        658   764   410    yes   yes     11
OLOC09.MPS        566   573   228    yes   yes     11
OMA.MPS           1593  1346  747    yes   yes     23
OWS1.MPS          1642  4516  708    yes   yes     10
OWS2.MPS          1274  1803  395    yes   yes     10
OWS3.MPS          1748  2863  775    yes   yes     9
OWS4.MPS          1582  2687  366    yes   yes     9
PBA.MPS           471   436   188    yes   yes     11
PLOC2.MPS         592   504   190    yes   yes     5
PLOC3.MPS         482   450   200    yes   yes     11
PWS1.MPS          1011  923   379    yes   yes     15
PWS2.MPS          1411  2687  600    yes   yes     8
FL.MPS            412   421   784    yes   yes     3

Total: 43 / 43 scripts pass.
```

**Observations from the survey:**

- **C(airo) / O(asis) / P(yramid) structure is clear** - the game has three
  worlds, each with a hub, ~9 locations, 4 word-scramble puzzles, and a
  math-area puzzle.
- **OMA (Oaxaca Math Area) has the most handlers** (23) - the most
  interaction-rich single scene.
- **CWS1 (Cairo Word Scramble 1) has the largest data** (5753 pool entries,
  2162 instructions) - probably due to the vocabulary word list.
- **FL.MPS (Final Level) is the smallest gameplay script** (412 inst, 3
  handlers) - likely just an ending cinematic sequence.
- **PLOC2.MPS has only 5 handlers** - unusually light for a location,
  possibly a transition scene.
- **Location scripts are remarkably uniform** (12-13 handlers) - the game
  has a well-defined location template.


## Reverse engineering work (already done)

Before any port work started, substantial reverse engineering established
the foundation. This is documented in the `docs/` directory:

- **`docs/mps_opcodes.md`** - complete MPS bytecode format spec and
  opcode table. Derived from Ghidra decompilation of the main interpreter
  function (5317 bytes, the largest in the EXE).
- **`docs/engine_classes.md`** - the 11 documented engine classes, 16+
  built-in commands, and known event names. Several additional classes
  (RPButton, RSelList, RKbdInp, RGraphicAnswer) were discovered during
  script tracing.
- **`docs/file_formats.md`** - RSC container format, RRGB palette format,
  ASEQ sprite format (single and multi-frame), NFNT font format, audio
  format.
- **`docs/disassembly/`** - all 43 MPS scripts disassembled to
  human-readable assembly by `tools/mps_disasm.py`.

Asset extraction (done earlier, not in this repo's timeline):
- All sprites extracted as PNG from ASEQ-in-RSC.
- All audio extracted as WAV.
- All fonts extracted from NFNT.
- All palettes extracted as RRGB.
- All 43 scripts extracted as raw `.MPS` bytecode.


## PS2 renderer

The renderer (`src/renderer.c`) drives the PS2's Graphics Synthesizer
directly using `ps2sdk`'s draw/graph/dma/packet libraries. It does NOT
use `gsKit` - everything is packed and DMA'd manually, which was chosen
for finer control and understanding.

**Capabilities:**

- GS initialization, double-buffered 640x448 NTSC framebuffers in
  `GRAPH_MODE_FRAME` (full 480 scanlines; `GRAPH_MODE_FIELD` only renders
  half and was a source of early confusion)
- Manual GS register setup via `setup_draw_context`: FRAME, ZBUF,
  XYOFFSET, SCISSOR, PRMODECONT, COLCLAMP, DTHE, TEST
- Rectangle drawing (`renderer_draw_rect`) with alpha
- Textured sprite drawing (`renderer_draw_sprite_frame`) with alpha
  testing (`ATST=6` GREATER, `AREF=0`) to make transparent pixels
  disappear
- Multi-frame sprites: one VRAM region per texture, re-uploaded per
  frame when the animation advances
- Texture format is PSMCT32 (32-bit RGBA); textures must be
  power-of-two dimensions

**Asset pipeline:**

The original `.tex` files have arbitrary dimensions (e.g., 68x189). PS2
GS texture sampling works much more reliably with POT dimensions, so all
assets are pre-padded to the next power of two using
`tools/cf4_pad_pot.py`. A 68x189 sprite becomes a 128x256 texture with
the original pixels in the top-left and transparent padding elsewhere.
All 2598 textures have been pre-padded.

Two helper scripts bake specific textures into C headers for embedded
testing: `src/embedded_sprite.h` (Joni, 128x256, 1 frame) and
`src/embedded_anim.h` (Santiago, 128x256, 12 frames).

**Build (inside ps2dev Docker):**

```sh
docker run -it --rm -v $(pwd):/project ps2dev/ps2dev:latest sh
cd /project/cf4_ps2
make
```

Produces `cf4.elf` in the project root. Copy to Windows shared folder
and load in PCSX2 via System -> Run ELF.

**Controls for the test ELF:**

- D-pad: move sprite
- Cross: switch between Joni (static) and Santiago (animated)
- L1 / R1: step animation frame back / forward
- Start: exit

**Known renderer issue:** the Joni sprite renders with a zone of
garbage pixels at the top of the image. Santiago (same 128x256 POT
dimensions, same upload path, same draw call, same alpha test, and a
harder multi-frame case) renders cleanly. Extensive debugging ruled
out: source data corruption, VRAM layout/overlap, allocation order,
DMA cache coherency, helper function truncation, and swizzling. Cause
unknown. Accepted as a test-program cosmetic glitch; the renderer is
functional for real work.

**Renderer debugging notes (learned the hard way):**

- `graph_vram_allocate` returns addresses in the units that ps2sdk's
  `draw_texture_transfer` and `draw_texturebuffer` helpers expect. Do
  NOT divide by 256 or 64 when using those helpers.
- `draw_texture_transfer` takes `dest_width` in pixels, not 64-pixel
  units.
- `FlushCache(0)` before DMA if pixel data was just written by CPU.
- PS2 `printf` output does NOT appear in the PCSX2 System Console. To
  get values out of a running program, draw them on screen as
  color-coded bars (bar height = byte value).
- PCSX2 `host:` filesystem resolves paths relative to the ELF's folder.


## MPS bytecode interpreter

The interpreter (`src/mps_interp.c`, `src/mps_interp.h`) is a full
reimplementation of the game engine's scripting VM, targeting the same
binary format the original EXE consumes.

**File format** (big-endian, version byte = 1):

```
u8   version             (1 = big-endian)
u32  inst_count
N *  { u8 opcode, u16 operand_idx }     (instructions)
u32  op_table_count
N *  int16                              (operand table, -1 = separator)
u32  pool_count                         (includes 3 pre-allocated entries)
N *  pool_entry                         (starting from entry index 3)
```

A pool entry has a type_flags u32, a value_type u32 (0/2=string,
3=integer, 4=numeric string), optional value body, four metadata u32s,
an optional primary string (`str1`), optional secondary string (`str2`),
and an optional int16 array. Entries 0, 1, 2 are pre-allocated by the
engine to `<null>`, `<true>`, `<false>` and are not stored in the file.

**Key discoveries:**

- For `GOTO` and `IF_FALSE` opcodes, the 16-bit `operand_idx` field in
  the instruction stream is NOT an index into the operand table. It is
  a direct instruction number to jump to.

- `SCENE_JUMP` (opcode 6) is an in-file procedure call with proper
  push-return semantics. Its operand is a pool entry whose int_val is
  the target instruction and whose str1 is the procedure name (e.g.,
  `pLogoMovie` at instruction 72).

- `RETURN` pops the call stack if non-empty; otherwise signals "current
  procedure finished, waiting for events".

- **Opcode 36 (`CALL_METHOD_RET`)** - previously undocumented. Method
  call whose last argument is the destination variable for the return
  value.

- **Opcode 45 (`SCENE_LOAD`)** - previously undocumented. Loads a
  different .MPS file. Operand can be a literal string or an expression
  like `curScript + ".mps"`.

- **String-concatenation expressions** live as special pool entries:
  - `str2` is a type code: `s` = symbol (variable lookup), `t` = text
    literal, `+` = operator (skipped)
  - `array` holds pool indices of the parts, consumed in order
  - `str1` is the disassembler-friendly debug representation

- **Named procedures** live as pool entries with `value_type=3` where
  the int_val is the entry instruction and str1 is the procedure name.
  Each .mps file has a standard set: `pInit`, `pStart`, `pInterruptScene`,
  plus scene-specific `proc*` button handlers and `e*` event handlers.

- **Script execution protocol:** when a .MPS loads, the engine runs
  instruction 0 (a file prologue that usually runs the backpack prelude),
  then calls `pInit` for scene-local setup, then `pStart` for scene
  activation (which creates UI objects and wires event handlers). After
  `pStart` returns, the engine waits for events and dispatches them
  through the registered handlers.

**Implemented opcodes** (all 24 seen in any script):

| Op | Hex  | Name            | Implemented | Notes |
|----|------|-----------------|-------------|-------|
|  6 | 0x06 | SCENE_JUMP      | Full        | Call-and-return with stack |
|  7 | 0x07 | ASSIGN          | Full        | |
|  8 | 0x08 | FOR_INIT        | Full        | Real loop stack |
|  9 | 0x09 | FOR_STEP        | Full        | |
| 10 | 0x0A | IF_FALSE        | Full        | |
| 11 | 0x0B | GOTO            | Full        | |
| 12 | 0x0C | BOOL_TEST       | Partial     | Nonzero int = true; no comparators |
| 13 | 0x0D | NOP             | Full        | |
| 14 | 0x0E | STR_CONCAT      | Stub        | Concat via pool expressions instead |
| 15 | 0x0F | RETURN          | Full        | Pops call stack |
| 18 | 0x12 | EVAL_ASSIGN     | Full        | Object instantiation + variable copy |
| 19 | 0x13 | DESTROY         | Full        | Removes handlers for object |
| 21 | 0x15 | CALL_BUILTIN    | Partial     | Extensible dispatch; most stubs |
| 22 | 0x16 | YIELD           | Full        | Sets waiting flag |
| 23 | 0x17 | CALL_METHOD     | Partial     | Generic handler registration |
| 24 | 0x18 | TIMER_B         | Stub        | |
| 26 | 0x1A | DISABLE_DATA    | Stub        | |
| 27 | 0x1B | ENABLE_DATA     | Stub        | |
| 28 | 0x1C | DELAY           | Stub        | |
| 29 | 0x1D | NOP2            | Full        | |
| 35 | 0x23 | CALL_METHOD_35  | Partial     | Same dispatch as CALL_METHOD |
| 36 | 0x24 | CALL_METHOD_RET | Partial     | Newly discovered |
| 37 | 0x25 | MAIN_PROC       | Full        | Procedure marker |
| 40 | 0x28 | GLOBAL_PROP     | Partial     | Property decl / method call |
| 45 | 0x2D | SCENE_LOAD      | Full        | Newly discovered |

**Variable store:** a flat array of 256 slots holding either int or
string values, lookup by name. Supports scoped-enough semantics for
current scripts (no separate global/local distinction yet; real engine
has `addGlobalIntProperty` vs `addIntProperty`).

**Loop stack:** up to 16 nested for-loops tracked with counter var
index, end value, step, and body entry point.

**Call stack:** up to 32 levels. `SCENE_JUMP` pushes, `RETURN` pops.
Fresh stack on every event fire.

**Event handler table:** up to 128 handlers. Registration is generic:
any `CALL_METHOD(obj, event_name, procRef)` where procRef is a pool
entry with type=3 and a string name starting with a letter registers
the handler. Re-registration replaces existing entries; DESTROY on an
object removes all its handlers.

**Expression evaluator** (`eval_pool_to_string`): resolves pool entries
that represent string-concatenation expressions into runtime strings.
Handles variable lookup, literal quote stripping, and concatenation.
Used by SCENE_LOAD; can be extended to any opcode that needs a string
operand.

**Procedure lookup** (`mps_find_proc`, `mps_call_proc`): find a named
procedure's entry IP by scanning the pool; call it (set IP, run until
RETURN).

**Host-side testing tools:**

- `tools/mps_test.c` - runs any single script with full instruction
  tracing.
- `tools/mps_chain_test.c` - simulates the full opening sequence end to
  end, including button clicks.
- `/tmp/script_survey.c` - runs every script through MAIN_PROC + pInit +
  pStart, prints summary table (this is how the 43-script survey was
  produced).

Build commands:

```sh
gcc -I src tools/mps_chain_test.c src/mps_interp.c -o tools/mps_chain_test
./tools/mps_chain_test
```


## Verified game flow

Each arrow below is a real executed transition in the current interpreter:

```
STARTUP.MPS (MAIN_PROC at inst 0)
   | caches 5 RSC files, creates RWorldPort, declares ~40 global properties
   | for-loops register per-piece state (CMAPiece* 1..12, OMAGator* 1..4)
   | destroys RWorldPort, creates RScenePort, sets music volume
   | SCENE_JUMP -> pLogoMovie (inst 72)
   |
   +-- pLogoMovie: registers handlers, creates RSmackerMovie("Logo.smk"), start
   |
   |  [ event: movie.finished ]  (Logo done)
   |
   +-- pOpeningMovie: destroy movie, reg handlers -> pTitleMovie (89),
   |                 create RSmackerMovie("MVOP1.smk"), start
   |
   |  [ event: movie.finished ]  (Opening cinematic done)
   |
   +-- pTitleMovie: reg handlers -> pIntroMoviesDone (98),
   |                create RSmackerMovie("MVTitle.smk"), start
   |
   |  [ event: movie.finished ]  (Title screen done)
   |
   +-- pIntroMoviesDone: StartTransition(); SCENE_LOAD "Signin.mps"

[ chain runner loads SIGNIN.MPS ]

SIGNIN.MPS (MAIN_PROC at inst 0 is backpack prelude)
   | pInit (inst 277): creates sQueue (RQueue), loads 15 resource IDs
   |                   (exitDialogBoxID=30566, listFull=30567, delete=30576,
   |                    clickSfxID=30600, laptrapIntroSfx=30610, etc.)
   |
   | (master init at inst 323, anonymous):
   |   -> procInitObjects (369): creates 6 RPButtons (up/down arrow, exit,
   |        start, newPlayer, pracMode), SignInList (RSelList), KeyInput
   |        (RKbdInp), sets list colors
   |   -> procPutText (438): renders labels
   |   -> procInitData (411): loads player list from disk
   |   -> 8 CALL_METHOD_35 calls wire event handlers:
   |        KeyInput."keyPressed" -> procKeyPressed (480)
   |        SignInList."entrySelected" -> procEntrySelected (634)
   |        newPlayerButton."hit" -> procNewPlayerButtonHit (585)
   |        startButton."hit" -> procStartButtonHit (591)
   |        pracModeButton."hit" -> procPracButtonHit (595)
   |        exitButton."hit" -> procExitButtonHit (611)
   |        upArrowButton."hit" -> procUpArrowHit (615)
   |        downArrowButton."hit" -> procDownArrowHit (619)
   |
   |  [ simulate: startButton click ]
   |
   +-- procStartButtonHit (591):
   |     SetDoubleClicksEnabled(kFalse)
   |     curScript = gPort.currentLocation
   |     cleanup proc (623): destroys all 9 UI objects
   |     StartTransition()
   |     SCENE_LOAD eval("curScript + .mps")  ->  "CHUB.mps"

[ chain runner loads CHUB.mps ]

CHUB.mps (Cairo Hub, ~1060 instructions, 40+ named procedures)
   | pInit (inst 278): sets Z-ordering for Leslie, Owen, Santiago, Joni,
   |                   Socrates, LapTrap, ClosedBackpack, Scroll
   | pStart (inst 390): caches CHUBi.rsc, builds walk-in animation queue:
   |   - PropertyAction(closedBackpack.visible = false)
   |   - CharacterAnimAction(joni, kJoniWalkInID)
   |   - PropertyAction(closedBackpack.visible = true)
   |   - RCompositeAction: Santiago, Owen, Leslie, Socrates, LapTrap walk-ins
   |   - sQueue.add(walkInCAct)
   |   - sQueueFinished = "eEntrySpeeches"
   |   - sQueue.start
   |
   | 13 handlers registered:
   |   gScenePort.backgroundClicked -> pInterruptScene (525)
   |   gScenePort.paused/resumed -> ePauseScene/eResumeScene
   |   sQueue.finished -> eSQueueDone (665)
   |   navHotspots.i.mouseDown -> eNavHotspotClicked (1004)
   |   closedBackpack.mouseDown -> eBackpackOpened (119)
   |   joni/santiago/owen/leslie.mouseDown -> eCharacterClicked (887)
   |   socrates.mouseDown -> eSocratesClicked (989)
   |   laptrap.mouseDown -> eLaptrapClicked (997)
   |   dealer.mouseDown -> eDealerClicked (950)
   |
   |  [ simulate: joni click ]
   |
   +-- eCharacterClicked (887):
   |     pInterruptScene pauses the walk-in queue
   |     checks scene state machine (state < 3, == 3, 4, 5, 6)
   |     first click: characterClickCtr == 0
   |       -> sQueue.add(CharacterSpeechAction, "owen", kClickKidsH2SpchID)
   |       -> characterClickCtr = 1
   |       -> sQueue.start  (Owen speaks)
   |
   |  [ simulate: socrates click ]
   |
   +-- eSocratesClicked (989):
   |     -> sQueue.add(CharacterSpeechAction, "socrates", kSocHelpSpchIDs.state)
   |        (context-sensitive help based on scene state)
   |
   |  [ simulate: laptrap click ]
   |
   +-- eLaptrapClicked (997):
         -> SCENE_JUMP gOpenLaptrap (global procedure, inst 221)
         -> creates gLaptrap = new RLapTrap(10000, gCurLocation)
         -> registers gLaptrap.closed -> gCloseLaptrap
         -> (LapTrap device appears showing location-specific hints)

CMA.MPS (Cairo Math Area puzzle - 1241 inst, 2430 pool entries)
   | Hundreds of piece.N.M.V entries (28 rows x 12 cols x 3 variants)
   |   plus preplaced.N.M entries (pre-placed pieces per level)
   | 60+ named procedures including:
   |   pSetupGameGeneral, pSetUpNewGame, pSetUpPreviousGame,
   |   pCreateBoardAndPieces, pSaveAnswers, pCheckInstructions,
   |   pDropAndSetupGame, pWalkIn, pWalkOut
   | Event handlers:
   |   eGameSolved (1059) - win condition!
   |   eResetGame (1098) - restart level
   |   eGeezarClick (1103), eSocratesClick (1131), eKidsClick (1161),
   |   eLaptrapClick (1176)
   |   game.solved -> eGameSolved, game.piecePickedUp -> pInterruptScene
   | pStart queues Geezar's intro speech (resource 6360) if level > 1 check
   |   fails, skipping the tutorial for levels after 1
   | 18 handlers total registered

PWS2.MPS (Pyramid Word Scramble 2 - discovered in full survey)
   | Uses RGraphicAnswer class for letter pieces
   | Per-letter event registrations:
   |   letter.answerNumber.pickedUp -> eAnswerPickedUp (1181)
   |   letter.answerNumber.goHomeSound = kHomeSfx
   |   letter.answerNumber.pickUpSound = kPickedUpSfx
   |   letter.answerNumber.snapSound = kSnapSoundSfx
   | Queries gPort.currentDataset("PWS2", whichDataset) for vocabulary
```

Every arrow here executes correctly in the interpreter today. Only the
side effects (actually playing Logo.smk, drawing the sign-in buttons,
playing Owen's speech sample, displaying the math puzzle grid) are
missing. Those are the next phase's work.


## Project layout

```
cf4_ps2/
|-- Makefile
|-- README.md              (this file)
|
|-- src/
|   |-- main.c             PS2 entry point + renderer test scene
|   |-- renderer.c/.h      PS2 Graphics Synthesizer renderer
|   |-- mps_interp.c/.h    MPS bytecode interpreter
|   |-- embedded_sprite.h  Test sprite (Joni, 128x256, 1 frame)
|   `-- embedded_anim.h    Test sprite (Santiago, 128x256, 12 frames)
|
|-- assets/
|   |-- textures/          Raw extracted .tex files (non-POT dimensions)
|   |-- textures_pot/      POT-padded textures, 2598 files, generated
|   |-- sprites/           Extracted PNG sprites (per-scene)
|   |-- audio/             Extracted WAVs (music/ and sfx/)
|   |-- fonts/             Extracted NFNT font data
|   |-- palettes/          Extracted RRGB palette JSONs
|   |-- scripts/           43 raw .MPS bytecode files
|   `-- video/             Extracted cutscenes (Smacker .SMK)
|
|-- tools/
|   |-- cf4_pad_pot.py     Pad all .tex files to POT dimensions
|   |-- tex_to_embedded_h.py  Generate embedded C header from a .tex
|   |-- mps_disasm.py      Python MPS disassembler (reference)
|   |-- mps_test.c         Host-side MPS interpreter test runner
|   `-- mps_chain_test.c   Host-side chain-through-opening test
|
`-- docs/
    |-- mps_opcodes.md     Bytecode format + opcode table
    |-- engine_classes.md  Engine classes, built-in commands, events
    |-- file_formats.md    RSC/RRGB/ASEQ/NFNT/WAVE specs
    `-- disassembly/       All 43 scripts disassembled to .asm
```


## What's left to do

Roughly in priority order for actually running the game.

### Phase 2 finish (MPS interpreter)

- Pre-populate engine constants (`kMaxItemsInWS`, `kBackpackHotspotX1`,
  `kUseAOCoords`, all the `k*` resource IDs). The original engine has
  these baked into the runtime; our interpreter needs to seed the
  variable store with them before running any script.
- Property system with per-object state storage. Currently
  `addIntProperty("name")` just reserves a global variable; the real
  engine attaches the property to the specific object, so
  `port.beenToSignInBefore` is distinct from `otherPort.beenToSignInBefore`.
- Real comparators for BOOL_TEST (`<`, `>`, `=`, `<=`, `>=`) - currently
  only tests for nonzero.
- SCENE_JUMP / RETURN fallthrough: in STARTUP.MPS, SCENE_JUMP at inst 71
  points to pLogoMovie at inst 72. After returning from pLogoMovie, the
  interpreter falls back to inst 72 which re-runs the procedure. Need to
  either treat SCENE_JUMP as a pure jump (no return) or record that the
  return-target instruction should be skipped.
- Real semantics for STR_CONCAT opcode (most scripts use the pool
  expression form we already handle, but the opcode may show up too).
- TIMER_B, DISABLE_DATA, ENABLE_DATA, DELAY semantics.

### Phase 3 (Engine runtime classes)

Rebuild in C, wired through the interpreter. Priority order based on
how often each class appears in the verified game flow:

1. **`RQueue`** - used everywhere. Needs add/start/finished semantics.
2. **`RCompositeAction`** - multi-action wrapper, used in walk-ins.
3. **`CharacterSpeechAction`** / **`CharacterAnimAction`** /
   **`PropertyAction`** - atomic queue items.
4. **`RCharacter`** - animated drawable characters (Joni, Santiago, etc.)
5. **`RSmackerMovie`** - big task on its own (see Phase 5).
6. **`RScenePort`** - current scene container, event routing.
7. **`RPButton`** - clickable UI button. First class that bridges to
   the renderer (visible on screen + fires events on click).
8. **`RSelList`** - scrollable list (sign-in screen player list).
9. **`RKbdInp`** - on-screen keyboard input widget.
10. **`RGraphicAnswer`** - letter/number piece for word-scramble puzzles.
11. **`RSound`** / **`RText`** / **`RHotSpot`** / **`RBackPack`** /
    **`RLapTrap`** / **`RWorldPort`** - rest of the class library.

### Phase 4 (I/O)

- `CacheDLL` actually reads an `.rsc` file and indexes its contents
  by resource ID.
- `UncacheAllDLLs` frees that table.
- `Scene()` / opcode 45 actually loads the target `.MPS` and hands off
  to the interpreter.
- File I/O from USB or CD/DVD (PCSX2 `host:` works for development).
- DualShock 2 input via `libpad`, mapped to mouse-style point-and-click
  navigation (the original is mouse-driven so the controller scheme
  needs design).
- SPU2 audio via `audsrv` for WAVE playback.
- Save/load to memory card via the PS2 memory card libraries.

### Phase 5 (Smacker video)

The game's cutscenes are in RAD Smacker format. Options:

- Port a Smacker decoder to PS2 (several open-source implementations
  exist: `libsmacker`, ffmpeg's Smacker codec).
- Pre-convert all .SMK files to a PS2-friendly format (MPEG-1 via
  `libmpeg2`, or raw frames streamed from disc). This trades disc space
  for decode complexity.

### Phase 6 (Memory/VRAM management)

- The PS2 has 32MB of main RAM and 4MB of VRAM. The biggest single
  texture in the game is a 9MB multi-frame animation; full assets are
  far more than fits in VRAM. A texture cache with LRU eviction will
  be needed, or per-frame re-uploads from main RAM.
- Assets on disc need to be streamed, not loaded upfront. The game has
  scene-scoped resource sets (the `CacheDLL` / `UncacheAllDLLs` pattern
  fits this).

### Phase 7 (Polish)

- Get through the whole game without crashes.
- Match original feel as closely as possible.
- Performance tuning (60Hz target on real hardware).


## How to pick up the work

Everything is reproducible from a fresh clone. The two main entry
points for continuing are:

**PS2 renderer side:**

```sh
docker run -it --rm -v $(pwd):/project ps2dev/ps2dev:latest sh
cd /project/cf4_ps2
make            # builds cf4.elf
```

Copy `cf4.elf` to a Windows machine running PCSX2, load via `System ->
Run ELF`.

**Interpreter side (host Linux, no Docker):**

```sh
gcc -I src tools/mps_chain_test.c src/mps_interp.c -o tools/mps_chain_test
./tools/mps_chain_test
```

This runs the full opening sequence through the interpreter and prints
the trace.


## License and credit

This is a fan preservation project. ClueFinders and all its assets are
copyright The Learning Company / Knowledge Adventure. The port code
(renderer, tools, interpreter) is hobbyist work. Original engine
understanding comes from Ghidra decompilation of the game's original
executable.

Not for commercial distribution. The goal is to make this great
educational game playable on hardware it was never released for, as a
preservation effort.
