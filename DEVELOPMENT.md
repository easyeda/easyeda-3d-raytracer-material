# Raytraced Material Resource Pack · Development Guide

> English | [简体中文](DEVELOPMENT.zh-CN.md)
>
> Two audiences:
> **A. Product / design** — understand what the resource pack is and how to make it take effect;
> **B. Material-pack authors / AI** — follow this guide to produce a photorealistic pack independently.
>
> This document covers **what goes into the pack, how to fill in each field, which values to use, and how to verify** — it does not cover code development.
>
> All values were measured in 2026-11 with client **4.1.109** and `smt-gl-engine 4.1.44`.
> Conclusions marked "measured" are local verification results. If an interface changes in a later version, the installed client's `easyeda-pro --help` / `easyeda-pro doc api` output wins.
> Mind the distribution: the international EasyEDA Pro executable is `easyeda-pro` (used throughout this document); the mainland China edition is `lceda-pro` — commands are identical, so substitute `easyeda-pro` with `lceda-pro` verbatim (e.g. `lceda-pro --help`, `lceda-pro doc api`). Use whichever edition is actually installed and never mix the two.

---

## Contents

1. What the resource pack is
2. Directory structure and file responsibilities
3. `material_table.json` — the color→PBR palette (the easiest part to get wrong)
4. `inner_material_table.json` — built-in material overrides
5. `param.json` — render parameters
6. Texture rules (channels, color space, size, naming)
7. HDR environment rules
8. Color space: everything is written as linear floats
9. `scene_model/`: whether to add it and how to develop it; why `debug/` is never included
10. Loading and activation mechanism (production path / hot reload)
11. Runtime behavior of the pack (no code required)
12. Full workflow for generating a pack with AI
13. Verification and tuning
14. Common pitfalls (summary)
15. Release checklist

---

## 1. What the resource pack is

When the ray tracer starts, it reads a **zip resource pack** from extension storage. The pack may contain:

- A material table (appearance parameters for every face/component)
- Built-in material overrides (PCB substrate / solder mask / silkscreen / finish)
- PBR textures (base color / normal / roughness / metallic)
- HDR environment map
- Render parameters (renderer / background / lights)
- Optional scene models (OBJ+MTL, e.g. desktop props)

The zip's **root directory contains these files directly** — no top-level folder.

### Size reference (measured)

A 154.7 MB set of PNG assets can be reduced to:

| Tier | Total size | Largest file | Notes |
|---|---:|---:|---|
| Original | 154.7 MB | 25.4 MB | Many RGBA images (alpha all 255), unoptimized, 3 duplicate file sets |
| Strict lossless | **56.5 MB** | 8.6 MB | Alpha removed, deduplicated, PNG level 9, pixel-identical |
| Near-lossless 1024 | 12.2 MB | ≤1.9 MB | **Not usable with this engine** (see below) |

> ⚠️ **Texture dimensions cannot be reduced.** The engine texture atlas unit is `UNIT_SIZE = 2048`, so textures must be 2048×2048, otherwise it throws:
> `Texture size mismatch: 1024×1024, expected 2048×2048`.
> Size optimization can therefore only be done at the **encoding layer** (strip alpha, deduplicate, PNG level 9). Ship the `lossless/` tier.

---

## 2. Directory structure and file responsibilities

```
raytracer-assets.zip
├─ material_table.json          # Required: color→PBR palette (array)
├─ inner_material_table.json    # Optional: override PCB built-in materials (object)
├─ param.json                   # Optional: background / lighting / render settings
├─ *.png / *.hdr                # Textures (referenced by filename in the material tables)
└─ scene_model/                 # Optional: scene props (OBJ+MTL)
    ├─ manifest.json
    ├─ <name>.obj
    └─ <name>.mtl
```

> This pack does not include `debug/` (engine debug output); see Chapter 9.

| File | Required | Purpose |
|---|---|---|
| `material_table.json` | ✅ Yes | Color→PBR palette; without it components fall back to matte gray |
| `inner_material_table.json` | No | Overrides named materials such as pads / substrate / solder mask / silkscreen |
| `param.json` | No | Renderer, background, lights |
| `*.hdr` | Conditional | Referenced by `background.texture` in `param.json` |
| `*.png` | Conditional | Referenced by material entries' `*Texture.name`; must be 2048×2048 |
| `scene_model/` | No | Scene props; see Chapter 9 for when to include |

> `debug/` is not shipped with the pack and does not need to be produced; see Chapter 9.

---

## 3. `material_table.json` — the color→PBR palette

### 3.1 What it actually is (the most misunderstood part)

It is **not** a per-face material list for the current board. It is the **color→material lookup table** the engine uses **while building geometry**:

```
MaterialComponent.getIndex(phong):
  1) phong hits a built-in material (the H6 dictionary) → use the PBR from inner_material_table
  2) otherwise, if material_table exists (hasMaterialTable)
        → find the nearest entry by baseColor distance, use that entry's PBR
  3) otherwise                                        → pbr = fromPhong(diffuse)
                                                         i.e. metallic 0 / roughness 1 (matte gray)
```

