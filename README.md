# MeshCore LoRa Relay Node Placement Tool
### Automated Emergency Network Design Along Virginia Hospital Corridors
> GIS 201 Final Project — Northern Virginia Community College, Spring 2026

---

## Overview

This project automates the placement of **MeshCore LoRa 915MHz** radio relay nodes along Virginia road corridors connecting hospitals. In disaster scenarios where cellular and internet infrastructure fails, a pre-planned mesh radio network enables emergency communications between medical facilities.

The tool processes 117 Virginia hospitals and 658,369 road segments to generate deployment-ready relay node placements, exported to KMZ for Google Earth field planning.

**[View in Google Earth](https://earth.google.com/earth/d/1ZBXHAT9JhQ6n_1q0mtfqxCBKBiCBAp9w?usp=sharing)**

---

## Radio Specifications

| Parameter | Value |
|-----------|-------|
| Protocol | MeshCore (NOT Meshtastic) |
| Frequency | 915MHz LoRa |
| Antenna | 30-foot pole-mounted, omni |
| Node Spacing | 3 miles (worst-case) |
| Design Conditions | Monsoon + dust storm attenuation |
| Coverage | 117 Virginia hospitals |

The 3-mile spacing is a conservative worst-case interval for 915MHz LoRa on pole-mounted antennas. Theoretical max is 10-15 miles line-of-sight under ideal conditions, but Virginia's Appalachian terrain and extreme weather conditions require significant margin.

---

## Results

| Metric | Value |
|--------|-------|
| Virginia hospitals | 117 |
| Road segments in network | 658,369 |
| Relay nodes generated | 600 |
| Manually digitized beam links | 19 |
| Estimated deployment cost | ~USD 202,800 |

---

## Data Sources

| # | Dataset | Source | Use |
|---|---------|--------|-----|
| 1 | HIFLD Hospitals and Medical Centers | [hub.arcgis.com](https://hub.arcgis.com/datasets/fedmaps::hospitals-medical-centers/explore) | 117 Virginia hospital locations |
| 2 | Virginia Road Centerlines 2026 Q1 | [vgin.vdem.virginia.gov](https://vgin.vdem.virginia.gov/datasets/cd9bed71346d4476a0a08d3685cb36ae/about) | 658,369 road segments for routing |
| 3 | Virginia State Boundary KML | [files.boundmaps.com](https://files.boundmaps.com/us/states/va) | Clip polygon for Virginia extent |
| 4 | Google Earth Pro | earth.google.com | KMZ visualization |
| 5 | BEAM_MESH_NODES | Manually digitized — ArcGIS Pro Editor | 19 inter-cluster beam links |
| 6 | World Topographic Basemap | ESRI ArcGIS Online | Background map |

---

## Methodology

### 1. Hospital Data Preparation
- Downloaded HIFLD national hospitals CSV
- Converted to spatial points using **XY Table To Point** (x/y fields, GCS WGS 1984)
- Clipped to Virginia using **Pairwise Clip** with KML-derived state boundary
- Result: 117 Virginia hospitals

### 2. Road Network Construction
- Copied VDOT VA_CENTERLINE (658,369 segments) into a Feature Dataset (NAD 1983 Virginia Lambert)
- Built routable **Network Dataset** (VA_Roads_ND) with Length travel mode
- Impedance: Shape_Length in meters — shortest distance routing
- All road types included — no filtering (hospitals near local roads must stay reachable)

### 3. Closest Facility Network Routing
- Used **Make Closest Facility Analysis Layer** with:
  - Travel Mode: Length (meters)
  - Cutoff: 500,000 meters (covers all of Virginia)
  - Facilities to Find: 2
- Loaded hospitals as **both Facilities AND Incidents** (same layer)
- This enables nearest-neighbor routing: each hospital searches for its nearest hospital via road

#### The Self-Match Problem and Fix
Since the same layer is used for both Facilities and Incidents, each hospital finds itself as rank 1 (zero length). Two settings fix this:

| Fix | Setting | Why |
|-----|---------|-----|
| 1 | Cutoff = 500,000m | Default cutoff was too small — real neighbors were being excluded |
| 2 | Facilities to Find = 2 | Forces solver to find rank 2 (real neighbor), not just rank 1 (self) |

Zero-length self-match routes produce zero points in Generate Points Along Lines and are effectively ignored in the output.

### 4. Node Placement
- **Generate Points Along Lines** at 3-mile intervals with endpoints included
- Output: `MESH_NODES` feature class (600 relay nodes)

### 5. Manual Beam Link Digitization
- Created `BEAM_MESH_NODES` as a Line feature class
- Manually digitized 19 long-distance directional beam links using ArcGIS Pro Editor
- These bridge geographically separated hospital clusters that nearest-neighbor routing doesn't connect
- Estimated range per beam link: 15-30 miles with directional 915MHz Yagi/panel antenna

### 6. Export
- Exported to KMZ format for Google Earth visualization
- Both MESH_NODES and BEAM_MESH_NODES exported as separate KMZ files

---

## ModelBuilder Pipeline

The entire workflow (except manual digitization) runs as an automated ModelBuilder model with 16 geoprocessing tools:

```
CSV → XY Table To Point → Hospitals
virginia.kml → KML To Geodatabase → virginia_boundary
Hospitals + virginia_boundary → Pairwise Clip → Hospitals_VA_Boundary (117)

VA_CENTERLINE → Copy Features → VA_Roads (in Road_Network feature dataset)
GDB → Create Feature Dataset → Road_Network
Road_Network → Create Network Dataset → VA_Roads_ND
VA_Roads_ND → Build Network

VA_Roads_ND → Make Closest Facility Analysis Layer → Closest Facility
Hospitals_VA_Boundary → Add Locations (Facilities) → Closest Facility
Hospitals_VA_Boundary → Add Locations (Incidents) → Closest Facility
Closest Facility → Solve → Network Analyst Layer
Network Analyst Layer → Select Data (Routes) → Routes
Routes → Generate Points Along Lines (3 miles) → MESH_NODES
MESH_NODES → Layer To KML → MESH_NODES.kmz
```

---

## Cost Estimate

| Component | Count | Unit Cost | Total |
|-----------|-------|-----------|-------|
| Relay nodes (Heltec LoRa32/LilyGo + pole + omni antenna) | 600 | USD 300 | USD 180,000 |
| Beam links (directional Yagi/panel, 2 radios per link) | 19 | USD 1,200 | USD 22,800 |
| **Total** | | | **~USD 202,800** |

---

## File Structure

```
GIS201-FINAL-Project-Meshcore/
├── GIS201-FINAL-Project-Meshcore.gdb/
│   ├── Road_Network/
│   │   ├── VA_Roads           (VDOT road centerlines)
│   │   └── VA_Roads_ND        (Network Dataset)
│   ├── Hospitals              (all US hospitals)
│   ├── Hospitals_VA_Boundary  (117 Virginia hospitals)
│   ├── virginia_boundary      (Virginia polygon)
│   ├── MESH_NODES             (600 automated relay nodes)
│   └── BEAM_MESH_NODES        (19 manual beam links)
├── Data/
│   ├── POI/
│   │   └── Structures_Medical_Emergency_Response_v1_*.csv
│   ├── Boundary/
│   │   └── Hospitals_VA_Boundary.shp
│   └── Road-centerlines/
│       └── Virginia_RCL_Dataset_2026Q1.gdb
└── virginia.kml
```

---

## Future Enhancements

- **Minimum Spanning Tree** — connect all hospital clusters into a fully linked network, eliminating inter-cluster gaps
- **Adaptive Node Spacing** — use USGS DEM elevation and NLCD land cover for variable spacing (tighter in mountains, wider on open highways)
- **Greedy Interval Algorithm** — Python script (`place_mesh_nodes.py`) in progress using `LOCAL_SPEED_MPH` as terrain proxy for dynamic hop ranges
- **Extended Coverage** — integrate HIFLD fire stations and emergency services data

---

## Tools Used

ArcGIS Pro, ModelBuilder, Network Analyst, Google Earth Pro

---

## References

- Federal Emergency Management Agency. *Hospitals and Medical Centers.* https://hub.arcgis.com/datasets/fedmaps::hospitals-medical-centers/explore
- Virginia Department of Emergency Management. *Virginia Road Centerlines (RCL) Dataset, 2026 Q1.* https://vgin.vdem.virginia.gov/datasets/cd9bed71346d4476a0a08d3685cb36ae/about
- BoundMaps. *Virginia State Boundary KML.* https://files.boundmaps.com/us/states/va
- Google LLC. *Google Earth Pro.* https://earth.google.com
- Wikipedia. *MeshCore.* https://en.wikipedia.org/wiki/MeshCore
