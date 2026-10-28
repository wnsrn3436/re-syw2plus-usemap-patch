# Seven Years War 2+ use map patch

This patch changes the executable of Seven Years War 2+: Chosun's Counterattack so that the units placed in the map appear when a normal game starts. It removes the six calls that create each player's starting units, and at the end of that function it jumps to code already in the game. That code walks the map's unit placement table and creates the units.


## Target

| Item | Value |
|---|---|
| Game | Seven Years War 2+: Chosun's Counterattack v1.1000 (+ HQ Net patch) |
| File | `syw2plus.exe` (1,032,192 bytes) |
| md5 | `ec9f2843de992af91c575bab2e9b64fe` |

The file is not packed and the image base is `0x400000`. Every change is inside the `.text` section, and `.text` maps virtual address `0x401000` to file offset `0x1000`. So **file offset = virtual address - 0x400000**.


## Changed bytes

| Virtual address | File offset | Original bytes | New bytes | Meaning |
|---|---|---|---|---|
| `0x0041F18F` | `0x1F18F` | `E8 FC 3F 02 00` | `90 90 90 90 90` | Remove a starting unit call |
| `0x0041F1CF` | `0x1F1CF` | `E8 BC 3F 02 00` | `90 90 90 90 90` | Remove a starting unit call |
| `0x0041F251` | `0x1F251` | `E8 3A 3F 02 00` | `90 90 90 90 90` | Remove a starting unit call |
| `0x0041F293` | `0x1F293` | `E8 F8 3E 02 00` | `90 90 90 90 90` | Remove a starting unit call |
| `0x0041F2E6` | `0x1F2E6` | `E8 A5 3E 02 00` | `90 90 90 90 90` | Remove a starting unit call |
| `0x0041F328` | `0x1F328` | `E8 63 3E 02 00` | `90 90 90 90 90` | Remove a starting unit call |
| `0x0041F356` | `0x1F356` | `C3 90 90 90 90 90` | `E9 A0 4B 0A 00 C3` | Jump to the map unit placement loop instead of `ret` |
| `0x004C3F73` | `0xC3F73` | `B9 70 67 95 00` | `E9 E3 B3 F5 FF` | When the loop ends, return to the `ret` at `0x0041F35B` |


## The starting unit function

When `[0x00975914]`, which appears to be the game type, is 3, `0x0041F0E0` calls `0x0041F360` and returns. Otherwise it falls through to `0x0041F0F8` and creates the starting units of the eight players.

```
0041F0E0  66 83 3D 14 59 97 00 03  cmp  word [0x00975914], 3
0041F0E8  75 0E                    jnz  0x0041F0F8
0041F0EA  8B 44 24 04              mov  eax, [esp+4]
0041F0EE  50                       push eax
0041F0EF  E8 6C 02 00 00           call 0x0041F360
0041F0F4  83 C4 04                 add  esp, 4
0041F0F7  C3                       ret
0041F0F8  53 55 8B 6C 24 0C 56 57  push ebx ; push ebp ; mov ebp, [esp+0xC] ; push esi ; push edi
0041F100  33 F6 33 FF              xor  esi, esi ; xor edi, edi
0041F104  BB 70 67 95 00           mov  ebx, 0x00956770    ; player 1 data
```

Player data is eight records starting at `0x00956770`, spaced `0x3ABC` bytes apart, and the first byte of each is the race. For each of races 1, 2, and 3 the function creates two unit types, and if `[0x00975914]` is 1 it creates one more unit of type `0x51`. The position comes from the x at `0x00B3DE58 + player index * 0x338` and the y at `0x00B3DE5C + player index * 0x338`.

| Call site | Unit type | Condition |
|---|---|---|
| `0x0041F18F` | `0x51` | `[0x00975914]` is 1 |
| `0x0041F1CF` | `0x31` | Race 1 |
| `0x0041F293` | `0x07` / `0x1F` | Race 1 / race 2 (both branches share this call) |
| `0x0041F251` | `0x3A` | Race 2 |
| `0x0041F2E6` | `0x46` | Race 3 |
| `0x0041F328` | `0x15` | Race 3 |

Each call looks like this. If `0x00442FA0(type)` returns a nonzero value, `0x00443190` is called with it.

