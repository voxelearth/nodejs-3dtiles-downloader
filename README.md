# Google Maps 3D Tiles Downloader — **Fast Node.js**

Download Google Maps 3D Tiles as `.glb` files around a lat/lng + radius, **then recenter, align “Up”, and bake transforms** so the glTF nodes end up with translation-only. This is a from-scratch Node.js translation of Lukas Lao Beyer’s original Python tool, with a focus on speed, reliability, and drop‑in output for DCC tools (Blender, etc.).

> ⚠️ **You need a Google Maps Platform API key with the 3D Tiles API enabled.** Make sure your usage complies with Google’s Terms of Service. The Elevation API is optionally used to estimate ground height for better culling.

---

## Install

```bash
# Node 18+ recommended
npm i
```

## Quick start

```bash
# Example: 800 m around (lat: 42.350148, lng: -71.069929), 16 parallel downloads
node download_and_rotate.js \
  --key $GOOGLE_MAPS_KEY \
  --lat 42.350148 --lng -71.069929 \
  --radius 800 \
  --out ./tiles \
  --parallel 16
```

**What you’ll get**

* A `tiles/` folder full of `.glb` files named by SHA‑1 (stable per tile).
* Logs like `ORIGIN_TRANSLATION [...]`, `ASSET_COPYRIGHT`, and `TILE_TRANSLATION` for each tile.
* Summary line `DOWNLOADED_TILES: ["....glb", ...]`.

> Tip: First run determines a shared **origin** from the first tile; subsequent tiles reuse it so the set is nicely aligned. You can override the origin; see `--origin` below.

---

## CLI

```text
--key       (string, required) Google Maps 3D Tiles API key
--lat       (number, required) Latitude  in degrees
--lng       (number, required) Longitude in degrees
--radius    (number, required) Radius in meters
--out       (string, required) Output directory
--parallel  (number, default: 10) Parallel download concurrency
--origin    (three numbers) ECEF origin x y z. Example:
            --origin 3383551.7246 2624125.9925 -4722209.0962
--help
```

### Examples

Minimal (10 parallel downloads):

```bash
node download_and_rotate.js \
  --key $GOOGLE_MAPS_KEY \
  --lat 37.7749 --lng -122.4194 \
  --radius 600 \
  --out ./sf_tiles
```

Provide your own origin (useful to keep multiple pulls aligned):

```bash
node download_and_rotate.js \
  --key $GOOGLE_MAPS_KEY \
  --lat 48.8583 --lng 2.2945 \
  --radius 1000 \
  --out ./paris \
  --parallel 24 \
  --origin 4200934.5 172560.7 4780095.3
```

---

## How it works (high level)

1. **Find ground elevation (optional):** Tries the Elevation API to better center the culling sphere.
2. **ECEF region sphere:** Converts lat/lng/(elev) → Earth‑fixed XYZ, radius in meters.
3. **Tileset BFS:** Starts at `.../v1/3dtiles/root.json`, follows `content/contents`, carrying over `key` + `session`.
4. **Culling:** Each node’s `boundingVolume.box` → approximate sphere; intersect with region sphere to prune.
5. **Leaves:** For `.glb` leaves, download; for nested tilesets, enqueue parse.
6. **Rotate & bake:** `rotateUtils` recenters to a shared origin, rotates geodetic Up → +Y, bakes rotation/scale into vertex data, normal/tangent corrected.
7. **Stable naming:** Tile URL → SHA‑1 filename. Existing files are skipped on rerun.

---

## Programmatic API (rotate step)

If you want to rotate/bake a GLB yourself:

```js
// CommonJS
const { rotateGlbBuffer } = require('./rotateUtils.cjs');

(async () => {
  const inBuf = fs.readFileSync('tile.glb');
  const { buffer: outBuf, positions, originUsed, scale } =
    await rotateGlbBuffer(inBuf, { origin: null, scaleOn: false });
  fs.writeFileSync('tile.rotated.glb', outBuf);
})();
```



## Output details

