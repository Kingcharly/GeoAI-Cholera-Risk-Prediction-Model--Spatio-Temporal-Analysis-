# 🗺️ GeoAI Cholera Risk Prediction Model (Spatio-Temporal Analysis)

## 1. 🎯 Project Goal & Context

This project develops a predictive model to identify Nigerian Local Government Areas (LGAs) at high risk of Cholera outbreaks. The approach leverages **Geographic Information Systems (GIS)** for advanced feature engineering and **Machine Learning (ML)** for predictive modeling, creating a robust, actionable **Spatio-Temporal Panel Dataset**.

| Metric | Target | Solution |
| :--- | :--- | :--- |
| **Problem** | High Cholera Case Fatality Ratio (CFR) and recurring outbreaks in Nigeria. | Predict **Risk Class** (0/1) based on static environmental and infrastructure factors. |
| **Output** | A **Predictive Risk Map** visualized in a GIS platform (ArcGIS Pro). | A trained **Random Forest Classifier** model. |
| **Methodology**| Overcome the challenge of scarce public health data by creating a **Spatio-Temporal Panel Dataset** (combining multiple years of data into a single, large training set). |

---

## 2. 📊 Data Engineering & Acquisition

This project required merging two disparate data sources: NCDC reports (the **Target Variable**) and national geospatial layers (the **Feature Variables**).

### 2.1. Target Variable (Y) Extraction: NCDC Reports

The target data was extracted from multiple published **NCDC Cholera Situation Reports** (e.g., 2021, 2024, 2025).

* **Target Selection:** Due to data incompleteness (reports only listing the "Top 14" affected LGAs), the project pivoted to a **Binary Classification** task.
    * **Positive Samples (Risk=1):** LGAs listed in the "Top 14" affected LGAs for a specific time period.
    * **Negative Samples (Risk=0):** LGAs that were *not* on the "Top 14" list for the same time period.
* **Methodology:** A **Spatio-Temporal Panel** approach was used, creating one row per **(LGA, Time\_Period)**.
* **Data Size:** The final dataset contains approximately **3800+ rows**, which is sufficient for robust classification modeling.

### 2.2. Feature Variable (X) Sourcing: GIS Data

The following public and estimated geospatial layers were acquired and processed using **ArcGIS Pro**:

* **Administrative:** LGA boundaries (774 LGAs).
* **Infrastructure:** Health Facilities, Roads, Water Points (reservoirs/wetlands), Waterways (rivers).
* **Demographics:** Estimated Population Data and LGA Area.

---

## 3. ⚙️ Feature Engineering & EDA

The core of the project involved transforming raw geospatial data into meaningful **risk features**.

### 3.1. Feature Creation (GIS Analysis)

| Feature Category | Feature Name | GIS Operation | Justification |
| :--- | :--- | :--- | :--- |
| **Water Risk** | `NEAR_DIST_Water` | Near Analysis | Distance to contamination source (rivers, open water). |
| **Water Risk** | `water_per_lga` | Spatial Join / Area Calculation | Concentration of water points/bodies within the LGA. |
| **Number of Health Facitilites** | `num_health_facilities` | Spatial Join  | Concentration of healthcare within the LGA. |
| **Density/Access**| `population_density`| Field Calculator (Pop / Area) | Strain on centralized sanitation systems. |
| **Hybrid** | `water_density_interaction` | `water_per_lga` $\times$ `log_popdensity` | Engineered feature to find areas that are **both** dense and water-rich. |

### 3.2. Exploratory Data Analysis (EDA) Insights

The EDA focused on visually separating the `Safe (0)` and `Outbreak (1)` classes using boxplots, leading to crucial feature refinement:

