# Contents
- [Crop Data Enrichment Pipeline](#crop-data-enrichment-pipeline)  
- [Configurable Crop Feature Engineering Pipeline](#configurable-crop-feature-engineering-pipeline)

# # Crop Data Enrichment Pipeline


## Purpose

The enrichment pipeline is the first stage of the crop modeling workflow. It transforms raw crop records into enriched, structured objects containing all necessary context and time-series data. The enriched records are the only valid inputs to the Configurable Crop Feature Engineering Pipeline (step 2).

## What Step 2 Expects
The Feature Engineering Pipeline consumes enriched crop records that must include:

- Identifiers (crop type, variety, location key, planting month, duration). -required
- Static context (soil_properties, elevation, region). 
- Configuration (defaults, crop-specific, or record overrides).
- Weather time series (Open-Meteo daily data, tagged by year). -- required
- Performance scores (calculated from NDVI scores, tagged by year like weather data) -- required

---

## Raw Input Structure

Raw input consists of crop definitions provided by region, planting time, and duration. Example:
> minimum allowed

```json
{
    "Maize": {
        "variety": "H614",
        "region": "Trans-Nzoia, Kenya",
        "coordinates": [-0.9711, 34.9586],
        "planting_season_month": 4,
        "duration_days": 90
        
    },
    "Sorghum": {
        "variety": "Serena",
        "region": "Eastern Kenya",
        "coordinates": [-1.0986, 37.0144],
        "planting_season_month": 4,
        "duration_days": 115
       
    }
}
```

> preferred

```json
{
    "Maize": {
        "variety": "H614",
        "region": "Trans-Nzoia, Kenya",
        "coordinates": [-0.9711, 34.9586],
        "planting_season_month": 4,
        "duration_days": 90,
        "yield_per_acre_kg": 90,
        "market_price_pkg": 53.33,
        "crop_type": "cereal"
    },
    "Sorghum": {
        "variety": "Serena",
        "region": "Eastern Kenya",
        "coordinates": [-1.0986, 37.0144],
        "planting_season_month": 4,
        "duration_days": 115,
        "yield_per_acre_kg": 910,
         "market_price_pkg": 80.00,
        "crop_type": "cereal"
    }
}
```

---

## Step 1. Location Key Computation

* Each crop is tied to a pair of latitude/longitude coordinates.
* A **location key** is computed to replace raw coordinates.
* Requirements for the key:

  * Compact, fast for lookup.
  * Reversible back to original coordinates.
  * Stable across datasets.

Example:

```python
location_key = encode_coordinates(lat, lon)
```

At this stage, the crop object no longer carries raw `coordinates`; it carries `location_key`.

---

## Step 2. Construct the Base Crop Object

From the raw input + location key, build a structured **Crop Object**.
This includes required and optional fields.

### Required fields

* `crop_name` (string)
* `crop_variety` (string)
* `location_key` (integer or encoded string)
* `crop_plant_month` (int: 1–12)
* `crop_field_duration` (int: days)

### Optional but useful fields

* `region` (string, human-readable)
* `yield_per_acre` (float)
* `market_price_kg` (decimal)
* `crop_type` (string, e.g. cereal, legume)

If any **required field** is missing → the object fails validation and does not proceed.

---

## Step 3. Apply Crop_Condition_Rule

For each **unique combination of crop variety, plant month, and location key**, the system triggers the enrichment process.

This ensures that:

* The same variety planted in two regions is treated as two separate enriched objects.
* Different planting months of the same variety are treated independently.

---

## Step 4. Fetch Weather and Climate Data

Data is fetched from the **Open Meteo Climate API**.
Since their historical endpoint is broken, the **climate model forecast API with historical values** is used.

* Query parameters:

  * `latitude`, `longitude` (reconstructed from location_key)
  * `start_date`, `end_date` (planting date → planting date + duration_days)
  * `daily` variables:

    * `temperature_2m_max`, `temperature_2m_min`, `temperature_2m_mean`
    * `relative_humidity_2m_mean`, `relative_humidity_2m_min`, `relative_humidity_2m_max`
    * `wind_speed_10m_mean`, `wind_speed_10m_max`
    * `precipitation_sum`, `rain_sum`
    * `soil_moisture_0_to_10cm_mean`

If weather API fails → **critical error** (since weather is a required component).

---

## Step 5. Fetch Soil Data (Optional)

* Soil data source is **currently unknown**.
* Schema placeholder is retained:

  ```json
  "soil_information": {}
  ```
* If a reliable source is later integrated, enrichment can automatically populate this object.
* Missing soil data → does not block enrichment.

---

## Step 6. Fetch Crop Health (NDVI from GEE)

* Monthly vegetation health is derived using **Google Earth Engine Sentinel-2 NDVI composites**.
* Data is aggregated over the crop’s `field_duration`.
* Result is a series of NDVI values tied to the same `(location_key, crop_variety, crop_plant_month)`.

If GEE fetch fails → NDVI array may be empty, but enrichment still proceeds.

---

## step 7. Crop Configuration

### Purpose_

The **configuration system** allows flexible handling of crop-specific requirements, such as season length, phenology stages, and stress thresholds.

* Each record can:

  1. Use **system defaults** (generic for all crops).
  2. Use **crop-type defaults** (e.g., maize_config, sorghum_config).
  3. Apply **record-level overrides** (custom values for a specific field).

This layering ensures robustness: if configuration is missing, defaults are still available.

---

### Example Configuration

```json
{
  "crop_type": "maize",
  "variety": "H614",
  "configuration": {
    "season_stages": {
      "germination": 14,
      "vegetative": 40,
      "flowering": 20,
      "grain_filling": 16
    },
    "stress_thresholds": {
      "temperature_max": 35,
      "soil_moisture_min": 0.2
    }
  }
}
```

---

## Step 8. Assemble the Enriched Crop Object

```json
{
  "identifiers": {
    "crop_type": "maize",
    "variety": "H614",
    "location_key": "loc_-0.9711_34.9586",
    "planting_month": 4,
    "duration_days": 90
  },
  "static_context": {
    "region": "Trans-Nzoia, Kenya",
    "soil_properties": {}
  },
  "configuration": {
    "season_stages": {...},
    "stress_thresholds": {...}
  },
  "weather_time_series": {
    "2022": [...],
    "2023": [...]
  },
  "performance_scores": {
    "2022": {"ndvi_score": 0.72},
    "2023": {"ndvi_score": 0.68}
  }
}
```
This object becomes the canonical enriched unit for downstream processing.

---



## Pseudocode Representation

```python
def enrich_crops(raw_crops):
    enriched_crops = []

    for crop_name, details in raw_crops.items():
        # Step 1: Compute location key
        location_key = encode_coordinates(details["coordinates"])

        # Step 2: Build base crop object
        crop_obj = {
            "crop_name": crop_name,
            "crop_variety": details["variety"],
            "region": details["region"],
            "location_key": location_key,
            "crop_plant_month": details["planting_season_month"],
            "crop_field_duration": details["duration_days"]
        }

        # Step 3: Enrichment trigger (unique combo)
        if not validate_required(crop_obj):
            raise ValueError(f"Critical failure: Missing required fields for {crop_name}")

        # Step 4: Fetch weather data
        weather_data = fetch_weather(
            location_key,
            crop_obj["crop_plant_month"],
            crop_obj["crop_field_duration"]
        )
        if not weather_data:
            raise RuntimeError(f"Weather API failed for {crop_name}")

        # Step 5: Fetch soil data (optional)
        soil_data = fetch_soil(location_key) or {}

        # Step 6: Fetch crop health (NDVI)
        ndvi_data = fetch_ndvi(
            location_key,
            crop_obj["crop_plant_month"],
            crop_obj["crop_field_duration"]
        ) or []

        # Step 7: Assemble enriched object
        enriched_crop = {
            **crop_obj,
            "weather": weather_data,
            "soil_properties": soil_data,
            "ndvi": ndvi_data,
            "configuration": crop_config
        }

        enriched_crops.append(enriched_crop)

    return enriched_crops
```


This way the enrichment process is **self-contained, modular, fault-tolerant, and deterministic**.

>  .  

>  .  

>.  



# Configurable Crop Feature Engineering Pipeline

## Purpose
Transform crop performance data from time-series weather format into machine learning–ready feature–label pairs, with full support for crop-specific configurations to handle diverse agronomic requirements.

## Problem Solved
Given historical crop data where each record contains:

- Multiple years of daily weather observations (ie 6+ months per year)  
- Annual performance scores (yield, quality, etc.)  
- Crop variety, location, and planting information  

We need to:

1. Extract meaningful agronomic features from raw weather time series  
2. Handle crop-specific thermal requirements, stress tolerances, and phenology  
3. Generate features that capture critical growth periods and stress events  
4. Pair features with labels for supervised learning  
5. Make the system adaptable to any crop type without code changes  

---


## Architecture

### Input Data Structure
```

Crop Record:
├─ Identifiers: crop_variety, plant_month, location_key
├─ Static Context: elevation, soil_properties
├─ Configuration: crop_config (OPTIONAL – overrides defaults)
├─ Weather Time Series: [{year: 2020, daily_data: [...]}, ...]
└─ Performance Scores: [{year: 2020, score: 4.5}, ...]

```

#### configuration object specification
``` 
crop_config = {
    
    // PHENOLOGY CONFIGURATION
    phenology: {
        // Growth stage boundaries (as percentages of season length, 0-100)
        early_vegetative: {start_pct: 0, end_pct: 25},
        vegetative: {start_pct: 25, end_pct: 50},
        reproductive: {start_pct: 40, end_pct: 75},
        maturation: {start_pct: 70, end_pct: 100},
        critical_period: {start_pct: 40, end_pct: 70},  // Most sensitive period
        
        // Alternative: Absolute day numbers (overrides percentages if provided)
        // early_vegetative: {start_day: 0, end_day: 30},
        // vegetative: {start_day: 30, end_day: 60},
        // etc.
    },
    
    // THERMAL TIME CONFIGURATION
    thermal: {
        base_temperature: 10,      // °C, minimum temp for growth
        optimal_temperature: 25,   // °C, optimal for growth
        max_temperature: 35,       // °C, growth stops above this
        gdd_calculation_method: "simple",  // "simple" | "modified" | "triangular"
        
        // For modified GDD calculation (optional)
        upper_threshold: 30,       // Cap daily temp at this value
    },
    
    // STRESS THRESHOLDS
    stress_thresholds: {
        // Heat stress
        heat_stress_temp: 32,           // °C, daily max above this = stress
        extreme_heat_temp: 38,          // °C, severe stress threshold
        heat_stress_duration: 3,        // consecutive days to count as event
        
        // Cold stress
        cold_stress_temp: 5,            // °C, daily min below this = stress
        frost_temp: 0,                  // °C, frost damage threshold
        freezing_damage_temp: -2,       // °C, severe damage threshold
        
        // Water stress
        drought_soil_moisture: 0.15,    // volumetric soil moisture threshold
        severe_drought_moisture: 0.10,  // severe drought threshold
        waterlogging_moisture: 0.35,    // upper threshold for waterlogging
        
        // Atmospheric stress
        high_vpd: 2.5,                  // kPa, high evaporative demand
        extreme_vpd: 4.0,               // kPa, severe stress
        
        // Precipitation
        dry_day_threshold: 1.0,         // mm, below this = dry day
        heavy_rain_threshold: 25,       // mm, above this = heavy rain event
        
        // Wind stress
        strong_wind_threshold: 15,      // m/s, lodging/damage risk
        extreme_wind_threshold: 20,     // m/s, severe damage risk
    },
    
    // FEATURE WEIGHTS (0-1, indicating importance)
    feature_importance: {
        weight_early_season: 1.0,       // How much to weight early features
        weight_reproductive: 1.5,       // Reproductive stage often most critical
        weight_maturation: 0.8,         // Late season may be less critical
        
        // Stress type importance
        heat_stress_weight: 1.0,
        cold_stress_weight: 1.0,
        drought_stress_weight: 1.2,     // Often most limiting factor
        vpd_stress_weight: 0.8,
    },
    
    // ROLLING WINDOW CONFIGURATION
    rolling_windows: {
        short_window: 7,                // days
        medium_window: 14,              // days
        long_window: 30,                // days
        critical_window: 7,             // days for "worst week" detection
    },
    
    // CROP-SPECIFIC BEHAVIORS
    crop_characteristics: {
        is_perennial: false,            // true for tree crops, false for annuals
        photoperiod_sensitive: false,   // true if day length affects development
        deep_rooted: false,             // if true, may need deeper soil moisture
        c4_photosynthesis: false,       // C4 crops have different heat tolerance
        frost_tolerant: false,          // can survive light frost
        flood_tolerant: false,          // can handle waterlogging
        
        // Expected season length (days) - used for validation
        typical_season_length: 120,     // days from planting to harvest
        min_season_length: 90,
        max_season_length: 150,
    },
    
    // INTERACTION EFFECTS
    interactions: {
        // Modify how stresses combine
        heat_drought_multiplier: 1.5,   // Combined heat+drought is worse
        wind_precipitation_factor: 0.8, // Wind during rain affects differently
    },
    
    // DATA QUALITY SETTINGS
    data_handling: {
        max_missing_days: 7,            // Max consecutive missing days allowed
        interpolate_missing: true,      // Whether to interpolate gaps
        outlier_detection: true,        // Flag and handle outliers
        
        // Outlier thresholds (in standard deviations)
        outlier_threshold: 4.0,
    }
}

```

#### Crop-type default configurtation examples:
This system is supposed to allow us to have predefined crop specific defaults just in case a crop does not have the configuration object:

```
MAIZE_CONFIG = {
    thermal: {base_temperature: 10, optimal_temperature: 25, max_temperature: 35},
    stress_thresholds: {
        heat_stress_temp: 32,
        drought_soil_moisture: 0.15,
    },
    phenology: {
        critical_period: {start_pct: 45, end_pct: 65}  // Tasseling/silking
    },
    crop_characteristics: {
        c4_photosynthesis: true,
        typical_season_length: 120
    }
}

WHEAT_CONFIG = {
    thermal: {base_temperature: 0, optimal_temperature: 20, max_temperature: 30},
    stress_thresholds: {
        heat_stress_temp: 30,
        frost_temp: -2,  // More frost tolerant
    },
    phenology: {
        critical_period: {start_pct: 50, end_pct: 70}  // Anthesis/grain fill
    },
    crop_characteristics: {
        frost_tolerant: true,
        typical_season_length: 150
    }
}

RICE_CONFIG = {
    thermal: {base_temperature: 10, optimal_temperature: 28, max_temperature: 35},
    stress_thresholds: {
        heat_stress_temp: 35,  // More heat tolerant
        waterlogging_moisture: 0.45,  // Can handle wet conditions
    },
    phenology: {
        critical_period: {start_pct: 50, end_pct: 65}  // Flowering
    },
    crop_characteristics: {
        flood_tolerant: true,
        typical_season_length: 120
    }
}
```

#### System default configuration
> Applys to all crops that dont have a crop type specific configuration
Defined in the psuedocode as default values

## psuedocode
> [crop agnostic pipeline pseudocode](./data_enrich.psuedo)

---

### Stage 1: Configuration Resolution
**Priority Hierarchy:**
1. Record-specific config (`crop_record.crop_config`)  
2. Crop-type defaults (e.g., `MAIZE_CONFIG`, `WHEAT_CONFIG`)  
3. System defaults (conservative values for any crop)  

**Merged Config Contains:**
- Phenology: growth stage boundaries (% or absolute days)  
- Thermal: base/optimal/max temps, GDD calculation method  
- Stress Thresholds: heat, cold, drought, VPD, wind limits  
- Feature Importance: weights for different periods/stresses  
- Rolling Windows: 7/14/30-day statistics windows  
- Crop Characteristics: perennial, photoperiod, root depth, etc.  
- Interactions: multipliers for combined stresses  
- Data Handling: quality checks, missing data tolerance  

**Output:** Validated, crop-specific configuration object  

---

### Stage 2: Static Feature Extraction
Features that remain constant across years:  

- **Direct:** elevation, soil properties (pH, organic matter, texture)  
- **Derived:** elevation zones, elevation risk scores  
- **Temporal:** planting month numeric, seasonality indicators  
- **Config-based:** typical season characteristics  

**Output:** Static feature dictionary (reused across all years)  

---

### Stage 3: Year-by-Year Time Series Processing
For each year in `weather_time_series`:  

**3A. Data Quality Validation**  
- Check season length  
- Detect missing data gaps  
- Flag outliers (if enabled)  
- Skip year if quality insufficient  

**3B. Match Performance Label**  
- Match corresponding score for year  
- Skip year if no label  

**3C. Define Growth Stages**  
- Use `config.phenology` to partition season  
- Calculate stage boundaries  
- Identify critical period  

**3D. Temporal Feature Generation (100+ features)**  
- **Cumulative:** GDD, precipitation, ratios, accumulation rates  
- **Stage-Specific Statistics:** temperature, soil moisture, humidity, wind  
- **Stress Events:** heat, cold, drought, VPD, waterlogging, wind  
- **Temporal Dynamics:** depletion rates, trends, variability  
- **Extreme Events:** peaks, driest period, heavy rain, last rain timing  
- **Interaction Features:** heat-humidity, evaporative stress, wind-chill, precipitation efficiency  
- **Rolling Window Stats:** 7/14/30-day averages, extremes, variability  

**3E. Feature Merging**  
- Combine static + temporal features  
- Add metadata (year, config summary)  

**3F. Create Feature–Label Pair**  
```

{ year, features, label, config_used }

```

---

### Output Data Structure
```

Transformed Record:
├─ Identifiers: crop_variety, plant_month, location_key
└─ Feature-Label Pairs:
[
  {
    year: 2020,
    features: {
        // Static
        elevation: 1650,
        soil_ph: 6.2,
        plant_month_numeric: 5,
        ...


       // Cumulative
       total_gdd: 2340,
       reproductive_precipitation: 145.5,
       ...

       // Stage-specific
       critical_period_temp_mean: 26.5,
       ...

       // Stress events
       heat_stress_days_critical: 8,
       drought_stress_days_weighted: 15.6,
       ...

       // Dynamics & extremes
       soil_moisture_depletion_rate: -0.003,
       last_rain_timing_pct: 85.3,
       ...

       // Interaction features
       heat_humidity_index_max: 28.5,
       evaporative_stress_mean: 125.3,
       ...

       // Rolling windows
       temp_max_rolling_7d_peak: 36.2,
       worst_week_timing_pct: 58.7,
       ...
     },
     label: 4.2,
     config_used: {
       base_temp: 10,
       heat_threshold: 32,
       critical_period: {start_pct: 40, end_pct: 70}
     }
   },
   { year: 2021, features: {...}, label: 3.5 },
   ...
 ]
```



---

## Key Features of the Pipeline

1. **Crop-Agnostic with Crop-Specific Flexibility**  
   - Sensible defaults for any crop  
   - Configurable via JSON (no code changes)  
   - Pre-built configs (maize, wheat, rice, etc.)  
   - Handles diverse types (annuals, perennials, C3/C4)  

2. **Comprehensive Feature Coverage**  
   - 100+ features: temporal, stress, interaction, statistical  
   - Captures variability, extremes, and critical timing  

3. **Phenology-Aware Processing**  
   - Growth stages by % of season or days  
   - Critical period detection  
   - Stage-specific features and weights  

4. **Robust Thermal Time Modeling**  
   - Multiple GDD calculation methods  
   - Crop-specific thresholds  
   - Temperate and tropical compatibility  

5. **Intelligent Stress Detection**  
   - Threshold-based with crop-specific values  
   - Duration-aware to avoid false positives  
   - Critical period weighting  
   - Combined stress interactions  

6. **Data Quality Management**  
   - Validates season length  
   - Handles missing/outlier data  
   - Graceful degradation for problematic years  

7. **Machine Learning Ready**  
   - Clean feature–label pairs  
   - Consistent naming  
   - Metadata included  
   - Easy conversion to DataFrames/arrays  

8. **Maintainability & Extensibility**  
   - Modular design  
   - Clear naming conventions  
   - Helper utilities for common ops  
   - Easily extensible feature sets  

---

## Configuration Flexibility

Users can override defaults at three levels:  

**Level 1: System Defaults** (built-in)  
- Conservative values suitable for most crops  

**Level 2: Crop-Type Defaults** (provided)  
- `MAIZE_CONFIG`: C4 crop, moderate heat tolerance  
- `WHEAT_CONFIG`: C3, cool season, frost tolerant  
- `RICE_CONFIG`: Heat and flood tolerant  
- `POTATO_CONFIG`: Cool season, moisture sensitive  
- `SORGHUM_CONFIG`: C4, heat/drought tolerant  
- `TOMATO_CONFIG`: Moderate requirements, frost sensitive  
- `COFFEE_CONFIG`: Perennial, shade/cool preference  

**Level 3: Record-Specific Overrides** (user-provided)  
- Fine-tune for specific varieties  

---

## Configuration Categories
- **Phenology:** stage boundaries, critical period  
- **Thermal:** base/optimal/max temp, GDD method  
- **Stress Thresholds:** heat, cold, drought, VPD, precipitation, wind  
- **Feature Importance:** stage and stress weighting  
- **Rolling Windows:** window sizes, critical detection periods  
- **Crop Characteristics:** lifecycle, tolerance, C3/C4 type, season length  
- **Interactions:** multipliers for combined stresses  
- **Data Handling:** missing data tolerance, outlier rules  

---

## Feature Summary (100+ Total)

**Static (10–20)**  
- Elevation, soil, planting month, derived zones  

**Cumulative (20–30)**  
- Total and stage-specific GDD  
- Precipitation totals and ratios  

**Stage-Specific (30–40)**  
- Temp, soil moisture, humidity, wind statistics  

**Stress Events (20–30)**  
- Heat, cold, drought, VPD, waterlogging, wind  
- Critical period–weighted versions  

**Temporal Dynamics (15–20)**  
- Depletion rates, accumulation velocities, variability  

**Extreme Events (10–15)**  
- Peak temp, driest period, heavy rains, last rain timing  

**Interaction Features (10–15)**  
- Heat–humidity index, evaporative stress, precipitation efficiency  

**Rolling Window Stats (10–15)**  
- 7/14/30-day rolling statistics  
- Worst week detection  

---

## Usage Patterns

**Basic Usage (with system defaults):**  
```python
# Example pseudocode
pipeline = CropFeaturePipeline()
features = pipeline.transform(crop_records)
````

