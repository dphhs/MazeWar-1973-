# Media assets

The root `README.md` references these files. Until they exist, those images render as broken links
on GitHub — so either add them or remove the corresponding `<img>` tags before making the repo public.

**Present:** `demo.gif` (README hero), `screenshot.jpg` (rendering section), `demo.mov` (source, linked as "full video").

| File | What it is | Notes |
|---|---|---|
| `demo.gif` | **Done.** Gameplay loop at the top of the README | 7 s from `demo.mov` (8s–15s), cropped to the screen, 10 fps, 48 colors, 520 px wide, 3.06 MB. Autoplays inline since it is under GitHub's 10 MB limit. Regenerate with the recipe below. |
| `architecture.svg` | Top-level block diagram | Export from `../TopLevelDiagram.drawio` (**not** `Schematic.drawio` — that one shows the proposed design that was never built). SVG scales cleanly and reads correctly on light and dark GitHub themes. |
| `screenshot.jpg` | Still of the VGA output | **Done.** Serves as the README hero until a GIF exists. Doubles as hardware proof, since the monitor bezel is in frame. |
| `board.jpg` | Photo of the DE1-SoC running the game | Proves real hardware rather than simulation. Cheap to capture, disproportionately convincing. |
| `rendering.svg` | Diagram of the perspective-band technique | Maze from above, the 16 distance bands, and the resulting on-screen trapezoids. Nothing in the repo currently illustrates the core idea. |
| `waveform.png` | ModelSim waveform | Optional. Concrete evidence of verification work. |

## Regenerating the GIF

Exact recipe used for the current `demo.gif`. The crop trims the phone-camera framing down to the
monitor screen; the two-pass palette keeps a flat-colour scene from banding.

```sh
VF="crop=432:254:107:59,fps=10,scale=520:-1:flags=lanczos"

ffmpeg -ss 8 -t 7 -i demo.mov -vf "$VF,palettegen=max_colors=48" -y palette.png
ffmpeg -ss 8 -t 7 -i demo.mov -i palette.png \
       -lavfi "$VF[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5" -y demo.gif
```

Size levers, in the order worth pulling: `max_colors` (the scene is mostly flat teal, so 48 is
plenty), `fps`, then `scale`. At 128 colours and 12 fps the same clip came out at 7.5 MB, so the
palette size matters more than the frame rate here.

Don't crop tighter than `432` wide. Beyond that it starts clipping the minimap's right border.

If `ffmpeg` isn't installed: `pip install imageio-ffmpeg` ships a binary, path via
`python -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())"`.
