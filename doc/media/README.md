# Media assets

The root `README.md` references these files. Until they exist, those images render as broken links
on GitHub — so either add them or remove the corresponding `<img>` tags before making the repo public.

**Present:** `demo.gif` (README hero), `screenshot.jpg` (rendering section), `demo.mp4` (linked as "full video"), `demo.mov` (uncropped original, keep as the source for re-cropping).

The VGA output is pillarboxed on a widescreen monitor, so the raw footage has black bars on **both**
sides of the picture. The crop below removes them. An earlier crop that included part of those bars
is what made the first GIF look badly framed.

| File | What it is | Notes |
|---|---|---|
| `demo.gif` | **Done.** Gameplay loop at the top of the README | 5.5 s from `demo.mov` (9s–14.5s), 12 fps, 80 colors, 540 px wide, 4.8 MB. Autoplays inline, being under GitHub's 10 MB limit. |
| `demo.mp4` | **Done.** Full clip, cropped, linked from the README | H.264 CRF 23, 0.42 MB for all 21.8 s. Roughly a tenth the size of the GIF for four times the duration and better picture. See the note on playable video below. |
| `architecture.svg` | Top-level block diagram | Export from `../TopLevelDiagram.drawio` (**not** `Schematic.drawio` — that one shows the proposed design that was never built). SVG scales cleanly and reads correctly on light and dark GitHub themes. |
| `screenshot.jpg` | Still of the VGA output | **Done.** Serves as the README hero until a GIF exists. Doubles as hardware proof, since the monitor bezel is in frame. |
| `board.jpg` | Photo of the DE1-SoC running the game | Proves real hardware rather than simulation. Cheap to capture, disproportionately convincing. |
| `rendering.svg` | Diagram of the perspective-band technique | Maze from above, the 16 distance bands, and the resulting on-screen trapezoids. Nothing in the repo currently illustrates the core idea. |
| `waveform.png` | ModelSim waveform | Optional. Concrete evidence of verification work. |

## Making the video actually play on the page

GitHub will not render a `<video>` tag pointing at a file in the repo, so `demo.mp4` is only a
download link. It *will* play inline if the file is served from GitHub's own attachment CDN:

1. Open any issue on the repo (it does not need to be submitted, or can be deleted after).
2. Drag `demo.mp4` into the comment box and wait for the upload to finish.
3. GitHub rewrites the box to a `https://github.com/user-attachments/assets/<uuid>` URL.
4. Put that URL in the README on a line of its own, or in a `<video src="...">` tag. It renders as a
   player with controls.

Limits are 10 MB on a free plan, 100 MB on a paid one. MP4, MOV and WebM are accepted, and H.264 is
the safest codec across browsers, which is what `demo.mp4` uses.

## Regenerating the GIF

Exact recipe used for the current `demo.gif`. The crop trims the phone-camera framing down to the
monitor screen; the two-pass palette keeps a flat-colour scene from banding.

```sh
VF="crop=390:274:129:49,fps=12,scale=540:-1:flags=lanczos"

ffmpeg -ss 9 -t 5.5 -i demo.mov -vf "$VF,palettegen=max_colors=80" -y palette.png
ffmpeg -ss 9 -t 5.5 -i demo.mov -i palette.png \
       -lavfi "$VF[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5" -y demo.gif
```

And the MP4:

```sh
ffmpeg -i demo.mov -vf "crop=390:274:129:49" -c:v libx264 -preset slow -crf 23 \
       -pix_fmt yuv420p -movflags +faststart -an -y demo.mp4
```

`crop=390:274:129:49` is the game picture with the pillarbox bars removed. Verify any change to it
against a frame before encoding: cropping past 390 wide clips the minimap's right border, and
leaving it wider pulls the black bars back in.

Size levers for the GIF, in the order worth pulling: `max_colors`, then `fps`, then `scale`. Clip
length costs the most of all, which is why the GIF is 5.5 s while the MP4 keeps all 21.8 s.

If `ffmpeg` isn't installed: `pip install imageio-ffmpeg` ships a binary, path via
`python -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())"`.
