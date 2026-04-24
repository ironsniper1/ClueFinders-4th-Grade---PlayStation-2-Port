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
  RAnimation, etc.
- **Event-driven execution model:** scripts register event handlers
  (`movie.finished`, `character.mouseDown`, etc.) and the engine pumps
  events into them.
- **Adaptive difficulty** with telemetry on wrong guesses, per-activity
  auto-leveling thresholds, and multi-dataset random selection.
- **Smacker (.SMK) video** for cutscenes.

Porting this to PS2 requires reverse-engineering the engine (via Ghidra
on the original EXE), rebuilding each piece in C targeting the PS2's GS
(Graphics Synthesizer), SPU2 (audio), and libpad (input) subsystems.


## Current state

**Overall project: ~38% complete**

| Phase | Area                    | Status   |
|-------|-------------------------|----------|
| 1     | PS2 renderer            | ~95%     |
| 2     | MPS bytecode interpreter| **complete** |
| 3     | Engine runtime classes  | 0%       |
| 4     | I/O (input/audio/files) | 0%       |
| 5     | Smacker video playback  | 0%       |
| 6     | Memory/VRAM management  | 0%       |
| 7     | Polish and full testing | 0%       |

**What runs today:**

- A PS2 ELF that boots on real hardware and in PCSX2, renders sprites
  with transparency and animation.
- A host-side MPS interpreter that loads every one of the game's 43
  scripts with correct semantics - 24 opcodes, full expression evaluator
  with all comparison/arithmetic/string operators, per-object events,
  cross-file scene chaining.
- All 43 scripts surveyed with each script's gameplay mechanics
  identified through systematic bytecode analysis.
- End-to-end opening sequence verified:
  - STARTUP.MPS -> Logo.smk -> MVOP1.smk -> MVTitle.smk -> SCENE_LOAD
    to Signin.mps
  - SIGNIN.MPS creates the sign-in UI (6 buttons, player list, keyboard
    input), wires 8 event handlers
  - Simulated Start click -> procStartButtonHit -> procEntrySelected
    -> "Limburger" cheat check (false) -> standard login path ->
    SCENE_LOAD curScript+".mps" resolves to CHUB.mps
  - CHUB.mps wires 13 handlers; character clicks (Joni, Socrates,
    LapTrap, Dealer, Backpack) correctly fire unique per-character logic
  - CMA.MPS math puzzle loads and initializes its 28-level board

**What doesn't run yet:** anything where the script's side effects need
to actually affect the screen, speakers, or file system. All "do something"
operations are currently logged stubs. Phase 3 (engine runtime classes)
is where that work begins.


## 43-script survey results

Every `.MPS` file in the game loaded, parsed, and executed its
MAIN_PROC + pInit + pStart protocol without errors.

```
script            inst  pool  handlers  notes
----------------- ----- ----- --------  ---------------------------
STARTUP.MPS       103   96    4         Boot: cache DLLs, create port
SIGNIN.MPS        704   563   1         Sign-in screen
CHUB.mps          1065  866   15        Cairo hub
OHUB.MPS          1470  1507  12        Oasis hub
CMA.MPS           1241  2430  20        Cairo math puzzle
OMA.MPS           1593  1346  23        Oasis catapult (most handlers)
CWS1.mps          2162  5753  15        Coffee-shop math (361 problems)
CWS2.mps          1481  3570  16        Tailor
CWS3.mps          1245  1945  18        Mason
CWS4.mps          1580  2930  16        Export/shipping
OWS1.MPS          1642  4516  10        Fixnuppan list puzzle
OWS2.MPS          1274  1803  10        Hieroglyph puzzle
OWS3.MPS          1748  2863  9         Bilko match
OWS4.MPS          1582  2687  13        Sentence building
PWS1.MPS          1011  923   15        Sphinxo
PWS2.MPS          1411  2687  7         Thoth letter puzzle
CBA1/2, PBA       ~700  ~900  ~14       Vehicle/build/transition
CLOC* x 14        ~650  ~720  ~12       Cairo locations (uniform)
OLOC* x 8         ~640  ~720  ~11       Oasis locations (very uniform)
PLOC2/3           ~540  ~480  ~12       Pyramid cutscene/transition
FL.MPS            412   421   3         Admin/level-select panel

Total: 43/43 scripts pass.
```


## Full game structure

### Cairo world (16 scripts)