EDA component 3D models carry no PBR data — only a base color (e.g. `0.208` gray = component body, `0.592,0.463,0` = plating, `0.855` gray = pins). The palette maps these encoded colors to real materials.

**Key trick**: an entry's `phong.diffuse` is used **only for matching**; `pbr.color` determines the **actual rendered color**. They may differ — e.g. map `0.208` gray to near-black epoxy `[0.078,0.063,0.063]`.

### 3.2 Structure

An array; each element has both a Phong (rasterized preview) and a PBR (raytraced) version:

```json
[
  {
    "phong": {
      "ambient":  [1, 1, 1],
      "diffuse":  [0.2078, 0.2078, 0.2078],
      "specular": [0.1255, 0.1255, 0.1255],
      "shininess": 5,
      "alpha": 1
    },
    "pbr": {
      "color": [0.0032, 0.0032, 0.0034],
      "metallic": 0,
      "roughness": 0.45,
      "ior": 1.5,
      "iorLevel": 0.5,
      "clearcoatTintColor": [1, 1, 1],
      "clearcoatIntensity": 0.15,
      "clearcoatRoughness": 0.12,
      "clearcoatIor": 1.5,
      "normalIntensity": 0.6,
      "normalTexture": null,
      "metallicTexture": null,
      "colorTexture": null,
      "roughnessTexture": null
    }
  }
]
```

**`phong` fields** (2D rasterized preview only — the ray tracer ignores them, except `diffuse`):

| Field | Type | Description |
|---|---|---|
| `ambient` | `[r,g,b]` | Rasterized ambient color; ignored by the ray tracer |
| `diffuse` | `[r,g,b]` | **Color matching key**. The engine uses it to find the palette entry; must match the component/model's encoded color — do not change |
| `specular` | `[r,g,b]` | Rasterized specular color; ignored by the ray tracer |
| `shininess` | number | Rasterized specular sharpness; ignored by the ray tracer |
| `alpha` | 0–1 | Opacity; the ray tracer treats surfaces as opaque — keep at 1 |

**`pbr` fields** (the parameters actually used by the ray tracer; all linear space):

| Field | Type / range | Description | Typical values |
|---|---|---|---|
| `color` | `[r,g,b]` 0–1 | Base color (albedo); for metals this is reflectance, not "paint color" | See Chapter 8 |
| `metallic` | 0–1 | Metalness; 0 = dielectric, 1 = pure metal | Metals 1.0; plastics/epoxy/solder mask 0 |
| `roughness` | 0–1 | Roughness; lower = glossier | Metals 0.2–0.35; epoxy 0.45–0.55; solder mask 0.35–0.45 |
| `ior` | ≥1 | Index of refraction; sets the base surface reflectance | Metals 1.5; solder mask 1.42–1.45; matte parts 1.0 |
| `iorLevel` | 0–1 | Fresnel strength, works with `ior`; too high washes out in bright environments | Metals 0.5; solder mask 0.15–0.25; matte 0 |
| `clearcoatTintColor` | `[r,g,b]` | Clearcoat tint (usually white) | `[1,1,1]` |
| `clearcoatIntensity` | 0–1 | Clearcoat strength; 0 = off | Epoxy 0.10–0.15; solder mask 0.06–0.18 |
| `clearcoatRoughness` | 0–1 | Clearcoat roughness; lower = shinier | 0.10–0.30 |
| `clearcoatIor` | ≥1 | Clearcoat IOR | 1.5 |
| `normalIntensity` | 0–2 | Normal strength; affects the built-in micro-normal when no normal map is present | Metals 0.3–0.4; plastics 0.6; none 1.0 |
| `colorTexture` | `{name, scale}` / `null` | Base color map (sRGB) | Used by scene props |
| `normalTexture` | `{name, scale}` / `null` | Normal map (linear, +Y up) | Same as above |
| `roughnessTexture` | `{name, scale}` / `null` | Roughness map (grayscale) | Same as above |
| `metallicTexture` | `{name, scale}` / `null` | Metallic map (grayscale) | Same as above |

> Texture objects have two sub-fields: `name` is the filename inside the zip (must match character-for-character), `scale` is the UV tiling multiplier (1 = no repeat).

### 3.3 Composition of this pack's palette (1,384 entries, 1,280 unique colors)

Measured statistics:

- The vast majority are colored dielectrics with `metallic 0 / roughness 0.25`;
- 170 metallic entries with `metallic 1 / roughness 0.25`;
- Grayscale coverage from 0.02 to 1.0 in 50+ steps, to hit component-encoded colors.

### 3.4 Recommended generation rules

Classify by `phong.diffuse` and write an independently designed `pbr`:

