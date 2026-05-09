# PanoStreetView — Geospatial Imagery & Orientation Pipeline

**Interactive QGIS plugin that turns vector panorama metadata + equirectangular imagery into a local, browser-served viewing experience with correct heading (yaw) and scene navigation.**

This repository is framed as a **small end-to-end data pipeline**: spatially indexed GIS features drive runtime JSON contracts, a lightweight HTTP serving layer exposes assets and metadata, and a web client (Pannellum) consumes those contracts for visualization. It demonstrates skills that transfer directly to data engineering—**source integration, indexing for low-latency lookup, deterministic transformations, schema-like payloads, and operational glue between batch and interactive workloads**.

---

## Core Data flows

| Theme | Functionality |
|--------|---------------------------|
| **Data sources** | Vector layer (shapefile) + filesystem imagery; explicit attribute contract (`Name`, `Azimuth`). |
| **Performance** | Pre-built **spatial index** (`QgsSpatialIndex`) for nearest-neighbor queries instead of full scans. |
| **Transformations** | Map click + bearing vector → angle math → **yaw offset** vs stored azimuth; CRS handling toward WGS84 where needed. |
| **Orchestration** | QGIS UI triggers indexing, then map-tool events trigger extract → serialize → serve → open consumer. |
| **Serving layer** | Embedded **Python `HTTPServer`** with a custom handler mapping `/images/*` to the user’s image directory. |
| **Contracts** | Version-stable JSON artifacts (`azimuth_data.json`, `Image_to_copy.json`) between Python and the browser. |
| **Consumer** | Async fetch + merge in `index_view.html`, scene graph for Pannellum. |

---

## Pipeline flow


![PanoStreetView pipeline — sources → QGIS → JSON → HTTP → Pannellum](pipeline-flow-diagram.png)

---

## Architecture at a glance

```mermaid
flowchart LR
  subgraph Sources["Data sources"]
    SHP[(Shapefile points)]
    IMG[/Panorama images folder/]
  end

  subgraph QGIS["QGIS plugin (Python)"]
    IDX[Spatial index build]
    NN[k-NN query + enrich]
    XF[Yaw / azimuth transform]
    SER[JSON serialization]
  end

  subgraph Serve["Serving layer"]
    HTTP[localhost:8030 HTTPServer]
  end

  subgraph Consumer["Presentation"]
    WEB[Pannellum viewer]
  end

  SHP --> IDX
  IDX --> NN
  NN --> XF
  XF --> SER
  IMG --> HTTP
  SER --> HTTP
  HTTP --> WEB
```

---

## Detailed pipeline flow

The following diagram mirrors the actual control flow in `PanoStreetView_2.py` and `index_view.html`.

```mermaid
flowchart TB
  START([User opens plugin]) --> DIALOG[Configuration dialog]
  DIALOG --> PICK_SHP[Select shapefile path]
  DIALOG --> PICK_IMG[Select images folder path]
  PICK_SHP --> BTN_INDEX[User clicks: build index]
  BTN_INDEX --> LOAD_LAYER[Load QgsVectorLayer from shapefile]
  LOAD_LAYER --> VALID{Layer valid?}
  VALID -->|no| ERR_LAYER[Error: invalid layer]
  VALID -->|yes| LOOP_FEAT[For each feature]
  LOOP_FEAT --> SI[QgsSpatialIndex.addFeature]
  SI --> PB[Update progress bar]
  PB --> IDX_READY[Index ready in memory]

  IDX_READY --> MAP_TOOL[Activate PointTool on map canvas]
  MAP_TOOL --> CLICK1[canvasPress: record point0 + marker]
  CLICK1 --> DRAG[canvasMove: rubber-band line to point1]
  DRAG --> RELEASE[canvasRelease]

  RELEASE --> ANGLE[Compute bearing angle from segment]
  ANGLE --> PARSE[Parse clicked map coordinates]
  PARSE --> Q1[nearestNeighbor point, 1 → anchor feature]
  Q1 --> QK[nearestNeighbor point, k → neighbor set]
  QK --> ENRICH[Read Name, Azimuth, geometry per id]

  ENRICH --> W1[Write Image_to_copy.json<br/>array of neighbor metadata]
  ANGLE --> MERGE[Merge azimuth from anchor + user yaw]
  MERGE --> W2[Write azimuth_data.json<br/>index, azimuth, yaw_value]

  W2 --> PANTRY{Image file resolvable?}
  PANTRY -->|yes| DAEMON[Start HttpDaemon QThread]
  PANTRY -->|no| ERR_PAGE[Open index_error.html]

  DAEMON --> CHDIR[Serve plugin directory as doc root]
  CHDIR --> ROUTE{Request path}
  ROUTE -->|/images/*| MAP[Map to user image_folder]
  ROUTE -->|*.json, *.html| STATIC[Static files from plugin dir]

  MAP --> OPEN[Open Chrome → localhost:8030/index_view.html]
  STATIC --> OPEN

  OPEN --> FETCH[Browser: parallel fetch JSON]
  FETCH --> JOIN[Find row where index matches azimuth_data.index]
  JOIN --> SCENES[Build Pannellum scene graph + hotspots]
  SCENES --> VIEW([360° viewer with forward/back scenes])
```

## Prerequisites

- **QGIS 3.x** with Python enabled  
- **Shapefile** (or compatible OGR source) of **point** features with at least:
  - `Name` — panorama filename  
  - `Azimuth` — numeric heading in degrees  
- **Folder** of equirectangular images whose filenames match `Name`  
- **Google Chrome** at the default path in `PanoStreetView_2.py` (or adjust `chrome_path`)  


---

## How to Install

To install the plugin manually:

1. Download the ZIP file from this repository.
2. Open QGIS.
3. Go to: `Plugins` → `Manage and Install Plugins` → `Install from ZIP`.
4. Select the downloaded ZIP file and install it.

<img width="647" height="308" alt="image" src="https://github.com/user-attachments/assets/b37e33cd-2678-4244-a862-c9ca50e20a10" />

## Usage

To use this plugin:

1. Ensure your panoramic images are **geotagged** and stored in a folder (preferably `.JPG` in equi-rectangular projection).
2. Create or load photo points in a **shapefile** (EPSG:4326 recommended) that contains:
   - A `Name` field matching each image filename.
   - Accurate coordinates for each photo location.
   - You can use the QGIS plugin **ImportPhotos** (available in the QGIS plugin repository) to generate this layer.
---
<img width="1919" height="1030" alt="image-1" src="https://github.com/user-attachments/assets/7c29808c-2e59-4f26-a704-d28de0550f45" />

## Output preview

<img width="1771" height="658" alt="image-2" src="https://github.com/user-attachments/assets/42df9a5d-2001-4881-922a-350f5b38b7f1" />


## Tech stack

- **Python 3 / PyQGIS** — vector layers, spatial index, map tools, threading  
- **`json`, `math`, `http.server`** — serialization, bearing math, embedded file server  
- **Qt (`QThread`)** — non-blocking HTTP daemon from the GIS process  
- **HTML / JS** — Pannellum 2.5.6 (CDN), `fetch`, async scene assembly  

---



## Author

**Sachin Kulal** — geospatial & data-oriented | R&D
