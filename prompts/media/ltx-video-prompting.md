---
name: ltx-video-prompting
purpose: media
kind: reference
applies_to:
  - "LTX-2.3 (22B audio-video DiT, open weights, ~Mar 2026) — primary target of this guide"
  - "LTX-2.5 — prompting doctrine transfers unchanged; mechanical deltas in §9"
does_not_apply_to:
  - "LTX 0.9.x — most LTX material online is 0.9.x and does NOT transfer"
verified: 2026-08-20
tags: [video-generation, ltx, comfyui, image-to-video, first-last-frame, prompting]
---

# LTX-2.3 / 2.5 — usage, motion, and prompting

Everything needed to drive LTX-2.3 properly from a cold start. Compiled against Lightricks'
official prompt, adherence, and audio guides, the ComfyUI-LTXVideo node source, and the LTX-2.3
model card, cross-checked with community testing.

**Read the version tags.** LTX changed its hard constraints between lines. A rule written for
0.9.x will quietly produce broken output on 2.3, and a 2.3 resolution rule will silently corrupt
a 2.5 graph. Every constraint below is tagged with the version it holds for.

## Version matrix

| Topic | 0.9.x | 2.3 | 2.5 |
| --- | --- | --- | --- |
| Prompting doctrine (§4–§6) | does not transfer | authoritative | unchanged from 2.3 |
| Dimension divisor | 32 | **32** | **64** (see §9) |
| Frame count | 8n+1 | 8n+1 | 8n+1 |
| Frame rate | 30 common | **24 / 25 / 48 / 50, never 30** | same as 2.3 |
| 9:16 vertical | varies | **1088×1920** | **1088×1920** (round-trips exactly) |
| Audio | separate | joint with video | joint with video |
| Per-generation values | n/a | widgets fine | **must arrive over a link** (see §9) |

One structural fact that shapes everything else: LTX has no in-model multi-shot storyboard. One
generation is one continuous shot, with audio generated jointly. Continuity across shots comes from
keyframe conditioning plus stitching; Extend-Video reaches longer pieces.

---

## 1. The motion fix — read this first

**Symptom.** Image-to-video clips come out static. A slow zoom or pan over a frozen frame, nothing
alive.

**Root cause.** First-frame conditioning strength. The ComfyUI inject node
(`LTXVImgToVideoInplace` / `LTXVImgToVideoConditionOnly`) builds a noise mask of `1.0 − strength`
over the conditioned frames. At strength 0.9 the model gets only a 0.1 noise budget to evolve the
frame, so it can barely move anything. Strength pinned high "to lock the look" is the single most
common cause of dead clips.

Apply these top-down, one at a time, on a fixed seed.

| # | Lever | Node / widget | Static default → motion value | Trade-off |
| --- | --- | --- | --- | --- |
| 1 | **First-frame strength** | `LTXVImgToVideoInplace` / `…ConditionOnly` · `strength` | **0.9 → 0.7** (0.65 for max motion) | Lower means more drift from the exact input frame. Keep the upscale / stage-2 re-inject at 1.0; it does not kill motion, which is already in the latent. |
| 2 | **Prompt the motion** | positive text encode | Name the action, ONE camera move, and an end-state; layer ambient motion | None. See §4. |
| 3 | **CRF / image compression** | `LTXVPreprocess` · `img_compression` | **18–25 → 28–35** (higher means more motion) | Softer, lower-fidelity first frame, slight artifacts. |
| 4 | **Frame rate (perceived)** | render fps + `CreateVideo` | Render **48–50**, then retime to 24/25 | More VRAM and time. Makes the image-to-motion transition less jarring. |
| 5 | **FETA enhance** | `LTXFetaEnhanceNode` · `feta_weight` | **4 → 6** | Too high causes temporal warble. |
| 6 | **Frame-rate conditioning** | `LTXVConditioning` · `frame_rate` | Declare a *lower* fps than real for bigger per-frame motion | A big mismatch causes judder. |
| 7 | **CFG** | `CFGGuider` / `STGGuider` · `cfg` | Distilled **1.0**; dev i2v **~3.0** | High CFG gives stiff, robotic motion and oversaturation. It is NOT sharper. |
| 8 | **Blur kernel** | `…Advanced` · blur radius | 0 → 1–3 | Softer anchor. |
| 9 | **Scheduler shift** | `LTXVScheduler` · `max_shift` | 2.05 → nudge up if still static | Aggressive values cause incoherence. |

