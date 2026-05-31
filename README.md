# Quantifying Clinical Gaps in Valvular Heart Disease: Using the Medication Burden Index (MBI) to Triage High-Complexity Intervention Candidates.
## 📄 Executive summary

| Section | Content |
|:---|:---|
| **Context**|International Cardiology mission phase 1 (2025 cohort) analysis of Valvular Heart Disease in León, Nicaragua|
| **Challenge** | In a 10-day high-volume "clinical sprint," cardiologists often skip fields for healthy valves: If a valve is normal, the entry is left blank to save time. This creates Missing Not At Random (MNAR) bias. |
| **Strategy** | Natural Normal Imputation: Gaussian noise centered around healthy means (e.g., RVSP $25 \pm 4 \text{ mmHg}$) to restore the statistical variance of a healthy population. |
| **Key metric** | Medication Burden Index (MBI): A quantitative surrogate for clinical complexity and pulmonary hypertension based on pharmacological intensity. |
| **Outcome** | MBI-based triage proposal for identification of complex and high risk patients. |

## 📑 Table of Contents

* [Outcome: MBI-driven Triage](#-outcome-mbi-driven-triage)
* [Data Context](#-data-context)
* [Technical Structure](#-technical-structure)
* [Database Schema and Governance](#-database-schema-and-governance)
* [Feature Engineering and Bias Mitigation](#-feature-engineering-and-bias-mitigation)
  * [The Medication Burden Index (MBI)](#the-medication-burden-index-mbi)
  * [Imputation to restore physiologic variance](#imputation-to-restore-physiologic-variance)
    * [Imputation sensitivity analysis](#imputation-sensitivity-analysis)

## 📊 Outcome: MBI-driven triage
The MBI score serves as a robust proxy for anatomical complexity and critical hemodynamic compromise (AUC: 0.82).

![MBI Zones](assets/output_triage_zones.png)

Based on the cohort analysis, these thresholds guide the intake staff in prioritizing patients.

| MBI Range | Triage Zone | Clinical Interpretation |
|---:|:---|:---|
| **< 4.0** | **Zone 1: Standard** | Likely compensated. |
| **4.0 - 5.25** | **Zone 2: Complex** | Optimal intervention area: High probability of hemodynamic complexity. |
| **5.25 - 5.5** | **Zone 3: Critical** | 85% Precision for Critical Pulmonary Hypertension ($RVSP > 60~mmHg$). |
| **> 5.5** | **Zone 4: Futility risk** | Intervention risk likely outweighs potential hemodynamic gain. |

## 🔎 Data context

This project analyzes real-world clinical data from a high-volume medical mission in León, Nicaragua. The cardiology brigade operates via a structured, three-phase clinical workflow designed to bridge the gap between primary screening and advanced cardiac intervention:

* **Phase 1 — General Cardiology Clinic:** High-throughput diagnostic screening, echocardiographic evaluation, and procedure prioritization.

* **Phase 2 — Electrophysiology:** Arrhythmia management, including cardiac ablations and permanent pacemaker/ICD implantation.

* **Phase 3 — Interventional Cardiology:** Percutaneous structural heart procedures and collaborative case reviews.

This analysis focuses on the Phase 1 - 2025 Cohort. To ensure clinical relevance and focus on adult structural heart disease, the population was filtered as follows:  

* **Initial Enrollment:** $N=187$ patients.

* **Exclusion Criteria:** Patients $< 15$ years old or those with entirely normal echocardiographic findings (no disease criteria met).

* **Final Analytical Sample:** $N=152$ patients.


## 💻 Technical structure
### Project Schema
```text
├── data/
│   ├── raw/                                    # Original CSV
│   └── processed/                              # Post-Transformation data
├── notebooks/
│   ├── 01_extraction_transformation.ipynb      # Extraction & Transformation
│   ├── 02_loading.ipynb                        # MySQL connector script
│   └── 03_analysis.ipynb                       # Data Analysis
├── sql/                                        # SQL-related files
│   └── schema_cardio_mbi.mwb                   # MySQL database schema
├── tableau/                                    # Tableau files
├── assets/                                     # Auxiliary files for documentation
├── config.py                                   # Python configuration dictionaries
├── requirements.txt                            # Python library dependencies
└── README.md
```

## 💽 Database schema and governance
To transition this dataset from a flat file to an analytics-ready warehouse, the database layer implements a **5-table Snowflake Schema** . While this structure enforces **Third Normal Form (3NF)** compliance across core tables to eliminate update anomalies, it implements a pragmatic, **denormalized hybrid strategy** within the associative entities to optimize analytic queries:

* **Entity Isolation:** Patient-level continuous demographics and baseline variables are strictly isolated within `fact_patient`, anchoring the baseline denominator ($N=152$).
* **Nomenclature Normalization:** Dimension tables `dim_diagnoses` and `dim_medications` function as a single source of truth, enforcing clean text constraints and eliminating manual string typographical variances.
* **Many-to-Many Resolution:** Complex multi-valvular disease records and polypharmacy exposures are decoupled into dedicated bridge tables (`bridge_diagnoses` and `bridge_medications`). This database design allows to execute localized cohort queries without duplicating core patient parameters.  
* **Programmatic Pipeline Execution:** The complete schema instantiation and transactional loading sequence are engineered natively in Python using `mysql-connector-python`.

![Database schema](assets/database_schema.png)

## 🛠️ Feature engineering and bias mitigation
### The Medication Burden Index (MBI)
The MBI transforms fragmented medication lists into a single, actionable score that quantifies how aggresively a patient is being medically managed, which serves as a proxy for both physiological severity and care complexity.

We define the score mathematically as:

$$MBI = \sum \left( \text{Class Weight} \times \frac{\text{Total Daily Dose}}{\text{Maximum Daily Dose}} \right)$$

* **Dose Ratio:** The total milligrams consumed by the patient in 24 hours. For example, a patient on Furosemide 40mg every 12h has a $TDD=80~mg$.

* **Weight:** Based on the drug's therapeutic impact for valvular heart disease (e.g., Loop Diuretics have a higher weight than Lipid Lowering agents).

| Weight | Priority | Medication Classes |
|---:|:---|:---|
| **3.0** | Critical | Loop diuretics, pulmonary vasodilators. |
| **2.0** | High | RAAS inhibitors, beta-blockers, SGLT2 inhibitors. |
| **1.0** | Moderate | Anticoagulants, calcium channel blockers. |
| **0.5** | Maintenance | Statins. |

**Implementation example: Normalizing calcium channel blockers (CCBs)**

To ensure the MBI reflects true clinical intensity rather than raw milligrams, we normalize dosages against their therapeutic ceilings. Consider two patients on Calcium Channel Blockers (CCBs):

| Patient | Medication | Raw dosage | Frequency | Total Daily Dose (TDD) | Maximum Daily Dose (MDD) | Dose ratio |
| :- | :- | -: | :- | -: | -: | -: |
| A | **Nifedipine** | 20 mg | BID | 40 mg | 120 mg | 0.33 |
| B | **Amlodipine** | 10 mg | QD | 10 mg | 10 mg | 1.00 |

**Clinical Insight:** Despite Patient A taking a higher milligram count (40mg vs 10mg), Patient B is at a higher therapeutic intensity (100% of max dose). The MBI correctly captures this higher therapeutic intensity for Patient B, which would be lost in a non-normalized dataset.

### Imputation to restore physiologic variance

#### The Problem: The "Silence of the Normal" (MNAR bias)

In a 10-day clinical sprint, data is recorded on physical paper charts at the bedside. This workflow creates a specific form of bias: Missing Not At Random (MNAR).

In high-volume brigade settings, "Missingness" is a clinical surrogate for "Normal." A cardiologist under extreme time pressure will prioritize documenting a pathological $1.2 cm^{2}$ Mitral Valve area but will likely leave the Aortic Valve field blank if it appears healthy. Standard imputation (mean/median) would erroneously assign "diseased" values to these healthy valves, leading to Model Alarmism and an overestimation of cohort severity.

#### The Strategy: "Natural Normal" Imputation

To preserve the statistical integrity of the cohort and prevent pathology bias, we utilize a **Natural Normal Imputation** strategy. Instead of treating nulls as errors, we treat them as "Healthy Proxies" and inject physiological noise centered around healthy clinical variables to approximate the natural variance of the population:

$$X_{imp} \sim \mathcal{N}(\mu_{healthy}, \sigma^{2}_{phys})$$

*Example:* For a missing Right Ventricular Systolic Pressure (RVSP), we assume a "normal" physiological state. Rather than imputing the cohort mean (which may be elevated due to severe mitral disease), we impute values centered around $25\text{ mmHg}$ with a small standard deviation ($\sigma \approx 4\text{ mmHg}$). This reflects a healthy pulmonary pressure range of $19\text{--}31\text{ mmHg}$.

#### Imputation sensitivity analysis
We plot **measured** (grey) vs. **imputed** (blue) to visually compare the distributions before and after the injection of Gaussian noise.
![Natural normal imputation](assets/output_kde_imputation.png)

| Parameter | Measured mean | Imputed mean | Delta |
| :- | -: | -: | :- |
| MS MG (mmHg)  | 11.4 | 3.8 | -7.6 | 
| AO V2 max | 4.3 | 1.6 | -2.6 | 
| RVSP | 51.0 | 33.6 | -17.5 | 
| LVIDd | 5.2 | 4.8 | -0.4 | 

The table reveals a Significant Shift in means for hemodynamic variables, such as RVSP dropping from 51.0 mmHg (measured) to 33.6 mmHg (imputed). This delta is the mathematical proof of a **High-Severity Documentation Threshold**. It confirms that clinicians only performed time-intensive quantitative measurements when a preliminary "Quick Scan" indicated significant pathology—leaving the healthy portion of the population "silent" but present.

[//]: # (MBI ROC for high RVSP)

[//]: # (Regression with outliers)

[//]: # (Tableau: Visualization)