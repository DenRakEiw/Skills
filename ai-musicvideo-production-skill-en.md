---
name: ai-musicvideo-production
description: Cut, grade and release AI-generated music videos using DaVinci Resolve over MCP, ffmpeg, and image/video generators (Flux, LTX, ComfyUI). Use whenever the user works on a music video edit, beat-synced cutting, lip-sync correction, grading of AI footage, thumbnails, or release copy.
---

# AI Music Video Production – Pipeline Skill

Hard-won practice from two finished productions. Every item below cost at least one
failed attempt. **The order of operations is part of the instruction, not just
structure** – getting it wrong is the single most expensive mistake in this work.

**Core principle:** AI footage is not film footage. It is already sharp, contrasty,
saturated and clean, and it lies about its own motion. Everything here – the grading
doctrine, the cutting doctrine, the verification discipline – follows from that one
fact. Treat it like film and you will fight it the whole way.

---

## 0. Order of Operations

1. **Audio first.** Derive the beat grid from the master before touching a single
   clip. Without a grid every cut is a guess.
2. **Measure lip-sync while there is still time to fix it.** Not at the end. If the
   measurement arrives after the edit is locked, the only remaining lever is keeping
   the weak inserts short.
3. **Write the cut plan as a Python file**, not by hand in the timeline. It will be
   revised five to ten times; it has to be reproducible.
4. **Bake elements with ffmpeg** (camera moves, wipes, effects), because Resolve over
   MCP has no keyframe API.
5. **Script the timeline**, render, verify.
6. **Finishing pass** (halation, grain) as a separate ffmpeg run over the master.

---

## 1. Beat Grid

Determine BPM and offset with librosa, then compute everything in **frames**, never
in seconds. At 144 BPM / 24 fps: bar = 40 frames, beat = 10, half-bar = 20.

```python
def b(n, beat=0):
    return OFFSET_FRAMES + (n-1)*FRAMES_PER_BAR + beat*FRAMES_PER_BEAT
```

Derive the song structure per bar from RMS plus low-band and high-band energy
(drop, break, bridge, total drop-out). **The loudest bar in the analysis is almost
always where the strongest image belongs** – find it before choosing shots.

---

## 2. DaVinci Resolve over MCP – Traps

- `AppendToTimeline`: **`endFrame` is exclusive.** Use `endFrame = startFrame + length`.
  Otherwise you silently get a one-frame gap at every cut.
- Timeline start frame is **86400**, not 0.
- **No keyframe API.** `SetProperty("ZoomX"/"Pan"/"Tilt")` only applies statically.
  Any movement must be baked in ffmpeg (Section 3).
- `SetCDL({NodeIndex, Slope, Offset, Power, Saturation})` and
  `GetNodeGraph().SetLUT(1, "MCP/name.cube")` work reliably.
- `generate_lut` writes to `…/Support/LUT/MCP`, referenced as `MCP/name.cube`.
- **Resolve will not import FLAC.** Transcode to 48 kHz `pcm_s24le` with ffmpeg first.
- `ExportCurrentFrameAsStill` can hang the MCP helper for minutes. Grab frames from
  the rendered file with ffmpeg instead.
- Container egress is often blocked (cloud storage, ComfyUI links). Run downloads on
  the user's own machine via the unsafe-script escape hatch.
- **File-commit tools may serve cached copies** when you write repeatedly to the same
  source path. Copy to a **new filename** before each commit (`be3.py` → `be4.py`).
  Otherwise you will debug the old version for hours.

---

## 3. ffmpeg – Concrete Traps

- **`geq` uses `N`** for the frame number, not `n`.
- **`rgbashift` rejects expressions** for `rh`/`bh`.
- **`gblur` rejects an expression for `sigma`.**
- **`boxblur` with an animated `luma_radius` plus `enable` fails**
  ("streams received no packets"). Stack fixed radii instead:
  `,boxblur=luma_radius=R:enable='eq(n,I)'` per frame.
- **`drawtext` `line_spacing` adds to the natural line height.** For credit blocks,
  draw **one `drawtext` per row at an explicit y position**. Multi-line text files
  will overlap your footer.
- Crop expressions must yield even dimensions: `w='2*floor(iw/Z/2)'`.
- Escape commas inside filtergraph expressions: `max(0\,1-n/4)`.

### Baking a camera move

```
crop=w='2*floor(iw/Z/2)':h='2*floor(ih/Z/2)':x='(iw-out_w)*PX':y='(ih-out_h)*PY',
scale=1920:1080:flags=bicubic
```

Z, PX and PY are expressions in `n`. **Make amplitude inversely proportional to shot
length** – a five-second hold should creep, not zoom.

### Masked wipes

```
color=black,format=gray,geq=lum='<shape expr>'  →  alphamerge onto B  →  overlay onto A
```

Procedural edges read well: diagonal with a sine disturbance, slats, iris, chevron.
**Object rotoscoping on AI material does not work** – the result is double-exposure mush.

---

## 4. Generation

### LTX lip-sync: the keyframe does the work, not the prompt

This is counter-intuitive and was learned the hard way, after a full set of
carefully written per-shot prompts was generated and then **thrown away unused**.

- **Control comes from the input image.** The keyframe must be a **close-up with the
  subject looking into the camera**. Wide shots, three-quarter angles and averted
  gaze fail no matter how the prompt is written. Fix the framing before touching the
  text.
