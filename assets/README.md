# EvalMonk image assets

The landing page references three ink-painting images. Drop them into this folder with these exact filenames:

| File | What it is | Used in | Recommended specs |
|---|---|---|---|
| `mountains.png` | Wide ink-painted mountain range | Fixed at the bottom of every viewport | ~2400 × 600 px, transparent or white background |
| `bamboo.png` | Tall vertical bamboo (1–2 stalks with leaves) | Left side of the "Start tracing" section | ~600 × 1800 px, transparent or white background |
| `crane.png` | Single ink-painted crane | Right side of the principle quote section | ~600 × 900 px, transparent or white background |

## Format notes

**Transparent background (best):** PNG or WebP with alpha channel. The page will composite the ink directly onto the cream paper.

**White background (works fine):** the CSS uses `mix-blend-mode: multiply` on each image, so any white in the image becomes effectively transparent — only the ink reads. This means a JPEG or a stock-photo PNG with a solid white background will look correct without any pre-processing.

## Where to source the images

You need images you have rights to use. Options:

- **Generate with an image model** (Midjourney, DALL·E, Stable Diffusion) — prompts like *"sumi-e ink painting of distant misty mountains, white background, minimal"* / *"sumi-e bamboo stalks with leaves, vertical composition, white background"* / *"sumi-e Japanese crane standing, white background, minimal ink wash"* work well
- **Licensed stock** (Adobe Stock, Shutterstock, iStock with appropriate license)
- **Public domain / CC0** sources (Unsplash, Pexels for some ink art, openverse.org for CC-licensed work)
- **Commission an artist** or paint your own

## After you add the files

The page works without changes — the `<img>` tags already point at `assets/mountains.png`, `assets/bamboo.png`, and `assets/crane.png`. Refresh the browser and the decorations appear.

## Tuning

Each image has a CSS opacity tuned for ink on cream paper:
- `.mist` (mountains) — `opacity: 0.7`
- `.begin-bamboo` — `opacity: 0.55`
- `.philosophy-crane` — `opacity: 0.5`

If your images are darker or lighter than expected, adjust these values in `index.html` (search for the class name).