| Script | Role | Characters | Mechanic |
|--------|------|-----------|----------|
| CHUB | Hub | Dealer | Scroll/clue/glyph collection with progressive cast reactions |
| CMA | Math puzzle | Geezar (villain's minion) | 28-level grid with piece variants |
| CBA1 | Vehicle puzzle | Dealer | Jeep exit mechanic |
| CBA2 | Build puzzle | Boat Girl | Boat assembly (sail/hull/motor/launch) |
| CWS1 | Commerce puzzle | Coffee Guy | 361 unique math problems, cups on tray sum to solution |
| CWS2 | Craft puzzle | Seller | Scissors on fabric (geometry) |
| CWS3 | Build puzzle | Mason | Containers + sled movement |
| CWS4 | Commerce puzzle | Exportress | Packages in/out |
| CLOC01/09/13 | Statue puzzle | - | Assembly of statue parts with state persistence |
| CLOC02-14 | Location | - | Basic navigation templates |

### Oasis world (9 scripts)

| Script | Role | Characters | Mechanic |
|--------|------|-----------|----------|
| OHUB | Hub | Purrina (cat) | Dissolving doors with escalating payoff speeches |
| OMA | Catapult puzzle | Crocodile (4-cycling dialogue) | Trajectory math with 7-notch ramp, 4-aim positions |
| OWS1 | Collect puzzle | Fixnuppan, Santiago | Add/remove answer list with adaptive difficulty |
| OWS2 | Glyph puzzle | Art Mouse, Joni | Hieroglyph replay + reward cutscene |
| OWS3 | Identify puzzle | Bilko, Leslie | Drag-to-match with wrong-answer handling |
| OWS4 | Sentence puzzle | Rocko | Sentence building + "mice trip" animation |
| OLOC02-09 | Location | - | Very uniform templates |

### Pyramid world (4 scripts)

| Script | Role | Characters | Mechanic |
|--------|------|-----------|----------|
| PWS1 | Pickup puzzle | Sphinxo | Generic pickup |
| PWS2 | Word puzzle | Thoth (god of writing) | Drag letter pieces into boxes |
| PBA | Transition | Person | Scene bridging |
| PLOC2 | Cutscene | - | Movie playback with interrupt controls |
| PLOC3 | Transition | Person | Scene bridging |

### Special

| Script | Role |
|--------|------|
| STARTUP.MPS | Boot - caches DLLs, declares ~40 global properties, plays 3 intro movies |
| SIGNIN.MPS | Sign-in screen - 6 buttons, player list, keyboard input, "Limburger" QA cheat |
| FL.MPS | Teacher/admin panel - 48-button grid (8x6) for level select, 98 player slots |


## Verified engine mechanics (discovered through script tracing)

### Character click escalation

Characters respond differently based on state or click counter:

- **Socrates click:** context-sensitive help based on scene state
- **Character click (CHUB):** first click has Owen speak, subsequent clicks cycle
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

`kLastClueSpchIDs.whichDataset` means **multiple possible endings**
depending on which randomly-selected dataset the game uses.

### Door-dissolve payoff (OHUB)

When a gem-collection puzzle completes, Purrina speaks in escalating
tiers based on `numOpenDoors`. The final (5th) door triggers a full-cast
celebration with Purrina, LapTrap, Owen, and Santiago each speaking.

### Adaptive difficulty / telemetry (OWS1)

Every puzzle tracks incorrect attempts and logs them:

- First round: 3 wrong tries triggers `gPort.incorrectGuess("OWS1")`
- Subsequent rounds: 6 wrong tries (grace period for learning curve)

STARTUP declares auto-level thresholds per activity (wsAutoLevelingA/B/X/Y)
that adjust difficulty based on accumulated telemetry.

### "Limburger" QA cheat

Typing "Limburger" as the player name on the sign-in screen shortcuts
the entry-selection path to `callQaCheatScript`, which activates a "QA"
player account and falls into the teacher panel flow.

### Multi-outcome collision (OMA)

When the catapult launches a rock, the rock can hit 4 distinct targets
each with its own handler:

- `pRockHitWater` - splash, try again
- `pRockHitStatue` - spawn `RAnimation` with `kRockCracksUpID[activeRock]`
  (different crack anim per rock type!), chain to `pRockHitLocation`
- `pRockHitLocation` - evaluates whether math answer matches
- `pRockStraightUp` - fell back down

### Object userData and string operators

The engine tracks per-object arbitrary data via `userData`. Strings like
`"FIRE_GEM#3"` encode multi-field data in a single property, parsed via:

- `@` (find): `"#" @ ud` - returns 1-based position of `#` in ud
- `|` (substring): `ud | 1 | p` - substring from position 1, length p


## Reverse engineering work

Before any port work started, substantial reverse engineering
established the foundation. This is documented in `docs/`:

- **`docs/mps_opcodes.md`** - bytecode format spec and opcode table
- **`docs/engine_classes.md`** - engine classes, built-in commands, events
- **`docs/file_formats.md`** - RSC/RRGB/ASEQ/NFNT/WAVE specs
- **`docs/disassembly/`** - all 43 MPS scripts disassembled

Additional classes discovered during script tracing (not in original
docs):

- **RPButton** - clickable UI button
- **RSelList** - scrollable selection list
- **RKbdInp** - on-screen keyboard input
- **RGraphicAnswer** - draggable letter/number piece
- **RAnimation** - animated sprite with play/pause control
- **RCompositeAction** - multi-action wrapper for queuing
- **CharacterSpeechAction** / **CharacterAnimAction** /
  **PropertyAction** / **SoundAction** / **AnimAction** - atomic queue items
- **RLapTrap** - the in-game hint device (context-sensitive per-location)

### Characters catalogued

**Main cast (6):** Joni, Santiago, Owen, Leslie, Socrates, LapTrap
**Villain:** Geezar
**NPCs (13):** Dealer, Boat Girl, Coffee Guy, Seller, Mason, Exportress,
Purrina, Fixnuppan, Art Mouse, Bilko, Rocko, Sphinxo, Thoth, crocodile


## PS2 renderer

The renderer (`src/renderer.c`) drives the PS2's Graphics Synthesizer
directly using `ps2sdk`'s draw/graph/dma/packet libraries.

**Capabilities:**

- GS initialization, double-buffered 640x448 NTSC framebuffers in
  `GRAPH_MODE_FRAME`
- Manual GS register setup (FRAME, ZBUF, XYOFFSET, SCISSOR, etc.)
- Rectangle drawing with alpha
- Textured sprite drawing with alpha testing (ATST=6 GREATER, AREF=0)
- Multi-frame sprite animation
- PSMCT32 32-bit RGBA texture format

**Asset pipeline:** All 2598 textures pre-padded to POT dimensions via
`tools/cf4_pad_pot.py`.

**Build:** Inside ps2dev Docker:

```sh
cd /project/cf4_ps2
make
```

Produces `cf4.elf`. Load in PCSX2 via System -> Run ELF.


## MPS bytecode interpreter

The interpreter (`src/mps_interp.c`) is a complete reimplementation of
the game engine's scripting VM.

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

### Opcode table

All 28 opcode values seen in any script are handled:

| Op | Name            | Status  |
|----|-----------------|---------|
|  6 | SCENE_JUMP      | Full (with return-target skip) |
|  7 | ASSIGN          | Full    |
|  8 | FOR_INIT        | Full (with skip-frame for undefined end) |
|  9 | FOR_STEP        | Full    |
| 10 | IF_FALSE        | Full    |
| 11 | GOTO            | Full    |
| 12 | BOOL_TEST       | Full (via expression evaluator) |
| 13 | NOP             | Full    |
| 14 | STR_CONCAT      | Stub (pool expressions used instead) |
| 15 | RETURN          | Full (with proc-entry skip fix) |
| 18 | EVAL_ASSIGN     | Full    |
| 19 | DESTROY         | Full    |
| 21 | CALL_BUILTIN    | Partial |
| 22 | YIELD           | Full    |
| 23 | CALL_METHOD     | Partial (generic handler registration) |
| 24 | TIMER_B         | Stub    |
| 26 | DISABLE_DATA    | Stub    |
| 27 | ENABLE_DATA     | Stub    |
| 28 | DELAY           | Stub    |
| 29 | NOP2            | Full    |
| 35 | CALL_METHOD_35  | Partial |
| 36 | CALL_METHOD_RET | Partial |
| 37 | MAIN_PROC       | Full    |
| 40 | GLOBAL_PROP     | Partial |
| 45 | SCENE_LOAD      | Full    |

### Expression evaluator

Pool entries encode expressions via `(str2, array)` pairs:

- **`str2`** is a stream of type codes and operator characters
- **`array`** holds the pool indices of symbolic/literal parts

Supported operators (with standard arithmetic precedence):

| Op | Name | Notes |
|----|------|-------|
| `+ - * /` | arithmetic | integer math |
| `= #` | equality | works for both ints and strings |
| `< >` | comparison | integer comparison |
| `@` | find | 1-based position of substring in string |
| `( )` | grouping | |

### Variable store

- 256 slots holding either int or string values
- Flat global namespace (per-object properties deferred to Phase 3)

### Loop stack

- Up to 16 nested for-loops
- Tracks counter, end value, step, body entry point
- Skip-flag for loops where FOR_INIT can't resolve its end operand

### Call stack

- Up to 32 levels
- SCENE_JUMP pushes, RETURN pops
- After return, if landed on a procedure entry point (collision), skip
  past it to avoid re-running the proc
- Fresh stack on every event fire

### Event handler table

- Up to 128 handlers
- Registration: any `CALL_METHOD(obj, event_name, procRef)` where
  procRef is a pool entry with type=3 and a string name registers the
  handler
- Re-registration replaces existing entries
- DESTROY on an object removes all its handlers

### Public API

- `mps_load`, `mps_step`, `mps_running`, `mps_free`
- `mps_call_proc(name)`, `mps_find_proc(name)`, `mps_fire_event`
- `mps_list_handlers`, `mps_pending_script`
- Variable helpers: `mps_var_find`, `mps_var_set_int/str`, `mps_var_get_int`

### Host-side testing tools

- **`tools/mps_test.c`** - runs any single script with full trace
- **`tools/mps_chain_test.c`** - simulates the full opening sequence
- **`tools/script_survey.c`** - runs every script through MAIN_PROC +
  pInit + pStart, prints summary table
- **`tools/survey_proc.c`** - per-procedure breakdown for diagnostics
- **`tools/scan_procs.py`** - pool scanner listing all named procedures
- **`tools/uniq_events.py`** - per-script unique-event fingerprint


## Project layout

```
cf4_ps2/
|-- Makefile
|-- README.md
|
|-- src/
|   |-- main.c             PS2 entry point
|   |-- renderer.c/.h      PS2 GS renderer
|   |-- mps_interp.c/.h    MPS bytecode interpreter
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
|   |-- mps_test.c
|   |-- mps_chain_test.c
|   |-- script_survey.c
|   |-- survey_proc.c
|   |-- scan_procs.py
|   |-- uniq_events.py
|   `-- apply_*.py         Patch scripts for interpreter updates
|
`-- docs/
    |-- mps_opcodes.md
    |-- engine_classes.md
    |-- file_formats.md
    `-- disassembly/       All 43 scripts disassembled
```


## What's left to do

### Phase 3 (Engine runtime classes) - 0% done

Rebuild in C, wired through the interpreter. Priority order based on how
often each class appears in the verified game flow:

1. **RQueue** - used everywhere. Needs add/start/finished semantics.
2. **RCompositeAction** - multi-action wrapper.
3. **CharacterSpeechAction** / **CharacterAnimAction** /
   **PropertyAction** / **SoundAction** / **AnimAction** - atomic queue items.
4. **RCharacter** - animated drawable characters.
5. **RAnimation** - standalone animated sprite.
6. **RScenePort** - scene container + event routing.
7. **RPButton** - clickable UI button (first renderer bridge).
8. **RSelList** - scrollable selection list.
9. **RKbdInp** - on-screen keyboard.
10. **RGraphicAnswer** - draggable puzzle piece.
11. **RLapTrap** - hint device.
12. **RSmackerMovie** - video playback (see Phase 5).
13. **RSound** / **RText** / **RHotSpot** / **RBackPack** / **RWorldPort**.

### Phase 4 (I/O) - 0%

- `CacheDLL` reads `.rsc` files; `UncacheAllDLLs` frees
- `SCENE_LOAD` hands off to new interpreter context
- USB / CD / DVD file I/O
- DS2 input via libpad
- SPU2 audio via audsrv
- Memory card save/load

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
docker run -it --rm -v $(pwd):/project ps2dev/ps2dev:latest sh
cd /project/cf4_ps2
make
```

**Interpreter side (host Linux):**

```sh
gcc -I src tools/mps_chain_test.c src/mps_interp.c -o tools/mps_chain_test
./tools/mps_chain_test
```

Full 43-script survey:

```sh
gcc -I src tools/script_survey.c src/mps_interp.c -o tools/script_survey
tools/script_survey 2>/dev/null | grep -E "inst="
```


## License and credit

This is a fan preservation project. ClueFinders and all its assets are
copyright The Learning Company / Knowledge Adventure. The port code
(renderer, tools, interpreter) is hobbyist work. Engine understanding
comes from Ghidra decompilation of the original executable.

Not for commercial distribution. The goal is to make this great
educational game playable on hardware it was never released for.