- **Short prompts beat long ones.** Elaborate, scene-specific prompts did not work.
  What worked was **one short universal prompt reused across every single clip**:

  ```
  character is aggressively rapping and moving lips in perfect sync with the fast audio beat
  ```

  Name the action, the energy and the sync – nothing else. Do not write bespoke
  prompts per shot; vary the **keyframe** instead and leave the text alone.
- **Keep input images small** (~640×480, max 768×432). High-resolution start images
  make the model output *a static image with a slight zoom* instead of motion. Along
  with a non-close-up keyframe, this is the most common cause of "the lip-sync isn't
  working".
- LTX clips are truncated at **6.0 s** but are frame-accurate.

**Budget implication:** spend the effort on generating good close-up keyframes, not
on prompt engineering. Prompt work here has close to zero marginal return.

### Image generation
- `flux-2-pro` **ignores `aspect_ratio`** and returns 1024×768 – centre-crop to 16:9
  yourself.
- `nano-banana-pro` accepts **only one image**. For two inputs (tight face reference
  plus wide keyframe) use `flux-2-pro` with `role:"image"` plus `role:"reference_image"`.

---

## 5. Measuring Lip-Sync

Mouth aperture as the dark-pixel fraction inside an ROI, cross-correlated against an
isolated vocal proxy (librosa HPSS harmonic + 300–3800 Hz bandpass).

**The ROI is the entire difficulty.** A Haar face box lands on the neck; temporal
variance lands on the hair. What works is **eye-anchored geometry**: mouth ≈ eye
midpoint + 1.05 × inter-ocular distance along the face axis.

**Always render a control image of the ROI and look at it before trusting any
numbers.** Two full measurement passes were worthless because the ROI sat wrong.

Realistic values: r ≈ 0.2–0.5. Below 0.2 the clip is unusable as a long insert – keep
it to ≤16 frames or drop it. A systematic picture lead of 3–8 frames is normal and is
corrected globally.

**Be honest about the ceiling.** Current lip-sync models produce *talking motion*, not
phoneme accuracy. Say so, and let it change the edit (shorter inserts, more cutaways)
instead of pretending it can be fixed in post – or by rewriting the prompt, which is
where effort tends to get wasted (Section 4).

---

## 6. Grading AI Footage

**Subtractive, not additive.** The film-grading reflex – add contrast, raise
saturation, drop a look on top – destroys AI material.

- Flatten first, then lift crushed shadows, then roll off clipped highlights.
- Saturation slightly **below** neutral (0.96–0.98).
- Let a split tone carry the look, not contrast.
- **The black-floor window is narrow:** ~0.030 is right. 0.00 crushes; 0.068 goes
  milky and magenta. Both errors were made and had to be undone.
- More important than any "look": **matching clips from different generators to each
  other.** An audience forgives a flat grade; it does not forgive two shots that
  clearly came from two different models.
- Grain (`noise=alls=5:allf=t+u`) and halation break the digital cleanliness.
  Halation: isolate highlights via `lutyuv`, `gblur=sigma=16`, `blend=screen:0.14`.
- **Never apply a treatment to ungraded source and then apply the LUT on top again.**
  That is exactly how a grading mismatch appears on freeze frames and inserts.

---

## 7. Cutting Doctrine

Derived from direct user feedback on finished cuts, not from theory. Treat as strong
defaults, not law.

- **Choreography is the spine; close-ups are accents** – at most one or two per
  eight-bar section, never back to back.
- **Dance shots need long holds** (three bars), or the choreography does not read.
- **Aggression comes from beat-level cuts between scene shots**, not from cutting to
  a face.
- **Freeze frames brake the flow.** Use scale kicks instead.
- **Static shots at uniform size read as boring** – bake camera moves on the long
  shots and give the short ones a static size rhythm.
- **Syncopate bursts**: 10/10/5/5/10 frames rather than an even run.
- **Keep source motion continuous**: track where each clip left off and resume there
  instead of restarting at 0; wrap rather than overrun.
- A lyric that belongs on a face belongs in a close-up. Users notice this instantly
  and will name the exact second.

---

## 8. Verification – Not Optional

Every check below caught a real error at least once:

- **Audio cross-correlation of every render against the master** at three points.
  Expect +0.0000 s.
- **Assert V1 is gapless** (`start == previous_end`).
- **Assert no source overrun** (`src + len <= media_len`).
- **Check overlaps** on the effect tracks.
- **Look at a contact sheet before accepting any effect.** Never trust a filter whose
  output you have not seen.

---

## 9. Release Assets

**Thumbnail:** a large face, one iconic object, strong colour contrast. Find
candidates from a contact sheet of the finished film rather than guessing. **Scale to
210 px and check** – that is where it is actually decided. Do not crop symmetrical
compositions; place the full frame in a 4:5 poster with a type block below it.

**YouTube:** keep a non-English title in its own language and put the translation
underneath; translating the title makes it read like a cover version. Tags: 500
characters max, specific ones first, tool tags next (they bring the most traffic on
AI videos). **Never tag other artists' names** – "inspired by" belongs in the
description.

**Instagram:** 4:5 for the feed, 9:16 with a blurred fill for stories, keep the bottom
~420 px clear for the link sticker.

---

## 10. Working With the User

- Feedback arrives short and blunt and is **always a diagnosis, never a mood**.
  "Feels boring" means structure, not grading. "Too many close-ups" means the
  choreography is not landing.
- When they name a timecode, it is correct. Go look there.
- **Deliver a 720p preview, not the 600 MB master.**
- State limits plainly. A model ceiling described honestly and worked around is worth
  more than an optimistic claim that the next pass will fix it.
