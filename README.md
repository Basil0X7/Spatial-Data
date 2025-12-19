# Spatial Data Analysis Portfolio

## Projects Included
- Jen and Barry’s Ice-Cream Site Selection  
- School Site Suitability Analysis  
- Tree Cutting Priority Assessment  
- LLM-Based SQL Query Generator  

---

## **Overview**

This portfolio presents four applied spatial analysis projects that demonstrate the use of modern GIS
technologies for real-world decision support. Each project focuses on a different problem domain and
utilizes appropriate tools including **PostGIS**, **GeoPandas**, **Python**, and **QGIS**.  

In addition, one project explores the integration of **Large Language Models (LLMs)** into GIS workflows
through the development of a custom QGIS plugin.

The projects progress from traditional database-driven spatial analysis to advanced multi-criteria
risk modeling and AI-assisted query generation.

---

## **Project Objectives**

### **Ice-Cream Shop Location Selection**

The goal of this project is to identify optimal cities in Pennsylvania for expanding an ice-cream
business. Candidate cities are evaluated using demographic, environmental, and infrastructure-based
criteria. The workflow applies attribute filtering followed by spatial proximity analysis to arrive
at a small set of optimal locations.

---

### **School Site Selection**

This project identifies parcels suitable for new school construction by analyzing land-use
designations, parcel size, building presence, and access to roads. The objective is to locate parcels
that meet safety, accessibility, and regulatory requirements while minimizing conflicts with existing
infrastructure.

---

### **Tree Cutting Priority Analysis**

This analysis determines priority zones for tree removal in the town of Fire Creek. By combining tree
mortality data with human activity, infrastructure, and emergency access layers, the project produces
a weighted priority surface that helps allocate limited tree removal resources efficiently.

---

### **LLM Query Generator Plugin**

This project investigates whether large language models can interpret visual spatial workflows and
translate them into executable PostGIS SQL queries. A QGIS plugin was developed to accept diagram inputs
and generate database queries automatically.

---
## **Evaluation Criteria**

### **Ice-Cream Site Selection**
- At least 500 farms producing milk  
- Labor force (ages 18–64) ≥ 25,000  
- Crime index ≤ 0.02  
- Population density < 150 people per square mile  
- Presence of a university or college  
- Recreation area within 10 miles  
- Interstate highway within 20 miles  
- Final selection limited to four cities  

---

### **School Site Selection**
- Acceptable land-use types: unused, agricultural, or commercial  
- Parcel area ≥ 5,000 m²  
- No existing buildings  
- Located within 25 meters of a road  

---

### **Tree-Cutting Priority**
- Tree mortality intensity  
- Distance to community features  
- Proximity to evacuation routes  
- Population density  
- Distance to electric utilities  
- Combined weighted priority value per grid cell  

---
## **Data Sources**

### **Ice-Cream Site Selection**

| Dataset | Description | Model | Type | Attributes |
|------|------------|-------|------|------------|
| cities | City locations | Vector | Point | Geom, Name, Population, Crime, University |
| counties | County boundaries | Vector | Polygon | Demographics, farms, density |
| interstates | Highway network | Vector | Line | Geom, Name |
| recareas | Recreation areas | Vector | Polygon | Area, Perimeter |

---

### **School Site Selection**

| Dataset | Description | Model | Type | Attributes |
|------|------------|-------|------|------------|
| buildings | Existing structures | Vector | Polygon | Owner, Type |
| landuse | Parcels | Vector | Polygon | Owner, Type, Area |
| roads | Road network | Vector | Line | Type, Length |

---

### **Tree-Cutting Priority**

| Dataset | Description | Model | Type | Attributes |
|------|------------|-------|------|------------|
| Tree mortality | Mortality intensity | Raster | Continuous | Total mortality |
| Community features | Public features | Vector | Point | Name, Weight |
| Egress routes | Evacuation routes | Vector | Line | Weight |
| Populated areas | Population density | Raster | Continuous | Pop per sq mile |
| Electric utilities | Power infrastructure | Vector | Line | Voltage, Type |

---

### **LLM Query Generator**

| Dataset | Description | Type |
|------|------------|------|
| Input diagrams | Workflow images | Image |
| Generated queries | SQL text | Text |
| Output layers | Query results | Vector / Raster |

---

## **Methodology**

### **Ice-Cream Shop Site Selection (PostGIS & QGIS)**

All datasets were imported into PostGIS using NAD27 coordinates (SRID 4267). Distance-based operations
required reprojection into Pennsylvania State Plane North.

The workflow first filtered counties using demographic and agricultural criteria. Cities within those
counties were then filtered by crime index and university presence. Distance-based spatial joins were
used to evaluate proximity to highways and recreation areas.

**Code**: `SQL`

```SQL
SELECT DISTINCT c.id, c.geom, c.name, c.population, c.crime_inde, c.university 

FROM cities c
JOIN counties co ON ST_Within(c.geom, co.geom) 
JOIN recareas r ON ST_DWithin(ST_Transform(c.geom, 2272), ST_Transform(r.geom, 2272), 52800)
JOIN interstates i ON ST_DWithin(ST_Transform(c.geom, 2272), ST_Transform(i.geom, 2272), 105600)

WHERE co.no_farms87 > 500 
  AND co.age_18_64 >= 25000 
  AND c.crime_inde <= 0.02 
  AND co.pop_sqmile < 150 
  AND c.university > 0
```

####

#### **School Site Selection (PostGIS \+ QGIS)**

