# Translating a Commercial Game (ROM Patching)

How to translate an existing NGPC game: find its text, make room, put the new text back,
redraw the graphics that carry words, and prove nothing else changed. Everything here was
learned on more than a dozen fan translations of retail games (Japanese to a European
language); a rule is listed when it held on several of them.

The deliverable is a **patch** (IPS) plus the tools that build it — never the ROM.

---

## 1. Ground rules

* **Translate from the original language.** When an English release exists, it is often the
  best *technical* base (Latin font already there, wider boxes, sometimes a bigger ROM), but its
  text is a translation of a translation, and it is not always right:
  * it may **merge variants** the original keeps apart (a partner who speaks differently
    depending on the player's gender: two entries in the table, both pointed at one English
    text);
  * it may leave original-language strings **still in use**;
  * it may get **values** wrong — check them in the code, not in the text.
* To align the two releases: tables that are parallel (same index), the sites that point at the
  strings (same code around them), and sequence alignment on the skeleton of control codes. Event
  tables often have identical headers in both releases.
* **Every line must make sense on its own**, with the speaker, in the target language —
  localise, do not word-for-word translate.

---

## 2. Making room

### 2.1 Grow the ROM to 4 MiB and put everything new on the second chip

A cartridge above 2 MB is two chips (see Storage and Saves §5.0e): the second one is at CPU
`0x800000`, file offset `0x200000` for a 4 MiB image. Growing the image to 4 MiB gives a
whole empty chip for new text, new fonts, new graphics and hook code, reachable with ordinary
32-bit pointers. This was the base of almost every project. Pad the file to exactly 4 MiB.

### 2.2 ⛔ The run of `FF` at the end of a chip is NOT free space

On a retail cartridge **the last blocks of the chip are the save**. In the ROM dump they read
`FF` and look empty. Writing new data there broke a game's records table and mileage display;
clues before it happens: the game's flash calls (BIOS `FLASHWRITE`/erase), "write error"
messages, a table of block numbers. Exclude the last blocks of each chip, and any block the
game writes.

### 2.3 The save block MOVES when the chip gets bigger

Games pass the save location to the BIOS as a **block number** (and often a size), hard-coded.
A block number is an address *for one chip size*: block 17 of an 8 Mbit chip is the 8 KiB block
at `0xFA000`; on a 16 Mbit chip block 17 is a 64 KiB block at `0x110000` — **in the middle of
your new data**, which the first save would erase. When you grow the ROM:

1. find every erase and write call (block number, offset, size pushed as constants);
2. re-point them to the equivalent block of the new size (e.g. the last 8 KiB blocks,
   `0x1FA000` area for 16 Mbit), including any size passed in hard (`pushw 8` → the new size);
3. copy the original save template to the new place;
4. neutralise hidden erases elsewhere;
5. verify a **write**, not only a read, on an emulator that models the flash.

### 2.4 Ship a blank save area

A dump may contain a previous player's save (a profile, even a dictionary of typed words copied
into the save). Reset the save blocks to `FF` in the build, and make the patch tool check the
source ROM's checksum *with* the save area blanked.

---

## 3. Finding the text

### 3.1 Follow the interpreter, do not scan

Story text usually lives inside a **bytecode** (a script with opcodes, operands and text
runs). Write the complete opcode → operand-length table and walk the scripts the way the game
does. A linear scan with heuristics ("looks like text") **loses strings silently** and invents
wrong block starts; one project had to redo its extraction for this reason.

### 3.2 The forms a reference to text takes

Search for all of them, not only the first:

* a 32-bit little-endian pointer (`0x20xxxx` for the first chip);
* `ld XHL,imm32` and friends in code;
* a 24-bit absolute address inside an instruction (`F2 aa bb cc 3x`, a load-address form);
* two pushed 16-bit halves, high then low: `0B 22 00 0B lo hi` for the address `0x22hilo`;
* 16-bit offsets relative to the start of a record;
* an accessor `base + (i−1) × size` computed with `mul` / `add` — the table has no pointers
  at all.

⚠️ The **base of a table** can look like a string start: re-pointing it moves the whole table.

### 3.3 False pointer sites

Any 4 bytes in graphics or in code operands can look like a pointer. Rewriting them damaged a
publisher logo, put a yellow pattern on a title logo, and blacked out an options screen.
Defences that worked:

* restrict the zones where pointers are looked for (known tables), and re-point **table
  entries** only;
