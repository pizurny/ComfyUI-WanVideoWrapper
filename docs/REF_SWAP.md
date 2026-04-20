# WanAnimate Reference Image Swap

Custom extensions that let you change the WanAnimate reference image mid-generation. Useful for multi-character videos, identity transitions, and any workflow where one ref image can't cover the whole shot.

These nodes are additive to the existing `ComfyUI-WanVideoWrapper` stack — existing WanAnimate workflows without swap nodes are unaffected.

---

## Quick start

Typical node chain:

```
WanVideoVAELoader ──► WanVideoAnimateEmbeds ──► WanVideoAnimateRefSwap ──► WanVideoSampler
                                                    │
                           (optional: chain more ─► WanVideoAnimateRefSwap ─► ...)
```

For a 120-frame shot where you want the character to change at frame 100:

1. `WanVideoAnimateEmbeds`: `num_frames=120`, `frame_window_size=77`, ref image = character A.
2. `WanVideoAnimateRefSwap`: `swap_frame=100`, `ref_image=` character B, `reset_temporal=False`.
3. Sampler runs 3 windows: `[0, 77)` character A, `[77, 100)` character A (short window), `[100, 120)` character B.

For a 60-frame shot with a swap at frame 30 (short video):

1. `WanVideoAnimateEmbeds`: `num_frames=60`, **`force_looping=True`** (required — see below), ref image = character A.
2. `WanVideoAnimateRefSwap`: `swap_frame=30`, `ref_image=` character B.
3. Sampler runs 2 windows, each ~30 frames.

---

## `WanVideoAnimateRefSwap`

Adds a new reference image that takes effect at a chosen output frame. Chainable — connect multiple swap nodes in series to stage multiple transitions.

