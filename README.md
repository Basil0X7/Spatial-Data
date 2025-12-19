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

**SQL Implementation**
```sql
SELECT DISTINCT c.id, c.geom, c.name, c.population, c.crime_inde, c.university
FROM cities c
JOIN counties co ON ST_Within(c.geom, co.geom)
JOIN recareas r ON ST_DWithin(ST_Transform(c.geom, 2272), ST_Transform(r.geom, 2272), 52800)
JOIN interstates i ON ST_DWithin(ST_Transform(c.geom, 2272), ST_Transform(i.geom, 2272), 105600)
WHERE co.no_farms87 > 500
  AND co.age_18_64 >= 25000
  AND c.crime_inde <= 0.02
  AND co.pop_sqmile < 150
  AND c.university > 0;
