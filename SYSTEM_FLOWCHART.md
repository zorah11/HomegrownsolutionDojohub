# COMPLETE AGRICULTURAL MONITORING & PREDICTION SYSTEM
## Data Flow Architecture

## INTEGRATED SYSTEM OVERVIEW

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    AGRICULTURAL INTELLIGENCE PLATFORM                            │
│                                                                                  │
│  ┌────────────────────────────────┐      ┌────────────────────────────────┐   │
│  │   SYSTEM 1: YIELD PREDICTION   │      │  SYSTEM 2: FARMER MONITORING   │   │
│  │   (ML-Based Forecasting)       │      │  (Multi-Crop Portfolio Mgmt)   │   │
│  └────────────────────────────────┘      └────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

## COMPLETE DATA FLOW (BOTH SYSTEMS INTEGRATED)

```
                        ┌──────────────────────────────────┐
                        │   DATA SOURCES (SATELLITE + API)  │
                        └──────────────┬───────────────────┘
                                       │
                ┌──────────────────────┼──────────────────────┐
                │                      │                      │
                ▼                      ▼                      ▼
    ┌──────────────────┐   ┌──────────────────┐   ┌──────────────────┐
    │   Sentinel-2     │   │   ERA5/CHIRPS    │   │   Sentinel-1     │
    │   Optical        │   │   Weather Data   │   │   Radar          │
    │   10m resolution │   │   Temperature    │   │   Soil Moisture  │
    │   NDVI           │   │   Precipitation  │   │   VV Backscatter │
    └────────┬─────────┘   └────────┬─────────┘   └────────┬─────────┘
             │                      │                      │
             │                      │                      │
┌────────────┴──────────────────────┴──────────────────────┴────────────┐
│                         DATA PROCESSING LAYER                          │
└────────────┬──────────────────────┬──────────────────────┬────────────┘
             │                      │                      │
             ▼                      ▼                      ▼

═══════════════════════════════════════════════════════════════════════════
  SYSTEM 1: YIELD PREDICTION PIPELINE (SINGLE FARM, ML-BASED)
═══════════════════════════════════════════════════════════════════════════

STEP 1: RAW NDVI EXTRACTION
┌────────────────────────────────────────────┐
│ Input: Sentinel-2 imagery (2020-2025)     │
│                                            │
│ Processing:                                │
│ • Filter by farm boundary                 │
│ • Calculate NDVI = (NIR-Red)/(NIR+Red)    │
│ • Compute statistics per date             │
│                                            │
│ Output: df_timeseries                     │
│ • 46 observations                         │
│ • Columns: date, cycle_name,              │
│           ndvi_mean, ndvi_std,            │
│           image_count                     │
└────────────────┬───────────────────────────┘
                 │
                 ▼
                 
STEP 2: WEATHER DATA EXTRACTION
┌────────────────────────────────────────────┐
│ Input: ERA5 (temp) + CHIRPS (precip)      │
│                                            │
│ Processing:                                │
│ • Extract for same dates as NDVI          │
│ • Point extraction at farm centroid       │
│ • Convert units (Kelvin → Celsius)        │
│                                            │
│ Output: df_weather                        │
│ • 46 records                              │
│ • Columns: date, cycle_name,              │
│           temperature (°C),               │
│           precipitation (mm)              │
└────────────────┬───────────────────────────┘
                 │
                 │
        ┌────────┴────────┐
        │ MERGE ON DATE   │
        │ Inner Join      │
        └────────┬────────┘
                 │
                 ▼
                 
STEP 3: COMBINED DATASET
┌────────────────────────────────────────────┐
│ Processing: Merge NDVI + Weather          │
│                                            │
│ Output: df_combined                       │
│ • 46 rows (all dates matched)             │
│ • Columns: date, cycle_name,              │
│           ndvi_mean, ndvi_std,            │
│           temperature,                    │
│           precipitation                   │
│                                            │
│ Data Ready for Variable Extraction         │
└────────────────┬───────────────────────────┘
                 │
                 │
        ┌────────┴────────────┐
        │ GROUP BY SEASON     │
        │ Aggregate Features  │
        └────────┬────────────┘
                 │
                 ▼
                 
STEP 4: VARIABLE EXTRACTION
┌────────────────────────────────────────────┐
│ Processing: Season-level aggregation      │
│                                            │
│ For each season (5 seasons):              │
│ • NDVI variables (15):                    │
│   - mean, max, min, std                   │
│   - peak_value, peak_timing               │
│   - slope, rate_increase, rate_decrease   │
│   - growing_season_length                 │
│   - etc.                                  │
│                                            │
│ • Temperature variables (7):              │
│   - mean, max, min, std                   │
│   - growing_degree_days                   │
│   - heat_stress_days                      │
│                                            │
│ • Precipitation variables (7):            │
│   - cumsum, mean, max                     │
│   - drought_days, excess_rain_days        │
│   - distribution patterns                 │
│                                            │
│ Output: df_variables_enhanced             │
│ • 5 rows (5 seasons)                      │
│ • 29 extracted variables per season       │
└────────────────┬───────────────────────────┘
                 │
                 │
        ┌────────┴─────────────────┐
        │ RANDOM FOREST RANKING    │
        │ Variable Importance      │
        └────────┬─────────────────┘
                 │
                 ▼
                 
STEP 5: VARIABLE SELECTION
┌────────────────────────────────────────────┐
│ Processing: Rank features by importance   │
│                                            │
│ Method: RandomForestRegressor             │
│ • Train on historical yields              │
│ • Extract variable_importances_           │
│ • Select top N variables                  │
│                                            │
│ Output: top_variable_names                │
│ • 5 variables selected (29 → 5)           │
│ • Top variables:                          │
│   1. ndvi_peak_value                      │
│   2. precipitation_cumsum                 │
│   3. temperature_growing_mean             │
│   4. ndvi_slope                           │
│   5. growing_season_length                │
│                                            │
│ Dataset: X_train_selected                 │
│ • 4 historical seasons × 5 variables      │
└────────────────┬───────────────────────────┘
                 │
                 │
        ┌────────┴───────────────────┐
        │ LEAVE-ONE-OUT CROSS-VAL    │
        │ Model Training & Validation │
        └────────┬───────────────────┘
                 │
                 ▼
                 
STEP 6: MODEL TRAINING & CROSS-VALIDATION
┌────────────────────────────────────────────┐
│ Algorithm: Random Forest Regressor        │
│ • n_estimators = 100 trees                │
│ • random_state = 42                       │
│                                            │
│ Validation: Leave-One-Out CV              │
│ • 4 training samples                      │
│ • Each sample tested once                 │
│ • Predictions: y_pred_cv                  │
│                                            │
│ Performance Metrics:                      │
│ • MAE (Mean Absolute Error): 467.5 kg    │
│ • RMSE (Root Mean Squared): 512.3 kg     │
│ • R² (Coefficient of Determination):     │
│   -0.527 (poor fit - limited data)       │
│                                            │
│ Model: rf_model (trained on all 4)        │
└────────────────┬───────────────────────────┘
                 │
                 │
        ┌────────┴──────────────────┐
        │ PREDICT CURRENT SEASON    │
        │ Apply trained model       │
        └────────┬──────────────────┘
                 │
                 ▼
                 
STEP 7: FINAL YIELD PREDICTION
┌────────────────────────────────────────────┐
│ Input: Current season (Jan-Jun 2025)      │
│ • 5 selected variables extracted          │
│                                            │
│ Prediction Process:                       │
│ • Run 100 trees in RF model               │
│ • Each tree votes                         │
│ • Average predictions                     │
│ • Calculate std deviation                 │
│                                            │
│ Output:                                   │
│ • Predicted Yield: 1440.0 kg             │
│ • Std Deviation: ±431.7 kg               │
│ • Confidence Interval (95%):             │
│   - Lower: 1008 kg                       │
│   - Upper: 1872 kg                       │
│                                            │
│ Status: Prediction complete ✓             │
└────────────────────────────────────────────┘


═══════════════════════════════════════════════════════════════════════════
  SYSTEM 2: MULTI-FARMER MONITORING (PORTFOLIO MANAGEMENT)
═══════════════════════════════════════════════════════════════════════════

STEP 1: FARM REGISTRATION & PORTFOLIO
┌────────────────────────────────────────────┐
│ Input: Farmer portfolio data              │
│                                            │
│ Farmer: Noah Natude                       │
│ Total Holdings: 46 acres                  │
│                                            │
│ Crops:                                    │
│ • Rice: 30 acres                          │
│   Coords: [(lat1, lon1), ...]            │
│                                            │
│ • Tomatoes: 10 acres                      │
│   Coords: [(lat2, lon2), ...]            │
│                                            │
│ • Beans: 6 acres                          │
│   Coords: [(lat3, lon3), ...]            │
│                                            │
│ Data Structure: farmer_portfolio dict     │
└────────────────┬───────────────────────────┘
                 │
                 ├──────────────┬──────────────┬──────────────┐
                 │              │              │              │
                 ▼              ▼              ▼              ▼
                 
STEP 2: NDVI HISTORY    STEP 3: SOIL         STEP 4: MOISTURE    STEP 5: ANOMALY
┌──────────────────┐   ┌──────────────┐     ┌──────────────┐    ┌──────────────┐
│ High-Res NDVI    │   │ Soil Chemistry│     │ Radar Moisture│    │ Health Alerts│
│ Timeline         │   │ Analysis      │     │ Monitoring    │    │              │
│                  │   │               │     │               │    │              │
│ Data Source:     │   │ Data Source:  │     │ Data Source:  │    │ Data Source: │
│ Sentinel-2       │   │ SoilGrids     │     │ Sentinel-1    │    │ NDVI Trend   │
│ (10-day steps)   │   │ ISRIC         │     │ C-band radar  │    │ Analysis     │
│                  │   │               │     │               │    │              │
│ Function:        │   │ Function:     │     │ Function:     │    │ Function:    │
│ extract_high_res │   │ get_static_   │     │ get_soil_     │    │ detect_pest_ │
│ _historical()    │   │ soil_health() │     │ moisture_     │    │ anomaly()    │
│                  │   │               │     │ trend()       │    │              │
│ Process:         │   │ Process:      │     │ Process:      │    │ Process:     │
│ • Query S2 for   │   │ • Extract pH  │     │ • Get VV      │    │ • Calculate  │
│   2020-2025      │   │ • Get N levels│     │   backscatter │    │   NDVI drops │
│ • 10-day         │   │ • Get SOC     │     │ • Recent 60   │    │ • Detect     │
│   intervals      │   │ • At centroid │     │   days        │    │   >15% drops │
│ • Per parcel     │   │ • Top 5cm     │     │ • Median      │    │ • Flag       │
│ • Rolling 30-day │   │ • Static data │     │   composite   │    │   stressed   │
│   average        │   │               │     │               │    │   areas      │
│                  │   │               │     │               │    │              │
│ Output:          │   │ Output:       │     │ Output:       │    │ Output:      │
│ • df_ndvi        │   │ • pH: X.XX    │     │ • VV: -XX dB  │    │ • Anomaly    │
│ • Columns:       │   │ • N: X.X g/kg │     │ • Status:     │    │   detected   │
│   - date         │   │ • SOC: X g/kg │     │   Adequate/   │    │ • Location   │
│   - crop         │   │ • Advice text │     │   Dry         │    │ • Severity   │
│   - ndvi         │   │               │     │               │    │              │
│   - ndvi_30day   │   │ Interpretation│     │ Alert if:     │    │ Alert if:    │
│   - status       │   │ • pH < 5.5:   │     │ • VV < -15 dB │    │ • Drop >15%  │
│                  │   │   Add lime    │     │ • Drought risk│    │ • Sustained  │
│ Timeline:        │   │ • N < 1.5:    │     │               │    │   decline    │
│ 500+ obs/crop    │   │   Add legumes │     │               │    │              │
└────────┬─────────┘   └──────┬───────┘     └──────┬────────┘    └──────┬───────┘
         │                    │                    │                    │
         │                    │                    │                    │
         └────────────────────┴────────────────────┴────────────────────┘
                                        │
                                        ▼
                          ┌─────────────────────────┐
                          │ MONITORING OUTPUTS      │
                          │                         │
                          │ For each farmer/crop:   │
                          │ • Current NDVI status   │
                          │ • Soil conditions       │
                          │ • Moisture levels       │
                          │ • Health alerts         │
                          │                         │
                          │ Portfolio totals:       │
                          │ • 6 farmers             │
                          │ • 14 crops              │
                          │ • 109 acres monitored   │
                          └─────────────────────────┘


═══════════════════════════════════════════════════════════════════════════
  UNIFIED OUTPUT: COMPLETE SYSTEM DATA MOVEMENT EXCEL
═══════════════════════════════════════════════════════════════════════════

┌───────────────────────────────────────────────────────────────────────────┐
│                  Complete_System_Data_Movement.xlsx                        │
└───────────────────────────────────────────────────────────────────────────┘

SHEET 1: Yield_Prediction_Flow (151 rows)
┌─────────────────────────────────────────────────────────────────────────┐
│ Step │ Transformation              │ Data Points │ Sample Values        │
├──────┼─────────────────────────────┼─────────────┼─────────────────────┤
│ 1    │ Satellite → NDVI            │ 46 obs      │ 0.643, 0.591, 0.492 │
│ 2    │ ERA5/CHIRPS → Weather       │ 46 records  │ 24.2°C, 33.5mm      │
│ 3    │ Merge on date               │ 46 combined │ NDVI+Temp+Precip    │
│ 4    │ Aggregate by season         │ 5 × 29 vars │ ndvi_mean=0.621     │
│ 5    │ RF ranking                  │ 29 → 5      │ Top 5 selected      │
│ 6    │ LOO Cross-validation        │ 4 samples   │ MAE=467.5 kg        │
│ 7    │ Final prediction            │ 1 result    │ 1440±432 kg         │
└──────┴─────────────────────────────┴─────────────┴─────────────────────┘

SHEET 2: Farmer_Monitoring_Flow (6 rows)
┌─────────────────────────────────────────────────────────────────────────┐
│ Step │ Analysis           │ Farmer       │ Crop(s)      │ Status      │
├──────┼────────────────────┼──────────────┼──────────────┼─────────────┤
│ 1    │ Registration       │ Noah Natude  │ Rice         │ Active      │
│ 2    │ Portfolio          │ Noah Natude  │ 3 crops      │ Monitored   │
│ 3    │ Soil Health        │ Noah Natude  │ Rice         │ Baseline    │
│ 4    │ Moisture           │ Noah Natude  │ Rice         │ Monitored   │
│ 5    │ Anomaly Detection  │ Noah Natude  │ Rice         │ Active      │
└──────┴────────────────────┴──────────────┴──────────────┴─────────────┘


═══════════════════════════════════════════════════════════════════════════
  DATA TRANSFORMATION METRICS
═══════════════════════════════════════════════════════════════════════════

YIELD PREDICTION SYSTEM:
• Input: 92 data points (46 NDVI + 46 Weather)
• Processing: 7 transformation steps
• Intermediate: 46 combined → 5 seasons → 5 variables
• Output: 1 prediction with confidence interval
• Time: ~200ms (using cached data)

FARMER MONITORING SYSTEM:
• Input: 6 farmers, 14 crops, 109 acres
• Processing: 5 parallel analyses
• Checks: NDVI timeline, Soil, Moisture, Anomalies
• Output: Monitoring reports with alerts
• Coverage: 500+ historical observations per crop

COMBINED EXCEL EXPORT:
• Total Rows: 157 (151 + 6)
• File Size: ~20 KB
• Sheets: 2 (separate but linked systems)
• Generation Time: ~290ms


═══════════════════════════════════════════════════════════════════════════
  KEY INSIGHTS
═══════════════════════════════════════════════════════════════════════════

SYSTEM 1 (Yield Prediction):
✓ Complete data lineage from raw satellite to final prediction
✓ All transformations documented with actual values
✓ Variable extraction: 29 variables → 5 most important
✓ Model validation: Cross-validated performance metrics
✓ Uncertainty quantification: ±432 kg confidence interval

SYSTEM 2 (Farmer Monitoring):
✓ Multi-crop portfolio management (14 crops, 109 acres)
✓ Parallel analysis streams (NDVI, Soil, Moisture, Anomaly)
✓ Historical baseline + real-time monitoring
✓ Alert system for drought, pests, soil deficiencies
✓ Scalable to multiple farmers (currently 6 farmers)

INTEGRATION POINTS:
• Both systems use same satellite sources (Sentinel-2)
• Weather data shared between systems
• NDVI calculations standardized
• Monitoring system can feed into prediction system
• Unified output format (Excel with both sheets)


═══════════════════════════════════════════════════════════════════════════
  END OF FLOWCHART
═══════════════════════════════════════════════════════════════════════════
Generated: February 5, 2026
Data Period: 2020-2025
Farm: Uganda
Systems: 2 (Prediction + Monitoring)
Total Data Points: 92 + 500+ monitoring observations
