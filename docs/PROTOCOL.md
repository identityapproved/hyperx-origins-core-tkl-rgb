# Protocol

Everything `hxoctkl` knows about the HyperX Alloy Origins Core, in enough
detail to write another implementation. Derived by querying the device on
Linux, with one USB capture of NGENUITY used to locate the animation store.

## Transport

Control interface: USB interface 2, HID usage page `0xFF13`, 64-byte reports,
no report IDs. Resolve the hidraw node by `HID_ID` plus `bInterfaceNumber`;
never hardcode `/dev/hidrawN`, since the numbering moves across replug.

Frame, as indices into the 65-byte buffer handed to `write()` - index 0 is the
report-number byte hidraw consumes:

```
[1] command   [2] index   [3] unused   [4] length   [5..] payload
```

Replies arrive with no report-number byte and echo the request header, so reply
`[1]` echoes request `[2]`, `[2]` echoes `[3]`, and `[3]` echoes `[4]`. Only
`0x10` writes a real length into reply `[3]`.

> [!IMPORTANT]
> A reply's byte 3 is **not** a length. It is an echo of what you sent. Reading
> it as a length will tell you a reply carries no data when it carries sixty
> bytes.

Status payloads begin `0xEC`:

| Payload | Meaning |
| --- | --- |
| `ec 00` | unsupported command |
| `ec 01` | command exists, parameters are wrong |
| `ec ac` | success |

Some commands (`0xA4`, `0xA6`) never reply at all, which is distinct from
`ec 00`.

## Opcodes

| Opcode | Meaning |
| --- | --- |
| `0x10` | device info: firmware at 4-5, name at 8-32, layout variant at 56 |
| `0x11` | per-profile, index 1-3, all zero on this unit |
| `0x13` / `0x93` | read / write effect GUID, index `profile << 4`, 32-byte payload holding a 16-byte GUID |
| `0x14` / `0x94` | read / write a 5-byte per-profile store, index 1-3 |
| `0x15` / `0x95` | get / set active profile; `[2]`=0, length 1, profile in payload |
| `0x16` | static constant `0x55` |
| `0x20` / `0xA0` | read / write per-profile config block, index 1-3 |
| `0x22` / `0xA3` | read / commit per-key colour bank, index `(profile << 4) \| chunk`; this is the key-press layer |
| `0x27` / `0xA7` | get / set brightness, 0-255 in five steps |
| `0x30` / `0xB0` | read / write a 780-byte per-profile store: index = profile, `[2]` = chunk 1-13, length `0x3c`. All zero on this unit |
| `0x31` | answers on all 256 indices, 20 bytes each, mostly `0xff` |
| `0x4f` | answers only at index `0xb0`, returns `0x0a` |
| `0xA2` | stream per-key colour to the live buffer, index = chunk |
| `0xA4` | animation upload bracket: `[1]`=profile, length 2, payload `55 <frames>` to begin, `aa <frames>` to end |
| `0xA5` | animation keyframe: `[1]` = `(profile << 4) \| chunk`, `[2]` = frame number, length `0x3c` |
| `0xA6` | exists, no getter, purpose unknown |

Getters and setters pair as **getter = setter - 0x80**, except `0xA3` to `0xA6`,
which have no getter and are actions rather than settings.

### Two traps

`0x30` is **not** a device reset. It is the read side of the 780-byte store,
and NGENUITY issues it thirteen times at startup with normal replies. Called
bare - no index, no chunk, no length - it faults the firmware into a USB
re-enumeration. That is a crash, not a documented command, though it is a
convenient way to force a reload.

The config block byte that reads `0x64` is the **animation frame count**, not a
speed or intensity. A 100-frame effect writes 100 there; a 56-frame one writes
56.

## Colour buffer

380 bytes covering 94 LED indices, in GRB planes:

```
pos = (idx // 12) * 48 + ((idx // 6) & 1) * 8 + (idx % 6)
green at pos, red at pos + 16, blue at pos + 32
```

Sent as seven chunks - six of 60 bytes, one of 20. Each write is `payload + 5`
bytes rather than a full report.

In live mode the keyboard reverts to its onboard profile a few seconds after
the stream stops, so the stream must be refreshed; 10ms is comfortable.

## The two layers

A profile renders two things:

- the **base layer** is an uploaded keyframe animation (`0xA4`/`0xA5`). A static
  colour is a one-frame animation; a frame count of 1 is accepted.
- the **key-press layer** is the per-key colour bank (`0x22`/`0xA3`). It renders
  its stored colour with a fade-to-black envelope that replaces the base for the
  pressed key.

No bank value makes a key press invisible. A bank matching the base gives a key
that dims; a black bank gives one that goes straight dark. Switching the
reactive behaviour off entirely would need a field that has not been located -
candidates are config block bytes 12 and 13, the 5-byte `0x14`/`0x94` store, or
the 780-byte `0x30`/`0xB0` store, all of which this tool writes blind.

`0xA5` has no getter: `0x25` answers `ec00`. The animation store cannot be read
back, so keyframe correctness is only verifiable by looking at the keyboard.

## Saving an animation

The full sequence, as NGENUITY performs it:

```
0x95  select profile            95 00 00 01 <profile>
0xB0  x13 write 780-byte store  b0 <profile> <chunk 1..13> 3c <60 bytes>
0x93  write effect GUID         93 <profile<<4> 00 20 <16-byte GUID> 00...
0xA4  begin upload              a4 <profile> 00 02 55 <frame count>
0xA5  x(7 * frames) keyframes   a5 <(profile<<4)|chunk> <frame 1..N> 3c <60 bytes>
0xA4  end upload                a4 <profile> 00 02 aa <frame count>
0xA0  write config block        a0 <profile> 00 17 09 18 00*8 <frames> 00...
0x95  select profile again
```

Each keyframe is a full 380-byte colour buffer in the same GRB packing as the
live stream, split into the same seven chunks.

The 780-byte store is all zero on this unit; `hxoctkl` reads it and writes back
what it read rather than inventing content for a store nobody understands.

The **effect GUID is metadata, not load-bearing**. An animation uploaded with
all sixteen GUID bytes zeroed renders and persists normally; the firmware does
not consult it when drawing. It exists so NGENUITY can identify which effect a
profile holds when it reads one back. `hxoctkl` writes zeros.

Upload pacing is not a constraint. A deliberately slowed upload - 700 reports at
9ms apart, 7.4 seconds total - renders correctly.

## Dead LED indices

Seven of the 94 indices have no physical key on an ANSI tenkeyless and are never
lit:

```
34, 44, 46, 47, 77, 83, 93
```

Laid out in index order, they fall exactly where other layouts put keys this one
does not have:

| Index | Between | Absent key |
| --- | --- | --- |
| 34 | `leftshift`, `z` | ISO extra key |
| 77 | `quote`, `enter` | ISO hash/tilde |
| 83 | `slash`, `rightshift` | ABNT2 / JIS extra |
| 44, 46, 47 | around `leftalt`, `space` | wider bottom rows |

So the 94-index buffer covers ANSI, ISO, ABNT2 and JIS positions, consistent
with the layout variant byte in `0x10` reporting `0x09` for ANSI and `0x10` for
ABNT2.

Indices run left half first (0-47), then right half (48-93), row by row within
each - a hardware scan order, not a visual one. `keymap.json` maps every index
to its HID usage and key name.

## Key-press notifications

The control interface volunteers reports with no request, which is how the Fn
layer reaches the host:

| Opcode | Payload | Meaning |
| --- | --- | --- |
| `0x73` | `[hid_usage, 01\|02]` | key event, `01` press, `02` release |
| `0x70` | `[profile]` | profile key pressed |
| `0x71` | `00` | follows every `0x70` |

`0x73` carries standard HID usage codes. `0x70` fires on every press of a
profile key, including re-selecting the active one, so it is not an edge
trigger.

Any code that sends a command and reads the next report can capture a
notification instead of its own reply. Filter replies on the echoed opcode in
byte 0.

## What NGENUITY's animations look like

Vendor presets are 100 frames by default - a handful use 13, 58 or 1 - and 93 to
100 of each animation's frames are unique. None is a brightness envelope over a
single base frame, and none is a cyclic shift of its first frame.

They are arbitrary per-key video rather than parameterised effects. Anything
trying to express them compactly will fail; a format should make simple things
simple and accept raw frames for everything else.

## The .hxp preset format

NGENUITY's own export format, partly decoded:

- 8-byte magic `ca 83 65 7d .. .. 00 00` then `ce 83 2c 4d`, a version field
  reading `09`, eight varying bytes, then byte `0x18` as the UTF-8 name length
  followed by the name
- **most of the file is an embedded Photoshop JPEG** - cover art for NGENUITY's
  UI, carrying Adobe XMP metadata. The same artwork appears at 320x160, 288x64,
  128x64 and 64x64, and is absent entirely from the smallest presets. It is not
  a render of the keyboard and cannot be sampled for key colours
- the lighting data is a small structured block: the frame count as a 32-bit
  value appearing twice, immediately followed by records at a 30-byte stride,
  each holding an RGBA colour, three doubles and an incrementing counter
- colours are stored verbatim, with no per-channel calibration between file and
  wire

Not determined: the exact field boundaries of the 30-byte record, and the
compositing model that turns a few hundred records into a hundred frames of 94
colours.

## A lesson worth repeating

This device indexes from `0x10`, not from `0`. `0x22` returned `ec 01` to
roughly 400 sampled parameter combinations because every one of them used small
values; an exhaustive `0x00`-`0xff` sweep of a single field found it
immediately.

**Sweep the full byte range before concluding a command is unreachable.**

## Reproducing this work

Capture on the Linux host with `usbmon` rather than inside a VM, even when the
keyboard is passed through - the host sees the same traffic without USBPcap's
framing. Change one setting per capture and note what changed.

A usbmon capture is bus-wide: the raw file contains every other device on that
bus, including whatever keyboard you were typing on. Filter it to the one
device before sharing it anywhere.

## Prior art

[OpenRGB](https://openrgb.org) supports this keyboard in Direct mode only and
has no onboard-profile save for any HyperX keyboard. Its colour packing - the
formula above - is correct and was the starting point here. The `0x22` read path
and the `0xA3` commit are not in OpenRGB.
