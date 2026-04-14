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
| `reset_temporal` | `BOOLEAN` | `False` | If `True`, clears the temporal seed from the prior window so the new identity starts cleanly instead of morphing from the old identity's last frame. Use for hard character cuts. |

**Output:** `image_embeds` — extended dict carrying a sorted `ref_swaps` list. Pass to the next swap node or directly to `WanVideoSampler`.

### Chaining semantics

Multiple swap nodes produce a list of entries in `image_embeds["ref_swaps"]`, sorted by `swap_frame`. Each entry is `{ref_image, swap_frame, reset_temporal}`. The sampler consumes this list to build a window schedule.

### Requirements

- Upstream `WanVideoAnimateEmbeds` **must** be in looping mode — either because `num_frames > frame_window_size`, or because you enabled `force_looping` on the embeds node. If not, the swap node logs a warning and the swap will be ignored at sample time.
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

Quantized `num_frames` → 57. `frame_window_size` is clamped to 57 by Embeds.

| iter | segment | window | `cur_frame_window_size` | sampled | decoded | drop | output |
|------|---------|--------|-------------------------|---------|---------|------|--------|
| 0 | `[0,30)` ref A | `(0, 30)` | 33 | 30 | 33 | 0 | 30 |
| 1 | `[30,57)` ref B | `(30, 57)` | 29 | 28 | 29 | 1 | 27 |

Total output: `30 + 27 = 57` frames.

The sampler logs the full schedule at startup so you can confirm the split before waiting for sampling:

```
WanAnimate: variable-window schedule with 2 windows from 1 swap(s): [(0, 30), (30, 57)]
WanAnimate: Applying ref swap idx 0 (requested frame 30, window starts at output frame 30)
```

### Latent quantization

The Wan VAE compresses temporally by 4×, so valid window frame counts are `{1, 5, 9, ..., 4k+1}`. A requested window of 30 frames is sampled as 33 and truncated; a requested window of 20 frames is sampled as 21 (plus 1 temporal overlap for non-initial windows). Extra frames are wasted compute; output is always exactly the requested size.

### `reset_temporal` in practice

Between windows, non-initial iterations seed the new window's temporal context from the **last frame of the previous window**. Without `reset_temporal`, the first few frames after a swap morph from the old identity toward the new ref. Setting `reset_temporal=True` replaces that temporal seed with the new ref image itself, giving a hard cut.

Use `reset_temporal=True` for:

- Hard character changes (different actor / species / style).
- Scene cuts where no visual continuity is desired.

Leave `reset_temporal=False` for:

- Same character, different wardrobe / lighting / expression.
- Smooth morphs between similar identities.

---

## Fallback rules

The variable-window path activates only when **all** of these are true:

- At least one `WanVideoAnimateRefSwap` node is connected.
- `wananim_ref_masks` is not in use.
- `samples` (input latents for vid2vid / denoise) is not in use.
- `uni3c_embeds` is not in use.
- `start_ref_image` on `WanVideoAnimateEmbeds` is not set.

If any of those is violated, the sampler logs:

```
WanAnimate: ref_swaps present but falling back to fixed-window stride because ref_masks / input samples / uni3c / start_ref_image are in use.
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

Keep this doc in sync whenever new behavior lands.
