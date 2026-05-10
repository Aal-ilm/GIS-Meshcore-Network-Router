# MeshCore LoRa Relay Node Placement Tool
### Automated Emergency Network Design Along Virginia Hospital Corridors
> GIS 201 Final Project — Northern Virginia Community College, Spring 2026

This started as a GIS final project but honestly turned into something that could actually be useful for real emergency communications planning. Given all the hospitals in Virginia, the tool automatically figures out where to place MeshCore radio relay nodes along the roads between them so they can all talk to each other when normal comms go down.

**[View in Google Earth](https://earth.google.com/earth/d/1ZBXHAT9JhQ6n_1q0mtfqxCBKBiCBAp9w?usp=sharing)**

---

## Overview

![Full ModelBuilder Pipeline](docs/screenshots/58_full_pipeline_canvas.png)

You drop in your boundary, your points of interest (hospitals in this case), and your road network. The tool figures out the nearest-neighbor road routes between all your POIs and places relay nodes at intervals along those routes based on your radio's real-world range.

No manual node placement. No guessing. Just run the model and get GPS-ready deployment coordinates.

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

The mesh is hive topology meaning there's no central hub. Every node just needs to reach its nearest neighbor and the whole network is connected. That's what makes placement so important. One gap and you lose the whole chain.

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

## How It Works

![All Virginia Hospitals Visible](docs/screenshots/24_hospitals_virginia_all_visible.png)

The pipeline runs entirely in ArcGIS Pro ModelBuilder with Network Analyst:

1. **Load hospitals:** HIFLD CSV converted to points via XY Table To Point
2. **Clip to boundary:** trim national dataset down to your study area
3. **Build road network:** VDOT centerlines built into a routable Network Dataset with distance-based travel mode
4. **Find nearest neighbors:** Closest Facility solver finds the shortest road route from each hospital to its nearest neighbor
5. **Filter routes:** removes zero-length self-matches
6. **Place nodes:** Generate Points Along Lines drops a node every 3 miles along each route
7. **Manual beam links:** 19 long-distance directional links digitized manually to bridge isolated clusters
8. **Export:** KML output for Google Earth field deployment planning

### ModelBuilder Pipeline

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

## Methodology and Tools

Every step runs inside a single ArcGIS Pro ModelBuilder pipeline. Here is each tool, what it does, and the exact parameters used.

---

### Step 1: XY Table To Point

Converts the HIFLD hospital CSV into a spatial point feature class.

![XY Table To Point ModelBuilder](docs/screenshots/01_xy_table_to_point_modelbuilder.png)

![XY Table To Point Parameters](docs/screenshots/04_xy_table_to_point_parameters.png)

| Parameter | Value |
|-----------|-------|
| Input Table | Structures_Medical_Emergency_Response_v1_-278061770013734790.csv |
| Output Feature Class | Hospitals (saved to project GDB) |
| X Field | x |
| Y Field | y |
| Coordinate System | GCS_WGS_1984 |

The CSV uses `x` and `y` as column names, not `longitude` and `latitude`. Using the wrong field names here causes all points to plot at 0,0 or not appear at all.

---

### Step 2: KML To Geodatabase

Converts the Virginia state boundary KML into a usable polygon feature class.

![KML To Geodatabase ModelBuilder](docs/screenshots/08_modelbuilder_kml_to_geodatabase.png)

![KML To Geodatabase Three Outputs](docs/screenshots/15_kml_to_geodatabase_three_outputs.png)

| Parameter | Value |
|-----------|-------|
| Input File | virginia.kml |
| Output Location | GIS201-FINAL-Project-Meshcore.gdb |
| Output Name | virginia_boundary |

This tool outputs three feature classes (points, lines, polygons). Only the polygon output is used as the clip boundary. KML To Layer was tried first but creates a folder structure instead of a direct feature class which caused downstream issues in ModelBuilder. KML To Geodatabase is the correct tool.

---

### Step 3: Pairwise Clip

Clips the national hospital dataset down to Virginia only.

![Hospitals Clipped to Virginia](docs/screenshots/16_hospitals_clipped_to_virginia.png)

| Parameter | Value |
|-----------|-------|
| Input Features | Hospitals |
| Clip Features | virginia_boundary |
| Output Feature Class | Hospitals_VA_Boundary.shp |

Pairwise Clip was used instead of regular Clip for better performance on large datasets. Result: 117 hospitals within Virginia boundary.

---

### Step 4: Create Feature Dataset

Creates a container inside the GDB to hold the road network. Network Datasets in ArcGIS Pro cannot be built directly in the GDB root, they require a Feature Dataset wrapper.

![Create Feature Dataset NAD1983](docs/screenshots/33_create_feature_dataset_nad1983.png)

| Parameter | Value |
|-----------|-------|
| Output Geodatabase | GIS201-FINAL-Project-Meshcore.gdb |
| Feature Dataset Name | Road_Network |
| Coordinate System | NAD_1983_Virginia_Lambert |

The coordinate system must match the road centerlines exactly or the network will not build correctly.

---

### Step 5: Copy Features

Copies the VDOT road centerlines into the Road_Network feature dataset so the Network Dataset can be built from them.

![VDOT Attribute Table Speed and Shape Length](docs/screenshots/21_va_centerline_attribute_table_2_speed_shape.png)

![VDOT Attribute Table MTFCC and Speed](docs/screenshots/23_va_centerline_attribute_table_4_mtfcc_speed.png)

| Parameter | Value |
|-----------|-------|
| Input Features | VA_CENTERLINE |
| Output Feature Class | Road_Network\VA_Roads |

The VDOT dataset has 658,369 road segments covering all of Virginia. All road classes are kept intentionally so the routing solver can reach hospitals that sit off major highways.

Key fields in the VDOT dataset:
- `Shape_Length` used as routing impedance
- `LOCAL_SPEED_MPH` available for future adaptive spacing
- `MTFCC` road classification (VDOT uses "US AND VA PRIMA", "LOCAL SECONDA" not Census S1100/S1200 codes)
- `ONE_WAY` directionality
- `DUAL_CARRIAGEWAY` divided highway flag

---

### Step 6: Create Network Dataset

Builds the routable network topology from VA_Roads.

![Road Network New Network Dataset Menu](docs/screenshots/38_road_network_new_network_dataset_menu.png)

![Create Network Dataset VA Roads Checked](docs/screenshots/84_create_network_dataset_va_roads_checked.png)

| Parameter | Value |
|-----------|-------|
| Target Feature Dataset | Road_Network |
| Network Dataset Name | VA_Roads_ND |
| Source Feature Classes | VA_Roads (checked) |
| Elevation Model | No elevation |

After creation, Network Dataset Properties must be manually configured:

**Travel Attributes (Costs tab):**
- Name: Length
- Units: Meters
- Data Type: Double
- Evaluator: Shape_Length field

![Travel Mode Length No UTurns](docs/screenshots/87_travel_mode_length_no_uturns.png)

**Travel Mode:**
- Name: Length
- Type: Driving
- Impedance: Length (meters)
- Time Cost: blank
- Distance Cost: Length (meters)
- U-turns: None
- Use Hierarchy: unchecked
- Simplify Output Geometry: unchecked

Important: This tool wipes travel mode settings on every run. After the first successful build, disable these setup tools in ModelBuilder and replace with a direct variable pointing to the existing VA_Roads_ND.

---

### Step 7: Build Network

Processes the network topology so the routing solver can traverse it.

| Parameter | Value |
|-----------|-------|
| Input Network Dataset | VA_Roads_ND |

Must run after Create Network Dataset and after any changes to Travel Attributes or Travel Modes. Without this step the solver returns a "network has no network elements" error.

---

### Step 8: Make Closest Facility Analysis Layer

Configures the routing problem. Finds the nearest hospital to each hospital via actual road network paths, not straight lines.

![Make Closest Facility Parameters](docs/screenshots/76_make_closest_facility_parameters.png)

| Parameter | Value |
|-----------|-------|
| Network Data Source | VA_Roads_ND |
| Layer Name | Closest Facility |
| Travel Mode | Length |
| Travel Direction | Toward Facilities |
| Cutoff | 500,000 meters |
| Number of Facilities to Find | 2 |

#### The Self-Match Problem and Fix

Since the same hospital layer is used for both Facilities and Incidents, each hospital finds itself as rank 1 (zero length). Two settings fix this:

| Fix | Setting | Why |
|-----|---------|-----|
| 1 | Cutoff = 500,000m | Default cutoff was too small, real neighbors were being excluded |
| 2 | Facilities to Find = 2 | Forces solver to find rank 2 (real neighbor), not just rank 1 (self) |

Zero-length self-match routes produce zero points in Generate Points Along Lines and are effectively ignored in the output.

---

### Step 9: Add Locations (Facilities)

Loads hospitals as the destination layer.

![Add Locations Configured](docs/screenshots/48_add_locations_configured.png)

| Parameter | Value |
|-----------|-------|
| Input Network Analysis Layer | Closest Facility |
| Sub Layer | Facilities |
| Input Locations | Hospitals_VA_Boundary.shp |
| Search Tolerance | 5000 Meters |
| Search Criteria | VA_Roads SHAPE |
| Match Type | Match to Closest |
| Append | Yes |

117 facilities located successfully.

---

### Step 10: Add Locations (Incidents)

Loads hospitals as the origin layer.

| Parameter | Value |
|-----------|-------|
| Input Network Analysis Layer | Closest Facility |
| Sub Layer | Incidents |
| Input Locations | Hospitals_VA_Boundary.shp |
| Search Tolerance | 5000 Meters |
| Search Criteria | VA_Roads SHAPE |
| Match Type | Match to Closest |
| Append | Yes |

Same hospital layer as Facilities. 117 incidents located successfully.

---

### Step 11: Solve

Runs the Closest Facility routing solver across all 117 hospital pairs.

![Solve Succeeded Canvas](docs/screenshots/73_solve_succeeded_canvas.png)

| Parameter | Value |
|-----------|-------|
| Input Network Analysis Layer | Closest Facility |
| Ignore Invalid Locations | Skip |
| Termination Condition | Terminate |

Solve time approximately 1 minute for 117 hospitals on the full Virginia network. Output includes a Routes sublayer containing road-following polylines between matched hospital pairs.

---

### Step 12: Select Data

Extracts the Routes sublayer from the Network Analyst layer so downstream tools can use it as a regular feature class.

![Select Data Parameters Lines](docs/screenshots/65_select_data_parameters_lines.png)

| Parameter | Value |
|-----------|-------|
| Input Data Element | Network Analyst Layer (from Solve) |
| Child Data Element | Routes |

The correct sublayer name is Routes not Lines. Lines is the OD Cost Matrix sublayer name. Using the wrong name returns an empty dataset.

---

### Step 13: Select by Attributes

Filters out self-matching routes where a hospital matched itself (Total_Length = 0).

| Parameter | Value |
|-----------|-------|
| Input Table | Routes (from Select Data) |
| Selection Type | New Selection |
| Expression | Total_Length > 0 |

Required cleanup step because Facilities and Incidents use the same hospital layer. Without this filter, MESH_NODES gets only 1-2 points instead of the full node network.

---

### Step 14: Generate Points Along Lines

Places relay nodes at fixed intervals along every route line.

![Generate Points Along Lines Parameters](docs/screenshots/57_generate_points_along_lines_parameters.png)

| Parameter | Value |
|-----------|-------|
| Input Features | Filtered Routes |
| Output Feature Class | MESH_NODES |
| Point Placement | By Distance |
| Distance | 3 Miles |
| Include End Points | Yes |
| Distance Method | Planar |

3 miles reflects the reliable MeshCore LoRa 915MHz range on 30ft pole-mounted antennas along Virginia's open highway corridors under worst-case monsoon and dust storm conditions. Result: 600 relay nodes.

---

### Step 15: Manual Beam Link Digitization

Created `BEAM_MESH_NODES` as a Line feature class using ArcGIS Pro Editor. 19 long-distance directional beam links were manually digitized to bridge geographically separated hospital clusters that nearest-neighbor routing does not connect.

Estimated range per beam link: 15-30 miles with directional 915MHz Yagi/panel antenna. Both MESH_NODES and BEAM_MESH_NODES were exported as separate KMZ files.

---

### Step 16: Layer To KML

Exports the node placement layer to KML for Google Earth field deployment reference.

![Layer To KML Error 584](docs/screenshots/59_layer_to_kml_error_584.png)

| Parameter | Value |
|-----------|-------|
| Input Layer | MESH_NODES |
| Output File | MESH_NODES.kmz |

Note: this tool shows Error 000584 in ModelBuilder at validation time because MESH_NODES does not exist yet. This is a timing issue only and the tool runs correctly at runtime.

---

## Cost Estimate

| Component | Count | Unit Cost | Total |
|-----------|-------|-----------|-------|
| Relay nodes (Heltec LoRa32/LilyGo + pole + omni antenna) | 600 | USD 300 | USD 180,000 |
| Beam links (directional Yagi/panel, 2 radios per link) | 19 | USD 1,200 | USD 22,800 |
| **Total** | | | **~USD 202,800** |

---

## Setup

### Requirements
- ArcGIS Pro 3.x
- Network Analyst extension
- Spatial Analyst extension (for future terrain analysis)

### Network Dataset
The road network needs to be built once before running the model:

1. Copy your road centerlines into a Feature Dataset inside your GDB
2. Right click the Feature Dataset → New → Network Dataset
3. In Properties → Travel Attributes → add a Length cost (meters, uses Shape_Length)
4. Travel Modes → add a mode with Length as impedance, no U-turns, no hierarchy
5. Build the network
6. Point the model variable directly at the built Network Dataset. Do not let the model rebuild it on each run or you will lose your travel mode settings.

### Running the model
1. Open the ModelBuilder model from the toolbox
2. Update the input variables (hospital CSV, boundary KML, Network Dataset path)
3. Run
4. MESH_NODES layer populates with relay node locations
5. Open the exported KMZ in Google Earth for field reference

---

## Output

The MESH_NODES feature class contains one point per relay node with attributes:
- Node ID
- Road name
- Distance to nearest neighbor node
- Coordinates for GPS deployment

Export to KML/KMZ for field use. Each point represents a physical location where a MeshCore node on a 30ft pole should be installed.

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

## Limitations

- Node spacing is currently fixed at 3 miles. Works well for open Virginia highways but is too conservative for urban segments.
- Closest Facility routing finds nearest neighbor pairs but does not implement a true Minimum Spanning Tree. Some routes may be redundant where hospitals are already reachable through existing mesh nodes.
- Terrain and canopy effects are not yet incorporated. Elevation and NLCD land cover layers are planned for a future version that will dynamically adjust spacing per segment.

---

## Future Enhancements

- **Minimum Spanning Tree** — connect all hospital clusters into a fully linked network, eliminating inter-cluster gaps
- **Adaptive Node Spacing** — use USGS DEM elevation and NLCD land cover for variable spacing (tighter in mountains, wider on open highways)
- **Greedy Interval Algorithm** — Python script (`place_mesh_nodes.py`) in progress using `LOCAL_SPEED_MPH` as terrain proxy for dynamic hop ranges
- **Extended Coverage** — integrate HIFLD fire stations and emergency services data

---

## Use Case

Built for emergency communications planning, specifically for scenarios where normal infrastructure (cell towers, internet) is unavailable and first responders need a mesh radio backbone between medical facilities. MeshCore hive topology means the network is resilient. No single point of failure, nodes route around gaps automatically.

Could be adapted for any POI type (fire stations, EOCs, shelters) and any LoRa frequency by adjusting the node spacing parameter.

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

---

## License

MIT. Use it, adapt it, deploy it. If you actually install nodes somewhere because of this I'd love to hear about it.