| Category | Criterion | metallic | roughness | clearcoat | pbr.color (linear) |
|---|---|---:|---:|---:|---|
| Black epoxy body | Neutral gray luminance < 0.16 | 0 | 0.45 | 0.15 | sRGB 0.045 → linear 0.0032 |
| Dark gray body | Neutral 0.16–0.32 | 0 | 0.50 | 0.10 | sRGB 0.12 → linear 0.0137 |
| Pins / pads / terminals | Neutral ≥ 0.32 | 1.0 | 0.33 | 0 | 0.66–0.76 |
| Bright metal | Neutral ≥ 0.88 | 1.0 | 0.28 | 0 | 0.74 (except pure white) |
| Gold plating | b < 0.10 and g/r ∈ [0.60, 0.92] | 1.0 | 0.26 | 0 | `[0.749,0.604,0.224]` |
| Copper | b < 0.10 and g/r ∈ [0.30, 0.60] | 1.0 | 0.30 | 0 | `[0.620,0.300,0.140]` |
| Colored dielectric | Everything else | 0 | 0.48 | 0.10 | Original sRGB → linear |
| White plastic / silkscreen | Pure white | 0 | 0.55 | 0.05 | `[0.90,0.895,0.88]` linear |

**Deduplicate**: keep only one entry per `phong.diffuse` (matching is nearest-neighbor, so duplicates are meaningless and only increase size).

### 3.5 Hard requirements

| Requirement | Description |
|---|---|
| Array must not be empty | An empty array means `hasMaterialTable = false` — no palette, all components fall back to matte gray |
| Keep `phong.diffuse` at the original encoding | It is the matching key; changing it to a design color makes matching fail and colors go wrong |
| One entry per `diffuse` | Matching is nearest-neighbor; duplicates add size and nothing else |
| Entry count and order are unconstrained | Matching is by color, not index, and is board-independent |

---

## 4. `inner_material_table.json` — built-in material overrides

### 4.1 Rules

An object whose keys must be engine built-in material names; unknown keys are skipped with a warning.

The fields are **identical** to the `pbr` described in §3.2 (there is no `phong`, since built-in materials already carry a Phong definition):

```json
{
  "gold": {
    "color": [0.749, 0.604, 0.224],
    "metallic": 0.85, "roughness": 0.30,
    "ior": 1.5, "iorLevel": 0.5,
    "clearcoatTintColor": [1,1,1],
    "clearcoatIntensity": 0, "clearcoatRoughness": 0, "clearcoatIor": 1.5,
    "normalIntensity": 0.25,
    "normalTexture": null, "metallicTexture": null,
    "colorTexture": null, "roughnessTexture": null
  }
}
```

### 4.2 Common built-in material names

**Actually used by PCBs (override all of these)**

| Group | Names |
|---|---|
| Surface finish | `gold` `silver` `aluminum` `osp` |
| Silkscreen | `silk_white` `silk_black` |
| Substrate | `substrate_yellow` (FR4) `substrate_white` (ceramic) `substrate_grey` `substrate_coppery` `substrate_fpc` |
| Solder mask | `solder_mask_<color>_with_copper` / `_without_copper`; colors: `green red blue yellow black purple white coppery soft_yellow` |

**Other valid names that are not PCB surfaces** (component or prop models may reference them; override as needed)

| Category | Names | Description |
|---|---|---|
| Metals | `brass` `bronze` `chrome` `coppery` | Brass / bronze / chrome / copper, used by component metal parts and props |
| Gems and minerals | `obsidian` `pearl` `jade` `emerale` `ruby` `turquoise` | Obsidian / pearl / jade / emerald / ruby / turquoise, decorative models |
| Plastics | `plastic_black` `plastic_white` `plastic_red` `plastic_green` `plastic_yellow` `plastic_cyan` | Common molded shell colors; component bodies may reference them |
| Rubber | `rubber_black` `rubber_white` `rubber_red` `rubber_green` `rubber_yellow` `rubber_cyan` | Cable jackets, feet and other flexible materials |
| FPC stiffeners | `stiffen_layer_black_film` `stiffen_layer_steel` `stiffen_layer_transparent_orange` `stiffen_layer_transparent_white` `stiffen_layer_transparent_yellow` | Flex-board stiffener layers, referenced only by FPC models |
| Transparent substrates | `substrate_transparent_acrylic` `substrate_fpc_acrylic` `substrate_glossy_transparent_pet` `substrate_frosted_transparent_pet` `substrate_glossy_transparent_pvc` `substrate_frosted_transparent_pvc` `substrate_black_transparent_acrylic` | Glossy and frosted acrylic / PET / PVC panels |
| Solder mask variants | `solder_mask_acrylic` `solder_mask_acrylic_with_copper` `solder_mask_acrylic_without_copper` | Acrylic-look solder mask, special board types only |
| Solder bumps | `raytracer_solder_bump` | BGA / CSP solder bumps |
| Occlusion | `absolute_black` | Pure black occlusion material |
| Placeholder | `zero` | Empty material placeholder; generally not overridden |

> The engine built-in dictionary has 81 names in total. A wrong name does not raise an error — it is silently skipped and the material keeps its default look.

### 4.3 `with_copper` vs. `without_copper`

The engine picks a variant based on whether copper lies under the solder mask:

