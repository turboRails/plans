# Micro-Py: 8-Bit Microprocessor Simulator

## Context

Build a Python-based 8-bit microprocessor simulator from scratch in an empty project directory. The simulator will be educational, demonstrating how a real CPU works (fetch-decode-execute cycle, registers, memory, I/O). Zero external dependencies — pure Python standard library.

## Architecture: MicroPy-8

- **8-bit data bus**, **16-bit address bus** (64KB memory)
- **Registers**: A (accumulator), B, C, D (general purpose), SP (stack pointer), PC (16-bit program counter), FLAGS (Z/C/N/O/H)
- **Memory map**: Zero page (0x0000-0x00FF), Stack (0x0100-0x01FF), Program+Data (0x0200-0xFEFF), I/O ports (0xFF00-0xFFFF)
- **I/O ports**: 0xFF00 = char output, 0xFF01 = char input, 0xFF02 = numeric output
- **~40 instructions**: data movement, arithmetic, logic, comparison, control flow, system
- **Variable-length encoding**: 1-3 bytes per instruction
- **Two-pass assembler** with labels, `.org`, `.db` directives

## File Structure

```
micro-pi/
├── pyproject.toml
├── micro_py/
│   ├── __init__.py          # Package + version
│   ├── __main__.py          # Entry: python -m micro_py
│   ├── cli.py               # argparse CLI (run/asm/debug/repl)
│   ├── opcodes.py           # Opcode table, enums, instruction defs
│   ├── alu.py               # Arithmetic/logic unit (stateless)
│   ├── memory.py            # 64KB memory + I/O interception
│   ├── cpu.py               # CPU: registers, fetch-decode-execute
│   ├── assembler.py         # Two-pass assembler
│   ├── disassembler.py      # Binary -> readable assembly
│   ├── loader.py            # Load .asm/.bin into memory
│   └── debugger.py          # Interactive debugger (cmd.Cmd)
├── programs/
│   ├── hello.asm            # Hello World via I/O port
│   ├── add.asm              # Add two numbers
│   ├── countdown.asm        # Loop countdown 10→0
│   ├── fibonacci.asm        # Fibonacci sequence
│   └── factorial.asm        # Factorial with subroutine
└── tests/
    ├── __init__.py
    ├── test_alu.py
    ├── test_memory.py
    ├── test_cpu.py
    ├── test_assembler.py
    └── test_integration.py
```

## Implementation Order

1. **`opcodes.py`** — Enums (RegisterID, AddressingMode), Instruction dataclass, opcode table
2. **`alu.py`** — Pure arithmetic/logic with flag computation
3. **`memory.py`** — 64KB bytearray, read/write, I/O port hooks
4. **`cpu.py`** — Registers, fetch-decode-execute loop, instruction dispatch
5. **`assembler.py`** — Tokenizer, two-pass assembly, label resolution
6. **`disassembler.py`** — Reverse lookup from bytes to assembly
7. **`loader.py`** — Glue: read file → assemble → load into memory
8. **`cli.py`** + **`__main__.py`** — CLI with `run`, `asm`, `debug`, `repl` subcommands
9. **`debugger.py`** — Interactive debugger (step, run, regs, mem, break, disasm)
10. **`programs/*.asm`** — Example assembly programs
11. **`tests/`** — Unit + integration tests
12. **`pyproject.toml`** — PEP 621 metadata

## Instruction Set Summary

| Category | Instructions |
|---|---|
| Data Movement | MOV, LDI, LDA, STA, LDR, STR, PUSH, POP |
| Arithmetic | ADD, ADI, SUB, SBI, INC, DEC, MUL |
| Logic | AND, OR, XOR, NOT, SHL, SHR |
| Comparison | CMP, CMI |
| Control Flow | JMP, JZ, JNZ, JC, JN, CALL, RET |
| System | NOP, HLT, BRK |

## Key Design Decisions

- **8-bit** for simplicity — all registers are single bytes, easy to reason about
- **Memory-mapped I/O** — teaches the concept, no separate I/O instructions needed
- **Variable-length instructions** — mirrors real architectures (x86, 6502)
- **Two-pass assembler** — simplest way to support forward label references
- **`cmd.Cmd` debugger** — stdlib, readline support for free
- **Max cycle guard** (default 1M) — prevents infinite loops from hanging

## Verification

1. `python -m micro_py run programs/add.asm` → prints `42`
2. `python -m micro_py run programs/hello.asm` → prints `Hello`
3. `python -m micro_py run programs/factorial.asm` → prints `120`
4. `python -m micro_py debug programs/fibonacci.asm` → interactive debugger works
5. `python -m pytest tests/` → all tests pass
