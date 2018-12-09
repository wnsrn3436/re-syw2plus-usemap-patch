# Seven Years War 2+ Use Map Patch

A byte patch for the executable of Seven Years War 2+: Chosun's Counterattack (HQ Team) that makes a normal game load the units placed in the map. Maps with units laid out in advance can be played as they are. The changed bytes and their assembly are described in [docs/usemap-patch.md](docs/usemap-patch.md).

**Compatible version**: Seven Years War 2+: Chosun's Counterattack v1.1000 (+ HQ Net patch), `syw2plus.exe` md5 `ec9f2843de992af91c575bab2e9b64fe`

<p>
  <img src="docs/screenshots/screenshot-1.jpg" width="480" alt="A game started with the patched executable">
</p>


## How to use

The patch is applied by hand from the table in the docs.

1. Back up `syw2plus.exe` in the game folder and check that its md5 matches the value above.
2. In a hex editor, go to each file offset in the table in [docs/usemap-patch.md](docs/usemap-patch.md), check the original bytes, and overwrite them with the new bytes.
3. With the patched executable, pick a map with placed units and start a normal game. The units from the map appear.


## How it works

**The map's units are created instead of the starting units.** When a game starts, the function that creates starting units goes through the eight players and creates the starting units for each race at the player's start location. Separately, the game contains a loop that goes through the map's unit placement table (800 entries of 16 bytes: type, x, y, owner) and calls the same creation function `0x00443190`. The patch replaces the six `CALL`s that create starting units with `NOP`s, turns the final `ret` of the starting unit function into a `JMP` to that loop, and jumps back to a `ret` where the loop ends.

```
0041F356  E9 A0 4B 0A 00        jmp  0x004C3EFB        ; to the map unit placement loop
0041F35B  C3                    ret
...
004C3F73  E9 E3 B3 F5 FF        jmp  0x0041F35B        ; back to ret when the loop ends
```

The creation function is cdecl, so the caller cleans up the arguments. Removing only the 5-byte `CALL` leaves the `push`es before it and the `add esp, 0x1C` after it paired up, and the stack stays balanced. The code inside the loop is what the game already had. The patch only jumps in and out with two jumps.


## Files

| Path | Contents |
|---|---|
| `docs/usemap-patch.md` | Target file, table of changed bytes, assembly notes |
| `docs/screenshots/` | Screenshots |


## License

zlib. See [LICENSE](LICENSE).
