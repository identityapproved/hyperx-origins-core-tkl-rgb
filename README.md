# hyperx-origins-core-tkl-rgb

HyperX says onboard profiles can only be written from Windows with NGENUITY.
`hxoctkl` writes them from Linux, into the keyboard's own flash, where they
persist with nothing running.

## (⌐■-■) Supported hardware

| Device                                 | USB ID      |
| -------------------------------------- | ----------- |
| HyperX Alloy Origins Core (tenkeyless) | `0951:16e6` |
| HP-branded Alloy Origins Core          | `03f0:098f` |

> [!WARNING]
> **Only these two devices.** Other Alloy models - the full-size Origins, the
> 60%, the 65%, the Elite - speak different protocols with different LED
> counts. This tool matches on product id and refuses to find anything else,
> deliberately: writing this layout's 380-byte colour buffer to a keyboard
> with a different LED count would corrupt an onboard profile that its owner
> could then only repair from Windows.

Developed against an ANSI layout, firmware 21.08.

## Status =ᗜωᗜ=

Working:

- live per-key control, equivalent to OpenRGB's Direct mode
- reading the stored per-key colour of all three onboard profiles
- **writing** animations and static colours into onboard profiles, persisting
  across power cycles
- device info, active-profile get and set, brightness get and set

Not here ( ꒪⌓꒪) yet:

- a TUI

## Install (￣ー￣)

Nothing to build. The hidraw node is root-only until a udev rule grants
access, so install the rule and the tool runs as your user:

```
./hxoctkl udev-rules | sudo tee /etc/udev/rules.d/60-hxoctkl.rules
sudo udevadm control --reload-rules && sudo udevadm trigger
```

Replug the keyboard, then:

```
./hxoctkl info
```

The rule uses `TAG+="uaccess"`, which grants an ACL to the user on the active
local seat rather than making the device world-writable. That is provided by
logind - systemd-logind, elogind or seatd - and does not apply over ssh, where
you still need root. Substitute `doas` for `sudo` if that is what your system
uses.

## ಠ_ಠ two layers

A profile renders two things, and they are written separately:

- the **base layer** is an uploaded keyframe animation. A static colour is
  simply a one-frame animation.
- the **key-press layer** is a per-key colour bank that flashes with a
  fade-to-black envelope when a key is pressed, replacing the base for that
  key.

`upload` writes the base layer and sets the key-press layer to match unless
told otherwise. This catches people out: writing only the base leaves whatever
the key-press layer last held flashing on every keystroke.

## Usage (๑´ᴗ\`๑)

```
./hxoctkl info
./hxoctkl read                        # what all three profiles hold
./hxoctkl upload 3 solid:ff0000       # static colour, persisted
./hxoctkl upload 3 breathe:00ff00 --frames 56
./hxoctkl upload 3 presets/gaming.json
./hxoctkl keypress 3 0 0 0            # key-press layer only
./hxoctkl live 0x00 0x00 0xff 8       # direct mode, not persisted
./hxoctkl brightness 128
./hxoctkl profile 2
./hxoctkl selftest 3                  # write, reset, verify it survived
```

`upload` takes `solid:RRGGBB`, `breathe:RRGGBB`, `gradient:RRGGBB`, or a path
to a JSON animation. `docs/PRESETS.md` covers the JSON format and authoring by
key name; `keymap.json` maps every LED index to its key.

Five examples ship in `presets/`: `gaming.json` picks out WASD and the arrows,
`vim.json` colours the board by what each key does in vim, `typing.json` gives
each finger column its own hue, `wave.json` sweeps a hue band across the whole
board, and `regions.json` colours each cluster so you can see how indices map
onto the keyboard.

`keymap.json` maps every LED index to its key; `layout.json` gives each key a
physical position, which is what anything spatial needs.

## How it works ( ̄□ ̄；)

The keyboard exposes a vendor control interface on USB interface 2, usage page
`0xFF13`, with 64-byte reports. Commands use a simple framed protocol; an
animation is uploaded as keyframes inside a begin/end bracket and committed to
flash.

`docs/PROTOCOL.md` documents the frame format, the opcode map, the colour
buffer packing, the upload sequence and the dead LED indices, in enough detail
to write another implementation.

All of it was derived on Linux by querying the device, with one USB capture of
NGENUITY used to locate the animation store.

## (人´ω\`\*) Credits

[OpenRGB](https://openrgb.org) is the prior art and the starting point. Its
Alloy Origins Core driver established the GRB colour buffer packing used here,
and its Direct mode is what live control reimplements. OpenRGB supports this
keyboard in Direct mode only and has no onboard-profile save for any HyperX
keyboard; the read path and the commit sequence in this tool are not in it.

The udev approach - `TAG+="uaccess"` rather than a group or a world-writable
mode - is also OpenRGB's.

## Licence (¬‿¬)

GPL-2.0-or-later. See `LICENSE`.

GPL was chosen to match OpenRGB, whose source this work builds on.

## Trademarks and affiliation (・へ・)

This project is not affiliated with, authorised by, or endorsed by HyperX, HP
Inc. or Kingston Technology. HyperX, Alloy Origins and NGENUITY are trademarks
of their respective owners, used here only to identify the hardware this
software works with.

> [!CAUTION]
> ⸜✞⸝( 𖦹 ﹏ 𖦹 )⸜✟⸝
>
> This software writes to your keyboard's firmware storage. That carries
> risk, and it is provided without warranty of any kind, as the licence
> states. The onboard profiles it overwrites can be restored from Windows
> with NGENUITY, but not by this tool alone.