Parcels were selected by filtering the land-use layer for unused, agricultural, and commercial
classifications, then excluding any parcels smaller than 5,000 square meters to ensure sufficient
development space. Parcels intersecting existing buildings were removed using spatial intersection
checks, leaving only vacant land suitable for construction. Finally, parcels within 25 meters of
road networks were retained to guarantee accessibility for transportation and emergency services.
The combined spatial criteria were applied in a single query, and results were visualized in QGIS to
analyze both compliance and spatial distribution relative to existing infrastructure.


**Code**: `SQL`

```SQL
SELECT l.id, l.type, l.area, l.owner, l.geom, ST_Centroid(l.geom)

AS centroid, MIN(ST_Distance(ST_Centroid(l.geom), r.geom)) AS min_road_dist

FROM "Landuse" l JOIN "Roads" r ON ST_DWithin(l.geom, r.geom, 25)

WHERE LOWER(l.type) IN ('un-used', 'agricultural areas', 'commercial lands') AND COALESCE(l.area, ST_Area(l.geom)) >= 5000 AND NOT EXISTS (SELECT 1 FROM "Buildings" b WHERE ST_Intersects(b.geom, l.geom))

GROUP BY l.id, l.type, l.area, l.owner, l.geom;
```

####

#### **Tree-Cutting Priority (GeoPandas \+ QGIS)**

Five risk factors were computed and normalized to a 0–1 scale. Tree mortality risk was derived by
converting the mortality raster into polygons and normalizing values by the dataset maximum.
Community feature risk was calculated using minimum distances from grid centroids, inverted and
weighted to reflect feature importance. Egress route risk was based on proximity to evacuation
routes, with distances transformed into weighted risk scores. Population density risk was obtained
by extracting and normalizing density statistics, assigning higher priority to denser areas.
Finally, electric utility risk was determined by converting distances to utility lines into
weighted scores, giving higher voltage infrastructure greater influence.


**Code**: `Python`

```python
crs     = "EPSG:3857"
layers  = [trees, community, roads, population, transmission, substations, grid, town_boundary]
layers  = [l.to_crs(crs) for l in layers]
trees, community, roads, population, transmission, substations, grid, town_boundary = layers

trees_r = to_raster(gdf=trees)
comm_r  = to_raster(gdf=community)
roads_r = to_raster(gdf=roads)
pop_r   = to_raster(gdf=population)

def euclidean(binary_mask): return distance_transform_edt(binary_mask == 0) * resolution

dist_trees = euclidean(binary_mask=trees_r)
dist_comm  = euclidean(binary_mask=comm_r)
dist_roads = euclidean(binary_mask=roads_r)
dist_pop   = euclidean(binary_mask=pop_r)
dist_util  = euclidean(binary_mask=util_r)

zs = zonal_stats(grid, priority_raster, transform, stats=["mean"], 0)

grid["priority"] = [z["mean"] for z in zs]
```

####

#### **LLM Query Generator Plugin Development**

The plugin was developed in Python for QGIS, including the main class, user interface, and metadata setup. 
The interface allows users to upload workflow images, view generated SQL queries, and execute them with Qt-based controls. 
Uploaded images are processed via the Gemini-Flash LLM API, which interprets the workflow and produces PostGIS SQL queries. 
The plugin parses the LLM output for valid SQL, allowing user review and modification. 
Queries are executed through QGIS's database connection, with error handling for syntax or database issues. 
Successful execution automatically generates new vector layers in QGIS with default styling.


**Code**: `Python`

```python
class SQLQueryGeneratorDialog(QDialog):
    def __init__(self, parent=None):
        super().__init__(parent)
        self.setWindowTitle("SQL Query Generator")
        self.setMinimumWidth(600)
        self.setMinimumHeight(400)
        self.setup_ui()
```
```python
class SQLQueryGeneratorPlugin:
    def __init__(self, iface):
        self.iface = iface
        self.dialog = None
```
```python
def classFactory(iface):
    return SQLQueryGeneratorPlugin(iface)
```

## **Results**

#### **Ice-Cream Shop Site Selection**
![assignment_1](assignment_1/ice_cream_qgis_results.png)

####  

#### **School Site Selection**
![assignment_2](assignment_2/school_selection_qgis_results.png)

####  

#### **Tree-Cutting Priority**
![assignment_3](assignment_3/assignment-3-qgis-results.png)

####  

#### **LLM Query Generator**

![LLM](LLM/llm_qgis_prompt.png)

####

![LLM](LLM/llm_qgis_results.png)

## **Conclusion**

These four projects demonstrate the practical use of GIS technologies to address real-world spatial analysis challenges. 
The ice-cream shop project automated site selection using PostGIS to evaluate multiple criteria, simplifying what would otherwise be a tedious manual process. 
The school site selection combined land-use rules, parcel size, and proximity calculations to identify suitable locations efficiently. 
For the tree-cutting priority project, multiple risk factors were combined using weighted scoring to allocate resources where they have the greatest impact. 
The LLM plugin provided a novel approach to streamline spatial analysis by converting visual workflows into executable SQL using the Gemini-Flash API.

Using PostgreSQL, PostGIS, Python, and QGIS together proved effective: PostGIS handled spatial queries, Python supported advanced computations, and QGIS enabled clear visualizations. 
Although each project posed unique challenges, all followed a consistent workflow: break down the problem, apply appropriate spatial operations, and present results in a decision-supportive format. 
These open-source tools allow these methods to be applied in business planning, urban development, or environmental management without reliance on costly proprietary software.


## **Contributors**

This project was developed and completed by:

- Basil Turk  
- Abdallah Khdaar
