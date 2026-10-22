# Writing presets

`hxoctkl upload` takes either a generated pattern or a JSON file.

The generators cover the cases that need no detail:

```
hxoctkl upload 3 solid:ff0000
hxoctkl upload 3 breathe:00ffd3 --frames 56
hxoctkl upload 3 gradient:8a2be2 --frames 40
```

JSON is for the thing generators cannot do: **choosing colours per key**.

## Format

```json
{
  "name": "gaming",
  "description": "optional, ignored by the tool",
  "frame_count": 1,
  "led_count": 94,
  "frames": [
    { "colors": ["#101010", "#00ffd3", "..."] }
  ]
}
```

A frame is either `{"colors": [...]}` with one `#rrggbb` per LED index, or
`{"all": "#rrggbb"}` as shorthand for a uniform frame. Missing entries are
padded with black. A bare array of frames of `[r, g, b]` triples is also
accepted, which is convenient when generating files from a script.

A **static colour is a one-frame animation**. There is no separate mode for it.

### The key-press layer

A preset may also carry the layer that flashes when a key is pressed:

```json
  "keypress": { "colors": ["#ffffff", "..."] }
```

or `{"all": "#ffffff"}` for a flat one. It is a single frame, not an
animation - the firmware supplies the fade. `upload` uses it in preference to
matching the base colour; `--keypress black` or `--keypress keep` still
override. `presets/wave.json` uses this to make every key flash the colour of
its own position.

## Index or key name?

The file is indexed by LED number, 0 to 93, and `keymap.json` in this
repository maps every index to its key:

```json
{ "0": {"usage": 41, "name": "escape"},
  "1": {"usage": 58, "name": "f1"}, ... }
```

Seven indices have no key on an ANSI tenkeyless and should stay black:

```
34, 44, 46, 47, 77, 83, 93
```

> [!NOTE]
> LED indices follow the keyboard's **hardware scan order**, not its physical
> layout - left half first (0-47), then right half (48-93), row by row within
> each. A gradient computed across index numbers does not look like a gradient
> across the keyboard.

For anything spatial, use `layout.json`, which gives each key a physical
position in standard key units:

```json
{ "26": {"name": "a", "row": 3, "x": 2.25, "w": 1, "u": 0.1233, "v": 0.6 } }
```

`u` and `v` are normalised 0 to 1 across and down the board, which is usually
all a sweep or a ripple needs. `row` and `x` are there when you want real
geometry - `x` is the key's centre and `w` its width, so a 6.25-unit spacebar
sits where it actually sits.

## A worked example

Building `gaming.json` - movement keys picked out against a dim ground:

```python
import json

keymap = json.load(open('keymap.json'))
index = {v['name']: int(k) for k, v in keymap.items() if v}
dead = {int(k) for k, v in keymap.items() if v is None}

# Start from a floor on every real key, then paint over it.
frame = ['#101010'] * 94
for i in dead:
    frame[i] = '#000000'

for name in ('w', 'a', 's', 'd'):
    frame[index[name]] = '#ff6a00'
for name in ('up', 'down', 'left', 'right'):
    frame[index[name]] = '#00ffd3'
frame[index['escape']] = '#ff0000'

json.dump({'name': 'gaming', 'frame_count': 1, 'led_count': 94,
           'frames': [{'colors': frame}]}, open('gaming.json', 'w'), indent=1)
```

> [!TIP]
> Start from a floor and paint over it, as above. Building the ground layer by
> listing categories - letters, digits, modifiers, nav - silently leaves out
> whatever you forget, and punctuation is the easy thing to forget. Two of the
> presets here shipped with `` ` - = [ ] \ ; ' , . / `` dark for exactly that
> reason.
>
> The invariant to check: the only unlit indices should be the seven dead ones.

Then `hxoctkl upload 3 gaming.json`.

## Shipped examples

| File | What it shows |
| --- | --- |
| `presets/gaming.json` | per-key static: WASD and arrows against a dim ground |
| `presets/vim.json` | a working keymap rendered as light, see below |
| `presets/typing.json` | one hue per finger column, home row bright, F and J brightest |
| `presets/wave.json` | a hue band sweeping the whole board, with a per-key press layer |
| `presets/regions.json` | a colour per keyboard cluster - useful as a map of index to position |

`vim.json` is the argument for per-key authoring. It colours the keyboard by
what each key *does* in vim:

| Colour | Keys | Meaning |
| --- | --- | --- |
| green, bright | `hjkl` | motion |
| green, dim | `w` `b` `e` `g` `0` `4` `6` `n` `f` `t` | other motions - same hue, lower level |
| red | `escape` | back to normal mode |
| orange | `d` `y` `p` `x` `u` `r` `c` | operators and edits |
| blue | `i` `a` `o` | enter insert |
| purple | `v` | visual |
| amber | `;` `/` | command line and search |
| grey | `space` | leader |

`4` and `6` carry `$` and `^`; the key lights, not the shifted symbol.
Everything else physical sits at a near-black floor so the keyboard still reads
as a keyboard in the dark. The seven dead indices stay fully off.

> [!NOTE]
> This was first built on a gruvbox palette and it did not work. Screen
> palettes are chosen for harmony - low saturation, closely spaced hues - and
> that is exactly wrong on RGB LEDs, where neighbouring hues wash toward white
> and adjacent keys become indistinguishable. The palette here picks six hues
> far apart and keeps the overall level low, using **brightness** rather than a
> seventh hue to separate primary from secondary motions.
>
> If you are authoring your own theme, budget for fewer colours than you think
> you need.

`regions.json` is worth uploading once even if you never use it. It colours
each cluster differently, so you can see at a glance how the index space maps
onto the physical keyboard.

## Size

A frame is 94 colours, so an animation is frames times 94 entries. The
40-frame `wave.json` is 59 KB; a 100-frame animation is around 140 KB.
That is fine for a file you keep, but it is a dump rather than a description -
the format is deliberately dumb so that anything can generate it.
