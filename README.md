# GBM-TxEval

![logo](https://github.com/user-attachments/assets/3f1724f1-835f-42eb-aa3a-fafe9a8e8ba2)

GBM-TxEval is a computational tool designed to evaluate transcriptional treatment responses in glioblastoma (GBM). It processes longitudinal gene expression data to calculate therapy-induced log2 fold changes, performs gene set enrichment analysis, and projects the results into principal component space. This enables stratification of samples into "Up" and "Down" responder subtypes, supporting the investigation of resistance mechanisms and selection of optimal models for preclinical drug evaluation.

## 📦 File Structure

- `R/utils_io.R`: File loading utilities.
- `R/utils_processing.R`: Core computation functions (normalization, filtering, log2FC, FGSEA).
- `server/`: Modular server logic for each step.
- `ui/`: Modular UI components for each step.
- `global.R`: Global configuration: packages, gene length tables, gene sets, PCA loadings, etc.
- `app.R`: Main entry point combining all modules into a seamless workflow.

## ⚙️ Workflow Overview

The analysis pipeline is divided into six modular steps:

### Step 1: Upload Datasets

The first step involves uploading two input files in CSV format: **sample metadata** and a **gene expression matrix**. These inputs are essential for downstream processing and must conform to specific format requirements.

![image](https://github.com/user-attachments/assets/4b204285-4b04-4720-a757-3ea18efaa6d4)

#### 📋 Sample Metadata

The metadata file must contain one row per sample and include the following mandatory columns:

- `Sample_ID`: Unique identifier for each sample.
- `Donor_ID`: Identifier linking paired samples (e.g., untreated/treated) from the same donor.
- `Model`: Label indicating the glioma model used.
- `Condition`: Treatment status of the sample.

The `Condition` column supports flexible input formats. It is case-insensitive and can accept the following values:

- `"Untreated"`, `"Primary"`, or `"0"` → interpreted as **Untreated**
- `"Treated"`, `"Recurrent"`, or `"1"` → interpreted as **Treated**

Optional metadata columns can also be included for annotation and filtering purposes in later steps.

#### 📊 Gene Expression Matrix

The expression matrix must be in **CSV format** and meet the following constraints:

- It must include exactly **one non-numeric column**, which is strictly required to be the **first column**. This column represents the gene identifier (e.g., Ensembl ID or HGNC symbol).
- All remaining columns must be numeric and correspond to sample names.
- Sample names (column headers) must match the values in the `Sample_ID` column of the metadata file. Partial overlap is permitted; unmatched samples will be excluded from analysis.

#### 🔎 Automatic Expression Type Detection

Upon upload, the app attempts to infer the type of expression data (Counts, FPKM, or TPM) using the following heuristic, applied to a random subset of two columns:

- If all values are integers, or if the proportion of non-integer values is between 5% and 80%, the **maximum value is below 20,000**, and all values are non-negative, the matrix is classified as **Counts** (this includes both raw and expected counts).
- If over 80% of values are non-integers, the **maximum value exceeds 10,000**, and all values are non-negative, it is classified as **FPKM**.
- If the sum of values in both sampled columns is approximately **1e6** (within a ±1e5 tolerance), it is classified as **TPM**.
- If these conditions are not met, or if fewer than 2 samples are available, the expression type is labeled **Unknown**.

The detected expression type can be overridden manually by the user.

------

### Step 2: Confirm Sample Matching

This step ensures proper alignment between the sample metadata and the expression matrix. The right panel provides a preview of the uploaded metadata, allowing users to **map column names** to their respective roles:

- `Sample_ID`
- `Donor_ID`
- `Model`
- `Condition` (e.g., untreated vs. treated)

![image](https://github.com/user-attachments/assets/61a77916-7639-4186-bb97-0ded4cab102f)

Users can also specify any number of **optional metadata columns** (referred to as "extra information"). These columns may include additional contextual details such as sample origin, treatment regimen, or sequencing batch. Selected extra information columns will:

- Be preserved in the filtered metadata file available for download in Step 4.
- Appear in the final visualization as part of the interactive hover text in Step 6.

There is no upper limit to the number of extra columns retained. Including such information is recommended when additional clinical or experimental annotations may aid interpretation.

------

### Step 3: Define Filtering Criteria

This step allows users to define one or more metadata-based **filtering conditions**. Any column in the uploaded metadata can be selected as a filtering variable—common examples include tumor purity, model subtype, treatment timepoint, or sequencing batch.

- **Zero or more filtering variables** may be selected.
- Filtering columns may be categorical or numeric; however, filtering operates via explicit selection of value levels.

![image](https://github.com/user-attachments/assets/3ca52516-810f-4ac5-bcba-6b9f45f2ddfe)

This design supports flexible sample selection for downstream analysis while allowing fine-grained control over cohort composition.

------

### Step 4: Apply Filters and Download Data

In this step, users specify which values within each selected filtering column should be retained. Filtering is performed using **checkbox selection**, where each level in a column can be toggled independently.

![image](https://github.com/user-attachments/assets/31f5bf19-c007-4133-aa43-9bf3b7fb7491)

> 💡 **Tip**: For numeric columns (e.g., purity scores), we recommend **creating binary classification columns** (e.g., `"HighPurity"` = TRUE/FALSE) in advance. This enables efficient selection via checkboxes and avoids the need for custom sliders.

Once filters are applied:

- The updated sample metadata is shown in the right panel for confirmation.
- Both the filtered metadata (`colData.csv`) and matched expression matrix (`countData.csv`) can be downloaded for external use or archival.

------

### Step 5: Process Data

In this step, GBM-TxEval performs core analytical operations including **expression normalization**, **low-expression filtering**, **log2 fold change calculation**, **PC1 score projection**, and **gene set enrichment analysis (GSEA)**. The overall analytical framework follows the strategy described in [Tanner *et al.*, *Genome Biology* 2024](https://genomebiology.biomedcentral.com/articles/10.1186/s13059-024-03172-3).

![image](https://github.com/user-attachments/assets/1a3a9ce8-66a9-4b42-a5b5-46b5155c0da1)

#### 🔬 Gene Filtering Options

Two gene filtering strategies are supported:

- **Default Filtering** (Recommended):
  - Expression values below a defined threshold are **set to zero**.
  - Users can choose from **preset thresholds** or specify a **custom value**.
  - If the input data type is **Counts**, the filtering is applied **before normalization**.
- **Advanced Filtering**:
  - Retains genes expressed **above the global first quartile (Q1)** in at least a user-defined **minimum proportion** of samples (e.g., 100%, 75%, etc.).
  - Designed for experienced users needing adaptive filtering based on data distribution.

![image](https://github.com/user-attachments/assets/a4bd9146-0daf-4d2a-9857-66faff09103b)

#### 📐 Expression Normalization

The normalization method depends on the input expression type:

- If the uploaded data is **TPM** or **FPKM**, the normalization method is fixed (i.e., no transformation is applied).
- If the input is **Counts**, users can choose to normalize the data using either:
  - **FPKM**: Fragments Per Kilobase of transcript per Million mapped reads
  - **TPM**: Transcripts Per Million

![image](https://github.com/user-attachments/assets/e44285cc-1cb2-4ca4-9fb5-1273863b1d4b)

Normalization is performed using uploaded gene length information, and both TPM and FPKM are computed with consistent formulae.

#### 📉 Log2 Fold Change Calculation

Log2 fold changes are computed **on normalized expression values** (either TPM or FPKM), not on raw counts. For count-based input, the data are first normalized according to the selected method, and then log2FC is calculated for each donor based on paired untreated vs. treated samples:

```
log2FC_gene = log2((expr_treated + 0.01) / (expr_untreated + 0.01))
```

A pseudocount of 0.01 is added to avoid division by zero.

#### 🧭 PC1 Score Calculation

After computing donor-specific log2FC vectors, each vector is projected into a predefined PCA space to calculate a **PC1 score**, which summarizes the direction and magnitude of the transcriptional response.

Specifically:

- A reference **PC1 rotation vector** (PCA loadings) is **precomputed from a cohort-wide log2FC matrix** derived from paired primary and recurrent glioblastoma bulk RNA-seq data.

- For each donor, their log2FC vector is **centered** and projected onto the PC1 axis using the formula:

  ```
  PC1_score = Σ (log2FC_gene × PC1_loading_gene)
  ```

- Only genes present in both the donor’s log2FC vector and the PCA rotation vector are used in the computation.

This PC1 score captures the principal axis of transcriptional variation observed across treatment conditions and serves as a **continuous quantitative representation** of response magnitude.

#### 🧬 Gene Set Enrichment Analysis

To evaluate transcriptional programs affected by treatment, GBM-TxEval performs **Gene Set Enrichment Analysis (GSEA)** using the FGSEA algorithm. For each donor, normalized log2FC values are ranked and used as input for enrichment testing against a predefined gene set (e.g., **JARID2 target genes**).

The gene ranking is performed as follows:

```
ranks = sort(log2FC, decreasing = TRUE)
```

Enrichment scores (ES) are calculated based on this ranking, with the top pathway's ES retained per donor.

To ensure compatibility between gene sets and input expression data, the application **automatically detects the type of gene identifier** in the expression matrix:

- If gene IDs match **Ensembl identifiers**, the Ensembl-based GMT file is used.
- If gene IDs match **HGNC symbols**, the corresponding symbol-based GMT file is used.

> ⚠️ **Note**: This automatic gene ID detection is highly recommended. Manual override is available but intended only for edge cases such as ambiguous gene formats or mismatched annotations.

#### ⚙️ Processing and Feedback

Once the “Start Processing” button is clicked:

- A progress bar appears, tracking analysis progress across donors.
- The current **Donor ID** and processing index are shown in real-time (e.g., “Donor ID: D123 [3/14]”).

![image](https://github.com/user-attachments/assets/4177dfee-e345-4af8-801a-b2c70600a15c)

#### 📤 Output

Upon completion, a preview of the results is displayed and a CSV download option is provided.

Each row in the output corresponds to a donor, containing the following columns:

- `Donor_ID`: Unique donor identifier
- `Model`: Glioma model associated with the sample pair
- `PC1_Score`: Projection of log2FC vector into the PC1 space
- `ES`: Enrichment Score from FGSEA (for the JARID2 pathway or selected gene set)
- `Responder`: Binary classification of transcriptional response (`Up` if ES ≥ 0; `Down` otherwise)

![image](https://github.com/user-attachments/assets/0bcd2437-e6c5-4eb5-a425-4fb2b92ee551)

The resulting dataset serves as input for the final step: interactive visualization and interpretation.

------

### Step 6: Visualization

This final step generates an interactive scatter plot that enables intuitive exploration of transcriptional responses across donors.

![image](https://github.com/user-attachments/assets/ccbeebe9-a864-4c88-a34b-52e752598939)

#### 📈 Plot Details

Each point in the plot represents a donor and is plotted as:

- **X-axis**: `PC1_Score` — summarizing the global transcriptional shift based on log2 fold changes
- **Y-axis**: `ES` — Enrichment Score from FGSEA for the specified gene set
- **Color and symbol**: Encodes `Model` type

The following result columns are visualized:

- `Donor_ID`
- `Model`
- `PC1_Score`
- `ES`
- `Responder`

#### 🖱️ Interactive Features

- **Hover Tooltips**: Hovering over a point reveals detailed information, including all retained metadata and extra user-specified columns from earlier steps.
- **Customization Options**:
  - `Point Size`: Adjustable via slider
  - `Point Opacity`: Adjustable via slider

These controls allow users to tune the visual clarity based on dataset size and density.

#### 💾 Export Options

The resulting interactive plot can be downloaded for presentation or publication. Users are encouraged to annotate or further customize the exported figure using external tools as needed.

## 🧠 Author & Acknowledgements

**Developer**: Bo Wang
**Affiliation**: University of Leeds
