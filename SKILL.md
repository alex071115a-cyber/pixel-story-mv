---
name: "pixel-story-mv"
description: "Make a story-driven pixel-art music video for the user's own song, drawn entirely in code (320x180 chibi scenes, LRC captions, beat-synced cuts, 1080p MP4), or review/improve one. Use for 像素风MV / 给歌做像素MV / MV评审."
---

# Pixel Story MV · 像素叙事 MV

Turn a song the user already has (audio + synced `.lrc`) into a story-driven pixel-art music video. Every frame is
drawn by Python/Pillow; nothing comes from an image or video model. The look: warm, detailed native-pixel scenes,
original chibi characters (about 37 px tall, big heads, dark outline, 2-3 tone shading, blush), a cream caption box
with a red ♪, and photo-album bookends. The bottom-right corner stays empty (no date stamp).

## Ground rules

- Code draws everything. No generative image/video models, no traced or copied artwork, original characters only.
- Never write out lyrics in chat. Refer to "line 7" or paraphrase. Caption text goes straight from the user's LRC
  into the renderer.
- You cannot hear the result. Cuts are aligned to the LRC and detected beats; always ask the user to check A/V sync.
- Show, don't tell: send contact sheets, stills and short previews. Get the character look approved before
  building scenes; send a 30-second representative segment before doing the whole song (unless told "just make it").
- When the user flags a time range, extract consecutive frames there (`ffmpeg -ss T -t .5 ... tile`) and find the
  real cause before changing anything.
- Every timestamp you quote to the user must come from a frame you actually looked at (see "QA" for the one reliable
  contact-sheet command). In one run an `fps=1/6,...,tile` sheet disagreed with `-ss` frames by several seconds.

## Setup (silent)

1. Make a project folder; copy the user's audio and LRC to `input/`.
2. Write `chars.py`, `engine.py` (and `demo.py` to smoke-test) from the code blocks at the end of this file.
3. `pip install --break-system-packages librosa` if missing; check `ffmpeg`; check the CJK font path in `engine.py`
   (`fc-list :lang=zh`) and change `FONT` if needed.
4. Run `python3 demo.py` once (it needs a `song.mp3` and a `song.lrc` with >= 5 lines in the folder; copy the user's
   files or make a 40 s sine with ffmpeg). It must print `bounds []`, `holes []`, then write `demo.png`, `demo_poster.png`
   and a 7 s `demo.mp4`.

## Workflow

1. **Analyse the song.** Duration (ffprobe), beats (`engine.beats`), LRC-vs-onset alignment, energy per 5 s
   (where verses, choruses, the climax and the quiet outro are). Ballads often detect at double tempo; a lyric line
   usually spans one bar. Note who sings to whom — that decides the cast (e.g. a sister singing to her brother needs
   child and adult versions of both).
2. **Story, line by line.** Pick a direction with the user (offer 2-3 with AskUserQuestion when available). Then
   give every lyric line one concrete, readable picture with cause and effect (what happened, who reacts, what prop
   shows it). Repeated lyrics get new situations that echo the first ones. Find a motif from the title and use it as
   bookends (intro / outro), plus callbacks (same door-frame years later, the same promise gesture, the same bench).
   About one shot per line, 2-4 s each; split long lines and instrumental passages into short varied shots.
   **Variety budget** (plan it before drawing; `film.shot_report()` checks it): at least one distinct location per
   ~25 s of song (4 min -> 9+), no non-motif location above ~25 % of the runtime, no more than two near-identical
   repeats of any set (a repeated motif must *change*: the jar fills, the sign flips, the weather turns). Mix the
   camera: wides, mediums, 2-3 prop/hand inserts, and one high or far viewpoint (rooftop, tower, bridge) so every
   shot is not eye-level. Alternate quiet interior with open exterior. Give wides life: extras, birds, drifting
   leaves, flags, lit windows, steam. Cast: the leads plus a few recurring side figures, each with a clear job.
   **Ending**: the emotional payoff is a character beat (a close-up or medium held >= 3 s), not an empty establishing
   shot; then pull out to the motif bookend, then an end card (MV credit). Never end on a bare fade
   of a static skyline. **Intro**: one title card (song, singer, 词/曲) held ~2.5 s beats four 2-second chips.
   Send the plan as a short paraphrased list.
3. **Character test.** Render a cast sheet (front/side/back, 4 walk frames, expressions) and one test scene with a
   caption. Get approval on the look before anything else.
4. **Segment by segment.** For each section: storyboard stills (one per shot) → fix → render. First deliver a
   ~30 s segment, then continue (or the whole song when asked). Keep each section in its own module with a list of
   `(start, end, fn, stamp)` shots (stamp is `None` unless the user asked for a date stamp); join sections at lyric
   start times into one `Film` for the full render.
5. **QA before every delivery** (do not skip):
   - Frame-accurate contact sheets of the whole render, 1-2 s steps (labels come from the decoded frame, so they can
     be trusted; do not use `fps=` + `tile`):
     `ffmpeg -i out.mp4 -vf "select='isnan(prev_selected_t)+gte(t-prev_selected_t,2)',scale=480:-1,drawtext=text='%{pts\:hms}':x=6:y=6:fontcolor=yellow:fontsize=22:box=1:boxcolor=black@0.6,tile=4x3" -an -vsync 0 sheet_%02d.png`
   - Full-size frames of every close-up and every contact action (hands meeting, hand-overs, props).
   - `film.check_bounds()` and `film.check_holes()` must both return `[]` (camera outside the world; pure-black holes:
     unfilled gaps between buildings, a layer that stops short of the edge, a pan that exposes the canvas).
   - `film.sync_report(beats)`: `pct` should clearly beat `chance_pct` (try `beats[::2]` for ballads); fix the
     listed off-beat starts or snap them with `snapper`. Lyric-driven cuts may legitimately sit off-beat.
   - `film.shot_report()`: `static` must be empty (add motion, a push-in, falling petals, or cut), and `locations`
     must meet the variety budget above.
   - Read the caption on the brightest and busiest frames; it must be legible at phone size, and the bottom-right
     corner must be empty.
   - Go through the action-logic checklist below.
   - Loudness: `ffmpeg -i out.mp4 -af ebur128=peak=true -f null -` → about -14 LUFS, true peak ≤ -1 dBFS.
6. **Deliver** the MP4 with SendUserFile and a short summary: what each section shows, what was checked, and that
   sync needs their ears. Frame 0 is black (fade-in), so also send `film.poster('cover.png', t)` as the cover. The
   encoder is x264 `-tune animation -crf 16` (flat pixel art needs the bits: platforms smear hard edges on re-encode).

## Visual spec

- **Canvas**: native 320x180, rendered at 24 fps to 1920x1080. Zoom 1 = x6 (wide), zoom 2 = crop 160x90, x12
  (medium / close). Only integer scales. Never shrink a rendered frame into a smaller image — it turns into mosaic.
- **Photos / album shots**: use a 640x360 'hi' canvas (x3). Paste 160x90 zoom-2 crops at 1:1 as polaroids; animate a
  photo flying into the album with integer steps 4→3→2→1 (`polaroid(src, k)`).
- **No dithering where zoom 2 can see it**: `gradient(dither=False)`, `light()` uses 3 clean bands. Screen-door dither
  becomes a checkerboard at x12. Dithered skies are OK only in zoom-1 wides.
- **Scenes**: layered — sky gradient, far silhouettes with lit windows, mid props, ground with tile/plank lines,
  warm light pools, readable shop signs (`pixtext`, size ≥ 11). Fill every region: gaps between buildings must be
  wall/alley, never black. **Pure #000 never appears in the art** (outline #2b2233, night sky e.g. #1a1424), which is
  what lets `check_holes()` catch gaps. One strong story prop per line (crumpled test paper with a red cross, two ice creams,
  a red charm, a guitar on its stand...). Callbacks reuse the same set.
- **Captions**: cream rounded box (247,237,216) with brown border and soft shadow, red ♪, Noto Sans CJK Bold 44 px,
  centred, box top at y=952. Credits lines (词/曲) without ♪. Faces and key action must stay above the caption: at
  zoom 2 keep feet/hands of important figures above `y0 + 79`; at zoom 1 above y = 158.
- **No date stamp.** The bottom-right corner stays empty; only the caption box lives at the bottom. Show time passing
  with the scene itself (season, light, hair length, a growing plant, a calendar prop). The engine still accepts a
  stamp string as the 4th element of a shot (None = nothing) and draws it with a dark ink outline, but use it only
  when the user explicitly asks for one.
- **One pixel grid per frame (no "mixels")**: every overlay — thought bubbles, rain, glints, glass edges, vignettes,
  title text — is drawn on the 320x180 world image, never at 1920x1080 after the upscale, and is at least 1 native
  pixel thick. Thought bubbles sit 8-12 px above `head_top(name, feet)`, never on the hat. Zoom-2 inserts are
  intentionally chunkier than wides: keep them short, cut into them on an action beat, and never mix zoom-1 and
  zoom-2 elements in one frame.
- **Title / credit / end cards**: `pixtext_c` on a real scene (shop front, skyline), not on pure black. Title card
  holds ~2.5 s; the end card carries the MV credit.
- **Transitions**: hard cuts on beats; dip-to-white into a memory; a freeze that becomes a polaroid; camera tilt to
  the sky to close a section.

## Character spec (chars.py)

- Sprite canvas 32x52, body centre x=16, shoe-bottom row y=45. `put(im, sprite, x, feet)` places it.
- Head = ellipse; adults rx 9.2 / ry 8.6, kids rx 8.4-8.6 / ry 7.8-8.0, teens 8.9 / 8.3. Torso 12 (adult) / 10
  (teen) / 8 (kid) rows; legs 7-8 / 6 / 4-5. Arms 3 px (adult) / 2 px (kid). Taller grown-up brother: leg_h 8.