* **Filenames:** `<sha1>.glb`, where the hash is computed from the tile URL **without** `key`/`session` so it’s stable and cacheable.
* **GLB contents:** Root nodes have translation only (rotation = identity, scale = 1). Geometry bounds are updated.
* **Logs:**

  * `ORIGIN_TRANSLATION [x,y,z]` on first tile (or when you pass `--origin`).
  * `ASSET_COPYRIGHT <file> <copyright>` if present in the GLB.
  * `TILE_TRANSLATION <file> [x,y,z]` per tile.
  * `DOWNLOADED_TILES: [ ... ]` at the end for workflow scripting.

---

## Tips for speed & reliability

* Increase `--parallel` on fast networks/CPUs (e.g. 24–64). Watch for quota and local bandwidth limits.
* Put the output on a fast SSD.
* If you plan huge areas, run in smaller tiles or rings to keep the working set small and resumable.
* For very large jobs, you can raise Node’s memory limit: `node --max-old-space-size=4096 download_and_rotate.js ...`.

---

## Troubleshooting

* **403/404/429** → Check your API key, referrer or IP restrictions, and that the 3D Tiles API is enabled. Also watch daily quotas.
* **`Elevation API error`** → We fall back to elevation 0. This only affects culling center; downloads still work.
* **`This GLB uses KHR_draco_mesh_compression. Install the decoder: npm i draco3d`** → Install `draco3d` (already in `package.json`).
* **`EXT_meshopt` not supported** → Use tools like `gltfpack`/`meshoptimizer` to re‑export without meshopt, or decode first.
* **`Mesh [...] referenced by multiple nodes`** → Our baking step requires a 1:1 node→mesh mapping. Duplicate the mesh per node (e.g. with glTF-Transform) before baking.

---

## Notes & limitations

* Vertex attributes must be uncompressed **FLOAT** accessors (sparse not supported for baking). Draco is auto‑decoded; **EXT_meshopt** is not.
* Skinned meshes (nodes with `skin`) aren’t supported by the baking routine.
* This tool is for **technical workflows**; ensure your use of Google data follows their ToS/licensing.

---

## Migrating from the Python CLI

Original Python (example):

```bash
python -m scripts.download_tiles -k <API_KEY> -71.069929 42.350148 -r 800 -o tiles
```

Node.js equivalent (lat/lng are named flags, plus optional `--parallel`):

```bash
node download_and_rotate.js \
  --key <API_KEY> \
  --lat 42.350148 --lng -71.069929 \
  --radius 800 \
  --out tiles \
  --parallel 16
```

---

## Why this is **much** faster than the original Python script

This Node.js version is designed for high‑throughput networking and parallel IO, and it makes a few pragmatic trade‑offs that drastically reduce end‑to‑end time:

1. **Aggressive parallelism**
   *Tileset discovery* (JSON traversal) runs concurrently, and *downloads* run concurrently with a configurable `--parallel` concurrency (defaults to 10). Node’s event loop + non‑blocking IO shines here.

2. **HTTP/HTTPS keep‑alive**
   Reuses sockets across many small requests (`http.Agent`/`https.Agent` with `keepAlive: true`), removing connection setup overhead on every tile.

3. **Coarse, cheap culling**
   Converts each tileset `boundingVolume.box` to an **approximate sphere** and intersects it against a **single region sphere** (your lat/lng/radius in ECEF). This prunes entire subtrees quickly with minimal math and fewer requests.

4. **Session reuse across the tree**
   Propagates Google’s `session` query param when encountered, minimizing auth churn, redirects, and cache misses.

5. **Idempotent, cache‑friendly output**
   Every `.glb` is named by a **SHA‑1 of its tile URL (sans key/session)**. Reruns **skip** files that already exist and simply log the metadata—great for incremental pulls.

6. **In‑process geometry processing**
   The `rotateUtils` step decodes **KHR_draco_mesh_compression** (via `draco3d`) and directly rewrites vertex buffers in memory. No shelling out, no temp files, no external CLI.

7. **Baked transforms = less post‑work**
   Global recenter + “geodetic Up → +Y” rotation gets **baked into vertex data**. Root nodes end up with identity rotation/scale (translation only), which loads faster and is easier to compose in downstream tools.

> Real‑world speedups vary by network + scene complexity, but these changes usually slash both the count of requests and the cost per request, while avoiding redundant work between runs.

---

## Acknowledgement

This project is a Node.js translation inspired by and adapted from **Lukas Lao Beyer’s** work: *3dtiles-dl*. Huge thanks to Lukas for the original idea and reference implementation.
