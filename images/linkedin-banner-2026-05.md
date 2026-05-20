# LinkedIn Banner Prompt — 2026-05

Source: ChatGPT image generation, paired with a Forgejo contribution-heatmap screenshot pasted as visual reference (21,759 contributions in the 12 months ending May 2026).

Image: `linkedin-banner-2026-05.png` (2206 × 713 — not exactly 4:1, but close enough that LinkedIn's auto-crop landed cleanly)

Status: Not used on the blog. Used as a LinkedIn cover photo. Kept here so the prompt and the generated asset stay versioned together for future re-use or re-generation.

## Prompt used

> Generate a polished 3D cartoon illustration designed as a **LinkedIn personal profile banner** for a DevOps blog called **"Homelab Adventures with a splash of AI"**. The attached image is a real Forgejo contribution heatmap showing **21,759 contributions in the last 12 months** — use it as direct visual reference for the colors (orange/amber squares ranging from pale to deep burnt-orange on a soft grey grid) and the calendar-grid layout style.
>
> **Canvas shape (critical, read first)**: this is an **ultra-wide, very short banner — exactly 1584 × 396 pixels, strict 4:1 aspect ratio**. It is roughly four times as wide as it is tall. Do not generate a landscape 16:9 or 2:1 image. The vertical space is narrow; compose everything horizontally across the width. If unsure, picture a long thin letterbox strip, not a normal landscape photo.
>
> **Profile photo safe zone (critical)**: a circular LinkedIn profile photo will be overlaid on the **bottom-left of the banner**, covering roughly the leftmost 15% horizontally and overlapping into the bottom half. **The lower-left quadrant must contain NOTHING important** — no faces, no text, no character bodies, no props. Treat it as pure dark background. Anything placed there will be hidden by the user's profile photo. The visible band of "real estate" is: the entire top half, plus the right 85% of the bottom half.
>
> **Layout (left to right across the wide canvas)**:
>
> 1. **Far-left zone (roughly the leftmost 30% of the width)**: mostly dark navy/charcoal background. In the **upper portion** of this zone (top third of the canvas, well above the profile-photo safe zone), place the text **`hai.wxs.ro`** (spelled H-A-I, dot, W-X-S, dot, R-O — render as the literal lowercase string `hai.wxs.ro` with two periods between segments). Modern futuristic sans-serif typeface, bold, glowing soft amber color, with a faint amber glow bleeding into the surrounding dark. Roughly the same visual weight as the title on the right. Do not hyphenate or space out individual letters; render as a normal lowercase word.
>
> 2. **Center zone (roughly 30% to 70% across)**: the **glowing wall-sized contribution heatmap**, matching the layout and color gradient of the attached reference. Months labeled across the top ("Jun" through "May", spelled J-U-N, J-U-L, A-U-G, S-E-P, O-C-T, N-O-V, D-E-C, J-A-N, F-E-B, M-A-R, A-P-R, M-A-Y), weekday labels on the left ("Mon", "Wed", "Fri"). The squares emit a soft amber glow, with darker burnt-orange cells radiating slightly brighter, as if the wall is energized. The grid sits on a dark navy/charcoal wall, not pure black.
>
> 3. **Right-of-center (roughly 60% to 85% across)**: a middle-aged man with closely buzzed dark hair, a short-trimmed dark beard with some grey, and silver aviator-style metal-framed glasses, wearing a dark hoodie. Three-quarter view, looking confidently toward the viewer with a friendly, slightly proud expression. Head and shoulders fit within the vertical space; do not let the head touch the top edge or the hoodie touch the bottom edge.
>
> 4. **Far-right zone (rightmost 15% of width)**: a friendly cartoon robot companion (sleek glossy white-and-orange plating, glowing blue eyes) standing at the man's shoulder, holding a small tablet emitting a soft blue glow.
>
> 5. **Top-right corner**: a tagline **"HOMELAB ADVENTURES WITH A SPLASH OF AI"** (spelled H-O-M-E-L-A-B, A-D-V-E-N-T-U-R-E-S, W-I-T-H, A, S-P-L-A-S-H, O-F, A-I) in clean modern sans-serif type, amber and white. Below or beside it: a bold luminous number **"21,759"** (spelled 2-1, comma, 7-5-9) in large modern sans-serif amber, with smaller subtitle **"CONTRIBUTIONS IN 12 MONTHS"** (spelled C-O-N-T-R-I-B-U-T-I-O-N-S, I-N, 1-2, M-O-N-T-H-S). All text rendered as normal words; do not hyphenate or space out individual letters anywhere in the image.
>
> **Lighting and mood**: dark room atmosphere, warm amber rim-light from the heatmap wall on the man's and robot's shoulders, subtle cool blue fill from the robot's tablet, faint amber glow around the `hai.wxs.ro` text on the left. Calm, confident, slightly cinematic.
>
> **Final reminders before generating**:
> - Strict **4:1 aspect ratio, 1584 × 396 pixels**. Wider than 16:9. Very short vertically.
> - **Lower-left quadrant must be empty dark background** — profile photo overlay will cover it.
> - Place `hai.wxs.ro` in the **upper-left**, not the lower-left.
> - Style: polished 3D cartoon illustration, dark background, orange and blue accent lighting.

## Notes for next time

- ChatGPT returned **2206 × 713** (≈ 3.1:1), still not the requested **1584 × 396** (4:1), but much closer than the first attempt (which came out 2:1). Adding the explicit "ultra-wide, very short, picture a letterbox strip" framing helped.
- The composition rule about leaving the bottom-left quadrant empty was largely honored this time — `hai.wxs.ro` landed in the upper-left, leaving the lower-left dark for the profile circle.
- All five layout zones (far-left text, heatmap center, man right-of-center, robot far-right, title top-right) were respected.
- For an exact 4:1 banner, this image still needs slight cropping or padding before upload. The image at 2206×713 has aspect 3.09:1; padding to ~2852×713 (or cropping to ~2206×552) would hit true 4:1.
