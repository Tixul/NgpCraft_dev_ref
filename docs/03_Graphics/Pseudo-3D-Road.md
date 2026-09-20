# Forward-View Pseudo-3D Road — two shipped engines, measured

A forward-facing road that fans out in perspective is the single most-asked-for effect on
this machine, and there is more than one way to do it. Two **commercially released**
cartridges solve it differently, and both were read out of a running emulator rather than
guessed: one is a driving game, the other a rail-vehicle simulation. Neither is named here
— what follows is the mechanism, which is what transfers.

Everything below was measured against the running game (per-scanline register logs,
frame-by-frame table dumps) and then re-verified by porting it. Where a figure comes from
our own port rather than from a cartridge, it says so.

> **Prerequisite:** [Effects and Raster](Effects-and-Raster.md) §1.4 for the per-scanline
> scroll mechanism itself, §1.4b for how a *hill* works, §1.4c for how to debug one.

---

## 1. The principle, in one sentence

One background plane is **sheared**: every scanline gets its own horizontal scroll value, so
a plane that holds a straight strip of tarmac bends into a perspective road. The world image
never changes — verified on one cartridge as **zero bytes touched over 600 frames**. What
changes is a table of registers, once per frame.

The two engines differ in **what fills that table**.

---

## 2. Engine A — sample a profile at a quadratic index

The cheaper of the two. It does not *compute* a curve; it **samples a signed profile table
at a quadratic index**.

```
index(k) = (k * k) / DIV                 k = 1..BANDS
scrollX[LINES - k] = profile[base + index(k)] - profile[base]
base = profile_pointer + corner_offset
```

Three things make this work, and all three are easy to get wrong.

**The index is quadratic, and the window is small.** On the cartridge, `index(k) = k²/64`
for 48 bands, so only **37 profile entries are ever read**. A whole "segment type" therefore
fits in 37 signed bytes. Verified **597 times out of 600** against the table the game itself
produced, live.

**The law generalises by keeping that 37-entry window:**

```
DIV = BANDS² / 36
```

48 bands → 64 (the cartridge). 56 bands → 87 (our port).

**The table fills DOWNWARD.** Band `k` lands on line `LINES − k`, so `k` grows toward the
top of the screen: zero deviation under the player, maximum at the horizon. That is what
puts the vanishing point where the player is, and reversing it puts it on the bumper.

**Advance and steering are the SAME control.** A single offset slides the sampling base
along the profile. While the window sits in a flat region the road is straight; it curves as
soon as the window reaches a ramp. There is no separate "corner engine" — *the profile IS
the track*, and a long profile with long ramps and long flats is a real circuit.

**One plane, not two.** Measured over 600 frames of driving: the cartridge **never** writes
the per-line table of the second background plane. There is no line-scroll parallax. The
other plane is a fixed layer (HUD, or a ground plane).

⚠️ **Align the bands to the road you actually drew.** If the bands overrun the painted
tarmac, the part of the fan that falls in the sky is wasted. Our port uses horizon 96 with
56 bands, which is exactly lines 96-151 of the art.

---

## 3. Engine B — a staircase of slices, with a wider per-line table

The other cartridge spends more per line and buys hills with it.

**Eight bytes per scanline**, not one scroll value: `SCY`, `SCX`, and **three 15-bit
colours**. The tarmac that streams past is a background palette entry rewritten every line;
the haze bands come from the same table.

**The road is a staircase.** 48 slices of two map rows each, walked from the bumper line
upward. Each slice is granted **0, 1 or 2 screen lines** by adding a per-gradient *rate*
twice into an 8-bit accumulator and counting the carries. The gradient model is the ratio
described in [Effects and Raster](Effects-and-Raster.md) §1.4b.

**Curvature is linear ramps that sum to a parabola.** Each slice adds a 16-bit word from the
segment's curvature profile into an accumulator; `SCX` is the high byte. The 21 profiles are
straight lines:

```
profile[k][i] = round(0.75 * k * i)
```

so the curvature increment of a slice is **proportional to its distance**. Accumulating from
the bumper upward makes the total grow as **i²** — the parabola of a corner, with **no
multiply at runtime**.