```
0041F21F  6A 3A                 push 0x3A
0041F221  E8 7A 3D 02 00        call 0x00442FA0
0041F226  0F BF C8              movsx ecx, ax
0041F229  83 C4 04              add  esp, 4
0041F22C  85 C9                 test ecx, ecx
0041F22E  74 29                 jz   0x0041F259
0041F230  0F BF C6              movsx eax, si
0041F233  69 C0 38 03 00 00     imul eax, eax, 0x338
0041F239  57                    push edi               ; player index
0041F23A  6A 64                 push 100
0041F23C  66 8B 90 5C DE B3 00  mov  dx, [eax+0x00B3DE5C]  ; y
0041F243  66 8B 80 58 DE B3 00  mov  ax, [eax+0x00B3DE58]  ; x
0041F24A  6A 01                 push 1
0041F24C  52                    push edx
0041F24D  50                    push eax
0041F24E  51                    push ecx
0041F24F  6A 3A                 push 0x3A              ; unit type
0041F251  E8 3A 3F 02 00        call 0x00443190        ; five NOPs in the patch
0041F256  83 C4 1C              add  esp, 0x1C
```

`0x00443190` is cdecl. The caller cleans up the arguments (`add esp, 0x1C`), so the stack stays balanced even when only the `CALL` is replaced with `NOP`s.

The function restores its registers and ends with `ret`. The patch turns that `ret` into a `JMP` to the map unit placement loop and puts a new `ret` in the last of the five `90` bytes that follow, as the place to come back to.

```
0041F352  5F 5E 5D 5B           pop edi ; pop esi ; pop ebp ; pop ebx
0041F356  C3                    ret                    ; original
0041F356  E9 A0 4B 0A 00        jmp  0x004C3EFB        ; patched
0041F35B  C3                    ret                    ; patched (was 90)
```


## The map unit placement loop

The function `0x004C3E50` contains a loop that goes through the map's unit placement table and creates the units. The table starts at `0x00B3F818` and holds 800 entries of 16 bytes.

| Offset in entry | Size | Contents |
|---|---|---|
| `+0x0` | 4 | Unit type (0 and 1 are skipped) |
| `+0x4` | 2 | x |
| `+0x8` | 2 | y |
| `+0xC` | 2 | Owner player |

```
004C3EF9  33 FF                 xor  edi, edi
004C3EFB  BE 20 F8 B3 00        mov  esi, 0x00B3F820   ; where the patch jumps in
004C3F00  8B 46 F8              mov  eax, [esi-8]      ; unit type
004C3F03  83 E8 00              sub  eax, 0
004C3F06  74 60                 jz   0x004C3F68
004C3F08  48                    dec  eax
004C3F09  74 5D                 jz   0x004C3F68
004C3F0B  66 8B 46 FC           mov  ax, [esi-4]       ; x
004C3F0F  66 8B 0E              mov  cx, [esi]         ; y
004C3F12  66 85 C0              test ax, ax
004C3F15  7C 51                 jl   0x004C3F68
004C3F17  66 3B 05 34 DE B3 00  cmp  ax, [0x00B3DE34]
004C3F1E  7D 48                 jge  0x004C3F68
004C3F20  66 85 C9              test cx, cx
004C3F23  7C 43                 jl   0x004C3F68
004C3F25  66 3B 0D 36 DE B3 00  cmp  cx, [0x00B3DE36]
004C3F2C  7D 3A                 jge  0x004C3F68
004C3F2E  66 8B 56 04           mov  dx, [esi+4]       ; owner player
004C3F32  52                    push edx
004C3F33  6A 64                 push 100
004C3F35  6A 01                 push 1
004C3F37  51                    push ecx
004C3F38  50                    push eax
004C3F39  66 8B 46 F8           mov  ax, [esi-8]
004C3F3D  57                    push edi
004C3F3E  50                    push eax
004C3F3F  E8 4C F2 F7 FF        call 0x00443190        ; same creation function as the starting units
004C3F44  83 C4 1C              add  esp, 0x1C
...
004C3F68  47                    inc  edi
004C3F69  83 C6 10              add  esi, 0x10
004C3F6C  66 81 FF 20 03        cmp  di, 800
004C3F71  7C 8D                 jl   0x004C3F00
004C3F73  B9 70 67 95 00        mov  ecx, 0x00956770   ; original
004C3F73  E9 E3 B3 F5 FF        jmp  0x0041F35B        ; patched
```

With the end of the loop (`0x004C3F73`) turned into a jump to the `ret` at `0x0041F35B`, the starting unit function creates the units placed in the map and then returns to its caller.


## Left for further investigation

- The jump lands at `0x004C3EFB`, the instruction after `xor edi, edi`. The loop counter `edi` starts from the caller's `edi` value, that is, the value restored at `0x0041F352`.
- `0x004C3E50` also runs through this loop, so with the patch applied it goes to `0x0041F35B` once the loop ends. Which screen's path runs through `0x00424D30`, the caller of that function.
- Which game type each value of `[0x00975914]` stands for, and where the code that fills the table is.


## Patching by hand

1. Check that the md5 of `syw2plus.exe` is `ec9f2843de992af91c575bab2e9b64fe`.
2. In a hex editor, go to each file offset in the table above and check that the original bytes match.
3. Overwrite them with the new bytes. Each replacement is the same length as the original.
