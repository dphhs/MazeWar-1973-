# Media assets

The root `README.md` references these files. Any that don't exist render as broken images on GitHub,
so add them or drop the corresponding `<img>` tag before showing the repo to anyone.

The README hero is **a video, not a file in this folder**. It is served from GitHub's attachment CDN
and appears in the README source as a bare URL. See the section below for why, and how to replace it.

The VGA output is pillarboxed on a widescreen monitor, so the raw footage has black bars on **both**
sides of the picture. `crop=762:582:222:118` removes them. An earlier crop that kept part of those
bars is what made the first attempt look badly framed.

| File | What it is | Notes |
|---|---|---|
| `demo.mp4` | **Upload this one** to produce the hero video | 762×582, H.264 High, CRF 19, 2.6 MB for 21.8 s. Cropped from `demo_source.mov`. |
| `demo_source.mov` | Uncropped 1280×720 original from the phone | The only source with the full frame. Needed to redo the crop. |
| `screenshot.jpg` | **Done.** Still of the VGA output, used in the rendering section | A crisp still beats a paused video for detail, and it sits beside the text explaining that each wall step is one distance-table entry. |
| `architecture.svg` | Top-level block diagram | Export from `../TopLevelDiagram.drawio` (**not** `Schematic.drawio` — that one shows the proposed design that was never built). SVG scales cleanly and reads correctly on light and dark GitHub themes. |
| `board.jpg` | Photo of the DE1-SoC running the game | Proves real hardware rather than simulation. Cheap to capture, disproportionately convincing. |
| `rendering.svg` | Diagram of the perspective-band technique | Maze from above, the 16 distance bands, and the resulting on-screen trapezoids. Nothing in the repo currently illustrates the core idea. |
| `waveform.png` | ModelSim waveform | Optional. Concrete evidence of verification work. |

## The hero video

The current one is:

```
https://github.com/user-attachments/assets/cb78a6dd-839b-4528-a9c5-445bc36303fd
```

That bare URL sits on its own line in the root README. GitHub turns it into a `<video>` player with
controls.

### Upload MP4, never MOV

GitHub serves an uploaded file under a content type derived from its extension, and that decides
whether browsers can play it:

| Uploaded as | Served as | Result |
|---|---|---|
| `.mp4` | `video/mp4` | plays everywhere |
| `.MOV` | `video/quicktime` | **Firefox will not play it**, even though the stream inside is H.264 |

A `.MOV` upload looks fine in Chrome and Safari, so the breakage is easy to miss. Convert to MP4
first; the container change alone fixes it, and re-encoding is not strictly required.

Check which one a live hero is by decoding the `response-content-type` in the rendered URL:

```sh
curl -s -H "Accept: application/vnd.github.html" \
     https://api.github.com/repos/dphhs/MazeWar-1973-/readme | grep -o 'response-content-type=[^"&]*'
```

### Swapping in a different clip

Upload the new one and replace the URL. Then confirm GitHub actually resolved it, rather than
trusting that it looks right in the source:

```sh
curl -s -H "Accept: application/vnd.github.html" \
     https://api.github.com/repos/dphhs/MazeWar-1973-/readme | grep -o '<video[^>]*>'
```

A rendered `<video>` whose `src` points at `private-user-images.githubusercontent.com/...<uuid>.mp4`
means it worked. No match means GitHub did not recognise the URL, and the hero is silently empty.

Repo-hosted video cannot do this. Tested against GitHub's own rendering API rather than assumed, and
every form fails:

| Written in the README | What GitHub renders |
|---|---|
| `<video src="doc/media/demo.mp4">` | tag stripped, nothing left |
| `<video>` with a `raw.githubusercontent.com` URL | tag stripped |
| `<video>` with a `github.com/.../raw/...` URL | tag stripped |
| A bare URL on its own line | an ordinary text link |
| `![demo](demo.mp4)` | `<img src="...mp4">`, which no browser can play |

`<video>` is not on GitHub's HTML allowlist for markdown, so where the file is hosted makes no
difference. Hence the attachment-URL route above.

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
ffmpeg -i demo_source.mov -vf "crop=762:582:222:118" -c:v libx264 -preset veryslow -crf 19 \
       -pix_fmt yuv420p -profile:v high -movflags +faststart -an -y demo.mp4
```

`crop=762:582:222:118` is the game picture on the 1280×720 source with the pillarbox bars removed.
The result is 762×582, close to the 4:3 of the real 640×480 output, which is the sanity check that
the crop is right. Re-measure rather than guess if the source is ever re-shot; camera framing moves
between recordings. Overlay a grid on a still to do it:

```sh
ffmpeg -ss 2 -i demo_source.mov -frames:v 1 -vf "drawgrid=w=128:h=72:t=2:c=red@0.9" grid.png
```

Trim with `-ss <start> -t <duration>` for a shorter clip. The whole 21.8 s costs 2.6 MB, well inside
the 10 MB limit, so there is little reason to.

### If a GIF is ever needed instead

Only worth it somewhere that can't host the video, since the GIF was **six times the size for a
quarter of the duration**. Two-pass palette, otherwise flat colour bands badly:

```sh
VF="crop=762:582:222:118,fps=12,scale=540:-1:flags=lanczos"

ffmpeg -ss 9 -t 5.5 -i demo_source.mov -vf "$VF,palettegen=max_colors=80" -y palette.png
ffmpeg -ss 9 -t 5.5 -i demo_source.mov -i palette.png \
       -lavfi "$VF[x];[x][1:v]paletteuse=dither=bayer:bayer_scale=5" -y demo.gif
```

Size levers, in the order worth pulling: `max_colors`, then `fps`, then `scale`. Clip length costs
the most of all. That combination gave 4.8 MB for 5.5 s.

If `ffmpeg` isn't installed: `pip install imageio-ffmpeg` ships a binary, path via
`python -c "import imageio_ffmpeg; print(imageio_ffmpeg.get_ffmpeg_exe())"`.