🔑 **The ramp index is the GLOBAL slice index, not an index local to the segment.** A distant
corner is therefore sampled in the strong part of its ramp automatically, and it "arrives" on
its own as the slices come closer. Get this wrong and corners snap into existence instead of
approaching.

---

## 4. The track format that makes both cheap: four bytes per segment

| byte | bits | field |
|---|---|---|
| 0 | 0-4 + sign in 5 | **gradient**, sign-magnitude |
| 0 | 6-7 | high bits of a roadside-scenery code |
| 1 | 0-4 + sign in 5 | **curvature**, sign-magnitude |
| 1 | 6-7 | low bits of that code (one value = finish line) |
| 2 | 0-5 | parameter 1 |
| 3 | 0-5 | parameter 2 |
| 2, 3 | 6-7 | event code (the centred corner sign; one value clears it) |

Two details worth stealing:

⚠️ **Sign-magnitude, not two's complement.** Bit 5 is the sign, `AND 0x1F` the value — so
"+0" and "−0" both exist and the decoder has to mean it. A classic sign-extend reads
garbage.

🔑 **The reversed circuit is FREE.** A mirror flag makes the loader `XOR 0x20` the curvature
field (flipping its sign bit) and take `0x3C − p1` for the parameter. One dataset, two
racing directions.

**And the author writes ramps, never steps.** On a measured track the gradient takes only
eleven distinct values and always arrives as `0, +2, +5, +10, …, +10, +5, +2, 0`; corners
likewise `0, 1, 2, 4, 4, 4, 2, 1, 0`. A hill or a corner is never dropped in as a single
segment — the ramp *is* the shape.

---

## 5. Where the gameplay comes from: the same bytes

The most transferable idea in either engine. **There is no separate physics data.** The
bytes that draw the road are the bytes that drive the car.

**Gradient → speed, and it is deliberately asymmetric.** Reading the gradient of the nearest
segment: climbing decelerates by `1.5 × gradient`, descending accelerates by `8 × |gradient|`,
both added into an 8-bit accumulator that only acts on carry (so the gain is `B/256` speed
units per frame).

> **A descent gives back 5.3 times what a climb costs, at equal gradient.** At −10:
> +0.31 units per frame. At +10: −0.06. That ratio is an arcade choice and it is what makes
> a descent an *event*. Measured against the running game: over the 200 frames of a descent
> ramping −2 to −10, speed rose +0.215/frame, exactly what the gradient ramp predicts.

**The crest jump is a subtraction.** The car takes off when the gradient **drops** from one
segment to the next, and the airtime is `speed × drop`, gated by a minimum. No "bump" test,
no geometry: with a writing scale of 0, ±2, ±5, ±10, ±15, ±20 you need a drop of at least two
steps (`+10 → 0`) and some speed.

**A corner pushes from a distance that depends on speed.** The push is read from the
curvature of segment `n`, where `n = min(speed / 16, 7)`. Faster means the corner starts
pushing from further away — the same table, sampled deeper.

---

## 6. Traps, all of them paid for

- ⛔ **Quantising the gradient without a dead band flickers.** See
  [Effects and Raster](Effects-and-Raster.md) §1.4b.
- ⛔ **A descent gives nothing if the top row of the sky board is not a uniform colour** —
  repeating it reads as a hard-edged stripe, which forces you to clamp descents and costs
  you half the relief. Fix the art before tuning the numbers.
- ⚠️ **A short hill is not a weak hill.** The intuition that a segment shorter than the
  look-ahead only unfolds a fraction of the relief is **wrong**: measured, a 400-unit bump
  reaches the same full deflection as an 1800-unit plateau. It just takes ~300 units to get
  there.
- ⚠️ **A hill sells itself by how much road it removes**, so it needs road to take. The same
  numbers read as timid on a 32-slice strip and strong on a 48-slice one. If your relief
  looks weak, check the strip before the constants.
- ⚠️ **Clamp the table index.** Gradient tables are indexed `centre + gradient`; a track
  byte outside the intended range walks straight off the end.

---

## See Also

- [Effects and Raster](Effects-and-Raster.md) — the per-scanline mechanism, hills, and how to
  debug a raster effect
- [Tilemaps and Scrolling](Tilemaps-and-Scrolling.md) — the plane registers being written
- [DMA](DMA.md) — driving the per-line table at zero CPU cost per line