A working baseline for lively but controlled motion: strength 0.7 for image-to-video and 0.7 for
first-last-frame guides, CFG 1.0 on distilled, CRF left at default unless the frame is flat, 24fps
or 48 retimed, and a prompt carrying one named camera move, an end-state, and two or three layered
ambient motions.

Change one lever per test. Blame the strength first and the prompt second.

---

## 2. What it is, and how to run it

LTX-2.3 is a 22B audio-video diffusion transformer, up from 19B in LTX-2. It generates video and
synchronized stereo audio jointly in one pass. Speech, foley, and music share roughly 48 transformer
blocks through cross-modal attention, with audio decoded by an audio VAE into a HiFi-GAN vocoder at
24 kHz. Modes are image-to-video, text-to-video, audio-to-video, keyframe interpolation, lip-dub,
and video-to-video. The text encoder is Gemma 3 12B.

What 2.3 changed over LTX-2: a rebuilt high-fidelity VAE for sharper edges, hair, and fabric; a text
connector four times larger, which markedly improves adherence on complex, spatial, and
multi-subject prompts; native 9:16 portrait trained on portrait rather than crops; cleaner audio;
and reduced texture drift on longer shots. There is no public 2.1 or 2.2. The jump is LTX-2 to
LTX-2.3.

### Node map (ComfyUI)

Install the core plus the official `Lightricks/ComfyUI-LTXVideo` pack. The official 2.3 templates
live under `example_workflows/2.3/`.

| Role | Node |
| --- | --- |
| Image-to-video inject (first frame) | `LTXVImgToVideoInplace` / `LTXVImgToVideoConditionOnly` — `strength` is the motion lever |
| Keyframe / first-last / mid | `LTXVAddGuide` — image at any `frame_idx`, the main directing tool |
| Preprocess (CRF and blur motion knobs) | `LTXVPreprocess` |
| Checkpoint | `CheckpointLoaderSimple` |
| Text encode | `LTXAVTextEncoderLoader` (Gemma 3) → `CLIPTextEncode` → `LTXVConditioning` (binds text plus fps) |
| Sampler | `SamplerCustomAdvanced` ← `KSamplerSelect` (`euler` / `euler_ancestral_cfg_pp`) + `ManualSigmas` or `LTXVScheduler` + `CFGGuider` |
| Empty latent | `EmptyLTXVLatentVideo` |
| Audio | `LTXVEmptyLatentAudio` → `LTXVConcatAVLatent` → sample → `LTXVSeparateAVLatent` → `LTXVAudioVAEDecode` |
| IC-LoRA control | `LTXICLoRALoaderModelOnly` + `LTXAddVideoICLoRAGuide` — applies a reference downscale, NOT the normal LoRA loader |
| Upscale | `LTXVLatentUpsampler` (spatial ×2) plus the temporal upscaler |
| Output | `VAEDecodeTiled` → `CreateVideo` → `SaveVideo` |

There is no `generate_audio` checkbox in ComfyUI. Audio is on whenever the audio node chain is
wired, so drop the decode and save branch for silent output. Hosted API endpoints do expose a
`generate_audio` boolean.

### Weights, quantization, VRAM

The Hugging Face repo is `Lightricks/LTX-2.3`. The distilled 8-step checkpoint at CFG 1 is the
practical default. A distilled LoRA variant is required for the two-stage pipeline, alongside the
spatial and temporal ×2 upscalers. Quantized repos cover fp8 and nvfp4 for RTX 50-series, plus
community GGUF builds.

Budget the Gemma 3 12B text encoder on top of the transformer or offload it.

