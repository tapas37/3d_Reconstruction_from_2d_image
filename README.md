
---

## Features

- **Depth-guided smart cropping** — Replaces naive center-crop with a percentile-based foreground mask from Depth Anything V2. Keeps off-center subjects in frame.
- **Robust shape generation** — Runs Hunyuan3D-2 in fp16 with configurable seed, guidance scale, and octree resolution. Automatic OOM fallback (384 → 256) keeps inference alive on a 16 GB T4.
- **Full post-processing** — Component selection, normal/winding repair, hole filling, Laplacian smoothing with volume constraint, and quadric-edge-collapse decimation. Reduces ~1.4M raw faces to a portable 80k without losing silhouette.
- **Two coloring modes**
  - **Dominant color** — KMeans (k=10) over non-background pixels picks a single representative RGB, applied uniformly. Clean, minimal, always correct.
  - **Vertex projection** — Maps each mesh vertex to a pixel in the original photo. Exported as both `normal.glb` and `flipped.glb` to work around Hunyuan3D-2's un-guaranteed handedness.
- **Reproducible** — Seeded generators, pinned dependency versions, environment verification cell.

---

## Quick start

Open `notebooks/templeforge_colab.ipynb` in Google Colab with a **T4 GPU runtime**.

Run the cells top to bottom:

1. Install dependencies → **restart runtime** after Cell 1
2. Verify GPU
3. Load Depth Anything V2
4. Upload a temple photo
5. Load Hunyuan3D-2 (downloads ~10 GB on first run)
6. Generate the shape (~4-5 min on T4)
7. Post-process
8. Export colored `.glb` to your Colab files panel

Preview the result at [gltf-viewer.donmccurdy.com](https://gltf-viewer.donmccurdy.com).

---

## Requirements

- Google Colab with **T4 GPU** (or better)
- ~15 GB free disk for model weights
- ~12 GB VRAM peak (fits in 16 GB with fp16 + OOM fallback)

Core dependencies (installed automatically by the notebook):

| Package | Purpose |
|---|---|
| `hy3dgen` | Hunyuan3D-2 inference |
| `transformers` + `timm` | Depth Anything V2 |
| `trimesh` | Mesh loading, cleaning, export |
| `pymeshlab` | Quadric edge-collapse decimation |
| `scikit-learn` | KMeans dominant-color extraction |
| `torch` (CUDA) | GPU runtime |

---

## Design decisions

**Why depth-guided crop instead of center crop?**
Center crop assumes the subject is centered. A tall temple photographed at an angle is often off-center and can be clipped. Depth masking auto-finds the foreground regardless of position.

**Why Hunyuan3D-2?**
Strongest public open-weight image-to-3D shape quality at time of build, with a clean pipeline API.

**Why fp16?**
T4 has 16 GB. Hunyuan3D-2 fp32 needs ~10 GB before activations — won't fit. fp16 halves that.

**Why 50 diffusion steps and octree 384?**
Hunyuan3D-2's defaults. 100 steps roughly doubles time for marginal perceptual gain. Octree 384 → ~1.4M faces, which decimates cleanly to 80k.

**Why decimate to 80k faces?**
1.4M faces → ~30 MB `.glb` that stutters in browsers. 80k → ~1.5 MB that loads in under 2 seconds. Below 40k you see facets; above 200k you see no improvement.

**Why generate both `normal.glb` and `flipped.glb`?**
Hunyuan3D-2 has no guaranteed handedness — the mesh may or may not match the input image's left-right orientation. Detecting this programmatically requires rendering (EGL is fragile on Colab T4), so we ship both and let the user pick in any `.glb` viewer.

**Why vertex colors and not UV texture baking?**
Hunyuan3D-Paint (the texture model) is a separate ~5 GB model. On a T4 with 16 GB, loading it alongside the shape model is a real risk. Vertex colors are simpler, well supported by `.glb`, and look acceptable at 80k vertex density. Texture baking is the natural upgrade path.

---

## Known limitations

- **Occlusion hallucination** — the back of the temple (not visible in the photo) is generated, not reconstructed. Plausible, not ground-truth.
- **Depth-based crop is a heuristic** — the closest object is assumed to be the subject. Foreground clutter (people, pillars) can confuse it. A segmentation model (SAM) would be more precise.
- **Mirror ambiguity is unresolved** — handled via user choice (normal vs. flipped export) rather than automatic detection.
- **Vertex colors are lower-resolution than baked textures** — the color resolution is bounded by vertex count.
- **Single input image only** — no multi-view fusion.

---

## Future work

- Replace the depth-crop heuristic with SAM segmentation
- Integrate Hunyuan3D-Paint for UV texture baking
- Auto-resolve mirror ambiguity via silhouette IoU against the input image
- Add quantitative evaluation (SSIM, LPIPS, Chamfer distance) with a test set
- Multi-view input for better occlusion handling
- FastAPI service + minimal frontend for turnkey demos

---


---

## License

MIT — see `LICENSE`.

---

## Acknowledgements

- [Tencent Hunyuan3D-2](https://github.com/Tencent/Hunyuan3D-2) — shape generation
- [Depth Anything V2](https://github.com/DepthAnything/Depth-Anything-V2) — monocular depth estimation
- [trimesh](https://github.com/mikedh/trimesh), [PyMeshLab](https://github.com/cnr-isti-vclab/PyMeshLab), [HuggingFace Transformers](https://github.com/huggingface/transformers)