| Variant | Physical meaning | Suggested parameters |
|---|---|---|
| `_with_copper` | Solder mask over traces; denser coating | roughness 0.36–0.40, clearcoat 0.12–0.18, color may be ×1.05 brighter |
| `_without_copper` | Solder mask over substrate; more matte | roughness 0.42–0.46, clearcoat 0.06–0.10 |

Do not let the two diverge too much, or copper and non-copper areas of the same board will show visible banding.

### 4.4 Recommended parameters (best measured appearance)

| Material | color (linear) | metallic | roughness | ior | iorLevel | clearcoat |
|---|---|---:|---:|---:|---:|---:|
| gold | `0.749,0.604,0.224` | 0.85 | 0.30 | 1.5 | 0.5 | 0 |
| silver | `0.820,0.820,0.824` | 0.85 | 0.24 | 1.5 | 0.5 | 0 |
| aluminum | `0.520,0.520,0.540` | 1.0 | 0.32 | 1.5 | 0.5 | 0 |
| silk_white | sRGB 0.90 → linear 0.787 | 0 | 0.50 | 1.5 | 0.5 | 0 |
| substrate_yellow | sRGB 0.72,0.64,0.42 → linear | 0 | 0.78 | 1.5 | 0.5 | 0 |
| Solder mask (with copper) | sRGB target color → linear ×1.05 | 0 | 0.38 | 1.45 | **0.20** | 0.14 |
| Solder mask (without copper) | Same as above | 0 | 0.44 | 1.42 | **0.20** | 0.08 |

> `iorLevel` is the easiest place for solder mask to go wrong: 0.5 is a common configuration, but under a "bright softbox HDR" it washes out the whole board. Measured 0.15–0.25 keeps the lacquer-like Fresnel without overexposure.

---

## 5. `param.json` — render parameters

```json
{
  "renderer": {
    "rendererMaxSample": 4096,
    "rendererEnableDenoise": true,
    "displayDevice": "sRGB",
    "viewTransform": "ACES",
    "lutName": "", "lutIntensity": 1,
    "maxRayBounces": 6,
    "hideRenderPlane": false,
    "hideRenderTriangle": false,
    "hideUnselected": false,
    "selectMaterialId": 0,
    "useBlueNoise": true
  },
  "background": {
    "useColor": false,
    "texture": "studio.hdr",
    "intensity": 0.35,
    "rotation": 0,
    "hide": true,
    "hideColor": "#000000"
  },
  "light": {
    "a0": { "color": "#ffffff", "power": 0.8, "normalize": false,
            "size": {"width": 0.011, "height": 0.011},
            "position": [-0.06, 0.23, 508.9], "rotate": [-45, 0, 25],
            "hideRender": false, "cameraVisible": true, "castShadow": true,
            "rayVisibilityDiffuse": true, "rayVisibilityGlossy": true,
            "spread": 80, "colorMapTileId": -1 },
    "a1": { "...": "defaults are fine" },
    "a2": { "...": "defaults are fine" }
  }
}
```

### 5.1 Renderer

| Field | Suggested value | Purpose | Notes |
|---|---|---|---|
| `rendererMaxSample` | 1024–65536 | Sample cap | Progressive accumulation; a high value does not slow rendering — it only sets "how clean it can get" |
| `rendererEnableDenoise` | true | Denoising | Enables usable output at low sample counts; keep on |
| `displayDevice` | `sRGB` | Display color space | `Display p3` also available |
| `viewTransform` | `ACES` | Tone mapping | `Standard` is flatter; ACES gives photographic highlight rolloff |
| `lutName` | `""` | LUT file | Empty = unused; the file must be included in the pack |
| `lutIntensity` | 1 | LUT strength | Only effective when `lutName` is non-empty |
| `maxRayBounces` | 6 | Max bounces | Diminishing returns above 8, only more time |
| `hideRenderPlane` | false | Hide plane | Debug use; keep false for final renders |
| `hideRenderTriangle` | false | Hide triangles | Debug use |
| `hideUnselected` | false | Render selection only | Debug use |
| `selectMaterialId` | 0 | Highlight selected material | Debug use; keep 0 for final renders |
| `useBlueNoise` | true | Blue-noise sampling | Reduces low-frequency noise; keep on |

### 5.2 Background

| Field | Description |
|---|---|
| `useColor` | true = solid-color background (uses `color`); false = use HDR environment |
| `color` | Background color (linear) when `useColor:true`; mutually exclusive with HDR |
| `texture` | HDR filename, **must match the file inside the zip exactly** |
| `intensity` | HDR energy multiplier. Measured 0.3–0.5 is most stable; >0.8 easily washes out solder mask |
| `rotation` | Horizontal rotation (radians), used to reorient the key light |
| `hide` | true = **lighting only, not visible** (product shots often use black); false = background visible |
| `hideColor` | Background color when `hide:true`, **interpreted as linear**: `#0B0C0F` renders around `#3A3A3A`; use `#000000` for true black |

### 5.3 Lights (important: version differences)

The pack can configure three area lights `a0` / `a1` / `a2`:

