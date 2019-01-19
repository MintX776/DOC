# Event Code Master Doc

The goal here is to have a complete and detailed documentation of *every* event codes handled by the base game.

When this is done, I will also post that on FEU. It isn't done yet tho so it will be only here for now.

Example and other event scene source will be written in event assembler language with no macros(*), and using [my experimental EA standard raws](https://github.com/StanHash/EAStandardLibrary/tree/experimental).

(*): As sole exception, I will be using the `sX` aliases for slot identifiers, to put emphasis on what is a slot identifier and what is a number.

## Generalities

The binary representation of each code is as follows:

	bits 0-3  | subcode : "subcode" identifier
	bits 4-7  | size    : byte size / 2
	bits 8-15 | id      : code identifier
	[code-dependent layout]

The size of a code in memory is *always* to be considered as equal to `size*2` bytes (header included), no matter what code this is. This is in contrast with FE6 and FE7 where to each code id is assigned a fixed size. This could allow for some funky obfucation shenanigans if you felt like doing that.

In this document, I will always use the size the code would have been if seen in vanilla events, which afaik is always its minimal size, aligned 4, for a given code id (and not subcode!).

The code identifier essentially dictates which routine the event engine will can to handle the code. The "subcode" doesn't have any meaning for the event engine itself, but usually it is used by the handling routine to know what kind of operation it needs to do.

For example, the code id `0x33` handles a whopping 9 subcodes, each of which reads some kind of information related to a unit. You can see that the general thing this code does is "reading information from a unit", but the subcode dictates which information to read (if it exists, if it is alive, which is its allegiance, its class id, its luck stat...).

## The Doc

This lists all event codes from 00 to 45 (hex) by id. The 80 to CD range may or may get documentation here in the future, just know that those are all world map related and probably not relevant to most people.

<details>
<summary>00 : do nothing (NOP)</summary>

```
[0020] NOP @ does nothing
```

Does nothing, doesn't seem to have any use.

Takes no arguments.

---

</details>

<details>
<summary>01 : end (ENDA, ENDB)</summary>

```
[0120] ENDA @ ends event subscene
[0121] ENDB @ ends event subscene and calling subscene(s)
```

- `ENDA` ends the current event subscene. This means that if you called your event scene using the `[0A40] CALL` event code, control will be given back to the calling event scene.
- `ENDB` does the same as `ENDA`, but clears the event call stack beforehand, meaning that if this event scene was called, control *won't* be given back.

If the current event scene "type" is "chapter events", and there is no calling event scene to return to, a special "cleanup" event scene will be called. Here's the full source for that scene:

<details>
<summary>Events</summary>

```
scAfterEnd:
  CHECK_EVBIT 10
  BNE $0 sC s0

  CALL scAfterEnd_Clean
  GOTO 1

LABEL $0
  CALL scAfterEnd_CleanResetMap

LABEL $1
  ENDA

scAfterEnd_Clean:
  CHECK_EVBIT 8
  BNE $0 sC s0

  CHECK_EVBIT 7
  BNE $63 sC s0

  FADI 0x10

LABEL $0
  CLEAN

  FADU 0x10

LABEL $63
  ENDA

scAfterEnd_CleanResetMap:
  CHECK_EVBIT 8
  BNE $0 sC s0

  FADI 0x10

LABEL $0
  CHECK_EVBIT 11
  BEQ $1 sC s0

  CHECK_CHAPTER_NUMBER

  SADD s2 sC s0 // s2 = sC
  SCOORD sB [0, 0]

  LOMA (-1)

LABEL $1
  ENDA
```

</details>

---

</details>

<details>
<summary>02 : set eid and evbits (ENUT, ENUF, EVBIT_T, EVBIT_F)</summary>

```
[0220] EVBIT_F Evbit @ sets given Evbit to 0/false
[0228] EVBIT_T Evbit @ sets given Evbit to 1/true
[0221] ENUF Eid @ sets given Eid to 0/false
[0229] ENUT Eid @ sets given Eid to 1/true
```

The argument is allowed to be `(-1)`, in which case the target Evbit/Eid will be read from `s2`. EA provides the parameterless `ENUF_SLOT2` and `ENUT_SLOT2` codes for convenience (no such code exist for the Evbit variants as of now).

---

</details>

<details>
<summary>03 : get eid and evbits (CHECK_EVBIT, CHECK_EVENTID)</summary>

```
[0320] CHECK_EVBIT Evbit @ gets Evbit state in sC
[0321] CHECK_EVENTID Eid @ gets Eid state in sC
```

The argument is allowed to be `(-1)`, in which case the target Evbit/Eid will be read from `s2`.

---

</details>

<details>
<summary>04 : get random number (RANDOMNUMBER)</summary>

```
[0420] RANDOMNUMBER Max @ gets random integer in interval [0-Max] in sC
```

This uses the game's primary random number generator (the same one used for battle calculations). This is contrast to FE6 and FE7 where similar codes used the secondary ("cosmetic") random number generator.

If `Max` is 0, then the result will be `0`, and no random number will be generated.

`Max` is fixed and there is no way of having it read from an event slot.

---

</details>

<details>
<summary>05 : set slot value (SVAL)</summary>

```
[0540] SVAL Slot Value @ sets slot to given value
```

Setting `s0` has no effect (it will be reset to 0 before the next code is handled).

EA standard raws provide `SMOV` and `SETVAL` as straight alias to `SVAL`. Experimental raws provide `SCOORD slot [X, Y]` and `SPTR slot label` for convenience.

---

</details>

<details>
<summary>06 : math on slots (SADD, SSUB, SMUL, SDIV, SMOD, SAND, SORR, SXOR, SLSL, SLSR)</summary>

```
[0620] SADD sDest sSrc1 sSrc2 @ sDest <- sSrc1 + sSrc2
[0621] SSUB sDest sSrc1 sSrc2 @ sDest <- sSrc1 - sSrc2
[0622] SMUL sDest sSrc1 sSrc2 @ sDest <- sSrc1 * sSrc2
[0623] SDIV sDest sSrc1 sSrc2 @ sDest <- sSrc1 / sSrc2
[0624] SMOD sDest sSrc1 sSrc2 @ sDest <- sSrc1 % sSrc2
[0625] SAND sDest sSrc1 sSrc2 @ sDest <- sSrc1 & sSrc2
[0626] SORR sDest sSrc1 sSrc2 @ sDest <- sSrc1 | sSrc2
[0627] SXOR sDest sSrc1 sSrc2 @ sDest <- sSrc1 ^ sSrc2
[0628] SLSL sDest sSrc1 sSrc2 @ sDest <- sSrc1 << sSrc2
[0629] SLSR sDest sSrc1 sSrc2 @ sDest <- sSrc1 >> sSrc2
```

Having destination as `s0` has no effect (it will be reset to 0 before the next code is handled).

All operations are done as if those were C expressions with `int`s as operands, which notably means that:

- Divisions round towards 0 (as opposed to down).
- Right shift is an arithmetic shift, which means that the msb aka "sign bit" will be repeated however many bits the value has been shifted (as opposed to having the extra bits filled with 0).

---

</details>

<details>
<summary>07 : queue operations (SENQUEUE, SDEQUEUE)</summary>

```
[0720] SENQUEUE Slot @ enqueues value of given slot
[0721] SENQUEUE @ euqueues value of s1
[0722] SDEQUEUE Slot @ dequeues value in given slot
```

Remember that operations on the queue modify `sD`.

TODO: link to the queue stuff

---

</details>

<details>
<summary>08 : label (LABEL)</summary>

```
[0820] LABEL Identifier @ marks the position of a label
```

Labels are used to mark the target location of jump operations. Label lookup is done by looking for each event code from the *start* of the current scene (and not the current code!) and getting the first matching label code. Do note that only labels with the standard size of `2` will be consided for the search.

Do note that the lookup doesn't check for ends and whatnot. You could very well get the game into an endless loop if you look for a label that doesn't exist! (That or jump into a completely unrelated scene which may have interesting effects).

The label code itself does nothing (much like `NOP`).

---

</details>

<details>
<summary>09 : jump to label (GOTO) </summary>

```
[0920] GOTO Identifier @ jumps to the identified label
```

Jump to given label unconditionally. For more information on how label lookup is done, see `08 : label (LABEL)`.

---

</details>

<details>
<summary>0A : call subscene (CALL)</summary>

```
[0A40] CALL TargetOffset @ calls event scene at TargetOffset
```

Pushes current event scene start/cursor on the event call stack and jumps to the given offset. The event call stack has room for 8 start/cursor pairs.

If TargetOffset is negative, the target event scene address will be read from `s2`. EA standard raws provide the parameterless `TUTORIAL_CALL` that handles just that (EA doesn't support negative "offset" values so this is required).

---

</details>

<details>
<summary>0B : enqueue event engine invocation (TODO: figure out)</summary>

TODO: add to experimental

- variant 0 equeues an event engine invocation. This means that the target event scene will be called *after the current event scene ends*. If other event engine invocations are already queued, this will be put at the end of the queue (because it's a queue).
- variant 1 does tutorial event stuff, TODO: need to investigate further.

If TargetOffset is negative, the target event scene address will be read from `s2`.

---

</details>

<details>
<summary>0C : conditional jump to label (BEQ, BNE, BGE, BGT, BLE, BLT)</summary>

```
[0C40] BEQ Identifier Slot1 Slot2 @ jumps to the identified label if (Slot1 == Slot2)
[0C40] BNE Identifier Slot1 Slot2 @ jumps to the identified label if (Slot1 != Slot2)
[0C40] BGE Identifier Slot1 Slot2 @ jumps to the identified label if (Slot1 >= Slot2)
[0C40] BGT Identifier Slot1 Slot2 @ jumps to the identified label if (Slot1 > Slot2)
[0C40] BLE Identifier Slot1 Slot2 @ jumps to the identified label if (Slot1 <= Slot2)
[0C40] BLT Identifier Slot1 Slot2 @ jumps to the identified label if (Slot1 < Slot2)
```

For more information on how label lookup is done, see `08 : label (LABEL)`.

Comparisons are done as if those were C expressions with `int` operands. Which means numbers in slots are to be considered signed.

---

</details>

<details>
<summary>0D : call native function (ASMC)</summary>

```
[0D40] ASMC FuncOffset @ calls function
```

In C terms, the function pointer has the following signature:

	void(*)(struct EventEngineProc*);

In other words, The called function takes one argument: the pointer to the running event engine [proc](https://feuniverse.us/t/guide-doc-asm-procs-or-6cs-coroutines-threads-fibers-funky-structs-whatever/3352?u=stanh). This is useful if you need to make something that will *pause* the execution of the event engine until it is done (typically some cosmetic effect).

The `ASMC` code will *always* stop the event engine execution for the current frame. This means that every time you use `ASMC`, no matter what function you call, you will always "loose" at least one frame.

There is no way of reading the function address from a slot. `ASMC` won't call anything if the function pointer is 0.

---

</details>

<details>
<summary>0E : sleep (STAL, STAL2, STAL3) </summary>

```
[0E20] STAL Duration @ pauses events for Duration frames
[0E21] - Duration @ pauses events for Duration frames (with dialogue behavior)
[0E22] STAL2 Duration @ pauses events for Duration frames (less on game speed fast)
[0E23] STAL3 Duration @ pauses events for Duration frames (less on game speed fast) (with dialogue behavior)
```

No sleeping will occur if events are currently being skipped (evbit 2 set).

- "with dialogue behavior" means that the sleep will be over if either the dialogue skipping evbit (evbit 3) is set, or the B button is pressed.
- "less on game speed fast" means that the "counting down" will be 4 times faster if either the "fast game speed" option is set, or the A button is held.

If Duration is 0, the game will hang.

There is no way of getting the duration from an event slot.

---

</details>

<details>
<summary>0F : counter operations (COUNTER_CHECK, COUNTER_SET, COUNTER_INC, COUNTER_DEC)</summary>

```
[0F20] COUNTER_CHECK Identifier @ gets identified counter value into sC
[0F21] COUNTER_SET Identifier Value @ sets identified counter value to given value
[0F22] COUNTER_INC Identifier @ increments identified counter value
[0F23] COUNTER_DEC Identifier @ decrements identified counter value
```

Event counters are small variables that can hold values ranging from 0 to 15 included. There is a total of 8 of them (identified by integers from 0 to 7) and they are saved between scenes and on suspend (much like chapter eids).

`COUNTER_INC` and `COUNTER_DEC` will not allow the counter value to overflow or underflow. If the value is 0 and `COUNTER_DEC` is used, the value will still be 0. If the value is 15 and `COUNTER_INC` is used, the value will still be 15.

There is no way of having `COUNTER_SET` get its Value argument from a slot.

---

</details>

<details>
<summary>10 : configure skipping (EVBIT_MODIFY)</summary>

```
[1020] EVBIT_MODIFY Configuration @ configures skipping
```

This will configure what kind of "skips" the player is allowed to do. Here's the list of allowed configuration identifiers:

- `0` allows scene skipping, dialogue skipping and dialogue fast-forwarding.
- `1` disallows scene skipping, dialogue skipping and dialogue fast-forwarding.
- `2` allows scene skipping and dialogue skipping, but disallows dialogue fast-forwarding.
- `3` disallows scene skipping, but allows dialogue skipping and dialogue fast-forwarding.
- `4` disallows scene skipping and dialogue skipping, but allows dialogue fast-forwarding.

Any configuration that isn't in this list will have the game hang.

If the configuration is nonzero, and if the player is currently skipping the scene, the "skipping" evbit (evbit 2) will be cleared and the "faded in" evbit (evbit 8) will be set.

There is no way of getting the configuration from an event slot.

_**Note**: "scene skipping" refers to pressing start to skip a scene. "dialogue skipping" refers to pressing B or start to skip a dialogue. "dialogue fast-forwarding" refers to pressing A, B or any D-pad direction to have the current text box content display entirely immediately._

---

</details>

<details>
<summary>11 : set ignored key presses (STORETOSOMETHING)</summary>

TODO: add better alias to experimental

```
[1120] STORETOSOMETHING Mask @ Sets key press ignore mask
```

`Mask` is a bitset where each bit maps to a button of the GBA console:

	bit 0 | A button
	bit 1 | B button
	bit 2 | select button
	bit 3 | start button
	bit 4 | right D-pad button
	bit 5 | left D-pad button
	bit 6 | up D-pad button
	bit 7 | down D-pad button
	bit 8 | R button
	bit 9 | L button

You can use the EA `1100001100b` notation (`b` suffix) to make it easier to visualize in your source (this example would ignore buttons select, start, R and L).

---

</details>

<details>
<summary>12 : smooth BGM start/change (MUSC)</summary>

```
[1220] MUSC SongId @ Sets current BGM to the m4a song identied by SongId
```

Smoothly switches BGM. A song id of `0x7FFF` (`INT16_MAX`) is silent.

This code stops the event engine for a single frame, but then the scene continues to be executed while the transition is occuring (no further waiting occurs).

If SongId is negative, then the song id will be read from `s2`.

---

</details>

<details>
<summary>13 : BGM start/change (MUSCFAST, MUSCMID, MUSCSLOW)</summary>

```
[1322] MUSCFAST SongId
[1324] MUSCMID SongId
[1326] MUSCSLOW SongId
```

Stops current BGM if any is playing, and starts new BGM with a volume fade in. A song id of `0x7FFF` (`INT16_MAX`) is silent.

- If the new BGM is not silent, then the old BGM will be stopped immediately (no smooth volume transition). If the scene is being skipped, no song switch occurs.
- If the new BGM is sient, then the old BGM will be stopped with smooth volume transition (unless the scene is being skipped, in which case the transition is instant).

It is the *subcode* that dictates which speed to fade the new BGM volume in. The lower the faster. As you can see above, MUSCFAST has subcode 2, MUSCMID has subcode 4 and MUSCSLOW has subcode 6. For reference, the transition speed of `[1220] MUSC` is 3.

This code stops the event engine for a single frame, but then the scene continues to be executed while the transition is occuring (no further waiting occurs).

If SongId is negative, then the song id will be read from `s2`.

---

</details>

<details>
<summary>14 : BGM overwrite (MUSS, MURE)</summary>

```
[1420] MUSS SongId @ plays SongId and remember current song
[1421] MURE Speed @ plays remembred song
```

Songs get proper volume fade in/out.

You can only "remember" one song. If you stack MUSSes, then only the *last* overwritten song will be "remembered".

This code stops the event engine for a single frame, but then the scene continues to be executed while the transition is occuring (no further waiting occurs).

For `MUSS`, if SongId is negative, then the song id will be read from `s2`. For `MURE`, there is no way of getting Speed from a slot.

---

</details>

<details>
<summary>15 : make/unmake BGM quieter (MUSI, MUNO)</summary>

```
[1520] MUSI @ make music quieter
[1521] MUNO @ restore music volume after MUSI
```

Volume transition is smooth.

`MUSI` does nothing if the scene is being skipped. `MUNO` sets music volume to normal immediately if the scene is being skippied.

This code stops the event engine for a single frame, but then the scene continues to be executed while the transition is occuring (no further waiting occurs).

---

</details>

<details>
<summary>16 : play arbitrary song (SOUN)</summary>

```
[1620] SOUN SongId @ plays identified song
```

Only plays song if scene is not being skipped, dialogue is not being skipped and the "sound effect" player option is set.

This code doesn't stops the event engine (no waiting occurs).

If SongId is negative, the song id is read from `s2`.

---

</details>

<details>
<summary>17 : fade in/out (FADU, FADI, FAWU, FAWI)</summary>

```
[1720] FADU Speed @ fades oUt of Dark
[1721] FADI Speed @ fades Into Dark
[1722] FAWU Speed @ fades oUt of White
[1723] FAWI Speed @ fades Into White
```

Hopefully it being said like that can make for decent mnemonics?

No fade happens if the scene is being skipped. Each fade code sets/clears the "faded in" evbit (evbit 8). The event engine will wait for the fade to end before continuing.

---

</details>

<details>
<summary>18 : arbitrary color effect (STARTFADE, ENDFADE, FADECOLORS)</summary>

```
[1860] STARTFADE
[1861] ENDFADE
[1862] FADECOLORS Target Speed Red Green Blue
```

TODO: figure more out

`STARTFADE` sets up the fade buffer from current palette.

`Target` is a pair of bytes: the first byte is the index of the first targetted palette, the second byte is the number of targetted palettes.

If scene-skipping or faded in, then `FADECOLORS` is instant.

---

</details>

<details>
<summary>19 : CHECK_[MODE|CHAPTER_NUMBER|HARD|TURNS|ENEMIES|OTHERS|SKIRMISH|TUTORIAL|MONEY|EVENTID|POSTGAME]</summary>

```
[1920] CHECK_MODE @ gets current mode identifier in sC
[1921] CHECK_CHAPTER_NUMBER @ gets current chapter identifier in sC
[1922] CHECK_HARD @ gets "is difficult mode" boolean in sC
[1923] CHECK_TURNS @ gets current turn number in sC
[1924] CHECK_ENEMIES @ gets current alive enemy count in sC
[1925] CHECK_OTHERS @ gets current alive npc count in sC
[1926] CHECK_SKIRMISH @ gets current battle map "type" in sC
[1927] CHECK_TUTORIAL @ gets "is easy mode" boolean in sC
[1928] CHECK_MONEY @ gets party gold amount in sC
[1929] CHECK_EVENTID @ gets eid identifier linked to running event is sC
[192A] CHECK_POSTGAME @ gets "is postgame" boolean in sC
```

Is of note:

- `CHECK_MODE` gets 0 for "prologue" chapters, 1 for eirika route, 2 for ephraim route.
- `CHECK_CHAPTER_NUMBER` doesn't care for `LOMA`.
- `CHECK_SKIRMISH` gets 0 for story chapters, 1 for tower/ruins and 2 for skirmishes.
- `CHECK_MONEY` gets 0 on chapter 5x.
- `CHECK_EVENTID` gets the eid given to the "condition" of the event (the `TURN`, `AFEV`, `CHAR` that had this event called). Gets 0 for events called otherwise.

---

</details>