* require a neighbour that is itself a pointer (and not loaded by code), or a terminator just
  before the target;
* check statistically that the hits are real: compare the hit rate of true string starts with
  that of **witness offsets** (start + 7). In one game, 32-bit references matched 24.8 % of true
  starts against 0.4 % of witnesses — real; a 24-bit search in 2 MB gives ~12 % collisions by
  chance alone.

### 3.4 Cracking an unknown character table

* Screenshot a dialogue, note which characters repeat and where, and search the ROM for that
  **pattern of repetitions** (equal codes at the same distances): it usually has a single hit.
* Find the glyph address in the drawing routine (`glyph = base + 16 × code`: look for a
  `mul`/shift by 16 then `add`). Check your font sheet against the screen — one project's first
  sheet was off by one cell.
* To find where a displayed text comes from: read the tiles it uses in VRAM, and search their
  16 bytes in the ROM.
* ⚠️ Many games draw **one new tile per character** (the map holds sequential tile numbers),
  so the map tells nothing about the font — and every box has a **tile budget** (one game: 47
  characters, beyond which the text overwrote the background).
* Menus may be stored as a set of glyphs plus **lists of glyph indices**, or as glyph
  *addresses*: searching the bytes of the text finds nothing.
* A menu in the BIOS font uses the BIOS's own character codes (the code is the tile number,
  0–255; see BIOS).

---

## 4. Putting the text back

### 4.1 Jump out of the script, and come back

The most useful technique: replace a text command in the bytecode with a **jump to a new block
on the second chip**, which holds the new text (any length, extra boxes if needed) and jumps
back. No more length limit. Pitfall: many interpreters have **a single return slot** — no
nested "call" from inside a relocated block.

### 4.2 Or hook the routine that fetches the text

Patch the routine that expands or copies a string: look the original address up in a **sorted
table [original address → new address]** with a binary search, and use the new text; an
address not in the table keeps the original. Good for text referenced from many places.

### 4.3 What must NOT move

* In a text read **sequentially**, the terminator must stay where it was: the next byte is the
  next opcode. Pad to the exact length, and after each build compare the positions of every
  terminator between the original and the patched ROM.
* Lists read **by number** keep their order and their count.
* Names inserted into other strings go through **fixed RAM buffers**: a longer name overwrote
  the next variable and reset the console; a name of length 0 started a 64 KiB copy. Read the
  copies into RAM before lengthening a name table.

### 4.4 Wider windows and menus

Look for the rectangle descriptors, the size parameters pushed before the frame-drawing call,
and the cursor X hard-coded in the menu code. Clear routines must be widened with their
window. Some frames are fixed tile art: there, abbreviate.

### 4.5 Layout traps met on several games

* leading spaces fall outside the visible area;
* menus with a cursor have **fixed columns**: align on the original's columns;
* a full line followed by a line break gives an empty line;
* a character past the box width wraps onto the **start of the same line** and overwrites it;
* a shorter text must be padded — the routine does not clear what was there;
* some windows have a single line (battle messages).

### 4.6 Duplicates

The same original text must get the same translation: propagate automatically. Watch for dead
duplicate blocks (never displayed) and whole repeated sections.

---

## 5. Fonts and accents

* **Accents go in cells that became useless** (kana cells of a Latin font, unused symbols),
  after checking that the extracted text never uses those codes. The font often exists in
  **two copies** (story and menus, colour and mono): patch both.
* **Variable-width text:** find the single instruction that advances the cursor after a
  character and replace it with a call to a routine that reads a **width table**. Pitfalls:
  * fast drawing modes may assume X is a multiple of 4;
  * compose the glyphs of a tile in a buffer and write the tile **once** — drawing straight
    into VRAM flickers;
  * keep an eye on the typing speed;
  * measure the safe line width on screen (136–144 px on the games done).
* **BIOS font in menus:** the game calls the font-loading system call, after which the tile
  number is the character code. Hook that call and copy the accented glyphs over unused codes
  right after it.

---

## 6. Graphics

* **Tiles have hidden users.** Tile ids that no known map displays are reused by the next screen
  (it keeps VRAM), by a logo on the same tile set, by a mirrored copy, or shared between colour
  and mono. Recycle only tiles **proven** to belong to the maps you rewrite, read them from the
  **original** ROM (not an already-patched one), and write **transactionally**: compute
  everything, check that every new tile found a place, then write — a re-encoder that wrote as
  it went left a half-modified ROM.