| Configuration | VRAM |
| --- | --- |
| Full bf16 | 32 GB and up |
| fp8 dev | ~24 GB |
| Distilled / fp8 / GGUF-Q4 | 12–16 GB with VAE tiling and offload |

Tiling is usually required to avoid running out of memory. The environment wants Python 3.12 or
newer, CUDA above 12.7, and PyTorch around 2.7.

### Sampler settings

| | Distilled (recommended default) | Full dev |
| --- | --- | --- |
| Steps | **8** in stage one, plus 3–4 in the stage-two upscale, on baked sigmas | 20–50, with 25–35 the sweet spot; 60–80 for smoother image-to-video |
| CFG | **1.0**, with CFG and STG off, intrinsic to distilled | **~3.0** for image-to-video, range 2–5; text-to-video tolerates more |
| Sampler | `euler` / `euler_ancestral_cfg_pp` | Same; the high-quality path uses `res_2s` |
| Scheduler | `ManualSigmas` on the LTX list, or `LTXVScheduler` | Same |

Only the distilled 8-steps-at-CFG-1 figure is verbatim official. The dev step and CFG numbers are
community guidance.

**Use the two-stage pipeline.** Run stage one at half resolution with the image injected at strength
0.7, spatially upscale ×2, then refine in stage two for about three steps with the image re-injected
at 1.0. The two-stage spatial and temporal upscale is the single biggest perceived-quality win, and
the LTX temporal upscaler beats RIFE for smoothness.

---

## 3. Hard constraints (LTX-2.3)

These are official and unforgiving.

- Width and height must be divisible by **32**. On 2.5 this becomes 64, for the reason in §9.
- Frame count must be **8n+1**, so 9, 17, 97, 121, 257, and so on. Duration is (frames − 1) ÷ fps.
- 9:16 vertical is **1088×1920**, not 1080×1920. 1080 is not divisible by 32, while 1088 is 32×34
  and 1920 is 32×60. The low-VRAM fallback is 768×1280. In image-to-video the aspect ratio is
  inherited from the source image, so author keyframes at 1088×1920.
- Frame rate must be **24, 25, 48, or 50**. Not 30, which was the 0.9.x convention. Rendering at 48
  gives smoothness without the soap-opera look.
- A single generation stays coherent to roughly 10–15 seconds, with a stable base around 121 frames.
  Quality drifts past that, so switch to Extend-Video.
- 4K, 50fps, and 20-second outputs are hosted-tier ceilings. Local open weights realistically reach
  720p–1080p base plus upscale.

---

## 4. Prompting recipe

Write one compact, flowing, present-tense paragraph.

For image-to-video and first-last-frame work, the conditioning images already own the look, layout,
and style. The positive prompt should mostly describe **what changes**. Use enough nouns to identify
the visible scene, then spend the rest on physical motion, one camera move, the endpoint, ambient
motion, and diegetic sound.

Word count is a ceiling, not a quota.

| Clip length | Target words |
| --- | --- |
| 2–3 seconds | 25–55 |
| 4–6 seconds | 45–85 |
| 7–10 seconds | 70–120 |
| Any | never above 180 without splitting the beat |

The official 2.3 adherence order is: main action, then chronological mechanics, then only the
necessary character and environment locks, then explicit camera behavior and its landing, then
audio. For image-conditioned clips, omit camera and lighting restatement when the frame already
owns it.

### Every generation is contextless

At prompt time a generation sees only its own conditioning frames and its own prompt text. It does
not see the previous video, the previous prompt, or any workflow metadata. So every clip prompt must
stand alone, describing the visible start state, the target state, the motion, and the sound.

Never write continuity shortcuts such as "continue the previous shot", "as before", or "same as the
last clip". Never include frame identifiers, file names, node names, upstream model names, or other
pipeline labels in the text sent to the model.

### Relevant detail is rewarded, padding is not

The larger text connector holds spatial and action detail far better than 0.9.x did, but boilerplate
still competes with motion for the model's budget. Keep details the model can act on: visible object
relationships, left and right, foreground and background, material, action mechanics, endpoint, and
sound.