| Input | Type | Default | Description |
|---|---|---|---|
| `embeds` | `WANVIDIMAGE_EMBEDS` | — | Upstream `WanVideoAnimateEmbeds` (or another swap node). |
| `ref_image` | `IMAGE` | — | New reference image that will drive generation from `swap_frame` onward. Should match the width/height of the original ref. |
| `swap_frame` | `INT` | `0` | Output frame at which the new reference takes over. The sampler forces a window boundary here and starts using `ref_image` as the static reference from this frame. |
| `reset_temporal` | `BOOLEAN` | `False` | If `True`, clears the temporal seed from the prior window so the new identity starts cleanly instead of morphing from the old identity's last frame. Use for hard character cuts. Only applied at the full-swap boundary, not during a blended transition. |
| `transition_frames` | `INT` | `0` | If `> 0`, the sampler inserts N intermediate micro-windows over this many output frames ending at `swap_frame`, each using a linearly-blended ref latent. `0` = hard swap (default). See [Smooth transitions](#smooth-transitions) below. |
| `transition_steps` | `INT` | `0` | Number of micro-windows for the identity transition. `0` = auto-compute from `transition_frames` (~1 step per 10 frames, no cap). Non-zero overrides. More steps = finer gradient; too many steps shrinks each micro-window and reduces temporal context. |
| `transition_curve` | enum | `linear` | Weight distribution across the micro-windows. `linear` (evenly spaced), `smoothstep` (concentrates the transition in the middle — slow start/end, fast middle), `ease_in` (slow start, fast end — holds old identity longer), `ease_out` (fast start, slow end — reaches new identity quickly). |

**Output:** `image_embeds` — extended dict carrying a sorted `ref_swaps` list. Pass to the next swap node or directly to `WanVideoSampler`.

### Chaining semantics

Multiple swap nodes produce a list of entries in `image_embeds["ref_swaps"]`, sorted by `swap_frame`. Each entry is `{ref_image, swap_frame, reset_temporal, transition_frames, transition_steps, transition_curve}`. The sampler consumes this list to build a window schedule.

### Requirements

- **Looping mode is auto-enabled.** If upstream `WanVideoAnimateEmbeds` is in non-looping mode (short videos where `num_frames <= frame_window_size` and `force_looping` is off), the swap node rewrites the embeds dict into looping shape on the fly — you don't have to remember to check `force_looping`. The runtime log reports this as:
  ```
  WanVideoAnimateRefSwap: auto-enabled looping mode (num_frames X -> Y, num_refs=Z, mask=extracted|none).
  ```
  `force_looping` on Embeds is optional. It's still useful if you want looping without a swap (e.g. a short-video workflow that just needs multiple windows for some other reason).
- **Masked workflows are supported.** When Embeds has a `mask` input, the auto-conversion extracts the latent-space `bg_mask` from the non-looping combined `ref_latent` (channels 0-3 of `ref_latent[:, 1:]`) and re-exposes it as `ref_masks` in looping shape. Variable-window mode honors it via per-iteration `start_latent:end_latent` slicing of the pingpong-padded mask.
- Incompatible features (see [fallback rules](#fallback-rules) below) cause the swap to fall back to coarse window-boundary snapping instead of frame-accurate placement.

---

## `force_looping` on `WanVideoAnimateEmbeds`

Added so short videos (`num_frames ≤ frame_window_size`) can still use the looping/windowed sampler path, which is where ref swaps actually take effect.

| Input | Type | Default | Description |
|---|---|---|---|
| `force_looping` | `BOOLEAN` | `False` | Force looping mode even when `num_frames ≤ frame_window_size`. Required when you want `WanVideoAnimateRefSwap` to split a short video into multiple windows. |

If you're rendering > `frame_window_size` frames, you don't need to touch this — looping activates automatically.

---

## How the variable-window schedule works

When swaps are present and the gating rules pass, the sampler builds a list of `(out_start, out_end, active_ref_idx)` tuples:

1. Collect boundaries: `[0] + [s.swap_frame for each swap if 0 < swap_frame < total_frames] + [total_frames]`.
2. Each adjacent boundary pair becomes a **segment**.
3. Each segment is sub-divided into one or more **windows**, each at most `frame_window_size` frames (or `frame_window_size - refert_num = 76` for non-initial windows).
4. Each window is sampled with a `cur_frame_window_size` rounded up to the nearest VAE-aligned frame count (`4k + 1`). After decoding, the output is truncated to the exact requested size.

### Worked examples

**Example A — `num_frames=120`, `swap_frame=100`, `frame_window_size=77`:**

Quantized `num_frames` → 117 (Wan VAE requires `(n-1) % 4 == 0`).

| iter | segment | window | `cur_frame_window_size` | sampled | decoded | drop | output |
|------|---------|--------|-------------------------|---------|---------|------|--------|
| 0 | `[0,100)` ref A | `(0, 77)` | 77 | 77 | 77 | 0 | 77 |
| 1 | `[0,100)` ref A | `(77, 100)` | 25 | 24 | 25 | 1 | 23 |
| 2 | `[100,117)` ref B | `(100, 117)` | 21 | 18 | 21 | 1 | 17 |

Total output: `77 + 23 + 17 = 117` frames. Swap takes effect at output frame 100.

**Example B — `num_frames=60`, `swap_frame=30`, `frame_window_size=77`, `force_looping=True`:**

In looping mode `num_frames` is preserved exactly as the user set it — no quantization, no clamp on `frame_window_size`.

| iter | segment | window | `cur_frame_window_size` | sampled | decoded | drop | output |
|------|---------|--------|-------------------------|---------|---------|------|--------|
| 0 | `[0,30)` ref A | `(0, 30)` | 33 | 30 | 33 | 0 | 30 |
| 1 | `[30,60)` ref B | `(30, 60)` | 33 | 31 | 33 | 1 | 30 |

Total output: `30 + 30 = 60` frames — exactly two 30-frame windows, which is what the user asked for.

The sampler logs the full schedule at startup so you can confirm the split before waiting for sampling:

```
WanAnimate: variable-window schedule with 2 windows from 1 swap(s): [(0, 30, -1, 0.0), (30, 60, 0, 1.0)]
WanAnimate: Applying ref swap idx 0 fully (window starts at output frame 30, requested swap_frame 30)
```

### Latent quantization

The Wan VAE compresses temporally by 4×, so valid window frame counts are `{1, 5, 9, ..., 4k+1}`. A requested window of 30 frames is sampled as 33 and truncated; a requested window of 20 frames is sampled as 21 (plus 1 temporal overlap for non-initial windows). Extra frames are wasted compute; output is always exactly the requested size.

**Total `num_frames` is preserved in looping mode.** `WanVideoAnimateEmbeds` no longer rounds `num_frames` down to the nearest `4k+1` when looping is active — per-window output truncation handles alignment so the schedule ends exactly at the user's requested total. Non-looping mode (single-window VAE encode) still quantizes, since its encode pass requires a `4k+1` total length.

### `reset_temporal` in practice

Between windows, non-initial iterations seed the new window's temporal context from the **last frame of the previous window**. Without `reset_temporal`, the first few frames after a swap morph from the old identity toward the new ref. Setting `reset_temporal=True` replaces that temporal seed with the new ref image itself, giving a hard cut.

Use `reset_temporal=True` for:

- Hard character changes (different actor / species / style).
- Scene cuts where no visual continuity is desired.

Leave `reset_temporal=False` for:

- Same character, different wardrobe / lighting / expression.
- Smooth morphs between similar identities.

When combined with `transition_frames > 0`, `reset_temporal` is applied **only at the full-swap boundary** (the window where `apply_weight = 1.0` — i.e. the first window starting at `swap_frame`), never during the blend micro-windows. If you want a hard cut with no blend, set `transition_frames=0` and `reset_temporal=True`.

---

## Smooth transitions

`transition_frames` lets you dial in a gradual identity shift instead of an abrupt swap. When `> 0`, the sampler subdivides the transition region `[swap_frame - transition_frames, swap_frame)` into **N micro-windows**, each sampled with a blended `ref_latent = (1 - w) · ref_old + w · ref_new`. The blend happens in VAE latent space — the denoiser's own conditioning shifts as identity — not in pixel space (pixel-space cross-fades produce ghosting at mid weights on different characters, so we don't do that).

- `N = transition_steps` if the input is non-zero, otherwise `max(1, round(transition_frames / 10))` (no cap).
- `N = min(N, transition_frames)` — can't have more steps than frames.
- Each micro-window `k ∈ [0, N)` covers `transition_frames / N` output frames and uses weight `w_k = curve((k+1) / (N+1))`, where `curve` is one of `linear` / `smoothstep` / `ease_in` / `ease_out` per the `transition_curve` input.
- The post-transition window starts exactly at `swap_frame` with `apply_weight = 1.0`. `ref_latent` is permanently replaced by the new ref here, and `reset_temporal` (if set) fires.
- `ref_latent` is **not** overwritten during blend windows, so subsequent blend windows keep interpolating between the original pre-swap latent and the fully-new latent.

Compute cost is ~1× — same as a normal swap. Each micro-window is a full sampling window (not two passes).

### Worked example — latent-blend transition

`num_frames=50` (→ 53 after `AVHandlesAdd`), `swap_frame=25`, `transition_frames=20`, `transition_steps=10`, `transition_curve=smoothstep`, `frame_window_size=77`:

`N = 10` micro-windows of 2 frames each, covering `[5, 25)`. Boundaries from `{0, 5, 7, 9, 11, 13, 15, 17, 19, 21, 23, 25, 53}`.

Smoothstep weights: `[0.028, 0.104, 0.216, 0.352, 0.500, 0.648, 0.784, 0.896, 0.972, 0.993]` (roughly).

| iter | window | target | `apply_weight` | ref_latent used |
|------|--------|--------|----------------|-----------------|
| 0 | `(0, 5)` | — | 0.0 | `ref_A` (original) |
| 1 | `(5, 7)` | swap 0 | 0.028 | `0.972·A + 0.028·B` |
| 2 | `(7, 9)` | swap 0 | 0.104 | `0.896·A + 0.104·B` |
| 3 | `(9, 11)` | swap 0 | 0.216 | `0.784·A + 0.216·B` |
| 4 | `(11, 13)` | swap 0 | 0.352 | `0.648·A + 0.352·B` |
| 5 | `(13, 15)` | swap 0 | 0.500 | `0.500·A + 0.500·B` |
| 6 | `(15, 17)` | swap 0 | 0.648 | `0.352·A + 0.648·B` |
| 7 | `(17, 19)` | swap 0 | 0.784 | `0.216·A + 0.784·B` |
| 8 | `(19, 21)` | swap 0 | 0.896 | `0.104·A + 0.896·B` |
| 9 | `(21, 23)` | swap 0 | 0.972 | `0.028·A + 0.972·B` |
| 10 | `(23, 25)` | swap 0 | 0.993 | `0.007·A + 0.993·B` |
| 11 | `(25, 53)` | swap 0 | 1.0 | `ref_B` (persisted; `reset_temporal` applies here if set) |

Expected log output during sampling:
```
WanAnimate: variable-window schedule with 12 windows from 1 swap(s): [(0, 5, -1, 0.0), (5, 7, 0, 0.028), (7, 9, 0, 0.104), ..., (23, 25, 0, 0.993), (25, 53, 0, 1.0)]
WanAnimate: Encoded new ref for swap idx 0
WanAnimate: Blending toward ref swap idx 0 at weight 0.028 (window 5-7)
WanAnimate: Blending toward ref swap idx 0 at weight 0.104 (window 7-9)
...
WanAnimate: Applying ref swap idx 0 fully (window starts at output frame 25, requested swap_frame 25)
```

### Picking `transition_steps`

- **`0` (default)**: auto `round(transition_frames / 10)`. Good for rough-and-ready — quick morphs over short regions.
- **Match `transition_frames`**: one micro-window per frame. Finest possible gradient, but each micro-window is 1 frame and has almost no temporal attention context. Quality degrades.
- **~half of `transition_frames`**: 2-frame micro-windows. Nice balance — smooth gradient, model still has a little temporal context per step.
- **Large transitions (50+ frames)**: 5-10 steps is usually plenty.

### Transition caveats

- **Latent linear interpolation isn't visually guaranteed.** For similar poses and lighting, the blend produces a smooth identity shift. For drastically different subjects (different species / pose / lighting / camera angle), the intermediate latents can look noisy. For those cases, the only alternative is chained manual intermediates — generate morph frames externally (FILM / RIFE / image-morph tools) and chain multiple `WanVideoAnimateRefSwap` nodes at close frames.
- **Very short micro-windows reduce temporal context.** Each micro-window is an independent sampling window. Windows under ~5 frames give the transformer very little to work with. Balance `transition_steps` against window quality.
- **VAE temporal quantization rounds window sizes up.** Each micro-window is sampled at the nearest `4k+1` frame count and truncated, so a 2-frame micro-window samples 5 frames internally. Not a correctness issue; just extra compute.
- **Overlapping transitions are not supported.** If two swaps' transition regions would overlap, the sampler logs a warning and drops the later swap's transition.
- **`reset_temporal`** applies only at the full-swap boundary window (`apply_weight = 1.0`), never during blend windows.
- **Clamped at frame 0.** If `swap_frame - transition_frames < 0`, the transition starts at frame 0 and is truncated.
- **Fallback mode ignores it.** `transition_frames` has no effect when the sampler falls back to fixed windows (see [Fallback rules](#fallback-rules)).

---

## Fallback rules

The variable-window path activates only when **all** of these are true:

- At least one `WanVideoAnimateRefSwap` node is connected.
- `samples` (input latents for vid2vid / denoise) is not in use.
- `uni3c_embeds` is not in use.
- `start_ref_image` on `WanVideoAnimateEmbeds` is not set.

Spatial `mask` input on Embeds is OK — variable-window mode handles it now via per-iter `start_latent:end_latent` slicing.

If any of the remaining fallbacks are violated, the sampler logs:

```
WanAnimate: ref_swaps present but falling back to fixed-window stride because input samples / uni3c / start_ref_image are in use.
```

In fallback mode, swaps still work but snap to the fixed 77-frame window boundaries (so `swap_frame=30` in a 150-frame render still fires at frame 77, not 30). Supporting variable windows alongside masks / uni3c / input samples would require re-deriving per-latent index math for variable window sizes — out of scope for now.

---

## 5B model support

The swap code path uses the sampler-local `vae_upscale_factor` (16 for Wan 5B, 8 otherwise) when reconstructing pixel-space dimensions for the swap ref encode. Earlier versions of the node hardcoded `lat_h * 8` which produced a shape mismatch on 5B — fixed in commit `96badcb`.

---

## Known limitations

- **Sub-4-frame granularity.** Output boundaries are exact; internally windows always cover a `4k+1` frame count. Only matters if you're counting compute cost.
- **No frame-accurate swaps with masks/uni3c/start_ref_image.** See fallback rules above.
- **RoPE is precomputed for the full video length.** Variable window sizes slice a prefix of the same positional encoding table; unlikely to matter for windows ≥ 5 frames.
- **Short windows have less temporal context.** The model still runs but very small windows (< 10 frames) give the transformer limited context. Larger windows produce better temporal consistency.

---

## Commit history for these features

- `c5bb7ca` — initial `WanVideoAnimateRefSwap` node.
- `96badcb` — fixed the hardcoded 5B upscale factor, added `reset_temporal`, tooltip + logging improvements.
- `942ed32` — variable-window schedule path, `force_looping` on `WanVideoAnimateEmbeds`, frame-accurate swap placement.
- `3a92dd7` — `transition_frames` for smooth ref-image transitions via latent interpolation over micro-windows.
- `fad4a3e` — preserve user's `num_frames` in looping mode; `60` + `swap_frame=30` now produces two exact 30-frame windows instead of `30 + 27`.
- `542c1b5` — auto-convert non-looping Embeds to looping when `WanVideoAnimateRefSwap` is connected, so the swap always takes effect without the user having to toggle `force_looping`.
- `58b8a3a` — support masked Embeds in the variable-window path; `_convert_to_looping` extracts `bg_mask` from the non-looping combined `ref_latent`, sampler gating drops the `wananim_ref_masks is None` requirement, and per-iter `start_latent` / `end_latent` are computed for correct mask slicing.
- `2eb7c5e` — replace `transition_frames` latent-blend micro-windows with pixel-space cross-fade (A/B sibling iterations, shared noise, per-frame smoothstep blend). Per-frame smooth; 2× compute in the transition region only. **Reverted in `0752487`** — pixel cross-fade ghosts at mid weights on different identities.
- `0752487` — revert the pixel cross-fade; restore latent-blend transition with `transition_steps` (no N=5 cap) and `transition_curve` (linear / smoothstep / ease_in / ease_out) tuning controls.

Keep this doc in sync whenever new behavior lands.