* **The palette is attached to each map cell.** A cell you add gets palette 0. Stay inside the
  original's non-empty cells, or set the palette cell by cell.
* **Colour games have separate mono assets** (maps and tiles for the original black-and-white
  console, with different ids). Writing mono data at the colour ids damages other tiles. Test in
  mono AND in colour; in compatibility mode the colour palette RAM is not used.
* Compressed graphics (LZSS is common): recompress in place, or relocate the stream to the
  second chip and re-point it. Vertical text: rotate, retouch, rotate back. A title can grow with
  free sprites on top of the planes. A font for a logo can be harvested from the letters already
  drawn in the game.

After every graphics module: a full run original vs patched with the same inputs, then the
**neighbouring screens** (logos, title, the screen after), in colour and in mono.

---

## 7. Proving nothing else changed

### 7.1 The identity build

Build the patched ROM with **the original text passed through every mechanism** (every jump,
every hook, every relocation): it must be identical to the original **to the pixel** over
hundreds of screenshots of an automated run (or to the byte where possible). This catches false
pointer sites, broken relocations and wrong terminators before any translated text is involved.
⚠️ Compare against the original **grown to the same size**: adding the second chip shifts
start-up by one or two frames.

### 7.2 Read the built ROM back

The builder checks every string in the result: **placed / refused / ANOMALY**, with a log of
which step wrote each byte. It refuses two writes to the same byte, never truncates silently,
and lists every error at once instead of stopping at the first.

### 7.3 Savestates are not portable between builds

A savestate keeps **VRAM and character RAM** — the old font and the old tiles. It cannot compare
two ROMs and it lies about a new font. For regression, start from power-on (or inject the font).
Some games do not reload their screen from a state at all.

### 7.4 Reach the content without playing for hours

* poke RAM bytes (story step, chapter, money, day counter, affection);
* in a test copy, redirect an entry reached early to the message you want to see;
* start a given scene instead of the prologue;
* automatic pilots: random walks, scripted inputs, a driving bot, auto-advance of dialogues;
* look for **hidden debug menus**: debug/test strings, a flag tested but never set, the
  joypad-reading routine and the combinations it tests, an orphan script section, OPTION held at
  power-on. Conclude "no menu" only after reading the joypad code. A separate "debug" patch
  helps testers.

### 7.5 Detectors

* a **leftover-original-script detector** on screen (e.g. kana-shaped tiles) while every scene
  is launched in turn;
* a full game original vs patched with identical inputs: records, saves and scores must stay
  identical;
* a detector of plausible text outside the catalogue (strings the extraction missed).

---

## 8. A reproducible release

* **IPS patch** (with RLE), verified by re-applying it.
* The repository rebuilds the patched ROM **byte-identical** (CRC32 printed); the tool takes
  `--rom` and checks the source by CRC/SHA-1 (save area blanked).
* Translation files hold `@ADDRESS<TAB>translation` — **not the original text**.
* README (in the target language and in English), the format of the files, a licence.
* **No ROM, no build output**; purge `__pycache__` (compiled files carry the absolute path of the
  machine that built them).
* Hook code assembled with a second assembler is shipped **pre-assembled** too, so the
  assembler is optional for rebuilders.
* One source of truth: after every fix, re-sync the release repository.

---

## 9. Translating a large script

* Batches by scene, with a speaker/portrait column; a style guide and a glossary.
* An automatic checker at zero errors before any build: widths, line counts per box, control
  codes, same speaker as the original.
* A small trial batch, review, then the rest; review against the original language;
  harmonise; apply corrections by priority.

---

## 10. Tooling traps

* An editing tool that strips trailing spaces breaks padded strings: pad with an explicit
  code instead.
* Backslashes in a shell here-document can become control bytes in the file (see Build
  Toolchain §8.5e).
* Write files atomically; run Python isolated (`-I -B`) with absolute paths.
* A batch-splitting script run again may renumber the batches.
* Several helpers writing drafts to the same folder overwrite each other: one sub-folder each.

---

## See Also

- [Storage and Saves](../05_Systems/Storage-and-Saves.md) — two-chip cartridges, save geometry
- [Localization](../05_Systems/Localization.md) — language byte, BIOS font, accented glyphs
- [BIOS](../01_Hardware/BIOS.md) — font-loading system call
- [Debug Tools](../05_Systems/Debug-Tools.md) — verifying a port against its original