Cut audit filler such as "preserve the exact composition", "approved frame", "grounded photoreal
texture", and repeated style blocks when the image already supplies the look. Surgical means precise,
not vague.

**First-last-frame especially: do not re-itemize every component.** Both keyframes already pin the
look, layout, geometry, and palette. Spend the words on what changes between them.

### Lead with the action

Use present-tense physical-mechanics verbs. The official example is instructive: "she shifts her
weight to her left foot and turns her head slowly toward the camera" works, while "she moves
gracefully" does not. Express speed in natural language such as "slowly", "suddenly", or "drifts",
never in numbers. Motion is driven by verbs, and a still-photo-style prompt yields a near-static
clip.

### One named camera move, with an end-state

Vocabulary the model knows: dolly in and out, push in, pull back, pan or dolly or track left and
right, tilt up and down, jib up and down, crane up and back, arc or orbit, handheld, "slow rising
drone-like camera", static or locked-off, plus shot scales from wide establishing through medium and
close to extreme macro.

Always give the move a destination, such as "settling on a close-up of the flame". Skipping the
endpoint is a leading cause of weak, unfinished motion.

Never combine two camera moves. Pan plus zoom plus handheld produces mushy motion and artifacts.

Map unsupported terms onto known ones: truck becomes dolly left or right, pedestal or boom becomes
jib, drone becomes "slow rising drone-like camera". A bare scale-zoom just inflates the subject over
a static background, so name a real move and give the camera a relationship to the subject.

### When the subject does not act, put the motion in the environment

For still-life and object-centric shots, layer two or three continuous ambient motions so the frame
is alive rather than frozen under a zoom: flame flicker and lean, smoke or vapour curling and
rising, dust motes drifting in a light beam, water beading and sliding, cloth or page edges
stirring, shifting shadows, a travelling highlight, light pulsing.

Use restrained descriptors such as "subtle", "gentle", and "gradual" to keep the mood slow without
going dead.

Then sequence the move across the clip with a soft closing beat: "the camera holds, then begins a
slow dolly-in as the vapour thickens, and settles as the flame steadies." Combined with ambient
layering, this kills the dead time where the model finishes early and drifts.

### Anti-patterns

Vague motion such as "make it dynamic". Numeric camera specifications. Past tense or imperatives.
Contradictions such as "still lake with crashing waves". Many simultaneous subjects. Emotion labels
standing in for motion. Complex or chaotic physics such as juggling. Readable in-frame text or
logos.

### Keep negatives out of the positive prompt

This one is counterintuitive and cost real render time to learn. Phrases like "no jumps", "no
fades", "no music", "do not tilt down", "without people", or "no writing appears" can make the model
attend to the forbidden noun or action and produce exactly that behavior.

The positive prompt should describe only the desired shot physics: "one continuous take", "stable
camera", "visible from the first instant", "period objects only", "the camera settles on...".

### Split the negative prompt

Keep shared base negatives for quality, style, and audio, plus a short per-generation negative for
that clip's likely failure modes. Do not promote a scene-specific control to the global list unless
every clip needs it. Long negative inventories pull attention toward the forbidden objects and
degrade motion.

| Layer | Target size | Example |
| --- | --- | --- |
| Base | 8–12 generic tokens | `blurry, warped geometry, jitter, watermark, readable text, subtitles, logo, CGI, modern objects` |
| Audio base | 3 tokens | `music, soundtrack, score` |
| Per-generation | 3–5 local failures | `malformed fingers, vertical screwdriver, pierced dome, explosion` |

---

## 5. Control and integration

In priority order.

1. **Keyframes via `LTXVAddGuide`.** Native, highest leverage, nothing to download. It VAE-encodes a
   guide image into the conditioning and latent at an arbitrary `frame_idx`, where negative counts
   from the end. For guides of nine frames or more, `frame_idx` must be divisible by 8, and guide
   length follows 8n+1. Strength runs 0 to 1. Chain multiple guide nodes, each taking the previous
   node's positive, negative, and latent, to storyboard a single clip with three or more keyframes
   while the model interpolates between them. This is how first-last-frame and first-mid-last-frame
   work. A reasonable starting set is first at 0.8–1.0, last at 0.6–0.9, mid around 0.5.