| Field | Type | Description |
|---|---|---|
| `color` | hex string | Light color (linear interpretation) |
| `power` | number | Power; the engine scales internally by `power × scene scale²` — do not think in physical watts |
| `normalize` | boolean | Whether to normalize; leave false |
| `size` | `{width, height}` | Emitter size; same scale as `position` |
| `position` | `[x, y, z]` | Position relative to the scene center |
| `rotate` | `[x, y, z]` | Emitter orientation (degrees) |
| `hideRender` | boolean | Hide the emitter while rendering; true = emits but is invisible |
| `cameraVisible` | boolean | Whether the camera can see the emitter |
| `castShadow` | boolean | Whether it casts shadows |
| `rayVisibilityDiffuse` | boolean | Whether it contributes to diffuse lighting |
| `rayVisibilityGlossy` | boolean | Whether it contributes to specular/glossy lighting |
| `spread` | number (degrees) | Emission angle; 180 = hemisphere, smaller = tighter |
| `colorMapTileId` | number | Light texture index; -1 when no texture |

Version behavior:

- The `a0` key light configured in the pack's `param.json` **is applied at initialization**;
- **Measured on 4.1.109**: runtime `setRenderConfigurations({ light })` reads back successfully, but exported frames are pixel-identical; `light` also has no effect during hot reload via `importPackage`;
- Therefore **lighting should rely primarily on the HDR**, with lights as optional fill, and always verify on the target version.

Light-unit hint: the engine rescales by scene scale internally (`power * scale²`, normalized `position`). The recommended combination is `power 0.8 / size 0.011 / position z=508` — **do not scale up arbitrarily based on physical intuition**.

---

## 6. Texture rules

| Use | Format | Color space | Notes |
|---|---|---|---|
| Base color `colorTexture` | PNG | **sRGB** | Name the file with `basecolor` / `Color` |
| Normal `normalTexture` | PNG | **Linear** | Tangent space, **+Y up** (OpenGL convention) |
| Roughness `roughnessTexture` | PNG | **Linear** | Grayscale, single channel is enough |
| Metallic `metallicTexture` | PNG | **Linear** | Grayscale, single channel is enough |
| Environment `background.texture` | HDR / RGBE | Linear | Equirectangular, **not part of the atlas** (1024×512 / 2048×1024 both fine) |

> The first four PBR texture types go into the engine texture atlas and **must be 2048×2048**; HDR is sampled separately and is not subject to that limit.

### 6.1 Size and file size

| Requirement | Description |
|---|---|
| Strictly 2048×2048 | Engine atlas `UNIT_SIZE = 2048`; other sizes throw `Texture size mismatch` |
| Compress only at the encoding layer | RGB→L grayscale, strip alpha, PNG level 9; **never reduce resolution** |
| Do not output RGBA | Alpha is usually all 255 — pure overhead; 8-bit is enough, 16-bit roughly quadruples size |

### 6.2 Naming and references

```json
"colorTexture": { "name": "my_mask_basecolor.png", "scale": 1 }
```

| Sub-field | Type | Description |
|---|---|---|
| `name` | string | Filename inside the zip; must match exactly |
| `scale` | number | UV tiling multiplier; 1 = as-is, 2 = 2×2 tiling |

| Rule | Description |
|---|---|
| `name` matches character-for-character | Must equal the filename inside the zip exactly, including Chinese characters, case and spaces |
| Missing files do not error | The texture simply does not show — materials look all black or flat-colored |
| One file can be reused | Multiple materials may reference the same file; do not package duplicates (they only bloat the zip) |

### 6.3 UV and tiling

| Rule | Description |
|---|---|
| `scale` is a tiling multiplier | When >1 the texture must be **seamlessly tileable**, otherwise seams appear |
| No alpha cutout | The ray tracer treats surfaces as opaque; cutouts require geometry |
| UV-dependent | PCB body UVs are generated by the engine; adding a normal map to solder mask may stretch — verify first |

---

## 7. HDR environment rules

### 7.1 Specification

| Item | Specification |
|---|---|
| Projection | Equirectangular: `az = u·360° − 180°`, `el = (0.5 − v)·180°` (v=0 is zenith) |
| Format | Radiance `.hdr` (`#?RADIANCE` / `FORMAT=32-bit_rle_rgbe`), per-scanline RLE |
| Size | 1024×512 or 2048×1024 recommended, 2:1 aspect ratio |
| Orientation | Keep standard `-Y +X`; other orientation variants are not supported in the current version |

### 7.2 A directly reusable "studio" environment

Four softboxes; measured natural look with little overexposure:

| Role | az | el | Angular size aw×ah | Intensity I | Soft edge |
|---|---|---:|---:|---:|---:|
| Key | −40° | 40° | 26°×17° | 22 | 11° |
| Top | −8° | 76° | 64°×14° | 4.5 | 12° |
| Fill | 62° | 10° | 34°×20° | 7 | 11° |
| Rim | 165° | 30° | 38°×12° | 13 | 11° |

Base environment (avoids pure black and dead shadows):

```
base = 0.004 + 0.007·(el>0 ? 1 : 0.5) + 0.006·exp(-|el|/18)
R = base, G = base·0.985, B = base·1.015
```

