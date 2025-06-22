# QGIS to Determine 15-Minute Accessibility of Complete Street Alternatives

A step-by-step tutorial for conducting multimodal accessibility analysis on various complete streets alternatives using free QGIS network analysis tools, SQL, and a bit of Python.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Overview](#overview)
- [Step 1: Data Collection](#step-1-data-collection)
- [Step 2: Network Preparation](#step-2-network-preparation)
- [Step 3: Speed Assignment](#step-3-speed-assignment)
- [Step 4: Origin Points](#step-4-origin-points)
- [Step 5: Network Analysis](#step-5-network-analysis)
- [Step 6: Results Processing](#step-6-results-processing)
- [Step 7: Comparative Analysis](#step-7-comparative-analysis)
- [Troubleshooting](#troubleshooting)

## Prerequisites

**Software Requirements:**
- QGIS 3.28+ (with Network Analyst plugin)
- QuickOSM plugin installed
- Python Console access

**Knowledge Requirements:**
- Basic QGIS navigation
- Understanding of coordinate reference systems (CRS)
- Basic SQL for field calculations

**Hardware:**
- 8GB+ RAM recommended for large networks
- SSD storage for faster processing

## Overview

This methodology measures multimodal accessibility by calculating the total length of street network segments reachable within 15 minutes from origin points. The analysis supports multiple transportation modes and infrastructure scenarios, making it ideal for complete streets evaluation. The network uses projected coordinates to ensure accurate distance measurements. Cycling analysis incorporates Level of Traffic Stress (LTS) to account for comfort and safety impacts on cycling speeds.

**Key Concepts:**
- **Accessibility**: Network length reachable within time threshold
- **Service Areas**: Geographic extent of network accessibility  
- **Speed Assignment**: Mode and infrastructure-specific travel speeds
- **Level of Traffic Stress (LTS)**: Cycling comfort assessment (1=very comfortable, 4=very stressful)
- **Multimodal Analysis**: Separate analysis for walking, cycling, and transit

## Step 1: Data Collection

### 1.1 Install QuickOSM Plugin

1. Open QGIS
2. Go to **Plugins → Manage and Install Plugins**
3. Search for "QuickOSM"
4. Click **Install Plugin**
5. Close the plugin manager

### 1.2 Download Street Network Data

1. **Set Study Area**:
   - Zoom to your area of interest
   - Ensure the view covers your complete study area

2. **Launch QuickOSM**:
   - Go to **Vector → QuickOSM → QuickOSM**

3. **Configure Download**:
   - **Key**: `highway`
   - **Value**: [leave empty]
   - **In**: Canvas Extent
   - Click **Run Query**

4. **Wait for Download**:
   - Processing time varies with area size
   - Larger areas may take several minutes

### 1.3 Filter Network Data

1. **Open Layer Properties**:
   - Right-click the `highway` layer → **Properties**
   - Go to **Source** tab → **Query Builder**

2. **Apply Filter**:
   ```sql
   "highway" IN ('primary','secondary','tertiary','residential','trunk',
                 'motorway','unclassified','footway','path','pedestrian',
                 'steps','cycleway','track')
   ```

3. **Test and Apply**:
   - Click **Test** to verify syntax
   - Click **OK** to apply filter

### 1.4 Save Network Layer

1. **Export Filtered Network**:
   - Right-click filtered layer → **Export → Save Features As**
   - **Format**: GeoPackage
   - **File name**: `network.gpkg`
   - **Layer name**: `network`
   - Click **OK**

## Step 2: Network Preparation

### 2.1 Coordinate Reference System Check

**Important**: Use a projected coordinate system (meters) to avoid degree-to-meter conversion issues.

1. **Check Current CRS**:
   - Right-click `network` layer → **Properties → Information**
   - Note the CRS (should be projected, not geographic)

2. **Reproject if Needed**:
   - If CRS shows EPSG:4326 (geographic), reproject:
   - Right-click layer → **Export → Save Features As**
   - **CRS**: Click globe icon, search for local UTM zone or use EPSG:3857
   - **File name**: `network_projected.gpkg`
   - Use the projected version for all analysis


### 2.2 Add Level of Traffic Stress (LTS) Analysis

Level of Traffic Stress (LTS) quantifies cycling comfort based on infrastructure quality, traffic volume, and speed. This enhances cycling speed assignments by incorporating safety and comfort factors.

**LTS Scale:**
- **LTS 1**: Very comfortable (suitable for all ages)
- **LTS 2**: Comfortable (suitable for most adults)  
- **LTS 3**: Somewhat stressful (experienced cyclists only)
- **LTS 4**: Very stressful (strong and fearless cyclists only)

1. **Add LTS Field**:
   - Right-click `network_projected` layer → **Properties → Fields**
   - Click **pencil icon** (Toggle editing)
   - Click **New field** button
   - **Name**: `LTS`
   - **Type**: Integer
   - **Width**: 1
   - Click **OK**

2. **Calculate LTS Values**:
   - Open **Field Calculator**
   - **Update existing field**: `LTS`
   - Use this Conveyal-based logic:

```sql
CASE 
WHEN "highway" IN ('cycleway', 'footway', 'path', 'pedestrian') THEN 1
WHEN "highway" = 'service' THEN 3
WHEN "highway" IN ('residential', 'living_street') THEN 1
WHEN "highway" IN ('unclassified', 'tertiary', 'tertiary_link') THEN 2
WHEN "highway" IN ('primary', 'primary_link', 'secondary', 'secondary_link', 'trunk', 'trunk_link') THEN 4
ELSE 3
END
```

3. **Verify LTS Assignment**:
   - Check attribute table for LTS values
      - Right-click the `network_projected layer` → "Open Attribute Table"
      - Scroll right in the table to find the LTS column
      - Most segments should have LTS 1-4
      - No NULL values should remain

### 2.3 Add Speed Fields

1. **Enable Editing**:
   - Right-click `network_projected` layer → **Properties → Fields**
   - Click **pencil icon** (Toggle editing)

2. **Add Speed Fields** (add these one by one):
   - Click **New field** button
   - **Name**: See field list below
   - **Type**: Decimal number (real)
   - Click **OK**

**Required Speed Fields** (15 total):
```
Walking Fields:
- walk_base
- walk_c_brt  
- walk_bi_brt
- walk_s_brt
- walk_enhanced

Cycling Fields (LTS-Enhanced):
- bike_base
- bike_c_brt
- bike_bi_brt  
- bike_s_brt
- bike_enhanced

Transit Fields:
- bus_base
- bus_c_brt
- bus_bi_brt
- bus_s_brt
- bus_enhanced

LTS Analysis Field:
- LTS
```

3. **Save Changes**:
   - Click **OK** in Properties dialog
   - Save edits (disk icon)

## Step 3: Speed Assignment

These SQL formulas are used in QGIS Field Calculator to assign travel speeds for different transportation modes across five complete streets alternatives. All speeds are in km/h.
You may alter speed assignments based on your specific complete streets alternative designs. As an example, these alternatives will be evaluated:

**Alternatives:**
1. **Baseline**: Painted bike lanes + standard sidewalks + regular bus
2. **Center BRT**: Center-running BRT + separated bike lanes + wider walkways
3. **Bidirectional BRT**: Bidirectional BRT + separated bike lanes + wider walkways  
4. **Side BRT**: Side-running BRT + separated bike lanes + wider walkways
5. **Enhanced Active**: Regular bus + separated bike lanes + wider walkways

**Open Field Calculator**:
- Right-click `network_projected` layer → **Open Attribute Table**
- Ensure editing mode is on (pencil icon)
- Click **Field Calculator** (calculator icon)

**Apply speed fields in this sequence:**

1. **LTS field** (integer)
2. **Walking fields** (5 fields): walk_base, walk_c_brt, walk_bi_brt, walk_s_brt, walk_enhanced
3. **Cycling fields** (5 fields): bike_base, bike_c_brt, bike_bi_brt, bike_s_brt, bike_enhanced  
4. **Transit fields** (5 fields): bus_base, bus_c_brt, bus_bi_brt, bus_s_brt, bus_enhanced

**Total fields created**: 16 (1 LTS + 15 speed fields)

---

## Walking Speed Fields

### walk_base (baseline with standard sidewalks)
```sql
CASE 
WHEN "highway" = 'footway' THEN 3.5
WHEN "highway" = 'pedestrian' THEN 3.5
WHEN "highway" IN ('primary','secondary','trunk') THEN 2.0
WHEN "highway" = 'tertiary' THEN 2.5
WHEN "highway" = 'unclassified' THEN 3.0
WHEN "highway" = 'residential' THEN 3.5
WHEN "highway" = 'path' THEN 3.5
ELSE 2.5
END
```

### walk_c_brt (Center BRT + wider walkways)
```sql
CASE 
WHEN "highway" = 'footway' THEN 4.5
WHEN "highway" = 'pedestrian' THEN 4.5
WHEN "highway" IN ('primary','secondary','trunk') THEN 4.2
WHEN "highway" = 'tertiary' THEN 3.8
WHEN "highway" = 'unclassified' THEN 3.8
WHEN "highway" = 'residential' THEN 3.5
WHEN "highway" = 'path' THEN 4.2
ELSE 3.8
END
```

### walk_bi_brt (Bidirectional BRT + wider walkways)
```sql
CASE 
WHEN "highway" = 'footway' THEN 4.5
WHEN "highway" = 'pedestrian' THEN 4.5
WHEN "highway" IN ('primary','secondary','trunk') THEN 3.8
WHEN "highway" = 'tertiary' THEN 3.5
WHEN "highway" = 'unclassified' THEN 3.5
WHEN "highway" = 'residential' THEN 3.5
WHEN "highway" = 'path' THEN 4.0
ELSE 3.5
END
```

### walk_s_brt (Side BRT + wider walkways)
```sql
CASE 
WHEN "highway" = 'footway' THEN 4.5
WHEN "highway" = 'pedestrian' THEN 4.5
WHEN "highway" IN ('primary','secondary','trunk') THEN 3.5
WHEN "highway" = 'tertiary' THEN 3.5
WHEN "highway" = 'unclassified' THEN 3.5
WHEN "highway" = 'residential' THEN 3.5
WHEN "highway" = 'path' THEN 4.0
ELSE 3.2
END
```

### walk_enhanced (enhanced active)
```sql
CASE 
WHEN "highway" = 'footway' THEN 4.0
WHEN "highway" = 'pedestrian' THEN 4.0
WHEN "highway" IN ('primary','secondary','trunk') THEN 3.5
WHEN "highway" = 'tertiary' THEN 3.5
WHEN "highway" = 'unclassified' THEN 3.5
WHEN "highway" = 'residential' THEN 3.5
WHEN "highway" = 'path' THEN 4.0
ELSE 3.0
END
```

---

## Cycling Speed Fields (LTS-Enhanced)

**Note**: These formulas require the LTS field to be calculated first.

**Speed Methodology:**
- **LTS 1 (Very Comfortable)**: 20-25 km/h - separated facilities, very low stress
- **LTS 2 (Comfortable)**: 15-23 km/h - some protection, manageable stress  
- **LTS 3 (Somewhat Stressful)**: 12-21 km/h - minimal protection, higher stress
- **LTS 4 (Very Stressful)**: 8-18 km/h - no protection, high stress environment

## Level of Traffic Stress (LTS) Formula

**Apply this first before cycling speed assignments:**

```sql
CASE 
-- Exclude segments where bikes cannot go
WHEN "highway" IN ('footway', 'steps', 'pedestrian') THEN NULL

-- Bike-friendly infrastructure  
WHEN "highway" IN ('cycleway', 'path') THEN 1
WHEN "highway" IN ('residential', 'living_street') THEN 1

-- Evaluate actual roads for cycling stress
WHEN "highway" IN ('unclassified', 'tertiary', 'tertiary_link') THEN 2
WHEN "highway" IN ('primary', 'primary_link', 'secondary', 'secondary_link', 'trunk', 'trunk_link') THEN 4
WHEN "highway" = 'service' THEN 3
ELSE 3
END
```

---

### bike_base (painted bike lanes - baseline scenario)
```sql
CASE 
WHEN "LTS" = 1 THEN 20
WHEN "LTS" = 2 THEN 15
WHEN "LTS" = 3 THEN 12
WHEN "LTS" = 4 THEN 8
WHEN "highway" = 'footway' THEN 0.1
ELSE 10
END
```

### bike_c_brt (Center BRT + separated bike lanes)
```sql
CASE 
WHEN "LTS" = 1 THEN 25
WHEN "LTS" = 2 THEN 23
WHEN "LTS" = 3 THEN 21
WHEN "LTS" = 4 THEN 18
WHEN "highway" = 'footway' THEN 0.1
ELSE 20
END
```

### bike_bi_brt (Bidirectional BRT + separated bike lanes)
```sql
CASE 
WHEN "LTS" = 1 THEN 25
WHEN "LTS" = 2 THEN 20
WHEN "LTS" = 3 THEN 19
WHEN "LTS" = 4 THEN 16
WHEN "highway" = 'footway' THEN 0.1
ELSE 18
END
```

### bike_s_brt (Side BRT + separated bike lanes)
```sql
CASE 
WHEN "LTS" = 1 THEN 25
WHEN "LTS" = 2 THEN 18
WHEN "LTS" = 3 THEN 16
WHEN "LTS" = 4 THEN 14
WHEN "highway" = 'footway' THEN 0.1
ELSE 16
END
```

### bike_enhanced (enhanced active with separated lanes)
```sql
CASE 
WHEN "LTS" = 1 THEN 25
WHEN "LTS" = 2 THEN 20
WHEN "LTS" = 3 THEN 15
WHEN "LTS" = 4 THEN 12
WHEN "highway" = 'footway' THEN 0.1
ELSE 15
END
```

---

## Transit Speed Fields

### bus_base (regular buses)
```sql
CASE 
WHEN "highway" IN ('primary','secondary','trunk') THEN 25
WHEN "highway" = 'tertiary' THEN 20
WHEN "highway" = 'unclassified' THEN 18
WHEN "highway" = 'residential' THEN 15
ELSE 15
END
```

### bus_c_brt (Center BRT - highest speeds)
```sql
CASE 
WHEN "highway" IN ('primary','secondary','trunk') THEN 45
WHEN "highway" = 'tertiary' THEN 35
WHEN "highway" = 'unclassified' THEN 30
WHEN "highway" = 'residential' THEN 25
ELSE 20
END
```

### bus_bi_brt (Bidirectional BRT - moderate complexity)
```sql
CASE 
WHEN "highway" IN ('primary','secondary','trunk') THEN 38
WHEN "highway" = 'tertiary' THEN 30
WHEN "highway" = 'unclassified' THEN 26
WHEN "highway" = 'residential' THEN 22
ELSE 18
END
```

### bus_s_brt (Side BRT - intersection constraints)
```sql
CASE 
WHEN "highway" IN ('primary','secondary','trunk') THEN 32
WHEN "highway" = 'tertiary' THEN 26
WHEN "highway" = 'unclassified' THEN 23
WHEN "highway" = 'residential' THEN 20
ELSE 16
END
```

### bus_enhanced (enhanced regular service)
```sql
CASE 
WHEN "highway" IN ('primary','secondary','trunk') THEN 28
WHEN "highway" = 'tertiary' THEN 22
WHEN "highway" = 'unclassified' THEN 20
WHEN "highway" = 'residential' THEN 18
ELSE 15
END
```

---

## Speed Assignment Rationale

### Walking Speed Differentiation
- **Center BRT**: Optimal pedestrian environment with maximum separation from traffic
- **Bidirectional BRT**: Good separation but more complex crossings at stations
- **Side BRT**: Protected walkways but more intersection conflicts  
- **Enhanced Active**: Walkway improvements without BRT investment
- **Baseline**: Standard sidewalks with arterial proximity impacts

### Cycling Speed Differentiation  
- **LTS Integration**: Complete streets infrastructure improves cycling stress levels
- **Center BRT**: Optimal cycling environment enabling highest speeds across all LTS levels
- **Bidirectional/Side BRT**: Good protection with varying operational complexity
- **Enhanced Active**: Infrastructure benefits without BRT
- **Baseline**: Maintains existing high-stress conditions with painted lanes

### Transit Speed Differentiation
- **Center BRT**: Complete grade separation and optimal signal priority (20-45 km/h)
- **Bidirectional BRT**: Efficient operations with moderate complexity (18-38 km/h)
- **Side BRT**: Dedicated lanes but frequent intersection delays (16-32 km/h)
- **Enhanced Regular**: Improved operations in mixed traffic (15-28 km/h)
- **Baseline**: Standard mixed traffic operations (15-25 km/h)

---

## TCQSM Alignment

**Walking Speeds**: Range from 2.0-4.5 km/h, approaching TCQSM standard of 5.0 km/h with infrastructure improvements

**Cycling Speeds**: Protected alternatives achieve portions of TCQSM 20-25 km/h range through LTS improvements  

**Transit Speeds**: BRT alternatives reflect TCQSM guidance that "BRT should emulate rapid transit quality"

---

*Note: These speed assignments are based on Transit Capacity and Quality of Service Manual (TCQSM) standards, Level of Traffic Stress research, and BRT design guidelines. All speeds reflect the relationship between infrastructure quality and travel performance for multimodal accessibility analysis.*

**Save All Changes**:
- After completing all speed assignments, save edits (disk icon)
- Stop editing mode

## Step 4: Origin Points

### 4.1 Create Residential Origins

1. **Create New Layer**:
   - **Layer → Create Layer → New Shapefile Layer**
   - **File name**: `residential_origins.shp`
   - **Geometry type**: Point
   - Click **OK**

2. **Add Origin Points**:
   - Select the new layer
   - Click **Toggle Editing** (pencil icon)
   - Click **Add Point Feature**
   - Click on map to add points in residential areas
   - Add 5-10 points across your study area
   - Save edits and stop editing

### 4.2 Create Transit Stop Origins

1. **Create Transit Stops Layer**:
   - **Layer → Create Layer → New Shapefile Layer**
   - **File name**: `transit_stops.shp`
   - **Geometry type**: Point
   - Click **OK**

2. **Add Transit Points**:
   - Follow same process as residential origins
   - Place points along major streets where BRT stops would be located
   - Save edits and stop editing

## Step 5: Network Analysis

### 5.1 Access Network Analysis Tools

1. **Open Processing Toolbox**:
   - **Processing → Toolbox**

2. **Find Service Area Tool**:
   - Expand **Network analysis**
   - Look for **"Service area (from layer)"**

### 5.2 Run Walking Accessibility Analysis

**For each walking scenario (5 analyses total):**

1. **Open Service Area Tool**:
   - Double-click **"Service area (from layer)"**

2. **Configure Parameters**:
   - **Input layer**: `residential_origins`
   - **Vector layer representing network**: `network_projected`
   - **Path type**: **Fastest**
   - **Travel cost**: `0.25` (15 minutes = 0.25 hours)

3. **Set Advanced Parameters**:
   - Expand **Advanced Parameters**
   - **Speed field**: Select appropriate walking field:
     - Analysis 1: `walk_base`
     - Analysis 2: `walk_c_brt`
     - Analysis 3: `walk_bi_brt`
     - Analysis 4: `walk_s_brt`
     - Analysis 5: `walk_enhanced`

4. **Name Output**:
   - **Service area (lines)**: `Walk_[Scenario_Name]`

5. **Run Analysis**:
   - Click **Run**
   - Wait for completion

### 5.3 Run Cycling Accessibility Analysis

**Repeat the process for cycling scenarios:**
- Use same parameters as walking
- **Input layer**: `residential_origins`
- **Speed field**: Use cycling fields (`bike_base`, `bike_c_brt`, etc.)
- **Output names**: `Bike_[Scenario_Name]`

### 5.4 Run Transit Accessibility Analysis

**For transit scenarios:**
- **Input layer**: `transit_stops` (different from walking/cycling)
- **Speed field**: Use transit fields (`bus_base`, `bus_c_brt`, etc.)
- **Output names**: `Transit_[Scenario_Name]`

**Total Analyses**: 15 service area analyses (5 scenarios × 3 modes)

## Step 6: Results Processing

### 6.1 Calculate Accessibility Metrics

1. **Open Python Console**:
   - **Plugins → Python Console**

2. **Run Analysis Code**:
   ```python
   def analyze_accessibility(mode_name, layer_names):
       print(f"\n{mode_name.upper()} ACCESSIBILITY (15 minutes)")
       print("-" * 50)
       
       # Get layers and calculate lengths
       layers = []
       lengths = []
       
       for name in layer_names:
           try:
               layer = QgsProject.instance().mapLayersByName(name)[0]
               layers.append(layer)
               total_length = sum([f.geometry().length() for f in layer.getFeatures()])
               lengths.append(total_length / 1000)  # Convert to kilometers
               print(f"{name.replace('_', ' ').title()}: {total_length/1000:.2f} km")
           except IndexError:
               print(f"Warning: Layer '{name}' not found")
               lengths.append(0)
       
       # Calculate improvements vs baseline
       if lengths[0] > 0:
           print(f"\nIMPROVEMENTS vs BASELINE:")
           baseline = lengths[0]
           scenarios = ['Center BRT', 'Bidirectional BRT', 'Side BRT', 'Enhanced Active']
           for i, scenario in enumerate(scenarios):
               if i+1 < len(lengths):
                   improvement = ((lengths[i+1] - baseline) / baseline) * 100
                   print(f"{scenario}: {improvement:.1f}% change")
       
       return lengths

   # Run analysis for all modes
   print("=" * 60)
   print("15-MINUTE ACCESSIBILITY ANALYSIS RESULTS")
   print("=" * 60)

   # Walking analysis
   walk_results = analyze_accessibility("Walking", [
       'Walk_Baseline', 'Walk_Center_BRT', 'Walk_Bidirectional_BRT', 
       'Walk_Side_BRT', 'Walk_Enhanced'
   ])

   # Check LTS distribution
   lts_layer = QgsProject.instance().mapLayersByName('network_projected')[0]
   lts_counts = {}
   for feature in lts_layer.getFeatures():
       lts_value = feature['LTS']
       lts_counts[lts_value] = lts_counts.get(lts_value, 0) + 1
   
   print("\nLTS Distribution:")
   for lts in sorted(lts_counts.keys()):
       count = lts_counts[lts]
       pct = count / lts_layer.featureCount() * 100
       print(f"LTS {lts}: {count} segments ({pct:.1f}%)")

   # Cycling analysis with LTS context  
   bike_results = analyze_accessibility("Cycling", [
       'Bike_Baseline', 'Bike_Center_BRT', 'Bike_Bidirectional_BRT',
       'Bike_Side_BRT', 'Bike_Enhanced'
   ])

   # Transit analysis
   transit_results = analyze_accessibility("Transit", [
       'Transit_Baseline', 'Transit_Center_BRT', 'Transit_Bidirectional_BRT',
       'Transit_Side_BRT', 'Transit_Enhanced'
   ])

   print("\n" + "=" * 60)
   print("ANALYSIS COMPLETE")
   ```

### 6.2 Visual Analysis

1. **Style Service Areas**:
   - Right-click each service area layer → **Properties → Symbology**
   - Set different colors with 50% transparency
   - Use consistent color schemes across modes

2. **Compare Visually**:
   - Toggle layers on/off to compare coverage
   - Look for differences in network reach
   - Identify areas with improved accessibility

## Step 7: Comparative Analysis

### 7.1 Create Summary Table

**Document results in a table format:**

| Scenario | Walking (km) | Cycling (km) | Transit (km) | Overall Rank |
|----------|--------------|--------------|--------------|--------------|
| Baseline | [value] | [value] | [value] | - |
| Center BRT | [value] | [value] | [value] | [rank] |
| Bidirectional BRT | [value] | [value] | [value] | [rank] |
| Side BRT | [value] | [value] | [value] | [rank] |
| Enhanced Active | [value] | [value] | [value] | [rank] |

### 7.2 Calculate Composite Scores

**Optional: Create weighted accessibility score:**
```python
def calculate_composite_score(walk_km, bike_km, transit_km, 
                             walk_weight=0.4, bike_weight=0.3, transit_weight=0.3):
    """Calculate weighted accessibility score"""
    return (walk_km * walk_weight + 
            bike_km * bike_weight + 
            transit_km * transit_weight)

# Example usage
scenarios = ['Baseline', 'Center BRT', 'Bidirectional BRT', 'Side BRT', 'Enhanced']
for i, scenario in enumerate(scenarios):
    if i < len(walk_results) and i < len(bike_results) and i < len(transit_results):
        score = calculate_composite_score(walk_results[i], bike_results[i], transit_results[i])
        print(f"{scenario}: {score:.2f} composite accessibility score")
```

## Troubleshooting

### Common Issues and Solutions

**Issue: "No OSM data matched the query"**
- Solution: Zoom to a smaller area with visible streets
- Check that you're querying an urban area with roads

**Issue: Service areas appear too small**
- Check speed field values in attribute table
- Verify travel cost is set to 0.25 hours
- Ensure "Fastest" path type is selected
- Confirm network uses projected coordinates (meters)

**Issue: Speed field shows NULL values**
- Check CASE statement syntax in Field Calculator
- Verify highway field contains expected values
- Use "Show All Values" on highway field to see actual data

**Issue: Analysis runs slowly**
- Reduce network size by limiting study area
- Close unnecessary QGIS layers
- Increase available RAM
- Use SSD storage

**Issue: Layers not found in Python analysis**
- Check exact layer names in Layers Panel
- Ensure all analyses completed successfully
- Verify layer names match those in the Python code

### Data Quality Checks

**Before analysis, verify:**
- [ ] Network has no gaps in critical areas
- [ ] All speed fields contain realistic values (no NULL, no zeros except footway cycling)
- [ ] Origin points are well-distributed across study area
- [ ] Network uses projected coordinate system
- [ ] Study area is appropriate size (not too large for computer capabilities)

**After analysis, verify:**
- [ ] Service areas look realistic for 15-minute travel
- [ ] Results show logical differences between scenarios
- [ ] No error messages in processing log
- [ ] All expected output layers were created

## Best Practices

**Study Area Selection:**
- Keep initial study areas manageable (few square kilometers)
- Ensure area includes diverse land uses and demographics
- Include key destinations within or near study boundary

**Speed Assignment:**
- Base speeds on local conditions when possible
- Document assumptions and sources for speed values
- Test sensitivity to speed assumptions
- Consider local factors (topography, climate, demographics)

**Origin Point Placement:**
- Distribute points to represent population distribution
- Include various neighborhood types
- Consider accessibility needs of target populations
- Use at least 5-10 points for meaningful analysis

**Result Interpretation:**
- Focus on relative comparisons between scenarios
- Consider accessibility alongside other planning factors
- Account for analysis limitations in conclusions
- Validate results against local knowledge

## Next Steps

**Advanced Analysis Options:**
- Add destination weighting (gravity-based accessibility)
- Include multimodal trip chains (walk + transit)
- Analyze accessibility equity across demographics
- Incorporate real-time traffic or transit data
- Validate results with travel surveys or GPS data

**Integration with Planning:**
- Use results to support infrastructure investment decisions
- Incorporate findings into comprehensive planning processes
- Communicate results to stakeholders and public
- Monitor accessibility changes over time

---

**Citation**: This methodology builds upon the Transit Capacity and Quality of Service Manual (TCQSM) 3rd Edition and established transportation planning practices for accessibility analysis.
