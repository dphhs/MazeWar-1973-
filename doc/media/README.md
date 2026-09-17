# Media assets

The root `README.md` references these files. Any that don't exist render as broken images on GitHub,
so add them or drop the corresponding `<img>` tag before showing the repo to anyone.

The README hero is **a video, not a file in this folder**. It is served from GitHub's attachment CDN
and appears in the README source as a bare URL. See the section below for why, and how to replace it.

The VGA output is pillarboxed on a widescreen monitor, so the raw footage has black bars on **both**
sides of the picture. `crop=390:274:129:49` removes them. An earlier crop that kept part of those
bars is what made the first attempt look badly framed.

| File | What it is | Notes |
|---|---|---|
| `demo.mp4` | **Done.** The clip uploaded to GitHub to produce the hero video | H.264 CRF 18 at the native crop size, 0.76 MB for all 21.8 s. Keep it: re-uploading needs this file. Don't upscale it either, a 780 px encode cost 2.86 MB and added no detail the 390 px source didn't have. |
| `demo.mov` | Uncropped original from the phone | The only source with the full frame. Keep it if the crop might ever be redone. |
| `screenshot.jpg` | **Done.** Still of the VGA output, used in the rendering section | A crisp still beats a paused video for detail, and it sits beside the text explaining that each wall step is one distance-table entry. |
| `architecture.svg` | Top-level block diagram | Export from `../TopLevelDiagram.drawio` (**not** `Schematic.drawio` — that one shows the proposed design that was never built). SVG scales cleanly and reads correctly on light and dark GitHub themes. |
| `board.jpg` | Photo of the DE1-SoC running the game | Proves real hardware rather than simulation. Cheap to capture, disproportionately convincing. |
| `rendering.svg` | Diagram of the perspective-band technique | Maze from above, the 16 distance bands, and the resulting on-screen trapezoids. Nothing in the repo currently illustrates the core idea. |
| `waveform.png` | ModelSim waveform | Optional. Concrete evidence of verification work. |

## The hero video

The current one is:

```
https://github.com/user-attachments/assets/8940a5df-ba85-441f-8bde-de87af83c0e5
```

That bare URL sits on its own line in the root README. GitHub turns it into a `<video>` player with
controls. Verified in the rendered output, where it resolves to
`private-user-images.githubusercontent.com/.../8940a5df-....mp4`.

Repo-hosted video cannot do this. Tested against GitHub's own rendering API rather than assumed, and
every form fails:

| Written in the README | What GitHub renders |
|---|---|
| `<video src="doc/media/demo.mp4">` | tag stripped, nothing left |
| `<video>` with a `raw.githubusercontent.com` URL | tag stripped |
| `<video>` with a `github.com/.../raw/...` URL | tag stripped |
| A bare URL on its own line | an ordinary text link |
| `![demo](demo.mp4)` | `<img src="...mp4">`, which no browser can play |

`<video>` is not on GitHub's HTML allowlist for markdown, so the source of the file is irrelevant.
That is why the hero is a GIF.

### How repos with playable video actually do it

Authors never write the `<video>` tag. **GitHub generates it.** Checking a repo that has one
([tiny-tpu](https://github.com/tiny-tpu-v2/tiny-tpu)), its README source is a bare URL alone on a
line:

```
https://github.com/user-attachments/assets/b5d6aefe-4250-4c6d-866e-65d519e4de74
```

GitHub's renderer recognises the attachment URL and emits the player itself, with its own classes
and a short-lived signed URL on `private-user-images.githubusercontent.com`:

```html
<video src="https://private-user-images.githubusercontent.com/...mp4?jwt=..."
       controls="controls" muted="muted" class="d-block rounded-bottom-2 border-top width-fit">
```

So the rule is that only GitHub may emit `<video>`, and only for its own attachment URLs. A hand-written
tag is stripped no matter what it points at. The signed URL expires, but the
`user-attachments/assets/<uuid>` form in the README is stable; GitHub re-signs it on each render.

### Getting such a URL

There is no API for this. The upload happens through the web UI: 

1. Open any issue on the repo (it does not need to be submitted, or can be deleted after).
2. Drag `demo.mp4` into the comment box and wait for the upload to finish.
3. GitHub rewrites the box to a `https://github.com/user-attachments/assets/<uuid>` URL.
4. Put that URL in the README **bare, on a line of its own**. Don't wrap it in a `<video>` tag or
   markdown link syntax; GitHub only builds the player when it sees the raw URL.

Limits are 10 MB on a free plan, 100 MB on a paid one. MP4, MOV and WebM are accepted, and H.264 is
the safest codec across browsers, which is what `demo.mp4` uses.

## Regenerating the clip

Recipe used for the current `demo.mp4`, which is what gets uploaded to produce the hero:

```sh
ffmpeg -i demo.mov -vf "crop=390:274:129:49" -c:v libx264 -preset veryslow -crf 18 \
       -pix_fmt yuv420p -movflags +faststart -an -y demo.mp4
```

`crop=390:274:129:49` is the game picture with the pillarbox bars removed. Verify any change to it
against a still before encoding: cropping past 390 wide clips the minimap's right border, and
leaving it wider pulls the black bars back in.

Trim with `-ss <start> -t <duration>` if a shorter clip is wanted. The whole 21.8 s costs 0.76 MB,
so there is little reason to.

### If a GIF is ever needed instead

Only worth it somewhere that can't host the video, since the GIF was **six times the size for a
quarter of the duration**. Two-pass palette, otherwise flat colour bands badly:

```sh
VF="crop=390:274:129:49,fps=12,scale=540:-1:flags=lanczos"

ffmpeg -ss 9 -t 5.5 -i demo.mov -vf "$VF,palettegen=max_colors=80" -y palette.png
ffmpeg -ss 9 -t 5.5 -i demo.mov -i palette.png \
       -lavfi "$VF[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5" -y demo.gif
```

Size levers, in the order worth pulling: `max_colors`, then `fps`, then `scale`. Clip length costs
the most of all. That combination gave 4.8 MB for 5.5 s.

If `ffmpeg` isn't installed: `pip install imageio-ffmpeg` ships a binary, path via
`python -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())"`.