Softbox accumulation:

```
daz = |wrap180(az − az0)|
del = |el − el0|
fx  = max(0, 1 − max(0, daz − aw/2)/soft)
fy  = max(0, 1 − max(0, del − ah/2)/soft)
f   = fx·fy
core= max(0, 1 − max(daz/(aw/2), del/(ah/2))²)
I_eff = I · (0.65·f + 0.35·core·f)
RGB += I_eff · color
```

### 7.3 Energy and look

- The HDR is **the only reliable light source** (see 5.3);
- Brightness baseline: `background.intensity` 0.30–0.50 corresponds to a product-shot look with **board-surface median brightness p50 ≈ 16–25/255**;
- If the solder mask washes out: lower `intensity` first, then the solder mask `iorLevel`, and only then consider recoloring;
- To get environment reflections without a black background: set `hide:false`, but the softboxes become visible bright spots — product shots usually use `hide:true` plus compositing.

---

## 8. Color space: everything is written as linear floats

**This was the biggest failure point in the previous version.** The engine treats `pbr.color` and `hideColor` as **linear** values and feeds them straight into lighting without sRGB decoding.

| Approach | Result |
|---|---|
| Copy design-file sRGB values directly (e.g. `#12305E` → `0.070,0.188,0.369`) | Renders noticeably brighter, grayish/washed out |
| Convert to linear first (`#12305E` → `0.006,0.0296,0.1119`) | Matches the design file |

Conversion rule (sRGB component u, 0–1):

```
u ≤ 0.04045      →  u / 12.92
u > 0.04045      →  ((u + 0.055) / 1.055) ^ 2.4
```

For hex colors, normalize each component by /255 first, then apply the formula. Common conversions:

| Design color (sRGB) | Linear value |
|---|---|
| `#12305E` | `0.006, 0.0296, 0.1119` |
| `#0A1B38` | `0.0030, 0.0116, 0.0429` |
| `#C9A24A` (gold) | `0.5721, 0.3744, 0.0742` |
| `#909090` | `0.2789` |

Key points:

| Point | Description |
|---|---|
| A metal's `pbr.color` is reflectance | A physically correct value, not a "paint color"; gold is `[0.749,0.604,0.224]`, not bright yellow |
| Keep `phong.diffuse` at the original encoding | It is the matching key, not a display color |
| `hideColor` / `light.color` | Also interpreted as linear; use `#000000` for true black |

---

## 9. `scene_model/`: whether to add it and how to develop it

### 9.1 Whether to add it

| Need | Add? |
|---|---|
| Desktop props (soldering iron, solder wire, tweezers, desk mat, etc.) | Yes |
| Board-only product shots with a clean background | No (saves about 27 MB) |

Once added, props appear in the scene at their world coordinates alongside the PCB; without it, the scene contains only the board itself.

### 9.2 Directory and manifest

```
scene_model/
├─ manifest.json
├─ <name>.obj
└─ <name>.mtl
```

`manifest.json` lists all props:

```json
{
  "version": "1.0.0",
  "elements": [
    { "name": "Soldering Iron",
      "obj": "scene_model/scene_soldering_iron.obj",
      "mtl": "scene_model/scene_soldering_iron.mtl" }
  ]
}
```

| Field | Description |
|---|---|
| `name` | Display name; customizable |
| `obj` / `mtl` | Paths relative to the zip root; must match the actual files exactly |
| Order | Does not affect rendering |

### 9.3 OBJ requirements

| Item | Requirement |
|---|---|
| Coordinates | **World coordinates**, loaded with zero transform; placement comes entirely from the vertices |
| Units | Same as the PCB (mil); otherwise props are too large or too small |
| `mtllib` | Must point to the MTL in the same directory, with the same filename as in the manifest |
| `usemtl` | Material names referenced must have a matching `newmtl` in the MTL |
| Size | Keep under 50k vertices per prop; larger significantly slows loading |

### 9.4 MTL requirements

| Item | Requirement |
|---|---|
| Material name | Must match the OBJ's `usemtl` — **do not rename** |
| `Kd` | Determines the part's color encoding; the ray tracer matches `material_table.json` through it |
| Textures | Do not write `map_*` in the MTL; prop textures are referenced through the material table (see 9.5) |

### 9.5 How prop textures take effect

1. Set `Kd` in the MTL (e.g. `0.800 0.800 0.800`);
2. Add an entry with `phong.diffuse = [0.8, 0.8, 0.8]` to `material_table.json`;
3. Fill in `colorTexture` / `normalTexture` / `roughnessTexture` in that entry's `pbr`, with `name` matching files in the pack;
4. When the engine builds the prop it matches the entry by color, and the textures apply.

> Textures must be 2048×2048; the same set can be referenced by multiple parts.

### 9.6 Why `debug/` never appears in the pack

`debug/` (`camera.json`, `used_material.json`, `plane.json`, etc.) is debug data the engine generates **when exporting a scene**, and it is only valid for the board it was exported from (material slots, camera, and board-layer planes are bound to specific geometry).

