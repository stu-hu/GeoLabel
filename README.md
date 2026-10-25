# GeoLabel Studio

**A Remote Sensing and Emergency Annotation Workbench**

GeoLabel Studio is a locally deployable annotation workbench built with **FastAPI and React/Vite**. It is designed for remote sensing imagery, UAV imagery, disaster assessment, semantic segmentation, geospatial annotation, and vision-language dataset construction.

The workflow is straightforward:

**Import imagery and metadata → Normalize annotations → Edit on the original image → Compare changes → Save revisions → Export a reproducible dataset package.**

The application uses an interactive SVG canvas to maintain accurate image-coordinate mapping during zooming and panning. The backend is the authoritative implementation for rasterizing annotation geometry into segmentation masks.

## Features

### 1. Annotation and Image Editing

- Interactive polygon and vertex editing.
- Vertex insertion, deletion, dragging, and geometry replacement.
- Grouped vertex selection and movement.
- Local boundary correction, point addition, and polygon closure.
- Brush-based mask painting and erasing.
- Undo and redo, with a maximum of 50 history steps.
- Category and annotation metadata management.
- Automatic validation of geometry, holes, category references, and image boundaries.
- Original annotation baselines and revision history.

### 2. Image Comparison and Revision Management

- Compare the current annotation against the initial imported annotation.
- View original imagery with annotation overlays.
- Generate pixel-level difference visualizations:
  - Green: newly added foreground.
  - Red: removed foreground.
  - Yellow: foreground category changes.
- Save revisions without modifying the immutable initial baseline.
- Restore historical annotations as drafts and save them as new revisions.
- Detect conflicting updates through revision numbers and optimistic concurrency control.

### 3. Image Library and Mosaic Redaction

- Import and organize image samples.
- Move images to the recycle bin, undo deletion, and restore saved annotations.
- Create independent rectangular mosaic regions.
- Configure multiple mosaic rectangles and block sizes.
- Support undo and redo for mosaic editing.
- Export redacted images using their original filenames.
- Use redacted imagery for AI-assisted text input.

Mosaic redaction is independent of semantic annotation. Mosaic rectangles do not create semantic labels, polygon regions, or segmentation masks.

### 4. AI-Assisted Annotation

The application supports integration with locally hosted vision-language and segmentation models.

- Select available language-model configurations.
- Generate bilingual visual question-answering (VQA) candidates.
- Generate text-guided or box-guided segmentation candidates.
- Review segmentation proposals before accepting them.
- Convert accepted SAM candidates into editable polygon vertices.
- Inspect and revise generated questions, answers, and reasoning text.
- Maintain separate model inference and annotation-editing workflows.

Model inference is designed to start on demand and release GPU memory after a configurable idle period. Available models depend on the deployment configuration.

For operational details, see the deployment and service documentation maintained with the project.

### 5. Interface Themes and Navigation

Four interface themes are available:

- Bright Cloud
- Eye-Friendly Teal
- Warm Paper
- Deep Ocean

The selected theme is remembered automatically.

Additional interaction features include pointer-centered scroll zooming, region focusing, synchronized comparison viewports, and independent bilingual VQA editing.

---

## Quick Start

### Requirements

- Python 3.11–3.13
- Node.js 22.12 or later
- A supported operating system with Python and Node.js installed

### Windows

From the project root, run:

```powershell
./start.ps1 -Install -Demo
```

Open the application at:

- Web interface: `http://127.0.0.1:8000`
- API documentation: `http://127.0.0.1:8000/docs`

For subsequent launches, run:

```powershell
./start.ps1
```

The `-Demo` option creates a synthetic demonstration sample. This sample is for interface testing and is **not real remote sensing observation data**.

PowerShell execution policy settings may need to permit running local scripts.

### Manual Setup

Create a Python virtual environment and install the backend dependencies:

```powershell
python -m venv .venv

./.venv/Scripts/python -m pip install -r backend/requirements.txt

./.venv/Scripts/python scripts/export_schema.py

./.venv/Scripts/python scripts/demo.py --seed

./.venv/Scripts/python -m uvicorn backend.app:app --host 127.0.0.1 --port 8000
```

In a separate terminal, start the frontend development server:

```powershell
cd frontend
npm install
npm run dev
```

The development interface is available at `http://127.0.0.1:5173`, with API requests proxied through Vite.

For a locally served production build:

```powershell
cd frontend
npm run build
```

Restart the backend to serve the built frontend from `frontend/dist`.

For macOS and Linux, replace the Windows virtual-environment executable path with the appropriate path for the current platform.

---

## Architecture

The project separates the annotation interface, authoritative data model, image processing, persistence, and dataset export.

### Main Components

| Component | Responsibility |
|---|---|
| `backend/app.py` | File uploads, batch imports, sample retrieval, annotation updates, revision history, previews, diffs, and ZIP exports |
| `frontend/src/main.jsx` | Upload forms, image overlays, vertex editing, brush tools, comparisons, categories, and text annotation |
| `backend/models.py` | Authoritative annotation data models and validation |
| `schema/sample.schema.json` | Generated JSON Schema for the unified sample format |
| `GET /api/schema` | Provides the current schema through the API |

### Recommended Repository Layout