2. **IC-LoRA Union Control**, the closest thing to ControlNet for LTX. Depth, canny, and pose in one,
   driving geometry and motion from a reference video and locking objects against deformation. Wire
   `LTXICLoRALoaderModelOnly` into `LTXAddVideoICLoRAGuide`. When using a control IC-LoRA, drop
   motion words from the prompt and keep only style, light, and material. Motion-Track handles sparse
   point trajectories for choreographed object paths, and a community Cameraman IC-LoRA transfers a
   camera move from a three-second reference clip at strength 0.7–1.0.
3. **Per-move camera LoRAs.** These are 19B and only partially compatible with 2.3. Dolly works,
   lateral moves are often ignored, and dropping CFG to about 2.0 helps. Treat them as a fallback and
   prefer prompt plus strength, or the Cameraman IC-LoRA.
4. **Extend-Video.** A single generation reaches roughly 10–15 seconds. For longer pieces, generate a
   clean base and seed the next window from the last two or three seconds as tail context, masking to
   anchor appearance and audio. Cheaper and more coherent than two separate clips, though quality
   degrades over many cycles.
5. **Two-stage spatial and temporal upscale.** Always do this for finals. Biggest polish per unit
   cost.
6. **Video-to-video and motion transfer.** The same IC-LoRA machinery with a reference video. Keep
   the image-to-video anchor image and feed the motion source through a preprocessor into
   `LTXAddVideoICLoRAGuide`.

---

## 6. Audio

LTX generates speech, foley, and music jointly. In joint generation there is no hard switch to
disable music, so sound-effects-only output is steerable but not guaranteed. Verify per clip.

- Describe only diegetic and ambient sound, woven into the prompt as prose rather than under a
  separate "Audio:" label. Be specific about tone, intensity, and acoustic space. "The only sound is
  a hushed reverberant room tone and the faint hiss of rising vapour" beats "room tone, hiss." Put
  the sound in the same sentence as the visual event it belongs to.
- Do not write "no music" in the positive prompt. It belongs in the negative.
- Negative candidates: `music, soundtrack, score, background music, melody, instrumental`. Plausibly
  helps, though suppression is not officially confirmed.
- `modality_scale` around 3.0 improves audio-video sync, and 1.0 disables it. Audio tolerates a
  higher CFG than video.
- Official caveat: audio without speech may be lower quality. Sound-effects-only output is exactly
  that case, so budget for candidate selection and prefer the distilled checkpoint. If you need a
  hard guarantee, build the audio bed externally and treat LTX audio as ambience texture underneath.

---

## 7. Limitations and failure modes

Residual Ken-Burns drift in image-to-video when no move is named, fixed by naming a move or raising
strength. Temporal drift, morphing, and warble accumulating past roughly four seconds and on long
orbits, fixed with keyframes, IC-LoRA, splitting, or Extend. Warping and teeth artifacts on fine
detail. Identity drift from over-long descriptors, fixed by shortening and locking anchors. A pull
from stylized toward realism on contrasty prompts. Unreliable in-frame text and lip-sync.
Close-enough-but-not-exact object counts. High CFG stiffening motion, which is counterintuitive.
Lower audio quality when there is no speech.

---

## 8. Prompt-authoring boundary

Do not author final model text from inside a long production conversation. LTX gives every supplied
word influence over the generated result, so inherited creative context becomes accidental
conditioning.

Author each generation in a fresh context containing only the mode and duration, that generation's
named base frame and optional true same-scene end frame, one literal action beat, the audio policy,
and only the necessary end-state, continuity locks, and local failure risks. Return exactly the
motion text, the sound text, and the negative. Start a separate context for each independent
image-to-video generation; a first-last-frame transition is one generation and gets both endpoints
in one context.

---

## 9. LTX-2.5 deltas

The prompting doctrine above transfers to 2.5 unchanged. The differences are mechanical, they live
in how the graph is wired, and they are unforgiving. Read all three before touching a 2.5 graph.