1.  **Water is the Key Driver:** The `water_per_lga` feature was found to be the **strongest predictor**. Outbreak LGAs consistently had a significantly higher concentration of water points than safe LGAs.
2.  **Density Alone is Weak:** The raw and log-transformed `popdensity` feature showed minimal difference between the two classes. This validated the hypothesis that high-density areas are only high-risk when water/sanitation fails.
3.  **Engineered Feature Success:** The **`water_density`** feature was created to amplify the signal missed by individual features.
4.  **Transformation:** Features like `popdensity` and `water_density` were stabilized using **`np.log1p()` transformation** to mitigate the massive skew and impact of spatial outliers.
5.  **Failed Features:** Features like `vulnerability` and simple counts of health facilities were discarded due to poor separation and noise.

---

### 3.3. Correlation & Predictor Validation

Analysis of the correlation matrix provided the following insights:

* **Primary Predictor:** The **`water_density`** feature shows the **strongest linear correlation** with the target variables (`Case` count: **+0.37**; `Has_Outbreak`: **+0.18**), confirming the hypothesis that concentrated water sources drive risk.
* **Health Access Insight:** The distance to the nearest health facility (`NEAR_DIST_Health`) has a strong negative correlation with the outbreak status (**-0.55** with `Has_Outbreak`). This counter-intuitive finding suggests that outbreaks primarily occur in areas *with* health access (urban/peri-urban), potentially due to dense, unsanitary living conditions overwhelming nearby clinics.
* **Feature Consolidation:** The raw features (`popdensity` and `num_health_facilities`) were found to be highly correlated with their log-transformed or per-capita counterparts. To avoid **multicollinearity**, the raw versions are dropped in the final model in favor of the stabilized, more insightful versions.

---

## 4. Machine Learning Strategy

* **Problem:** Extreme Class Imbalance (Only 3.7% of LGAs had outbreaks).
* **Solution:** Applied **SMOTE (Synthetic Minority Over-sampling Technique)** to balance the training data.
* **Model:** **Random Forest Classifier** with Probability Threshold Tuning.
    * **Threshold Shift:** Lowered decision threshold from 0.5 to **0.2** to prioritize Recall (catching outbreaks) over Precision.

---

### 4.1. Pre-Modeling Pipeline

* **Target Type:** Binary Classification (`Has_Outbreak` 0/1).
* **Encoding:** The `State_Name` and `LGA_Name` identifiers were removed from the feature set to prevent overfitting, though they are retained as a **DataFrame Index** for post-model interpretability.
* **Scaling:** All final features will be normalized using **StandardScaler** to prepare them for the model.

### 4.2. Modeling Performance

| Metric | Score | Interpretation |
| :--- | :--- | :--- |
| **Recall** | **86.7%** | The model successfully identified **26 out of 30** actual outbreaks in the validation set. |
| **Precision** | **5.1%** | Expected trade-off for a vulnerability screener. The "False Positives" represent high-risk zones that require monitoring. |
| **Accuracy** | **35%** | Deliberately sacrificed to maximize sensitivity for public safety. |

---

## 🗺️ Operational Results: The "Silent Risk" Hierarchy

The model identified **480 vulnerable LGAs**. To make this actionable for the NCDC, I implemented a **Tiered Risk Scoring System**:

| Risk Tier | Count | Recommended Action |
| :--- | :--- | :--- |
| **🔴 Tier 1: Critical** | **116 LGAs** | **Immediate Action:** Deploy WASH teams and oral vaccines. |
| **🟠 Tier 2: High** | **188 LGAs** | **Priority Monitoring:** Increase surveillance frequency. |
| **🟡 Tier 3: Moderate** | **176 LGAs** | **Routine Surveillance:** Maintain standard reporting protocols. |

### 📍 Visual Output

![alt text](<Vulnerability Map.jpg>)

---

## 🛠️ 5. Tech Stack
* **GIS:** ArcGIS Pro (Spatial Join, Near Analysis, Density Calculation)
* **Python:** Pandas, Scikit-Learn, Imbalanced-Learn (SMOTE)
* **Visualization:** Matplotlib, Seaborn

## 🚀 How to Run
1.  Clone the repo.
2.  Install dependencies: `pip install -r requirements.txt`
3.  Run `notebooks/04_Modeling_RandomForest.ipynb` to retrain the model.