```text
GeoLabel-Studio/
├── README.md
├── LICENSE
├── backend/
│   ├── app.py
│   ├── models.py
│   └── requirements.txt
├── frontend/
│   ├── src/
│   │   └── main.jsx
│   └── package.json
├── schema/
│   └── sample.schema.json
├── scripts/
│   ├── demo.py
│   └── export_schema.py
├── services/
│   └── README.md
├── docs/
│   ├── VQA-AND-GEOMETRY.md
│   ├── THEMES-AND-RUNTIME.md
│   ├── MOSAIC.md
│   ├── IMAGE-LIBRARY.md
│   └── VALIDATION.md
└── examples/
```

The paths above describe the main project components. Additional files may be present in the complete repository.

---

## Supported Image Formats

The current prototype supports:

- JPEG
- PNG
- WebP
- BMP
- TIFF and GeoTIFF
- JPEG 2000 (JP2)

Supported geospatial workflows depend on the image metadata and installed processing libraries.

The following are not currently supported as complete native workflows:

- HEIC
- Camera RAW formats
- Video
- Point clouds
- ENVI products
- HDF and NetCDF
- Complex multi-file sensor products

### Image Display and Geospatial Metadata

- Single-band images are displayed as grayscale.
- TIFF and JP2 previews use the first one to three bands.
- Display bands are stretched using the 2nd and 98th percentiles.
- Original image files and their geospatial metadata are preserved.
- Display previews are derived artifacts and are not substitutes for the original imagery.

The current prototype does not implement SAR calibration, configurable multispectral band combinations, or image registration.

Automatic EXIF rotation is not applied to ordinary images, ensuring that existing annotations remain aligned with encoded pixel coordinates.

### Image Size Limits

The current prototype imposes the following limits:

- Maximum image size: 32 million pixels.
- Maximum individual file size: 128 MB.
- Maximum expanded batch archive size: 512 MB.
- Maximum samples per batch archive: 500.
- Maximum polygonized regions for highly fragmented masks: 30,000.

For larger orthomosaics, Cloud Optimized GeoTIFF (COG) pyramids and tile-based image access should be implemented instead of loading the entire image into browser memory.

---

## Unified Annotation Data Model

The unified JSON representation is the authoritative editable format. Dataset-specific adapters convert source annotations into this model and export them into supported target formats.

A simplified example:

```json
{
  "schema_version": "1.0",
  "id": "managed-id",
  "profile": "custom",
  "revision": 0,
  "coordinate_space": "pixel-edge",
  "image": {
    "filename": "uav.jpg",
    "width": 4000,
    "height": 3000,
    "crs": null,
    "transform": null
  },
  "annotation": {
    "categories": [
      {
        "id": 0,
        "name": "Background",
        "color": "#182534"
      },
      {
        "id": 1,
        "name": "Road blocked",
        "color": "#e45b70"
      }
    ],
    "regions": [
      {
        "id": "r1",
        "category_id": 1,
        "rings": [
          [
            [100, 100],
            [450, 100],
            [450, 300],
            [100, 300]
          ]
        ],
        "properties": {
          "damage": "major"
        }
      }
    ],
    "strokes": [],
    "image_labels": ["flood"],
    "instruction": "Mark the blocked road.",
    "response": "A road blockage is visible in the upper-left region.",
    "properties": {
      "pair_id": "event001"
    }
  },
  "source": {
    "dataset": "local"
  }
}
```

This is a simplified structural example rather than a complete schema specification.

### Data Model Conventions

- The origin is at the upper-left corner.
- The x-axis points right; the y-axis points down.
- Coordinates use pixel-edge space: `[0, width] × [0, height]`.
- Coordinates are not normalized.
- Polygon arrays do not repeat the first point at the end.
- `rings[0]` represents the exterior ring; subsequent rings represent holes.
- Category IDs must be unique, and background category ID `0` must exist.
- Every region must reference an explicitly defined category.
- Original metadata is preserved in `source.metadata` and annotation properties.
- Region-specific attributes are preserved in `regions[].properties`.

Pydantic validation supplements JSON Schema validation with geometry checks, hole validation, category-reference validation, finite-coordinate checks, and image-boundary checks.

### Coordinate Mapping and Rasterization

The SVG canvas converts screen coordinates to image coordinates using the inverse screen transformation matrix obtained through `getScreenCTM().inverse()`.

Zooming and panning therefore do not modify the underlying annotation coordinates.

The backend rasterizes annotation geometry into single-channel masks using the image's original dimensions. Regions are rasterized in sequence, with later drawing operations taking precedence over earlier ones. Brush strokes are rendered above polygon regions, and erasing writes the background value.

Vector regions and brush strokes are both retained in the editable annotation model, allowing masks to be regenerated during export.

Mask-to-polygon conversion uses `rasterio.features.shapes`. Polygon holes and disconnected components are preserved where representable.

The current implementation does not automatically infer RGB color-mask semantics or the meaning of an ignore value such as `255`.

---

## Dataset Import and Export

Dataset support is implemented through separate components:

1. Label profiles define available categories.
2. Format adapters parse source files.
3. Native dataset adapters map annotations to a specific dataset release or training pipeline.

Support for a dataset name does not imply complete compatibility with every release, training script, or annotation convention.

### Current Adapter Coverage

| Dataset / Project | Current Prototype Support | Remaining Work |
|---|---|---|
| RescueNet | Official semantic IDs 0–10, single-channel or palette-indexed PNG masks, editable polygons, original ID-mask export | Native directory manifests and training-pipeline integration |
| DisasterM3 | Example disaster-level profiles, xBD pixel-coordinate WKT import, unified polygons and text, generic VLM conversation export | Release-specific temporal image paths, task fields, and training-script compatibility |
| DORA | Preservation of original JSON properties, unified region and instruction/answer