This pack does not include `debug/` and it does not need to be generated. The resource pack is responsible only for appearance — it carries no camera or board-layer data.

---

## 10. Loading and activation mechanism

```
eext-common menu "Enable ray tracing"
  → if not installed, download the zip from modules.lceda.cn (streamed, with progress)
  → store in extension storage under 'raytracer resource pack'
  → eda.pcb_RayTracerEngine.init(uuid)
  → pro-api → pro-ui → pro-pcb → smt-gl-engine
  → look up the zip in IndexedDB by uuid
  → unpack, build a getAsset(path) closure
  → MaterialComponent.decode()
  → render
```

Decode order:

1. `inner_material_table.json` → override built-in materials (**first**)
2. `material_table.json` → clear and rebuild the palette
3. `debug/used_material.json` → restore material slots (if present in the pack; this pack has none)

### 10.1 Why "load at initialization" matters

`getIndex` color matching happens **while building geometry**. If the pack loads after geometry is built:

- The palette is not re-matched (components keep the old / `fromPhong` materials);
- `usedMaterials` slot indices still point at the old table, so a new table of different length produces **mismatched assignments**.

**Measured**: after hot-loading a new 5-entry table and set materials 0–4 to pure red, the entire board (substrate, silkscreen, pads, solder mask) turned red — proving the indices land directly on slots 0–4.

### 10.2 Correct use of hot reload (`importPackage`)

- Suitable for: packs where **entry count and order are unchanged** (e.g. export from this board → tweak parameters → re-import);
- Not suitable for: reusing across boards;
- To hot-verify a pack, the correct order is: **stop ray tracing → wait 2 seconds → restart (rebuild geometry) → then import the pack**;
- The safest path: write the pack into extension storage and restart ray tracing so it loads at initialization (production path).

---

## 11. Runtime behavior of the pack (no code required)

| Behavior | Description | Recommendation |
|---|---|---|
| Loads at initialization | Palette matching happens at geometry build time; swapping packs afterwards neither re-matches nor keeps material slots aligned | Load the pack as part of initialization |
| Hot reload has preconditions | Only suitable for packs with unchanged entry count and order | Use initialization loading when reusing across boards |
| Repeated initialization errors | Engine init is not idempotent; initializing again while rendering throws `this.currentTask !== null` | Stop ray tracing, wait, restart; or close and reopen the 3D page |
| Camera is not a pack capability | Runtime camera changes have no effect in 4.1.109 | This pack contains no `debug/`, so no camera handling is needed |
| Export resolution = canvas size | The pack has no resolution field | Adjust the window/canvas if a fixed size is required |
| Light activation timing | Lights in `param.json` apply at initialization; runtime light changes do not | Rely mainly on HDR lighting |
| Texture size | PBR textures in the atlas must be 2048×2048; HDR is outside the atlas | HDR may be 1024×512 |

---

## 12. Full workflow for generating a pack with AI

### 12.1 What AI can and cannot do

| Can generate | Notes |
|---|---|
| `inner_material_table.json` | ✅ Most recommended: reskin the PCB (finish, substrate, solder mask) |
| `material_table.json` | ✅ Batch-generate the palette from the "color→PBR" rules |
| PBR texture sets | ✅ AI image generation works; mind channels and color space |
| `param.json` | ✅ Background/renderer parameters |
| HDR environment | ✅ Procedural generation is most reliable (see Chapter 7) |
| `scene_model/` | ⚠️ Simple props are possible; geometric precision is not guaranteed |
| Replacing PCB geometry | ❌ Geometry is generated from EDA data |
| Deciding material slot indices | ❌ Baked by the engine via color matching |

### 12.2 Prompt template to hand to AI

```
Generate a general-purpose EasyEDA ray-tracing material pack, output as a zip (files at the root), containing:
material_table.json   — roughly 1,384 "color→PBR" palette entries
inner_material_table.json — 27 built-in material overrides
param.json            — studio render parameters
studio.hdr            — 1024x512 studio environment map

Hard constraints:
1. All colors are 0–1 linear float arrays; hex strings are forbidden. Convert design colors from sRGB to linear first.
2. For every material_table entry: keep phong.diffuse at the given encoded color (matching key), write the design color into pbr.color.
3. Metals (pads/pins): metallic=1, roughness 0.25–0.35; black epoxy: metallic=0, roughness 0.45–0.55.
4. Solder mask: roughness 0.36–0.46, ior 1.42–1.45, iorLevel 0.15–0.25, clearcoat 0.06–0.18.
5. HDR is equirectangular + RGBE; four softboxes, key light intensity 20–25, environment base 0.004–0.008.
6. Scene props are optional per Chapter 9; do not include debug/, camera, or board geometry.
7. JSON is UTF-8; follow the given schema strictly; extra fields may be ignored.
```

### 12.3 Self-check after generation