**Resolution must be divisible by 64.** The 2.5 image-to-video subgraph samples the latent at half
the requested size and then runs a ×2 latent upsampler, and LTX packs latents in 32-pixel blocks. So
each axis is halved and then floored to a multiple of 32. 1088×1920 round-trips exactly. 1280×720
silently becomes 1280×704. A "corrected" true-9:16 1080×1920 silently becomes 1024×1920. Have your
workflow builder reject anything not divisible by 64.

**Per-generation values must arrive over a link, not a widget.** A subgraph definition is shared by
every instance, so a value pinned inside it cannot vary per segment. An instance's `widgets_values`
do not survive the UI-to-API conversion either, because the converter emits the internal node's own
default. Only a real link from a top-level node carries a per-segment value through. Emit explicit
per-segment seed and duration nodes and link them in.

**Nothing is proven until the output is measured.** Both failures above are invisible in the spec,
in the instance widgets, and in a completion check. Two real batches make the point. In one, a
490-clip run rendered landscape while every visible layer read 1088×1920. In another, all 490
segments shared a single noise seed. Assert the emitted graph, the converted prompt, and the
rendered file itself.

---

## 10. Licensing

LTX-2 ships under the **LTX-2 Community License**, which is custom and neither Apache nor OpenRail.
OpenRail covered the 0.9.x line only.

Use is free for entities under 10 million dollars in annual revenue. At or above that threshold a
paid commercial license is required. Outputs are yours. Acceptable-use restrictions cover
impersonation and deepfakes, malware, and training competing models, among others.

Confirm which entity's revenue governs your use before relying on this, and read the full license
text. This is not legal advice.

---

## Sources

**Official.** [LTX-2 GitHub](https://github.com/Lightricks/LTX-2) ·
[ComfyUI-LTXVideo](https://github.com/Lightricks/ComfyUI-LTXVideo), where `latents.py`, `guide.py`,
and `tricks/` carry the strength-mask and CRF, blur, and FETA tooltips ·
[LTX-2.3 on Hugging Face](https://huggingface.co/Lightricks/LTX-2.3) ·
[image-to-video docs](https://docs.ltx.video/open-source-model/usage-guides/image-to-video), source
of "0.7 leaves room for natural motion" ·
[2.3 prompt guide](https://ltx.io/blog/ltx-2-3-prompt-guide) ·
[prompt adherence](https://ltx.io/blog/how-to-improve-ltx-2-3-prompt-adherence) ·
[negative prompts](https://ltx.io/blog/negative-prompts) ·
[audio guide](https://ltx.io/blog/a-guide-to-ltx-2-3-audio) ·
[reducing warble and artifacts](https://ltx.io/model/model-blog/how-to-reduce-warble-and-ai-pattern-artifacts-in-ltx-2) ·
[license](https://huggingface.co/Lightricks/LTX-2.3/blob/main/LICENSE).

**Control models.** [AddGuide docs](https://docs.comfy.org/built-in-nodes/LTXVAddGuide) ·
[Union Control IC-LoRA](https://huggingface.co/Lightricks/LTX-2.3-22b-IC-LoRA-Union-Control) ·
[Motion-Track](https://huggingface.co/Lightricks/LTX-2.3-22b-IC-LoRA-Motion-Track-Control) ·
[Cameraman IC-LoRA](https://huggingface.co/Cseti/LTX2.3-22B_IC-LoRA-Cameraman_v1) ·
[LatentUpsampler](https://docs.comfy.org/built-in-nodes/LTXVLatentUpsampler).

**Practical, cross-checked.** [CrePal](https://crepal.ai/blog/aivideo/blog-ltx-2-prompting-guide/) ·
[film.fun](https://www.film.fun/articles/ltx-2-prompting-guide) ·
[PixelDojo](https://pixeldojo.ai/guides/ltx-2-prompting-guide) ·
[WaveSpeed 9:16 workflow](https://wavespeed.ai/blog/posts/ltx-2-3-portrait-video-9-16-workflow-2026/) ·
[awesome-ltx2](https://github.com/wildminder/awesome-ltx2).
