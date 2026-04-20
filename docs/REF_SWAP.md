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

**Output:** `image_embeds` — extended dict carrying a sorted `ref_swaps` list. Pass to the next swap node or directly to `WanVideoSampler`.

### Chaining semantics

Multiple swap nodes produce a list of entries in `image_embeds["ref_swaps"]`, sorted by `swap_frame`. Each entry is `{ref_image, swap_frame, reset_temporal, transition_frames}`. The sampler consumes this list to build a window schedule.

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

When combined with `transition_frames > 0`, `reset_temporal` is applied **only after the cross-fade blend completes**, never during the pass-A or pass-B sampling. If you want a hard cut inside the transition region itself (no smooth blend), set `transition_frames=0` and `reset_temporal=True`.

---

## Smooth transitions

`transition_frames` lets you dial in a gradual identity shift instead of an abrupt swap. When `> 0`, the sampler samples the transition region **twice** (pass A with the prior ref, pass B with the new ref, both sharing the same noise and temporal seed), then **linearly blends the decoded pixels frame-by-frame** with a `smoothstep` curve. The result is per-frame smooth — no discrete weight steps, no mid-window identity snaps.

- The cross-fade region is `[swap_frame - transition_frames, swap_frame)`. Must fit inside one sampling window; if `transition_frames > frame_window_size - refert_num` it's clamped with a warning.
- Both passes share the same VAE noise tensor (saved during pass A and reused for pass B) so motion is identical and only the identity differs.
- Per-frame blend weight is `w(t) = smoothstep(t / (T-1)) = 3t² - 2t³` for `t ∈ [0, T-1]`. `w(0) = 0` → pass A, `w(T-1) = 1` → pass B. Smoothstep has zero first-derivative at both ends, so the transition eases in and out gently.
- `ref_latent` is permanently replaced by the new ref after the blend (during pass B's swap-apply). `reset_temporal` (if set) fires here, applied against the blended output.

Cost: ~2× sampling in the transition region only. Everything outside the transition runs once as normal.

### Worked example — smooth transition

`num_frames=50` (→ 53 after `AVHandlesAdd`), `swap_frame=25`, `transition_frames=20`, `frame_window_size=77`:

Cross-fade region: `[5, 25)` — 20 frames. Boundaries: `[0, 5, 25, 53]`.

| iter | window | target | `apply_weight` | role | ref_latent used | behavior |
|------|--------|--------|----------------|------|-----------------|----------|
| 0 | `(0, 5)` | — | 0.0 | — | `ref_A` (original) | normal sample |
| 1 | `(5, 25)` | — | 1.0 | `A` | `ref_A` (original) | sample, **buffer** output |
| 2 | `(5, 25)` | swap 0 | 1.0 | `B` | `ref_B` (new) | sample, **blend** with buffered A → append |
| 3 | `(25, 53)` | swap 0 | 1.0 | — | `ref_B` (persisted) | normal sample |

Expected log output during sampling:
```
WanAnimate: variable-window schedule with 4 windows from 1 swap(s): [(0, 5, -1, 0.0, None), (5, 25, -1, 1.0, 'A'), (5, 25, 0, 1.0, 'B'), (25, 53, 0, 1.0, None)]
WanAnimate: Encoded new ref for swap idx 0
WanAnimate: Applying ref swap idx 0 (pass B of crossfade) (window starts at output frame 5, requested swap_frame 25)
```

Frames `0..4` are pure ref A. Frames `5..24` smoothly cross-fade from A to B following the smoothstep curve. Frames `25..52` are pure ref B.

### Transition caveats

- **2× compute in the transition region only.** For `transition_frames=50` at 1 s/step, that's roughly 50 s added per transition. Everything outside the region runs once.
- **`transition_frames > frame_window_size - refert_num` gets clamped.** With the default `frame_window_size=77`, the max is 76 output frames. Multi-window cross-fade (spanning multiple sample windows) is not supported — you'd need to split the swap into several smaller transitions.
- **Overlapping transitions are not supported.** If two swaps' transition regions would overlap, the sampler logs a warning and drops the later swap's transition (B becomes a hard swap).
- **Shared-noise requirement.** The A/B passes reuse the same noise tensor so motion stays coherent. Pass B's schedule entry immediately follows pass A's in the schedule — don't rely on being able to interleave other iters between them.
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
- `2eb7c5e` — replace `transition_frames` latent-blend micro-windows with pixel-space cross-fade (A/B sibling iterations, shared noise, per-frame smoothstep blend). Per-frame smooth; 2× compute in the transition region only.

Keep this doc in sync whenever new behavior lands.
