# Loan_Default_Prediction_Project

Presented by Abhishek Kumar

📖 Overview
This project demonstrates a production-grade credit risk system built on the Databricks Lakehouse Platform. Unlike academic projects that focus solely on optimizing accuracy, this solution addresses "The Production Reality" of financial lending: the need for lineage, reproducibility, and strict governance .
The goal was to move beyond the "Academic Trap"—where accuracy exists without auditability—and build a system capable of defensive data engineering, auditable deployment, and real-time serving .

🏗 Architecture
The solution follows the Medallion Architecture, governed entirely by Unity Catalog , :
• Ingestion: Schema enforcement on source data.
• Bronze (Raw Truth): Raw data storage with metadata injection.
• Silver (Clean & Features): Handling nulls, normalization, and string parsing.
• Gold (Model Inputs): Aggregated features ready for training.
• MLflow & Serving: Model lifecycle management and REST API deployment.

🛠️ Key Components
1. Defensive Engineering (Bronze Layer)
We treat data ingestion as a defensive operation to prevent silent corruption :
• Explicit Schemas: We disable schema inference to ensure data types match expectations .
• Metadata Injection: We inject _source_file and ingested_at columns immediately for traceability .
• Raw Truth: Data is stored in Delta format to preserve the exact state of ingestion prior to transformation .
2. The Quality Audit & Transformation (Silver Layer)
An audit of the source data revealed over 1,322,708 null values in employment length and inconsistent string formats (e.g., "10+ years", "< 1 year", "n/a") .
To handle this, we implemented robust transformation logic , :
• Normalize: Converting empty strings to true NULLs.
• Extract: Using Regex (regexp_extract) to parse "10+ years" into integers.
• Safe Cast: Utilizing try_cast to prevent pipeline crashes if bad data persists.

```<language-identifier>
// Your code goes here
df_silver = df_silver.withColumn(
    "emp_length_num",
    F.expr("""
        CASE 
            WHEN emp_length IS NULL OR trim(emp_length) = '' THEN 0
            WHEN emp_length LIKE '%10+%' THEN 10
            WHEN emp_length RLIKE '^[1]+ year[s]?$' THEN 
                 try_cast(regexp_extract(emp_length, '([1]+)', 1) as int)
            ELSE try_cast(regexp_extract(emp_length, '([1]+)', 1) as int)
        END
    """)
)
```

3. Modeling & The Pipeline Approach
We faced a significant Target Imbalance, with only 13.02% of the dataset representing defaults . Consequently, we optimized for AUC (Area Under Curve) rather than accuracy to better capture the trade-off between sensitivity and specificity .
Critical Design Pattern: The Pipeline IS The Model To prevent training-serving skew, we do not fit encoders separately. We wrap feature engineering (StringIndexers, OneHotEncoders, VectorAssemblers) and the Classifier into a single Pipeline object .

```<language-identifier>
// Your code goes here
pipeline = Pipeline(stages=indexers + encoders + [assembler, classifier])
model = pipeline.fit(train_df)
predictions = model.transform(test_df)

```



This ensures inference data undergoes the exact same transformations as training data , .

5. Governance & Deployment
• Lineage: We utilize Unity Catalog to provide end-to-end traceability. We can trace a prediction from the endpoint back to the specific raw CSV file in the Bronze layer .
• MLflow Lifecycle:
    1. Experiment Tracking: Logging params and metrics (Baseline AUC: 0.73) , .
    2. Model Registry: Version control and stage promotion (Staging -> Production) , .
    3. Serving: The model is served via a REST API with auto-scaling .
       
Challenges & Learnings ("The War Story")
During deployment, we encountered a critical failure where the serving endpoint refused to start despite valid code.
• The Incident: py4j.protocol.Py4JError during serving initialization .
• The Diagnosis: A JVM version mismatch. The model was trained on a cluster running Java 8, but the serving environment used Java 11 .
• The Fix: We retrained on a Databricks LTS (Long Term Support) runtime to ensure environment parity between training and serving .
Key Takeaway: Runtime compatibility is a critical dependency .

📊 Business Impact
1. Risk Mitigation: Real-time scoring enables immediate credit decisions .
2. Auditability: Regulators can trace decisions back to specific lines of code and data sources .
3. Agility: MLflow enables safe A/B testing of challenger models without disrupting production .
   
🧰 Tech Stack
• Compute: Apache Spark
• Storage: Delta Lake
• Tracking: MLflow
• Governance: Unity Catalog
• Language: Python / PySpark
