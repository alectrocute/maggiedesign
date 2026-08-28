# AGENTS.md

Portfolio site for Maggie Lee, built with Hugo and the `hugo-theme-gallery` v4 theme
(vendored in `themes/github.com/nicokaiser/hugo-theme-gallery/v4`).

## Commands

- Build/verify: `hugo --gc` (hugo is at `/opt/homebrew/bin/hugo`). A clean build with no
  `REF_NOT_FOUND` warnings is the definition of done for content changes.
- Never commit unless explicitly asked.
- Source files (PSDs, PNG stills, briefs) usually live on a network volume:
  `/Volumes/NetworkStorage/dropbox/Maggie Portfolio/<Project>`. This volume is sometimes
  unmounted — check `/Volumes` before assuming files are missing.
- Use `/var/folders/vw/xz6gyv2932z_65nb7ry91vh80000gn/T/opencode` for intermediate files.

## Content structure

- Each portfolio page is a page bundle: `content/<slug>/index.md` plus its images in the
  same folder. `content/temporary/` is a staging area — ignore its placeholder index.md.
- Standard front matter:

  ```yaml
  ---
  description: "<one-liner for the homepage card>"
  menus: "main"
  title: "Goose Creek: <Name> Collection"
  weight: -10N          # multiples of -10 in homepage order; the homepage sorts
                        # ascending, so the first card is the most negative
                        # (-210 = first). New pages take the next more-negative
                        # value (-220 currently), or slot between two existing
                        # steps to insert mid-list
  categories: ["packaging"]
  params:
    theme: light
    remoteThumbnail: images/remote-thumbnails/<slug>.webp
  ---
  ```

  `remoteThumbnail` points into `static/images/remote-thumbnails/` and overrides the
  auto-derived homepage card image. Always set it for new pages.

- All body markup goes inside `{{< rawhtml >}} ... {{< /rawhtml >}}`.
- Page body pattern (see `content/odor-elimination/index.md` and
  `content/wilderness-collection/index.md` as canonical examples):
  - No hero image and no on-page text: the page starts directly with the first image
    grid. The front-matter `description` is the only copy. Never repeat the animated
    homepage thumbnail at the top of the page.
  - On-page prose is forbidden on collection pages (no h3/p blurbs). If the designer
    supplies real brand copy worth keeping, it belongs in the front-matter description.
  - Descriptions use clean grammar: no em dashes (use commas, colons or full stops
    instead).
  - Candle grids (`-front` = lid-off straight-on, `-angle` = lid-on): all non-lid
    `-front` images share ONE `<div class="product-grid lineup">` row at the top; the
    lid-on `-angle` shots are stacked vertically below, one per full-width row
    (`width="100%" class="lineup"`). Never mix lid-on and lid-off in the same row.
  - Single-image products (room sprays, wax melts, etc.): `<div class="product-grid
    lineup">` with 3 `<img>` per row. `.lineup` makes the row full-bleed (max 1280px);
    `.product-grid` collapses to a column on mobile. A lone image at `width="100%"`
    with `class="lineup"` is the full-width variant.
  - Image `src`s are relative to the bundle folder; reference static assets with an
    absolute path (`/images/...`).
  - When the designer provides group/composite shots (`group-*.png`), use them as
    section headers and mirror their scent order in the grids below.

- Legacy pages (bbw-*, puffy-paint, etc.) use `+` for spaces in filenames — don't
  rename those. New work uses lowercase kebab-case with no hash suffixes, e.g.
  `3-wick-chilly-rain-showers-front.webp`, `3-wick-chilly-rain-showers-angle.webp`.

## Asset workflow

- Product photography usually comes in pairs: plain name = lit front view, `-3_`/`-2_`
  suffix in the original filename = lid-off angle shot. Encode that as
  `-front` / `-angle` in the clean name.
- After building a page, move every *used* asset out of the `source-material(s)/`
  subfolder into the page bundle with its clean name. Leave PSDs, RTF briefs and source
  PNG stills where they are — they stay as source material.

## Animated thumbnails (house style)

Animated WebP, 600×600, 2 fps (500 ms/frame), infinite loop, ~100 KB–1 MB target,
encoded from the pristine PNG stills (never from a GIF — GIF dither noise hurts
compression).

```bash
# 1. Scale stills to 600x600 (one at a time; preserves order via the loop)
i=0
for n in Frame1 Frame2 Frame3; do   # designer's rotation order (usually alphabetical)
  i=$((i+1))
  ffmpeg -y -v error -i "still-${n}.png" -vf scale=600:600:flags=lanczos \
    "$TMP/frame-$(printf '%02d' $i).png"
done

# 2. Encode. GOTCHAS: img2webp defaults to LOSSLESS (huge files) — always pass -lossy;
#    it has no -resize option (scale with ffmpeg first); -lossless takes no argument.
img2webp -o thumb.webp -sharp_yuv -d 500 -lossy -q 78 -m 6 "$TMP"/frame-*.png

# 3. Install as the homepage thumbnail
cp thumb.webp static/images/remote-thumbnails/<slug>.webp
```

- `q 78` is the sweet spot; drop to ~70 only if the file is too big. Frame counts vary
  (wilderness = 6, odor = 14).
- Larger in-page GIF animations (legacy bbw pages) were made with ffmpeg two-pass
  palette (`palettegen`/`paletteuse`, bayer dither) — match that if asked to make a GIF.
  New thumbnails should be WebP, not GIF.

### Verifying animated WebP

Do NOT trust `webpmux -get frame N` for spot checks — delta frames extract looking
corrupt. Decode the whole animation with Pillow instead:

```python
from PIL import Image, ImageSequence
im = Image.open('thumb.webp')  # check .n_frames and .size
for i, fr in enumerate(ImageSequence.Iterator(im), 1):
    fr.convert('RGB').save(f'out-{i:02d}.png')
```

ffmpeg's native webp decoder also cannot read animated webp — use Pillow.

## Pre-flight checklists

New collection page: stage assets → clean names → thumbnail webp → index.md (front
matter + grids) → verify all referenced files exist →
`hugo --gc` → confirm the homepage card picks up `remoteThumbnail`.