```text
□ Structure: root contains only material_table.json / inner_material_table.json / param.json / *.hdr / *.png
□ Palette: array is non-empty; every phong.diffuse keeps its original encoding
□ Built-in materials: every key exists in the engine dictionary
□ Values: all color components 0–1; metallic / roughness 0–1
□ Metallic entries have metallic = 1; solder mask iorLevel ≤ 0.3
□ References: every *Texture.name and background.texture has a matching file in the pack
□ Textures: every atlas texture is 2048×2048
□ Look: after initialization — solder mask is dark with a lacquer-like sheen, pads have metallic reflections, component bodies are near-black, shadows are not dead black
```

---

## 13. Verification and tuning

### 13.1 Verification workflow

| Step | Action | Notes |
|---|---|---|
| 1 | Open the PCB 3D preview page | Ray tracing depends on it and cannot initialize while it is closed |
| 2 | Menu "Enable ray tracing" | Make sure the pack is loaded |
| 3 | Observe the image | A gradually cleaner image means sampling is accumulating |
| 4 | Wait 2–5 minutes, then Save As PNG | Check material depth and noise |
| 5 | Repeat 1–4 on another board | Confirm no banding, misalignment or stray props |

### 13.2 Tuning order (by impact, high to low)

| Order | Adjustment | Impact |
|---|---|---|
| 1 | Background `intensity` | Overall exposure; tune this first |
| 2 | HDR softbox intensity / soft edge | Light-dark structure and shadow softness |
| 3 | Solder mask `iorLevel` + `clearcoat` | Lacquer depth / whether it washes out |
| 4 | Palette classification thresholds | Whether bodies are black enough and pins look metallic |
| 5 | `viewTransform` | Standard is flatter; ACES is more photographic |
| 6 | `maxRayBounces` / samples | Only affects image cleanliness |

### 13.3 Diagnosing common symptoms

| Symptom | Check first |
|---|---|
| Whole board washed out / gray | Did you forget sRGB→linear? Is `intensity` too high? |
| Solder mask uniformly glossy | Is `iorLevel` ≥0.5? Is the HDR key too bright? |
| Pads look like yellow plastic | Palette lacks metallic entries, or `material_table.json` is missing |
| Component bodies are pale gray | Same as above — they fell through to `fromPhong`'s metallic 0 / roughness 1 |
| Entire image black | HDR filename mismatch / HDR not in the pack / `hide:true` with no other light |
| Board turns red / garbled after import | Hot-reload index mismatch; use dispose→init, then import |
| Extra components / props appear | The pack includes `scene_model/` |

---

## 14. Common pitfalls (summary)

| Symptom | Cause | Fix |
|---|---|---|
| Materials all black | Missing texture or wrong name; empty `material_table` | Verify filenames; ensure the array is non-empty |
| Textures gray / blown out | Base color not handled as sRGB (or vice versa) | Base color sRGB; normal/roughness/metallic linear |
| Bump direction reversed | Normal map Y axis flipped | Use +Y up (OpenGL) |
| Built-in materials not applied | Key spelling does not match the engine dictionary | See the list in 4.2 |
| Huge pack | 8K textures / RGBA / unoptimized | Keep 2048 + grayscale + strip alpha + PNG level 9 |
| `Texture size mismatch` | Texture is not 2048×2048 | Restore to 2048 (atlas unit is fixed at 2048) |
| Settings lost after restart | Not using the production path | Export with `exportPackage` and update the URL constant |
| Colors uniformly brighter | Colors written as sRGB | Convert to linear (Chapter 8) |
| Materials misaligned after hot reload | Indices are already baked | Load at initialization / dispose→init |
| Duplicate geometry when importing another board | Pack contains `scene_model/` | Remove the directory and manifest entries if props are not wanted |
| `init()` never returns | 3D preview page not open | Open the 3D preview first |
| Exported image size wrong | Resolution = canvas size | Fix the window size |
| Light changes have no effect | Runtime `light` has no effect in 4.1.109 | Rely mainly on HDR lighting |
| `Error: this.currentTask !== null` | Repeated `init()` calls: `startRender()` sees the render loop already running (no idempotency guard) | `dispose()` and wait before `init()`; or close/reopen the 3D page; avoid concurrent calls |

---

## 15. Release checklist

| Step | Action | Notes |
|---|---|---|
| 1 | Update `raytracer-assets/assets` | Put in the new material tables and textures |
| 2 | Run `npm run build:prod` | Produces the zip |
| 3 | Upload to `modules.lceda.cn/raytracer-assets/<version>.<hash>/` | Version directory and hash must correspond |
| 4 | Edit `eext-common/packages/raytracer/index.ts` | `RESOURCE_PACK_URL`, `RESOURCE_PACK_VERSION`, `RESOURCE_PACK_SIZE` (progress percentage depends on it; must stay in sync) |
| 5 | Rebuild the extension | Activates the new URL |
| 6 | Verify on two boards | Solder mask / substrate / silkscreen / pads / bodies look right, and no extra geometry |

---

## Appendix: versions used for this document

| Component | Version |
|---|---|
| EasyEDA Pro | 4.1.109 |
| smt-gl-engine | 4.1.44 |
| pro-api | 0.3.84 |

Interface behavior (especially `light`, `camera`, and `importPackage` index semantics) may change between versions — re-verify per Chapter 13 after upgrading.