- Auto outline (#2b2233) around the silhouette plus a chin line; hair has a shadow side and a highlight arc;
  clothes have a shaded side; blush under the eyes.
- Views: `front`, `side` (faces right; `mirror=True` to face left), `back`. Hair styles: `boy`, `pigtails`,
  `ponytail`. Outfits: `hoodie`, `tee_stripe`, `dress`, `cardigan`; bottoms `pants`, `shorts`, `dress`, `longskirt`.
- Expressions: neutral, smile, laugh, soft (+ `tearful` spec flag for a tear), sad, angry, surprise, cry (streaming
  tears, sobbing mouth), silly (wink + tongue); append `_blink` for a blink.
- Arms: `hands={'l': (dx, dy), 'r': ...}` (front/back) or `{'near': ..., 'far': ...}` (side) — hand offset from the
  shoulder. Natural hang ≈ (0, torso_h-3). Poses: `sit_side` (seated, shins hanging). Props: `ice=True|'melt'|'both'`.
- Walking: `walking=True, frame=walkframe(t, rate)`; `samehand=True` = same-hand-same-foot (同手同脚) gait.
- New people are colour variants of an existing build (`dict(SIS_ADULT, hair=..., top=...)`); register them in
  `engine.SPECS`.

## Action-logic checklist (hard-won — check every shot)

- **Contact is computed, not eyeballed.** Use `side_hand` / `front_hand` for both people; the two hands must land
  on the same pixel (±1) in x and y. Move the characters until they do. Hand offsets longer than ~10 px (adult) or
  ~7 px (kid) look like stretched arms.
- **Chibi reach limits.** Hands cannot reach the top of anyone's head — no head pats, no adult reaching over a
  child's head. A fully raised adult hand only reaches a child's head-top height. Design gestures that fit: a push
  on the shoulder, a hop to compare heights, thumbs up, holding a jacket between two raised hands over the kids.
- **Walking hand in hand (side view):** draw the far person first, the near person 2-3 px lower and on top; near
  person's *far* hand meets far person's *near* hand; spacing 12 px (kids) / 16 px (adults, arms out) so both faces
  stay visible.
- **Facing each other:** at least 26-28 px apart or the faces touch.
- **Hand-overs:** the object sits in the giver's hand, then in the receiver's; both hands meet at the swap moment.
- **Left/right continuity:** a close-up keeps who is on which side and their sleeve colours from the wide shot.
- **Prop continuity:** a broken umbrella stays on the ground in the next shot; a guitar waits on a stand before she
  plays it; she walks all the way to the mic.
- **Seated hands** are `leg_h - 1` px lower than standing (`seated=True`).
- **Stride matches speed:** adult ≈ 3 x rate px/s, kid ≈ 2 x rate px/s; running rate 14-18. Walking backwards
  uses the reversed cycle `(3 - walkframe(t)) % 4` — never a moonwalk.
- **Camera:** static framings for walking (characters cross the frame); never track slow walks — a stepping
  background judders, and `int()` vs `round()` mismatches make the characters shake. If you must track a fast run,
  use `x0 = round(x) - c`. Run `check_bounds()`.
- **Pacing:** no static shot longer than ~4 s unless it is a deliberate emotional hold or a slow tilt; break intros
  and instrumentals into 2-3 s shots with a new action in each.
- **Draw order:** a jacket held over kids goes on before the near holder (her face stays visible); rain skips
  sheltered areas (`rain(..., shelter=...)`); translucent memory figures (`ghost`) walk ahead in the lane, never on
  top of the living characters.
- **Distance:** a far-away figure is a hand-drawn mini figure with a face, not a scaled-up blob.
- **Accessories follow the view:** headphones in profile show one ear-cup.
- **Small marks need inserts:** pencil marks on a door-frame, text on paper, a phone screen — show the body action
  in the medium shot and the detail in a close-up insert.

## Close-up recipes (160x90 frames, zoom 2)

- `thumbs_press(q, L, R)`: two raised thumbs pressed tip to tip (the promise). Animate q 0→1, add a 0.12 s push and
  a flash ring on contact. Much clearer than a pinky promise at this resolution. Match-cut kid hands → adult hands.
- `clasped_hands(left, right)`: holding hands — fingers draped over the back of the other hand, thumb on top.
  A red charm with tassel can hang below the clasp.
- Each hand is its own `Canvas`, outlined separately, then layered (`layer`), so overlaps keep an ink line.
- Door-frame insert: wooden post down the middle, existing coloured marks with single-character labels on the left,
  the child's hair dome at the bottom, an adult hand with a pencil from the right drawing the new mark left to right.
- Phone video call: hand canvas behind, phone body, a 44x74 screen with the other person's sprite, fingers in front.

## Reviewing an existing MV (user uploads an .mp4 and asks for feedback)

Look first, then talk. Same tools, no rendering:
1. `ffprobe` (duration, fps, resolution, bitrate), then the frame-accurate contact sheets from QA (6 s step; 1-2 s
   around the intro and outro), then full-size frames for anything you will cite.
2. Hard cuts: `ffmpeg -i in.mp4 -vf "select='gt(scene,0.3)',showinfo" -an -f null -` (scene-detect cuts miss dissolves
   and camera moves; say so). Compare cut times with a beat grid (spectral-flux autocorrelation in numpy is enough when
   librosa is missing); report on-beat % against the chance level `2*tol/period`, and note tempo-doubling.
3. Audio: `ebur128=peak=true`, `silencedetect`, per-10 s RMS. You cannot judge lyric sync by ear; say so and flag only
   stretches with no caption that you should ask about.
4. Defects to look for: black strips or slivers at frame edges and between buildings; thin (sub-pixel) overlay lines
   next to chunky pixels; overlays covering heads; low-contrast date stamps; unreadable small credits; static holds
   over ~6 s; first-frame black cover; no end card; ending that fades on an empty shot.
5. Scene-richness inventory: list distinct locations, time-of-day/weather variants, shot types, repeated sets and
   their share of runtime, and background life; then name the 2-3 missing shot types that would help most.
Report: what works (one short paragraph), then fixes ranked by cost/benefit with verified timestamps, then what you
could not verify (sync, taste). Offer to patch the project if they have the source.

## Audio

`Film.render(gain_db=-4)` keeps 4 dB of headroom: loud masters otherwise come out of AAC with true peaks above
0 dBFS. If a delivered file measures over -1 dBFS, re-mux the audio only:
`ffmpeg -i v.mp4 -i song.mp3 -map 0:v -map 1:a -c:v copy -af "volume=-4dB,afade=t=in:d=0.6" -c:a aac -b:a 256k out.mp4`.

## Code: chars.py

```python
"""Parametric chibi pixel characters, drawn entirely in code (same look as the approved test).
Sprite canvas 32x52, shoe bottom row y=45, body centre x=16.
"""
import math
from PIL import Image

W, H = 32, 52
CX = 16
OUT = (43, 34, 51)


def hexc(s):
    s = s.lstrip('#'); return tuple(int(s[i:i + 2], 16) for i in (0, 2, 4))


BASE = dict(skin='#f2c6a0', skin_sh='#d99a7a', blush='#ee8f8a', mouth='#8a3d3d', tongue='#e0707a',
            tear='#9fd3f5', white='#f4f4f0')

BRO_ADULT = dict(BASE, name='bro', kind='adult', hair_style='boy', rx=9.2, ry=8.6, torso_h=12, torso_w=13, leg_h=8, arm_w=3,
                 outfit='hoodie', bottom='pants',
                 hair='#2c2c3e', hair_sh='#1d1d2b', hair_hi='#5a6283',
                 top='#3f6fb5', top_sh='#2f5590', top_hi='#5f8fd0', string='#eef0f2',
                 pants='#4d5a73', pants_sh='#3b465c', shoe='#eeeeee', sole='#a9adb5')

BRO_KID = dict(BASE, name='bro_kid', kind='kid', hair_style='boy', rx=8.4, ry=7.8, torso_h=8, torso_w=11, leg_h=4, arm_w=2,
               outfit='tee_stripe', bottom='shorts',
               hair='#2c2c3e', hair_sh='#1d1d2b', hair_hi='#5a6283',
               top='#f4f1e8', top_sh='#d6d0c2', top_hi='#ffffff', stripe='#4f86c6',
               pants='#c9a46a', pants_sh='#a8854f', shoe='#e9554a', sole='#9a3a33', sock='#f4f4f0')

SIS_KID = dict(BASE, name='sis_kid', kind='kid', hair_style='pigtails', rx=8.6, ry=8.0, torso_h=8, torso_w=11, leg_h=5, arm_w=2,
               outfit='dress', bottom='dress',
               skin='#f3c8a4', hair='#5a3a2c', hair_sh='#3f271d', hair_hi='#8a5e44', tie='#e2493f',
               top='#f08a5d', top_sh='#cf6a43', top_hi='#ffad85', collar='#fbf6ea',
               pants='#f08a5d', pants_sh='#cf6a43', shoe='#7a4ea0', sole='#4b2f66', sock='#f4f4f0')

SIS_TEEN = dict(BASE, name='sis_teen', kind='kid', hair_style='ponytail', rx=8.9, ry=8.3, torso_h=10, torso_w=12, leg_h=6, arm_w=2,
                outfit='dress', bottom='dress', skin='#f3c8a4', hair='#5a3a2c', hair_sh='#3f271d', hair_hi='#8a5e44', tie='#e2493f',
                top='#f2c14e', top_sh='#d4a037', top_hi='#ffe08a', collar='#fbf6ea',
                pants='#f2c14e', pants_sh='#d4a037', shoe='#7a4ea0', sole='#4b2f66', sock='#f4f4f0')

BRO_TEEN = dict(BASE, name='bro_teen', kind='kid', hair_style='boy', rx=8.9, ry=8.3, torso_h=10, torso_w=12, leg_h=6, arm_w=2,
                outfit='tee_stripe', bottom='pants', hair='#2c2c3e', hair_sh='#1d1d2b', hair_hi='#5a6283',
                top='#f4f1e8', top_sh='#d6d0c2', top_hi='#ffffff', stripe='#d2463c',
                pants='#4d5a73', pants_sh='#3b465c', shoe='#eeeeee', sole='#a9adb5', sock='#f4f4f0')

SIS_ADULT = dict(BASE, name='sis', kind='adult', hair_style='ponytail', rx=9.2, ry=8.6, torso_h=12, torso_w=13, leg_h=7, arm_w=3,
                 outfit='cardigan', bottom='longskirt',
                 skin='#f0c3a2', skin_sh='#d4977b', blush='#e98a84', tie='#e7c35a',
                 hair='#5a3a2c', hair_sh='#3f271d', hair_hi='#8a5e44', ear='#e7c35a',
                 top='#c8674a', top_sh='#a24f38', top_hi='#e0866a', inner='#f3e7cf',
                 skirt='#5f6b45', skirt_sh='#4a5536', shoe='#6b4630', sole='#3d2a1f',
                 pants='#5f6b45', pants_sh='#4a5536')


class G:
    def __init__(s): s.p = [[None] * W for _ in range(H)]
    def put(s, x, y, c):
        x, y = int(round(x)), int(round(y))
        if 0 <= x < W and 0 <= y < H: s.p[y][x] = c
    def get(s, x, y):
        return s.p[y][x] if 0 <= x < W and 0 <= y < H else None
    def rect(s, x, y, w, h, c):
        for yy in range(int(y), int(y + h)):
            for xx in range(int(x), int(x + w)): s.put(xx, yy, c)
    def ell(s, cx, cy, rx, ry, c, where=None):
        for y in range(H):
            for x in range(W):
                if ((x + .5 - cx) / rx) ** 2 + ((y + .5 - cy) / ry) ** 2 <= 1 and (where is None or where(x, y)):
                    s.put(x, y, c)
    def line(s, a, b, c, w=1):
        (x0, y0), (x1, y1) = a, b
        n = int(max(abs(x1 - x0), abs(y1 - y0))) + 1
        for i in range(n + 1):
            q = i / max(n, 1); x = round(x0 + (x1 - x0) * q); y = round(y0 + (y1 - y0) * q)
            s.rect(x - (w - 1) // 2, y - (w - 1) // 2, w, w, c)
    def outline(s):
        add = [(x, y) for y in range(H) for x in range(W) if s.p[y][x] is None and
               any(s.get(x + dx, y + dy) not in (None, 'OUT') for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)))]
        for x, y in add: s.p[y][x] = 'OUT'
    def image(s, sp):
        im = Image.new('RGBA', (W, H), (0, 0, 0, 0)); px = im.load()
        for y in range(H):
            for x in range(W):
                c = s.p[y][x]
                if c is None: continue
                if c == 'OUT': col = OUT
                elif c.startswith('#'): col = hexc(c)
                else: col = hexc(sp.get(c, '#ff00ff'))
                px[x, y] = col + (255,)
        return im


def layout(sp, sit=0):
    hip = 44 - sp['leg_h']
    sh = hip - sp['torso_h']
    ty = sh - round(2 * sp['ry']) + 1
    return hip, sh, ty


# ------------------------------------------------------------------ hair
def hair(g, sp, view, ty, cx):
    rx, ry = sp['rx'], sp['ry']; cy = ty + ry + .4
    er = round(ry * 1.28)                     # eye row (relative)
    st = sp['hair_style']
    hx, hy, hrx, hry = cx, cy - .6, rx + .9, ry + .7
    def rel(y): return y - ty
    if st == 'boy':
        if view == 'front':
            pat = [1, 1, 0, 0, 1, 0, -1, 0, 1, 0, -1, -1, 0, 1, 0, -1, 0, 0, 1, 1, 1]
            def hw(x, y):
                dx = x - cx
                if abs(dx + .5) >= rx - .5: return rel(y) <= er - 1
                return rel(y) <= er - 3 + pat[int(dx + 10) % len(pat)] - 1
            g.ell(hx, hy, hrx, hry, 'hair', hw)
            for dx, dy in ((-5, -1), (-4, -1), (1, -1), (2, -2), (3, -1)): g.put(cx + dx, ty + dy, 'hair')
        elif view == 'side':
            def hw(x, y):
                dx = x - cx
                if dx < -1: return rel(y) <= ry * 1.75
                if dx < 4: return rel(y) <= er - 4 + (1 if x % 3 == 0 else 0)
                return rel(y) <= er - 3 - (dx - 4) // 2 + (1 if x % 2 else 0)
            g.ell(hx - .4, hy, hrx, hry, 'hair', hw)
            for dx, dy in ((-4, -1), (0, -1), (1, -2), (2, -1), (9, 4)): g.put(cx + dx, ty + dy, 'hair')
        else:
            g.ell(hx, hy, hrx, hry, 'hair', lambda x, y: rel(y) <= ry * 1.75)
            for dx, dy in ((-5, -1), (1, -1), (2, -2)): g.put(cx + dx, ty + dy, 'hair')
    elif st == 'pigtails':
        tufts = []
        if view == 'front':
            def hw(x, y):
                dx = x - cx
                if abs(dx + .5) >= rx - 1.5: return rel(y) <= er + 1
                return rel(y) <= er - 3 - (1 if abs(dx) in (0, 5) else 0)
            g.ell(hx, hy, hrx, hry, 'hair', hw)
            tufts = [(cx - rx - 2.2, ty + er), (cx + rx + 1.8, ty + er)]
        elif view == 'side':
            def hw(x, y):
                dx = x - cx
                if dx < -1: return rel(y) <= ry * 1.6
                return rel(y) <= er - 3 - max(0, dx - 5) // 2
            g.ell(hx - .4, hy, hrx, hry, 'hair', hw)
            tufts = [(cx - rx - 1.4, ty + er)]
        else:
            g.ell(hx, hy, hrx, hry, 'hair', lambda x, y: rel(y) <= ry * 1.6)
            tufts = [(cx - rx - 2.2, ty + er), (cx + rx + 1.8, ty + er)]
            g.put(cx, ty + 1, 'hair_sh'); g.put(cx, ty + 3, 'hair_sh'); g.put(cx, ty + 5, 'hair_sh')
        for (tx, tyy) in tufts:
            side = 1 if tx < cx else -1
            g.ell(tx, tyy + 1.8, 2.7, 3.8, 'hair')
            g.put(tx - side, tyy + 4, 'hair_sh'); g.put(tx, tyy + 5, 'hair_sh'); g.put(tx - side, tyy, 'hair_hi')
            g.rect(int(round(tx + side * 1.6)) - (1 if side < 0 else 0), int(tyy) - 2, 2, 3, 'tie')
    elif st == 'ponytail':
        if view == 'front':
            def hw(x, y):
                dx = x - cx
                if abs(dx + .5) >= rx - 1.2: return rel(y) <= er + 2
                d = abs(dx + .5)
                return rel(y) <= 3 + (d >= 1.5) + (d >= 3.5) + (d >= 5.5) + 2 * (d >= 7)
            g.ell(hx, hy, hrx, hry, 'hair', hw)
            g.put(cx - 1, ty + 1, 'hair_sh')
        elif view == 'side':
            def hw(x, y):
                dx = x - cx
                if dx < -1: return rel(y) <= ry * 1.7
                if dx < 3: return rel(y) <= 6
                return rel(y) <= 5 - (dx - 3) // 3 + (dx > 6)
            g.ell(hx - .4, hy, hrx, hry, 'hair', hw)
            # ponytail hanging behind the head
            px0 = cx - rx - .2
            for i in range(11):
                w = 3 if i < 8 else 2
                g.rect(px0 - 2 + (1 if i > 5 else 0) - (i // 4), ty + 6 + i, w, 1, 'hair' if i % 4 else 'hair_sh')
            g.rect(px0 - 1, ty + 5, 2, 2, 'tie')
        else:
            g.ell(hx, hy, hrx, hry, 'hair', lambda x, y: rel(y) <= ry * 1.7)
            g.rect(cx - 1, ty + 9, 3, 12, 'hair'); g.rect(cx, ty + 12, 1, 8, 'hair_sh')
            g.rect(cx - 1, ty + 8, 3, 1, 'tie')
    # shading + highlights
    for y in range(H):
        for x in range(W):
            if g.get(x, y) != 'hair': continue
            dx, r = x - cx, rel(y)
            if view == 'front':
                if (dx >= 6 and r >= 3) or (r >= er - 2 and abs(dx) >= rx - 1.5): g.put(x, y, 'hair_sh')
            elif view == 'side':
                if dx <= -4 and r >= 5: g.put(x, y, 'hair_sh')
            else:
                if r >= er or dx >= 6: g.put(x, y, 'hair_sh')
    hi = {'front': [(-5, 2), (-4, 1), (-3, 1), (-2, 1), (-6, 3), (2, 2), (3, 2)],
          'side': [(-2, 1), (-1, 1), (0, 1), (1, 1), (-3, 2), (4, 2), (5, 3)],
          'back': [(-4, 2), (-3, 1), (-2, 1), (-1, 1), (2, 1), (3, 2), (-5, 3)]}[view]
    for dx, dy in hi:
        if g.get(cx + dx, ty + dy) in ('hair', 'hair_sh'): g.put(cx + dx, ty + dy, 'hair_hi')
    if view == 'back' and st == 'boy':
        for dx, dy in ((-3, 7), (-3, 8), (-2, 9), (3, 7), (3, 8), (2, 9), (0, 4), (0, 5)):
            if g.get(cx + dx, ty + dy) == 'hair': g.put(cx + dx, ty + dy, 'hair_sh')


def head(g, sp, view, ty, expr, t, look=0):
    rx, ry = sp['rx'], sp['ry']
    cx = CX if view != 'side' else CX
    cy = ty + ry + .4
    er = round(ry * 1.28)
    g.ell(cx, cy, rx, ry, 'skin')
    if view == 'side':
        g.ell(cx + 2.2, cy + 1.6, rx - 2.3, ry - 2.3, 'skin')
    if view == 'front':
        g.rect(int(cx - rx - .6), ty + er - 1, 1, 3, 'skin'); g.rect(int(cx + rx - .4), ty + er - 1, 1, 3, 'skin')
    hair(g, sp, view, ty, cx)
    if view == 'side':
        g.rect(cx - 2, ty + er - 1, 2, 3, 'skin'); g.put(cx - 2, ty + er, 'skin_sh')
    if view == 'front' and sp.get('ear'):
        g.put(int(cx - rx - .6), ty + er + 2, 'ear'); g.put(int(cx + rx - .4), ty + er + 2, 'ear')
    if view == 'front':
        for y in range(ty + er + 2, ty + er + 7):
            for x in range(cx + 5, cx + 10):
                if g.get(x, y) == 'skin' and ((x + 1.5 - cx) / rx) ** 2 + ((y + .5 - cy) / ry) ** 2 > 1: g.put(x, y, 'skin_sh')
    if view != 'back':
        face(g, sp, view, ty, cx, er, expr, t, look)


def face(g, sp, view, ty, cx, er, expr, t, look):
    ey = ty + er
    eyes = [(cx - 4 + look, ey), (cx + 3 + look, ey)] if view == 'front' else [(cx + 5, ey)]
    blink = expr.endswith('_blink'); e = expr.replace('_blink', '')
    for (x, y) in eyes:
        if blink:
            g.put(x, y + 1, 'OUT'); g.put(x + (1 if (x < cx or view == 'side') else -1), y + 1, 'OUT')
        elif e in ('smile', 'laugh', 'soft'):
            g.put(x - 1, y + 1, 'OUT'); g.put(x, y, 'OUT'); g.put(x + 1, y + 1, 'OUT')
        elif e == 'surprise':
            g.rect(x, y - 1, 1, 3, 'OUT')
        elif e == 'cry':
            inner = 1 if (x > cx or view == 'side') else -1
            g.put(x - inner, y, 'OUT'); g.put(x, y + 1, 'OUT'); g.put(x + inner, y + 1, 'OUT')   # squeezed shut
            g.put(x - inner, y - 2, 'OUT'); g.put(x, y - 2, 'OUT')
        elif e == 'silly':
            if x < cx and view == 'front':
                g.put(x - 1, y + 1, 'OUT'); g.put(x, y + 1, 'OUT'); g.put(x + 1, y + 1, 'OUT')   # wink
            else:
                g.rect(x, y - 1, 1, 3, 'OUT')
        elif e == 'angry':
            g.rect(x, y, 1, 2, 'OUT')
            inner = 1 if (x < cx and view == 'front') else -1
            if view == 'side': inner = 1
            g.put(x - inner, y - 2, 'OUT'); g.put(x, y - 2, 'OUT'); g.put(x + inner, y - 1, 'OUT')
        else:
            g.rect(x, y, 1, 2, 'OUT')
            if e == 'sad':
                inner = 1 if (x > cx or view == 'side') else -1
                g.put(x - inner, y - 1, 'OUT'); g.put(x, y - 2, 'OUT')
    if view == 'front':
        g.rect(cx - 6, ey + 3, 2, 1, 'blush'); g.rect(cx + 4, ey + 3, 2, 1, 'blush')
        my = ey + 4
        if e in ('smile', 'soft'): g.rect(cx - 1, my, 2, 1, 'mouth'); g.put(cx - 2, my - 1, 'mouth'); g.put(cx + 1, my - 1, 'mouth')
        elif e == 'laugh': g.rect(cx - 2, my - 1, 4, 1, 'mouth'); g.rect(cx - 1, my, 2, 1, 'tongue'); g.put(cx - 2, my, 'mouth'); g.put(cx + 1, my, 'mouth')
        elif e in ('sad', 'angry'): g.rect(cx - 1, my, 2, 1, 'mouth'); g.put(cx - 2, my + 1, 'mouth'); g.put(cx + 1, my + 1, 'mouth')
        elif e == 'surprise': g.rect(cx - 1, my, 2, 2, 'mouth')
        elif e == 'cry':
            open_ = int(t * 4) % 2
            g.rect(cx - 2, my - 1, 4, 2 + open_, 'mouth'); g.rect(cx - 1, my + open_, 2, 1, 'tongue')
        elif e == 'silly': g.rect(cx - 1, my, 3, 1, 'mouth'); g.rect(cx, my + 1, 2, 1, 'tongue')
        else: g.rect(cx - 1, my, 2, 1, 'mouth')
        if e == 'cry':                                     # streaming tears
            for (x, _) in eyes:
                for k in range(3):
                    yy = ey + 2 + (int(t * 10) + k * 2) % 6
                    g.put(x, yy, 'tear')
        if expr.startswith('soft') and sp.get('tearful'):
            g.put(cx + 3 + look, ey + 2 + int(t * 1.5) % 3, 'tear')
    else:
        g.rect(cx + 3, ey + 3, 2, 1, 'blush')
        g.put(cx + 8, ty + er + 1, 'skin'); g.put(cx + 8, ty + er + 2, 'skin')
        my = ey + 4
        if e == 'surprise': g.rect(cx + 6, my, 1, 2, 'mouth')
        elif e in ('smile', 'soft'): g.put(cx + 6, my, 'mouth'); g.put(cx + 7, my - 1, 'mouth')
        elif e == 'laugh': g.rect(cx + 6, my - 1, 2, 2, 'mouth'); g.put(cx + 6, my, 'tongue')
        elif e in ('sad', 'cry'): g.put(cx + 6, my, 'mouth'); g.put(cx + 7, my + 1, 'mouth')
        elif e == 'angry': g.put(cx + 6, my, 'mouth'); g.put(cx + 7, my, 'mouth'); g.put(cx + 8, my, 'skin'); g.rect(cx + 2, ey + 3, 3, 1, 'blush')
        else: g.put(cx + 6, my, 'mouth')
        if e == 'cry':
            for k in range(2): g.put(cx + 5, ey + 2 + (int(t * 10) + k * 3) % 6, 'tear')


# ------------------------------------------------------------------ body
WALK = [(3, -3, 0, 0), (1, -1, 0, 1), (-3, 3, 0, 0), (-1, 1, 1, 0)]


def torso_front(g, sp, view, sh, hip, tb):
    tw = sp['torso_w']; x0 = CX - tw // 2
    o = sp['outfit']
    bottom = hip if sp['bottom'] not in ('longskirt',) else hip - 2
    g.rect(x0, tb, tw, bottom - tb, 'top')
    g.rect(x0 + tw - 3, tb + 1, 3, bottom - tb - 1, 'top_sh')
    g.rect(CX - 1, sh - 1, 3, 1, 'skin')
    if view == 'front':
        g.put(CX - 1, sh - 1, 'skin_sh'); g.put(CX + 1, sh - 1, 'skin_sh')
    if o == 'hoodie':
        if view == 'front':
            g.rect(x0 + 2, tb, tw - 4, 1, 'top_sh')
            for dy in (1, 2, 3): g.put(CX - 2, tb + dy, 'string')
            for dy in (1, 2): g.put(CX + 2, tb + dy, 'string')
            g.rect(x0 + 2, hip - 5, tw - 4, 1, 'top_sh'); g.put(CX, hip - 4, 'top_sh'); g.put(CX, hip - 3, 'top_sh')
            g.put(x0 + 2, tb + 2, 'top_hi'); g.put(x0 + 1, tb + 3, 'top_hi')
        else:
            g.rect(CX - 3, tb, 7, 3, 'top_sh'); g.rect(CX - 2, tb + 3, 5, 1, 'top_sh')
        g.rect(x0, hip - 1, tw, 1, 'top_sh')
    elif o == 'tee_stripe':
        for y in range(tb + 2, hip, 2): g.rect(x0, y, tw, 1, 'stripe')
        if view == 'front': g.rect(CX - 1, tb, 3, 1, 'top_sh')
    elif o == 'dress':
        if view == 'front':
            g.rect(CX - 2, tb, 2, 1, 'collar'); g.rect(CX + 1, tb, 2, 1, 'collar'); g.put(CX - 2, tb + 1, 'collar'); g.put(CX + 2, tb + 1, 'collar')
            g.put(CX, tb + 3, 'top_sh'); g.put(CX, tb + 5, 'top_sh')
    elif o == 'cardigan':
        if view == 'front':
            g.rect(CX - 1, tb, 3, bottom - tb, 'inner')
            g.put(CX - 2, tb, 'top_hi'); g.put(CX + 2, tb, 'top_hi')
            for y in range(tb + 2, bottom, 3): g.put(CX - 2, y, 'top_sh')
        else:
            pass


def legs_front(g, sp, hip, lifts=(0, 0)):
    b = sp['bottom']; lh = sp['leg_h']
    if b == 'longskirt':
        for i, x0 in enumerate((CX - 4, CX + 1)):
            l = lifts[i]; g.rect(x0, 44 - l, 4, 1, 'shoe'); g.rect(x0, 45 - l, 4, 1, 'sole')
        for k, y in enumerate(range(hip - 2, 44)):
            w = 12 + k // 3; x0 = CX - w // 2
            g.rect(x0, y, w, 1, 'skirt'); g.rect(x0 + w - 2, y, 2, 1, 'skirt_sh')
        for k, y in enumerate(range(hip, 44)):
            g.put(CX - 2 - k // 4, y, 'skirt_sh'); g.put(CX + 2 + k // 4, y, 'skirt_sh')
        return
    kid = sp['kind'] == 'kid'
    lw = 3 if kid else 4
    xs = (CX - lw - 1 + (1 if kid else 0), CX + 1) if kid else (CX - 4, CX + 1)
    for i, x0 in enumerate(xs):
        l = lifts[i]
        if b == 'pants':
            g.rect(x0, hip, lw, 44 - hip - l, 'pants'); g.rect(x0 + lw - 1 if i else x0, hip, 1, 44 - hip - l, 'pants_sh')
        elif b == 'shorts':
            g.rect(x0, hip, lw, 2, 'pants'); g.rect(x0, hip + 2, lw, 44 - hip - 2 - l, 'skin')
            g.rect(x0, 43 - l, lw, 1, 'sock')
        else:   # dress: bare legs + socks
            g.rect(x0, hip, lw, 44 - hip - l, 'skin'); g.rect(x0, 42 - l, lw, 2, 'sock')
        sw = lw + 1
        g.rect(x0 - (1 if i == 0 else 0), 44 - l, sw, 1, 'shoe'); g.rect(x0 - (1 if i == 0 else 0), 45 - l, sw, 1, 'sole')
    if b == 'shorts':
        g.rect(xs[0], hip, xs[1] + lw - xs[0], 1, 'pants')
    if b == 'dress':        # flared hem
        tw = sp['torso_w']
        for k in range(3):
            w = tw + 2 + k; x0 = CX - w // 2
            g.rect(x0, hip - 1 + k, w, 1, 'top'); g.rect(x0 + w - 3, hip - 1 + k, 3, 1, 'top_sh')
        g.rect(CX - (tw + 4) // 2, hip + 2, tw + 4, 1, 'top_sh')


def arm(g, sp, shoulder, hand, sleeve, shade=False):
    aw = sp['arm_w']
    col = 'top_sh' if shade else 'top'
    sx, sy = shoulder; hx, hy = hand
    L = max(1, math.hypot(hx - sx, hy - sy))
    sl = {'long': 1.0, 'short': .38}[sleeve]
    mid = (sx + (hx - sx) * sl, sy + (hy - sy) * sl)
    if sleeve == 'long':
        g.line(shoulder, hand, col, aw)
    else:
        g.line(shoulder, mid, col, aw); g.line(mid, hand, 'skin_sh' if shade else 'skin', max(2, aw - 1))
    g.rect(hx - (aw - 1) // 2, hy, aw, 2, 'skin_sh' if shade else 'skin')


def sprite(sp, view='front', frame=0, walking=False, expr='neutral', t=0.0, hands=None, look=0, samehand=False,
           pose=None, prop=None, bob=0):
    """hands: dict side->(dx,dy) hand offset from shoulder (front: 'l','r'; side: 'near','far')."""
    g = G()
    hip, sh, ty = layout(sp)
    b = (1 if walking and frame in (1, 3) else 0) + bob
    kid = sp['kind'] == 'kid'
    sleeve = 'short' if sp['outfit'] in ('tee_stripe', 'dress') else 'long'
    aw = sp['arm_w']; alen = sp['torso_h'] - (2 if kid else 3)
    tw = sp['torso_w']
    hands = hands or {}
    if pose == 'sit_side':           # side view, sitting on bed edge, shins hanging
        drop = sp['leg_h'] - 1
        hip2, sh2, ty2 = hip + drop, sh + drop, ty + drop
        thigh = 8 if kid else 11
        legcol = 'pants' if sp['bottom'] in ('pants', 'shorts') else ('skirt' if sp['bottom'] == 'longskirt' else 'top')
        g.rect(CX - 3, hip2 - (1 if kid else 2), thigh, 2 if kid else 3, legcol)
        swing = round(math.sin(t * 6) * 1) if expr in ('laugh', 'smile') else 0
        shin = 4 if kid else 6
        kx = CX - 3 + thigh - (2 if kid else 3)
        shcol = 'skin' if sp['bottom'] in ('dress', 'shorts') else ('skirt' if sp['bottom'] == 'longskirt' else 'pants')
        g.rect(kx + swing, hip2 + 1, 2 if kid else 3, shin, shcol)
        if kid: g.rect(kx + swing, hip2 + shin - 1, 2, 1, 'sock')
        g.rect(kx + swing, hip2 + 1 + shin, 4, 1, 'shoe'); g.rect(kx + swing, hip2 + 2 + shin, 4, 1, 'sole')
        g.rect(CX - 4, sh2 + b, 9, hip2 - sh2 - b, 'top'); g.rect(CX - 4, sh2 + 1 + b, 2, hip2 - sh2 - 1 - b, 'top_sh')
        if sp['outfit'] == 'tee_stripe':
            for y in range(sh2 + 2 + b, hip2, 2): g.rect(CX - 4, y, 9, 1, 'stripe')
        hn = hands.get('near', (2, alen - 1))
        arm(g, sp, (CX, sh2 + 1 + b), (CX + hn[0], sh2 + 1 + b + hn[1]), sleeve)
        before = [r[:] for r in g.p]
        head(g, sp, 'side', ty2 + b, expr, t)
        _chin(g, before, ty2 + b)
        g.outline(); return g.image(sp)

    if view in ('front', 'back'):
        lifts = [(0, 1), (0, 0), (1, 0), (0, 0)][frame] if walking else (0, 0)
        legs_front(g, sp, hip, lifts)
        tb = sh + b
        torso_front(g, sp, view, sh + b, hip, tb)
        sw = [(1, -1), (0, 0), (-1, 1), (0, 0)][frame] if walking else (0, 0)
        for i, side in enumerate(('l', 'r')):
            sgn = -1 if side == 'l' else 1
            sx = CX + sgn * (tw // 2 + 1 + (aw - 1) // 2) - (1 if side == 'l' and aw == 3 else 0)
            if side == 'r' and aw == 3: sx -= 0
            shoulder = (sx, tb + 1)
            dx, dy = hands.get(side, (sgn * 1 if aw == 2 else 0, alen + sw[i]))
            arm(g, sp, shoulder, (sx + dx, tb + 1 + dy), sleeve)
        if prop: prop(g, sp, view, hands, tb)
        before = [r[:] for r in g.p]
        head(g, sp, view, ty + b, expr, t, look)
        _chin(g, before, ty + b)
    else:   # side, facing right
        fl, bl, lf, lb = WALK[frame] if walking else (0, 0, 0, 0)
        if kid: fl, bl = round(fl * .67), round(bl * .67)
        tb = sh + b
        sway = round(fl * .8) if walking else 0
        near_swing = (sway if samehand else -sway)
        hf = hands.get('far', (-near_swing, alen))
        arm(g, sp, (CX - 1, tb + 1), (CX - 1 + hf[0], tb + 1 + hf[1]), sleeve, shade=True)
        lw = 3 if kid else 4
        b_ = sp['bottom']
        if b_ == 'longskirt':
            for dx, lift, back in ((bl, lb, True), (fl, lf, False)):
                x0 = CX - 2 + dx // 2 + 1
                g.rect(x0, 44 - lift, 5, 1, 'sole' if back else 'shoe'); g.rect(x0, 45 - lift, 5, 1, 'sole')
            for k, y in enumerate(range(hip - 2, 44)):
                w = 9 + k // 3
                g.rect(CX - w // 2 + (1 if walking and frame in (0, 2) and k > 6 else 0), y, w, 1, 'skirt')
            for y in range(hip, 44): g.put(CX - 4 + (y - hip) // 4, y, 'skirt_sh')
        else:
            for dx, lift, back in ((bl, lb, True), (fl, lf, False)):
                x0 = CX - 2 + dx
                if not back:
                    for y in range(hip + 1, 45 - lift):
                        if g.get(x0 - 1, y) not in (None, 'OUT'): g.put(x0 - 1, y, 'OUT')
                        if g.get(x0 + lw, y) not in (None, 'OUT'): g.put(x0 + lw, y, 'OUT')
                shade = back
                if b_ == 'pants':
                    g.rect(x0, hip, lw, 44 - hip - lift, 'pants_sh' if shade else 'pants')
                elif b_ == 'shorts':
                    g.rect(x0, hip, lw, 2, 'pants_sh' if shade else 'pants'); g.rect(x0, hip + 2, lw, 44 - hip - 2 - lift, 'skin_sh' if shade else 'skin')
                    g.rect(x0, 43 - lift, lw, 1, 'sock')
                else:
                    g.rect(x0, hip, lw, 44 - hip - lift, 'skin_sh' if shade else 'skin'); g.rect(x0, 42 - lift, lw, 2, 'sock')
                g.rect(x0, 44 - lift, lw + 1, 1, 'sole' if back else 'shoe'); g.rect(x0 + lw + 1, 44 - lift, 1, 1, 'OUT' if back else 'shoe')
                g.rect(x0, 45 - lift, lw + 2, 1, 'sole')
        bottom = hip if b_ != 'longskirt' else hip - 2
        tws = tw - 4 if not kid else tw - 3
        x0 = CX - tws // 2 - 1
        g.rect(x0, tb, tws, bottom - tb, 'top'); g.rect(x0, tb + 1, 2, bottom - tb - 1, 'top_sh')
        g.rect(CX - 1, sh - 1 + b, 3, 1, 'skin')
        o = sp['outfit']
        if o == 'hoodie':
            g.rect(x0, tb, 4, 3, 'top_sh'); g.rect(x0, hip - 1, tws, 1, 'top_sh')
            for dy in (1, 2, 3): g.put(x0 + tws - 2, tb + dy, 'string')
        elif o == 'tee_stripe':
            for y in range(tb + 2, hip, 2): g.rect(x0, y, tws, 1, 'stripe')
        elif o == 'cardigan':
            g.rect(x0 + tws - 2, tb, 2, bottom - tb, 'inner')
        elif o == 'dress':
            g.put(x0 + tws - 1, tb, 'collar'); g.put(x0 + tws - 2, tb, 'collar')
            for k in range(3):
                g.rect(x0 - 1, hip - 1 + k, tws + 2 + k, 1, 'top')
            g.rect(x0 - 1, hip + 2, tws + 4, 1, 'top_sh')
        hn = hands.get('near', (near_swing, alen))
        nsx = CX - 1
        arm(g, sp, (nsx, tb + 1), (nsx + hn[0], tb + 1 + hn[1]), sleeve)
        if sleeve == 'long': g.rect(nsx + 1 if aw == 3 else nsx, tb + 1, 1, max(1, int(hn[1] * .8)), 'top_hi') if hn[0] == near_swing and abs(hn[0]) < 2 else None
        if prop: prop(g, sp, view, hands, tb)
        before = [r[:] for r in g.p]
        head(g, sp, 'side', ty + b, expr, t)
        _chin(g, before, ty + b)
    g.outline()
    return g.image(sp)


def _chin(g, before, ty):
    headpx = {(x, y) for y in range(H) for x in range(W) if g.p[y][x] != before[y][x] and g.p[y][x] is not None}
    for (x, y) in headpx:
        if (x, y + 1) not in headpx and g.get(x, y + 1) not in (None, 'OUT') and y + 1 > ty + 11:
            g.put(x, y + 1, 'OUT')


# ------------------------------------------------------------------ props
def icecream(side='r', melt=False):
    def draw(g, sp, view, hands, tb):
        if view == 'side':
            dx, dy = hands.get('near', (2, 6)); hx, hy = CX - 1 + dx, tb + 1 + dy
        else:
            sgn = 1 if side == 'r' else -1
            dx, dy = hands.get(side, (0, 6)); hx, hy = CX + sgn * (sp['torso_w'] // 2 + 1) + dx - (1 if side == 'l' else 0), tb + 1 + dy
        g.rect(hx, hy - 2, 2, 3, '#c98b4a'); g.put(hx, hy - 1, '#e0a865'); g.rect(hx - 1, hy - 3, 4, 1, '#e0a865')
        g.rect(hx - 1, hy - 6, 4, 3, '#f7c6da'); g.rect(hx, hy - 7, 2, 1, '#f7c6da')
        g.put(hx - 1, hy - 6, '#ffeef4'); g.put(hx, hy - 7, '#ffeef4'); g.rect(hx + 2, hy - 5, 1, 2, '#e195b5')
        if melt:
            g.put(hx - 1, hy - 2, '#f7c6da'); g.put(hx + 2, hy - 2, '#f7c6da'); g.put(hx + 2, hy - 1, '#f7c6da'); g.put(hx + 1, hy + 1, '#f7c6da')
    return draw


def flip(im):
    return im.transpose(Image.FLIP_LEFT_RIGHT)


# ------------------------------------------------------------------ later-life cast (colour variants of the same build)
MOM = dict(SIS_ADULT, name='mom', hair='#8e8782', hair_sh='#6e6762', hair_hi='#b4aea8', ear=None,
           top='#7a6a9a', top_sh='#5e5080', top_hi='#9a8aba', inner='#efe6d8', skirt='#4a4a5a', skirt_sh='#3a3a48', tie='#b4aea8')
DAD = dict(BRO_ADULT, name='dad', hair='#9a9690', hair_sh='#76726c', hair_hi='#c4c0ba',
           top='#8a5a3a', top_sh='#6a4428', top_hi='#a87452', string='#8a5a3a', pants='#4a4a5a', pants_sh='#3a3a48')
SIS_OLD = dict(SIS_ADULT, name='sis_old', hair='#cbc6c1', hair_sh='#a9a49f', hair_hi='#ebe7e3', top='#a06a58', top_sh='#80503f', top_hi='#bf8672')
BRO_OLD = dict(BRO_ADULT, name='bro_old', hair='#cbc6c1', hair_sh='#a9a49f', hair_hi='#ebe7e3', top='#4a6a8a', top_sh='#38526c', top_hi='#6a8aaa')
KID_G = dict(SIS_KID, name='kid_g', hair='#2c2c3e', hair_sh='#1d1d2b', hair_hi='#5a6283', tie='#4f86c6',
             top='#f2c94c', top_sh='#d9a92a', top_hi='#ffe28a', pants='#f2c94c', pants_sh='#d9a92a', shoe='#e9554a', sole='#9a3a33')
KID_B = dict(BRO_KID, name='kid_b', hair='#5a3a2c', hair_sh='#3f271d', hair_hi='#8a5e44', stripe='#5aa86a', pants='#4d5a73', pants_sh='#3b465c',
             shoe='#4f86c6', sole='#2f5590')
```

## Code: engine.py

```python
"""Pixel story-MV engine: native 320x180 canvas, integer zoom, lyric captions, hand close-ups, ffmpeg out (optional date stamps, off by default).
A film is a list of Shot(start, end, fn[, stamp]); fn(t) -> (world_image, (x0, y0, zoom)) with zoom 1, 2 or 'hi'."""
import functools, json, math, os, re, subprocess
from PIL import Image, ImageChops, ImageDraw, ImageFont
from chars import sprite, flip, layout, CX, icecream

FPS, OUT_W, OUT_H = 24, 1920, 1080
FONT = '/usr/share/fonts/opentype/noto/NotoSansCJK-Bold.ttc'
MONO = '/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf'
OUTLINE = (43, 34, 51)


# ---------------------------------------------------------------- small helpers
def hx(s): s = s.lstrip('#'); return tuple(int(s[i:i + 2], 16) for i in (0, 2, 4))
def lerp(a, b, q): return tuple(round(a[i] + (b[i] - a[i]) * q) for i in range(3))
def sm(t, a, b): q = min(max((t - a) / (b - a), 0), 1); return q * q * (3 - 2 * q)
def lin(t, a, b): return min(max((t - a) / (b - a), 0), 1)
def walkframe(t, rate=7): return int(t * rate) % 4


def load_lrc(path):
    out = []
    for ln in open(path, encoding='utf-8'):
        m = re.match(r'\[(\d+):(\d+\.\d+)\](.*)', ln.strip())
        if m: out.append((int(m[1]) * 60 + float(m[2]), m[3].strip()))
    return out


def beats(audio, cache='work/timing.json'):
    if os.path.exists(cache) and os.path.getmtime(cache) >= os.path.getmtime(audio): return json.load(open(cache))['beats']  # stale cache = wrong song
    import librosa
    y, sr = librosa.load(audio, sr=22050, mono=True)
    _, b = librosa.beat.beat_track(y=y, sr=sr, units='time')
    os.makedirs(os.path.dirname(cache) or '.', exist_ok=True)
    json.dump({'beats': [float(x) for x in b], 'duration': len(y) / sr}, open(cache, 'w'))
    return [float(x) for x in b]


def snapper(beat_list, tol=.22):
    def snap(t):
        b = min(beat_list, key=lambda x: abs(x - t)); return b if abs(b - t) <= tol else t
    return snap


# ---------------------------------------------------------------- drawing helpers (native pixels)
def gradient(im, box, stops, dither=False):
    """Vertical gradient. dither=False (default) gives clean bands — use it for anything seen at zoom 2."""
    x0, y0, x1, y1 = box; cols = [hx(c) for c in stops]; d = ImageDraw.Draw(im)
    B = [[0, 8, 2, 10], [12, 4, 14, 6], [3, 11, 1, 9], [15, 7, 13, 5]]
    for y in range(y0, y1):
        f = (y - y0) / max(1, y1 - y0 - 1) * (len(cols) - 1); i = min(int(f), len(cols) - 2); q = f - i
        if not dither: d.line((x0, y, x1 - 1, y), fill=cols[i + 1] if q > .5 else cols[i]); continue
        for x in range(x0, x1): im.putpixel((x, y), cols[i + 1] if q * 16 > B[y % 4][x % 4] + .5 else cols[i])


def light(im, cx, cy, r, strength=.3, col='#ffd99a'):
    """Warm light pool as 3 clean bands (never a dithered screen-door: it turns into a checkerboard at x12)."""
    px = im.load(); c = hx(col); W, H = im.size
    for y in range(max(0, int(cy - r)), min(H, int(cy + r))):
        for x in range(max(0, int(cx - r)), min(W, int(cx + r))):
            dd = math.hypot(x - cx, y - cy) / r
            if dd < 1:
                q = math.floor((1 - dd) ** 1.6 * strength * 3) / 3
                if q > 0: px[x, y] = lerp(px[x, y], c, min(q, .8))


def pixtext(im, xy, text, size, col):
    """Hard-edged CJK text at native resolution. Size 11 is the smallest that stays legible."""
    f = ImageFont.truetype(FONT, size); d0 = ImageDraw.Draw(im)
    m = Image.new('L', (int(d0.textlength(text, font=f)) + 4, size + 6), 0); d = ImageDraw.Draw(m)
    d.text((1, -size // 5), text, font=f, fill=255)
    im.paste(hx(col), (int(xy[0]), int(xy[1])), m.point(lambda v: 255 if v > 120 else 0))


def pixtext_c(im, y, text, size, col, shadow='#2b2233', cx=None):
    """Centred hard-edged text for title / credit / end cards, with a 1 px drop shadow so it reads on any background."""
    f = ImageFont.truetype(FONT, size); w = int(ImageDraw.Draw(im).textlength(text, font=f)); x = (im.width // 2 if cx is None else cx) - w // 2
    if shadow: pixtext(im, (x + 1, y + 1), text, size, shadow)
    pixtext(im, (x, y), text, size, col)


def loc(name):
    """@loc('candy_shop') tags a shot function with its location so shot_report() can show runtime per location."""
    def deco(fn): fn.loc = name; return fn
    return deco


def put(im, s, x, feet, mirror=False, clip=None):
    """Paste a 32x52 sprite so its shoe-bottom row (45) lands on y=feet, centred on x. clip=(x0,y0,x1,y1)."""
    if mirror: s = flip(s)
    px, py = int(round(x)) - 16, int(round(feet)) - 45
    cx0, cy0, cx1, cy1 = clip if clip else (0, 0, im.width - 1, im.height - 1)
    x0, y0, x1, y1 = max(px, cx0), max(py, cy0), min(px + s.width, cx1 + 1), min(py + s.height, cy1 + 1)
    if x1 > x0 and y1 > y0:
        part = s.crop((x0 - px, y0 - py, x1 - px, y1 - py)); im.paste(part, (x0, y0), part)


def shadow(im, x, feet, w=12): ImageDraw.Draw(im).ellipse((x - w // 2, feet - 1, x + w // 2, feet + 2), fill=hx('#4d3b3f'))


def ghost(im, draw_fn, alpha):
    """Translucent memory figures: draw_fn(layer) draws onto a copy, which is blended in."""
    if alpha <= 0: return im
    layer = im.copy(); draw_fn(layer); return Image.blend(im, layer, alpha)


# ---------------------------------------------------------------- cast registry + cached sprites
SPECS = {}


@functools.lru_cache(maxsize=4096)
def _spr(key):
    name, kw = key; kw = dict(kw)
    if kw.get('hands') is not None: kw['hands'] = dict(kw['hands'])
    ice = kw.pop('ice', False)
    if ice == 'both':
        a, b = icecream('l'), icecream('r'); kw['prop'] = lambda *z: (a(*z), b(*z))
    elif ice: kw['prop'] = icecream('l' if str(ice).endswith('_l') else 'r', melt=str(ice).startswith('melt'))
    return sprite(SPECS[name], **kw)


def spr(name, **kw):
    if kw.get('hands') is not None: kw['hands'] = tuple(sorted(kw['hands'].items()))
    if 't' in kw: kw['t'] = round(kw['t'] * 12) / 12
    return _spr((name, tuple(sorted(kw.items()))))


def head_top(name, feet): return feet - 45 + layout(SPECS[name])[2]


def side_hand(name, x, feet, off, mirror=False, seated=False):
    """World position of a side-view hand at offset `off` from the shoulder. seated=True adds the sit drop."""
    sp = SPECS[name]; hip, sh, ty = layout(sp)
    hx_, hy_ = CX - 1 + off[0], sh + 1 + off[1]
    if mirror: hx_ = 31 - hx_
    return x - 16 + hx_, feet - 45 + hy_ + ((sp['leg_h'] - 1) if seated else 0)


def front_hand(name, x, feet, side, off):
    sp = SPECS[name]; hip, sh, ty = layout(sp); tw, aw = sp['torso_w'], sp['arm_w']
    sgn = -1 if side == 'l' else 1
    sx = CX + sgn * (tw // 2 + 1 + (aw - 1) // 2) - (1 if side == 'l' and aw == 3 else 0)
    return x - 16 + sx + off[0], feet - 45 + sh + 1 + off[1]


# ---------------------------------------------------------------- free canvas with auto-outline (hand close-ups etc.)
class Canvas:
    def __init__(s, w, h): s.w, s.h = w, h; s.p = [[None] * w for _ in range(h)]
    def put(s, x, y, c):
        x, y = int(round(x)), int(round(y))
        if 0 <= x < s.w and 0 <= y < s.h: s.p[y][x] = c
    def get(s, x, y): return s.p[y][x] if 0 <= x < s.w and 0 <= y < s.h else None
    def rect(s, x, y, w, h, c):
        for yy in range(int(y), int(y + h)):
            for xx in range(int(x), int(x + w)): s.put(xx, yy, c)
    def ell(s, cx, cy, rx, ry, c):
        for y in range(max(0, int(cy - ry - 1)), min(s.h, int(cy + ry + 2))):
            for x in range(max(0, int(cx - rx - 1)), min(s.w, int(cx + rx + 2))):
                if ((x + .5 - cx) / rx) ** 2 + ((y + .5 - cy) / ry) ** 2 <= 1: s.put(x, y, c)
    def rrect(s, x, y, w, h, c, r=2):
        s.rect(x + r, y, w - 2 * r, h, c); s.rect(x, y + r, w, h - 2 * r, c)
        for cx, cy in ((x + r, y + r), (x + w - r - 1, y + r), (x + r, y + h - r - 1), (x + w - r - 1, y + h - r - 1)): s.ell(cx + .5, cy + .5, r + .4, r + .4, c)
    def line(s, a, b, c, w=1):
        (x0, y0), (x1, y1) = a, b; n = int(max(abs(x1 - x0), abs(y1 - y0))) + 1
        for i in range(n + 1):
            q = i / max(n, 1); s.rect(round(x0 + (x1 - x0) * q) - (w - 1) // 2, round(y0 + (y1 - y0) * q) - (w - 1) // 2, w, w, c)
    def outline(s):
        add = [(x, y) for y in range(s.h) for x in range(s.w) if s.p[y][x] is None and
               any(s.get(x + dx, y + dy) not in (None, 'OUT') for dx, dy in ((1, 0), (-1, 0), (0, 1), (0, -1)))]
        for x, y in add: s.p[y][x] = 'OUT'
    def image(s):
        im = Image.new('RGBA', (s.w, s.h), (0, 0, 0, 0)); px = im.load()
        for y in range(s.h):
            for x in range(s.w):
                c = s.p[y][x]
                if c is not None: px[x, y] = (OUTLINE if c == 'OUT' else hx(c)) + (255,)
        return im


def layer(im, cv):
    """Outline a canvas and paste it: draw each hand on its own canvas so overlaps keep an ink line."""
    cv.outline(); l = cv.image(); im.paste(l, (0, 0), l)


def _thumb_fist(cv, x, y, sk, sks, sl, sls, arm_from, flip_=False):
    def X(dx, w=1): return x + dx if not flip_ else x + 19 - dx - w + 1
    cv.line(arm_from, (X(2), y + 16), sl, 13); cv.line((arm_from[0], arm_from[1] + 3), (X(2), y + 19), sls, 6)
    cv.rect(X(-1, 4), y + 6, 4, 18, sls); cv.rrect(X(0, 20), y + 4, 20, 20, sk, 4); cv.rect(X(2, 14), y + 20, 14, 3, sks)
    for i in range(4):
        fy = y + 6 + i * 4; cv.rrect(X(13, 9), fy, 9, 4, sk, 1)
        if i: cv.rect(X(13, 8), fy, 8, 1, 'OUT')
    cv.rect(X(12, 1), y + 6, 1, 16, sks)
    bx, by, tx, ty = X(14), y + 5, X(22), y - 12
    cv.line((bx, by), (tx, ty + 3), sk, 7); cv.ell(tx + .5, ty + 2.5, 3.6, 3.6, sk)
    cv.rect(tx - 1 if not flip_ else tx, ty, 2, 3, '#f7dccb')


def thumbs_press(q=1.0, L=('#f0c3a2', '#d4977b', '#c8674a', '#a24f38'), R=('#e8b896', '#c98a6e', '#3f6fb5', '#2f5590')):
    """160x90 RGBA: two thumbs pressed tip to tip ('seal the promise'); q 0 = apart, 1 = touching.
    L/R = (skin, skin shade, sleeve, sleeve shade) — match each person's side in the previous wide shot."""
    gap = round(14 * (1 - q)); im = Image.new('RGBA', (160, 90), (0, 0, 0, 0))
    cr = Canvas(160, 90); _thumb_fist(cr, 85 + gap, 38, *R, (170, 100), flip_=True); layer(im, cr)
    cl = Canvas(160, 90); _thumb_fist(cl, 55 - gap, 38, *L, (-10, 100)); layer(im, cl)
    return im


def clasped_hands(left=('#a24f38', '#c8674a'), right=('#2f5590', '#3f6fb5')):
    """160x90 RGBA: holding hands — right person's fingers wrap the back of the left person's hand, left thumb on top."""
    im = Image.new('RGBA', (160, 90), (0, 0, 0, 0)); sk, sks, sk2, sk2s = '#f0c3a2', '#d4977b', '#e8b896', '#c98a6e'
    a = Canvas(160, 90); a.line((-10, 4), (56, 40), left[0], 15); a.line((-10, 0), (54, 36), left[1], 9); a.rect(50, 30, 6, 22, left[0])
    a.rrect(54, 30, 30, 26, sk, 6); a.rect(57, 51, 24, 3, sks); layer(im, a)
    b = Canvas(160, 90); b.line((170, 4), (110, 40), right[0], 15); b.line((170, 0), (112, 36), right[1], 9); b.rect(106, 30, 6, 24, right[0])
    b.rrect(84, 30, 24, 26, sk2, 6); b.rect(86, 51, 20, 3, sk2s)
    for i in range(4):
        fy = 32 + i * 6; b.rrect(62, fy, 28, 5, sk2, 2); b.rect(64, fy + 4, 22, 1, sk2s)
        if i: b.rect(62, fy, 26, 1, 'OUT')
    b.rect(89, 33, 1, 22, sk2s); layer(im, b)
    c = Canvas(160, 90); c.rrect(76, 24, 18, 8, sk, 3); c.rect(78, 30, 14, 1, sks); layer(im, c)
    return im


def rain(im, t, n=160, wind=4, col='#9fb3e0', shelter=None):
    """shelter=(x0, y_top, x1): no drops under an umbrella / jacket / roof."""
    d = ImageDraw.Draw(im); c = hx(col); W, H = im.size
    for i in range(n):
        x = (i * 37 + t * 60 * wind) % (W + 40) - 20; y = (i * 53 + t * 240) % (H + 20) - 10
        if shelter and shelter[0] - 4 <= x <= shelter[2] + 4 and y + 6 >= shelter[1]: continue
        d.line((round(x), round(y), round(x - wind), round(y + 6)), fill=c)


def polaroid(src, k=1):
    """White-bordered print of a native crop at integer scale k (shrink-into-album animations step 4->3->2->1)."""
    ph = src if k == 1 else src.resize((src.width * k, src.height * k), Image.NEAREST)
    b = 4 * k; card = Image.new('RGB', (ph.width + 2 * b, ph.height + 2 * b + 6 * k), hx('#fbf8f1')); card.paste(ph, (b, b))
    return card


# ---------------------------------------------------------------- captions, optional stamps, compositing, render
@functools.lru_cache(maxsize=64)
def caption_img(text):
    f = ImageFont.truetype(FONT, 44); tw = ImageDraw.Draw(Image.new('RGB', (1, 1))).textlength(text, font=f)
    note = text[:1] not in ('词', '曲', '编')
    w = int(tw + (140 if note else 60)); h = 78
    im = Image.new('RGBA', (w + 8, h + 8), (0, 0, 0, 0)); d = ImageDraw.Draw(im)
    d.rounded_rectangle((6, 6, w + 6, h + 6), 14, fill=(46, 30, 26, 200))
    d.rounded_rectangle((0, 0, w, h), 14, fill=(247, 237, 216, 255), outline=(138, 104, 72, 255), width=4)
    if note: d.text((30, 11), '♪', font=f, fill=(206, 72, 56)); d.text((92, 11), text, font=f, fill=(74, 52, 40))
    else: d.text((30, 11), text, font=f, fill=(74, 52, 40))
    return im


@functools.lru_cache(maxsize=64)
def stamp_img(text):
    """Optional orange film-camera date (off by default: shots pass stamp=None). The dark ink outline keeps it legible on warm backgrounds."""
    f = ImageFont.truetype(MONO, 40); im = Image.new('RGBA', (300, 64), (0, 0, 0, 0)); d = ImageDraw.Draw(im)
    for dx in (-3, -2, 0, 2, 3):
        for dy in (-3, -2, 0, 2, 3): d.text((12 + dx, 8 + dy), text, font=f, fill=(43, 34, 51, 235))
    d.text((12, 8), text, font=f, fill=(255, 168, 72, 255)); return im


def crop(world, cam):
    x0, y0, z = cam
    if z == 'hi': return world                       # 640x360 canvas, scaled x3 (album / photo shots)
    return world.crop((x0, y0, x0 + 320 // z, y0 + 180 // z))


class Film:
    def __init__(s, song, shots, caps, seg, fade_in=.7, fade_out=.65):
        """shots: list of (start, end, fn, stamp) — stamp is None, a string, or fn(t)->str|None.
        caps: list of (start, end, text). seg: (start, end) seconds of the song to render.
        fade_in / fade_out: seconds of dip-to-black at the ends (0 = none). Frame 0 is black when fade_in > 0, so make the
        cover with film.poster(), never from the first frame."""
        s.song, s.shots, s.caps, s.seg, s.fade_in, s.fade_out = song, sorted(shots, key=lambda r: r[0]), caps, seg, fade_in, fade_out

    def shot_at(s, t):
        for r in s.shots:
            if r[0] <= t < r[1]: return r
        return s.shots[-1]

    def frame(s, t):
        a, b, fn, st = s.shot_at(t)
        world, cam = fn(t)
        img = crop(world, cam).resize((OUT_W, OUT_H), Image.NEAREST).convert('RGBA')
        if callable(st): st = st(t)
        if st: S = stamp_img(st); img.alpha_composite(S, (OUT_W - S.width - 70, OUT_H - 120))
        for (c0, c1, text) in s.caps:
            if c0 <= t < c1: c = caption_img(text); img.alpha_composite(c, ((OUT_W - c.width) // 2, OUT_H - 128))
        img = img.convert('RGB')
        fi = sm(t, s.seg[0], s.seg[0] + s.fade_in) if s.fade_in > 0 else 1
        fo = 1 - sm(t, s.seg[1] - s.fade_out, s.seg[1] - .15) if s.fade_out > .2 else 1
        fade = min(fi, fo)
        return Image.blend(Image.new('RGB', img.size), img, max(fade, 0)) if fade < 1 else img

    def render(s, out, a=None, b=None, gain_db=-4.0):
        """Video + the song's own audio. gain_db: loud masters push AAC true-peak above 0 dBFS — keep headroom."""
        a = s.seg[0] if a is None else a; b = s.seg[1] if b is None else b; n = int(round((b - a) * FPS))
        cmd = ['ffmpeg', '-v', 'error', '-y', '-f', 'rawvideo', '-pix_fmt', 'rgb24', '-s', f'{OUT_W}x{OUT_H}', '-r', str(FPS), '-i', '-',
               '-ss', f'{a:.3f}', '-t', f'{b - a:.3f}', '-i', s.song,
               '-filter:a', f'volume={gain_db}dB' + (f',afade=t=in:st=0:d={s.fade_in * .85:.2f}' if s.fade_in > 0 else '')
               + (f',afade=t=out:st={b - a - s.fade_out - .05:.3f}:d={s.fade_out + .05:.2f}' if s.fade_out > .2 else ''),
               # -tune animation + crf 16: flat pixel art has hard edges that platforms smear on re-encode; spend the bits
               '-map', '0:v', '-map', '1:a', '-c:v', 'libx264', '-preset', 'medium', '-tune', 'animation', '-crf', '16',
               '-pix_fmt', 'yuv420p', '-movflags', '+faststart', '-c:a', 'aac', '-b:a', '256k', '-shortest', out]
        p = subprocess.Popen(cmd, stdin=subprocess.PIPE)
        for k in range(n):
            p.stdin.write(s.frame(a + k / FPS).tobytes())
            if k % 240 == 0: print(f'{k}/{n}', flush=True)
        p.stdin.close(); p.wait()

    def stills(s, out, times, cols=4):
        tiles = [s.frame(t).resize((480, 270), Image.LANCZOS) for t in times]
        rows = (len(tiles) + cols - 1) // cols
        sheet = Image.new('RGB', (cols * 486 + 6, rows * 276 + 6), (250, 247, 240))
        for k, im in enumerate(tiles): sheet.paste(im, (6 + (k % cols) * 486, 6 + (k // cols) * 276))
        sheet.save(out)

    def check_bounds(s, step=.25):
        """Every camera crop must stay inside its world image (out-of-bounds shows up as black bars)."""
        bad, t = [], s.seg[0]
        while t < s.seg[1]:
            a, b, fn, st = s.shot_at(t); w, (x0, y0, z) = fn(t)
            if z != 'hi' and (x0 < 0 or y0 < 0 or x0 + 320 // z > w.width or y0 + 180 // z > w.height): bad.append((round(t, 2), fn.__name__))
            t += step
        return bad

    # ------------------------------------------------------------ QA (run all of these before every delivery)
    def _view(s, t):
        a, b, fn, st = s.shot_at(t); world, cam = fn(t); return crop(world, cam).convert('RGB')

    def poster(s, out, t=None):
        """Cover / thumbnail frame with no fade (frame 0 of the file is black)."""
        s.frame(s.seg[0] + s.fade_in + .3 if t is None else t).save(out)

    def check_holes(s, step=1.0, min_px=6):
        """Pure black never appears in the art (outline is #2b2233; night sky should be e.g. #1a1424), so any #000 pixel is a hole:
        a gap between buildings, a layer that stops short of the camera edge, a pan that exposes the bare canvas.
        Returns [(t, shot_fn, n_pixels, bbox)]. Must be [] (intentional black cards: use #101018 instead)."""
        out, t = [], s.seg[0] + .05
        while t < s.seg[1]:
            r, g, b = s._view(t).split()
            m = ImageChops.lighter(ImageChops.lighter(r, g), b).point(lambda x: 255 if x == 0 else 0); n = m.histogram()[255]
            if n >= min_px: out.append((round(t, 2), s.shot_at(t)[2].__name__, n, m.getbbox()))
            t += step
        return out

    def sync_report(s, beat_list, tol=.08):
        """Are the shot starts on the beat grid? Returns dict(on_beat, total, pct, chance_pct, off=[(start, shot_fn, offset_s)]).
        pct should clearly beat chance_pct. Ballads often detect at double tempo, so also try beat_list[::2]."""
        bl = sorted(beat_list); gaps = sorted(y - x for x, y in zip(bl, bl[1:])); per = gaps[len(gaps) // 2]
        starts = [r for r in s.shots if r[0] > s.seg[0] + .5]; off, hit = [], 0
        for a, b, fn, st in starts:
            d = min(abs(a - x) for x in bl)
            if d <= tol: hit += 1
            else: off.append((round(a, 2), fn.__name__, round(d, 2)))
        return dict(on_beat=hit, total=len(starts), pct=round(100 * hit / max(1, len(starts))), chance_pct=round(100 * min(1, 2 * tol / per)), off=off)

    def shot_report(s, max_len=4.5, static_pct=1.0):
        """Pacing + variety. static = shots longer than max_len where under static_pct % of pixels change between start and end
        (add motion or cut); locations = (name, n_shots, seconds) sorted by runtime. Tag shots with @loc('name').
        Rule of thumb: >= 1 distinct location per ~25 s of song, no non-motif location over ~25 % of runtime."""
        static, share = [], {}
        for a, b, fn, st in s.shots:
            L = b - a; name = getattr(fn, 'loc', fn.__name__); e = share.setdefault(name, [0, 0.0]); e[0] += 1; e[1] += L
            if L > max_len:
                d = ImageChops.difference(s._view(a + .3), s._view(b - .3)).convert('L').point(lambda x: 255 if x > 8 else 0)
                ch = 100 * d.histogram()[255] / (d.width * d.height)
                if ch < static_pct: static.append((round(a, 2), round(L, 1), fn.__name__, round(ch, 2)))
        return dict(static=static, locations=sorted(((k, v[0], round(v[1], 1)) for k, v in share.items()), key=lambda r: -r[2]))
```

## Code: demo.py (smoke test, also a minimal example of a Film)

```python
from engine import *
from chars import SIS_KID, BRO_KID, SIS_ADULT, BRO_ADULT
SPECS.update(sis_kid=SIS_KID, bro_kid=BRO_KID, sis=SIS_ADULT, bro=BRO_ADULT)
L = load_lrc('song.lrc')
def bg():
    im = Image.new('RGB', (320, 180)); gradient(im, (0, 0, 320, 140), ['#5d4f86', '#c97f8f', '#f7c98a'])
    ImageDraw.Draw(im).rectangle((0, 140, 320, 180), fill=hx('#a89485')); light(im, 160, 60, 50, .3); pixtext(im, (130, 100), '早点铺', 11, '#2f5d55'); return im
BG = bg()
@loc('street')
def walk(t):
    im = BG.copy(); x = 100 + 15 * (t - 20.0); f = walkframe(t, 5)
    put(im, spr('bro', view='side', frame=f, walking=True, hands={'near': (-8, 8)}), x + 16, 150)
    put(im, spr('sis', view='side', frame=(f + 2) % 4, walking=True, expr='smile', hands={'far': (8, 4)}), x, 153)
    return im, (60, 80, 2)
@loc('close_up')
def promise(t):
    im = Image.new('RGB', (160, 90), hx('#4a3060')); h = thumbs_press(sm(t, 23.0, 23.5)); im.paste(h, (0, 0), h); return im, (0, 0, 2)
def end_card(t):                      # title / credit / end cards: centred pixtext on a real scene, never a pure-black card
    im = BG.copy(); pixtext_c(im, 78, 'MV 制作：你的名字', 12, '#fbf6ea'); return im, (0, 0, 1)
film = Film('song.mp3', [(20.0, 22.6, walk, None), (22.6, 24.5, promise, None), (24.5, 27.0, end_card, None)],
            [(L[3][0], L[4][0] - .2, L[3][1])], (20.0, 27.0))
print('bounds', film.check_bounds()); print('holes ', film.check_holes()); print('pacing', film.shot_report())
film.stills('demo.png', [20.5, 22.0, 23.2, 25.5]); film.poster('demo_poster.png', 21.0); film.render('demo.mp4')
```