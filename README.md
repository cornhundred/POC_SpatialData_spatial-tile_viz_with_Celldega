# SpatialData tiles with Celldega

This repository is a minimal proof of concept for proposed SpatialData storage changes. It loads a tiled SpatialData store, checks its geometry and CSC encodings, and visualizes it directly with Celldega `0.26.0a1` through a local HTTP server. It intentionally does not cover how the modified dataset was produced.

see example dataset: https://huggingface.co/datasets/cornhundred/SpatialData_with_spatial_tiles/tree/main

## Proposed storage changes

The SpatialData logical model is unchanged: the store still contains standard images, points, shapes, and an AnnData table and can be opened with `spatialdata.read_zarr`. The proposal changes or more narrowly specifies parts of their on-disk representation to support efficient visualization.

| Area | Current SpatialData convention | Proposed profile demonstrated here |
| --- | --- | --- |
| Spatial indexing | Points and shapes are stored in Parquet, but there is no standard mapping from a spatial region to particular files or row groups. | Assign points and shapes to a shared regular grid, store each logical tile in the corresponding Parquet row group, and record the grid-to-file mapping in `spatial_tiling` metadata (zarr.json). |
| Transcript columns | Point coordinates use standard `x` and `y` columns; no projection-oriented physical column order is prescribed. | Keep the standard point representation while placing commonly projected columns (`x`, `y`, and `feature_name`) first (for more efficient parquet_wasm column projection/filtering). |
| Cell geometries | Shapes may use WKB or the already-supported GeoArrow geometry encoding; GeoArrow is not required by default. | Store cell boundaries as native `geoarrow.polygon` columns so Arrow-compatible clients can read them without decoding WKB. |
| Expression matrix | AnnData permits CSR, CSC, dense matrices, and additional layers, but SpatialData does not prescribe a gene-major representation. | Keep the conventional CSR `X` matrix and add the equivalent CSC matrix as `layers["X_csc"]` for efficient access to individual genes. |

The column ordering, GeoArrow encoding, and CSC layer use capabilities already available in the underlying formats. The new convention introduced by this POC is the shared spatial tile and Parquet row-group mapping.

## Demo


https://github.com/user-attachments/assets/050be217-f7e6-496d-b11f-ce489be64f69

* zooming into tissue renders transcripts and cell boundaries using row-group Parquet spatial tiles
* clicking genes colors cells by their gene expression counts using the CSC layer

## Requirements

- macOS or Linux
- [`uv`](https://docs.astral.sh/uv/)
- [`hf`](https://huggingface.co/docs/huggingface_hub/guides/cli)
- Python 3.11 or 3.12 (managed automatically by `uv`)

## Install

From this directory:

```bash
uv sync
```

This creates `.venv`, installs the exact alpha release `celldega==0.26.0a1`, and installs JupyterLab. Confirm the version with:

```bash
uv run python -c "import celldega; print(celldega.__version__)"
```

## Download only `skin_adapt_v2.zarr`

The command below writes into this project instead of the global Hugging Face cache snapshot directory:

```bash
hf download cornhundred/SpatialData_with_spatial_tiles \
  --type dataset \
  --include 'skin_adapt_v2.zarr/**' \
  --local-dir data \
  --max-workers 1
```

The result is `data/skin_adapt_v2.zarr`. Downloads are resumable; rerun the same command after an interruption. This Zarr has many small files, so anonymous requests may hit a `429` rate limit. If that happens, authenticate once and rerun:

```bash
hf auth login
```

Do not add a trailing `.` to `hf download` to select an output directory. Use `--local-dir` explicitly. Without it, `hf` intentionally returns a path inside `~/.cache/huggingface/hub/`.

To inspect or remove a cached copy without manually deleting cache internals:

```bash
hf cache list --revisions
hf cache rm dataset/cornhundred/SpatialData_with_spatial_tiles --dry-run
hf cache rm dataset/cornhundred/SpatialData_with_spatial_tiles
```

## Run the notebook

```bash
uv run jupyter lab celldega_spatialdata.ipynb
```

Run the cells in order. The notebook:

1. loads `data/skin_adapt_v2.zarr` with `spatialdata.read_zarr`;
2. shows that transcripts use the NGFF points encoding with `x`/`y` columns and cell boundaries use the `geoarrow.polygon` Arrow extension;
3. confirms the additional CSC cell-by-gene layer;
4. starts Celldega's local server and creates a `dega.viz.Landscape` directly from the store URL.

The server uses an available ephemeral port and stops when the notebook kernel exits.
