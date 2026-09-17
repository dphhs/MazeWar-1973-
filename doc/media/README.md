# Media assets

The root `README.md` references these files. Until they exist, those images render as broken links
on GitHub — so either add them or remove the corresponding `<img>` tags before making the repo public.

| File | What it is | Notes |
|---|---|---|
| `demo.gif` | Gameplay loop, shown at the top of the README | 5–8 s, ~700 px wide, **under 10 MB** so GitHub autoplays it inline. Show walking a corridor, turning a corner, and the minimap tracking in sync. |
| `architecture.svg` | Top-level block diagram | Export from `../TopLevelDiagram.drawio` (**not** `Schematic.drawio` — that one shows the proposed design that was never built). SVG scales cleanly and reads correctly on light and dark GitHub themes. |
| `screenshot.png` | Still of the VGA output | First-person view and minimap together. A crisp still carries detail that a paused GIF loses. |
| `board.jpg` | Photo of the DE1-SoC running the game | Proves real hardware rather than simulation. Cheap to capture, disproportionately convincing. |
| `rendering.svg` | Diagram of the perspective-band technique | Maze from above, the 16 distance bands, and the resulting on-screen trapezoids. Nothing in the repo currently illustrates the core idea. |
| `waveform.png` | ModelSim waveform | Optional. Concrete evidence of verification work. |

## Making the GIF from a video

With `ffmpeg`, two-pass for a good palette:

```sh
ffmpeg -i demo.mp4 -vf "fps=15,scale=700:-1:flags=lanczos,palettegen" -y palette.png
ffmpeg -i demo.mp4 -i palette.png -lavfi "fps=15,scale=700:-1:flags=lanczos[x];[x][1:v]paletteuse" -y demo.gif
```

Trim first with `-ss <start> -t <duration>` to keep it under the size limit. Drop to `fps=12` or
`scale=600:-1` if it comes out too large.

Pull a still frame for `screenshot.png`:

```sh
ffmpeg -i demo.mp4 -ss 00:00:05 -vframes 1 screenshot.png
```
