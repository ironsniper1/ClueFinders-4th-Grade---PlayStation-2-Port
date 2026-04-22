# MPS Bytecode Opcodes

The MPS VM is a register-based bytecode interpreter with 21 opcodes.
The main interpreter function is 5317 bytes (the largest in the EXE).

## File Format (Binary, Big-Endian)

| Offset | Type   | Description |
|--------|--------|-------------|
| 0      | u8     | Version (1=BE, 2=LE) |
| 1      | u32 BE | Instruction count |
| 5      | 3*N    | Instructions: (opcode_u8, operand_idx_u16) |
| varies | u32 BE | Operand table entry count |
| varies | 2*N    | Operand table: int16 pool indices (-1 = separator) |
| varies | u32 BE | Pool entry count |
| varies | ...    | Pool entries (entries 0-2 are pre-allocated) |

## Pool Entry Format

Each pool entry contains:
- `u32` type_flags (field[6])
- `u32` value_type: 0/2=string, 3=integer, 4=numeric string
- [if type 3: `u32` integer value]
- [if type 4: `u16` length + ASCII string data]
- `u32` × 4 metadata fields
- `u32` string1_length (0xFFFFFFFF = none), then string1 data
- `u32` string2_length (0xFFFFFFFF = none), then string2 data
- `u32` array_length (0xFFFFFFFF = none), then int16 array data

## Opcodes

| Op | Hex  | Name           | Description |
|----|------|----------------|-------------|
|  6 | 0x06 | SCENE_JUMP     | Load location script |
|  7 | 0x07 | ASSIGN         | Copy values between variables |
|  8 | 0x08 | FOR_INIT       | Initialize for-loop (start, end, step) |
|  9 | 0x09 | FOR_STEP       | Increment loop counter, check bounds |
| 10 | 0x0A | IF_FALSE       | Skip next block if condition false |
| 11 | 0x0B | GOTO           | Set instruction pointer (branch) |
| 12 | 0x0C | BOOL_TEST      | Test value, set condition flag |
| 13 | 0x0D | NOP            | No operation |
| 14 | 0x0E | STR_CONCAT     | Concatenate string arguments |
| 15 | 0x0F | RETURN         | Return from function / end block |
| 18 | 0x12 | EVAL_ASSIGN    | Evaluate expression and assign |
| 19 | 0x13 | DESTROY        | Destroy / cleanup variables |
| 21 | 0x15 | CALL_BUILTIN   | Call built-in command |
| 22 | 0x16 | YIELD          | Pause execution |
| 23 | 0x17 | CALL_METHOD    | Call object method |
| 24 | 0x18 | TIMER_B        | Set timer B |
| 26 | 0x1A | DISABLE_DATA   | Disable data processing |
| 27 | 0x1B | ENABLE_DATA    | Enable data processing |
| 28 | 0x1C | DELAY          | Wait for specified time |
| 29 | 0x1D | NOP2           | No operation (variant) |
| 35 | 0x23 | CALL_METHOD_35 | Call method (variant) |
| 37 | 0x25 | MAIN_PROC      | Main procedure definition |
| 40 | 0x28 | GLOBAL_PROP    | Global property access |

## VM Context Structure (0x790 bytes)

| Offset | Type | Name | Notes |
|--------|------|------|-------|
| +0x018 | ptr  | opcode_array | 8 bytes per instruction at runtime |
| +0x01C | ptr  | operand_table | int16 indices into pool |
| +0x024 | str  | script_name | Current .mps filename |
| +0x228 | arr  | loop_counters | 10 bytes per loop level |
| +0x728 | int  | instruction_count | |
| +0x72C | int  | operand_count | |
| +0x730 | int  | loop_stack_ptr | Current loop nesting |
| +0x76C | int  | exec_state | 0=run, 0x309=done, 0x14D=paused |
| +0x770 | int  | instruction_pointer | Current position |
| +0x774 | int  | waiting | Delay/event wait |
| +0x77C | int  | condition_flag | Result of last test |
