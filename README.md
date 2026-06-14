# LLM-Assisted Big Data Integration

## 1. Project overview

This repository contains the implementation and evaluation of an **LLM-assisted Big Data Integration pipeline** developed for the *Advanced Topics in Computer Science* course.

The objective of the project is to compare two different approaches for integrating heterogeneous product data:

1. A **traditional baseline pipeline**, based on deterministic rules and string similarity.
2. An **LLM-assisted pipeline**, which uses a Large Language Model in selected stages of the integration process.

Both pipelines address the three main phases of Big Data Integration:

* Schema alignment.
* Record linkage or entity resolution.
* Data fusion or truth discovery.

The goal of the project is not to prove that LLMs are always superior, but to analyse where they provide useful semantic reasoning, where classical methods remain preferable, and which limitations arise from cost, latency, hallucinations and reproducibility.

---

## 2. Repository structure

```text
Big_Data_Integration/
│
├── 3ATICS.ipynb
├── 3atics.py
├── requirements.txt
├── README.md
├── ATICS_3_report.pdf
├── .gitignore
│
└── llm_bdi_project/
    │
    ├── data/
    │   ├── raw/
    │   └── processed/
    │       ├── attribute_stats.csv
    │       ├── canonical_baseline.csv
    │       ├── canonical_gold.csv
    │       ├── canonical_llm.csv
    │       ├── fusion_silver_ground_truth.csv
    │       ├── mediated_schema.csv
    │       ├── schema_ground_truth.csv
    │       └── work_records.pkl
    │
    ├── prompts/
    │   ├── schema_alignment_prompt.txt
    │   ├── record_linkage_prompt.txt
    │   └── data_fusion_prompt.txt
    │
    ├── results/
    │   ├── blocking_stats_baseline.csv
    │   ├── blocking_stats_llm.csv
    │   ├── candidate_pairs_baseline.csv
    │   ├── candidate_pairs_llm.csv
    │   ├── clusters_baseline.csv
    │   ├── clusters_llm.csv
    │   ├── error_examples_fusion_baseline.csv
    │   ├── error_examples_fusion_llm.csv
    │   ├── error_examples_linkage_baseline.csv
    │   ├── error_examples_linkage_llm.csv
    │   ├── fusion_eval_baseline.csv
    │   ├── fusion_eval_llm.csv
    │   ├── integrated_dataset_baseline.csv
    │   ├── integrated_dataset_llm.csv
    │   ├── integrated_dataset_llm_initial.csv
    │   ├── llm_logs.jsonl
    │   ├── metrics_summary.csv
    │   ├── record_linkage_baseline.csv
    │   ├── record_linkage_llm.csv
    │   ├── schema_alignment_baseline.csv
    │   ├── schema_alignment_llm.csv
    │   └── technical_report_base.md
    │
    └── submission_manifest.txt
```

The `llm_bdi_project` directory is generated automatically by the notebook when the complete experiment is executed.

---

## 3. Main files

### `3ATICS.ipynb`

This is the main executable version of the project.

The notebook is designed to run in **Google Colab** and performs the complete workflow:

* Installs the required dependencies.
* Downloads the public dataset.
* Creates the three heterogeneous source views.
* Builds the mediated schema and ground-truth mappings.
* Executes the traditional baseline.
* Executes the LLM-assisted pipeline.
* Computes all required evaluation metrics.
* Stores the prompts and LLM responses.
* Generates the final integrated datasets.
* Creates the results directory.
* Compresses the generated project into a ZIP file.

### `3atics.py`

This file is the Python export automatically generated from the Colab notebook.

The recommended execution method is the `.ipynb` notebook because the exported Python file contains Colab-specific commands such as:

```python
!pip install ...
```

and utilities from:

```python
google.colab
```

Therefore, the `.py` file is included mainly for code inspection and as an alternative representation of the notebook.

### `requirements.txt`

This file contains the Python dependencies required by the project.

### `ATICS_3_report.pdf`

The technical report includes:

* Dataset description.
* Methodology.
* Baseline pipeline.
* LLM-assisted pipeline.
* Experimental protocol.
* Evaluation metrics.
* Quantitative results.
* Error analysis.
* Limitations and discussion.
* GitHub repository link.

---

## 4. Dataset

The project uses the following public dataset:

```text
willcb/wdc-products-multi
```

The dataset is downloaded automatically from Hugging Face through the `datasets` library.

It contains product offers and a gold cluster identifier that indicates which records refer to the same real-world product.

The working subset contains approximately:

* 1,800 product records.
* 300 gold entities.
* Three source views.
* More than 300 positive matching pairs.
* More than 100 conflicting entity-attribute values.
* Five mediated attributes.

The mediated schema is:

```text
title
brand
description
price
priceCurrency
```

---

## 5. Construction of the sources

The Hugging Face dataset is originally provided as a unified table.

Since the assignment requires at least three heterogeneous sources, the notebook constructs three source views:

```text
source_alpha
source_beta
source_gamma
```

Records are distributed among these sources using a round-robin strategy within each gold entity.

Each source uses different attribute names.

### Source Alpha

```text
product_name
manufacturer
details
cost
currency
```

### Source Beta

```text
name
brand_name
description_text
amount
currency_code
```

### Source Gamma

```text
item_title
maker
specifications
listed_price
money
```

These source attributes are mapped to the following mediated schema:

| Mediated attribute | Source Alpha   | Source Beta        | Source Gamma     |
| ------------------ | -------------- | ------------------ | ---------------- |
| `title`            | `product_name` | `name`             | `item_title`     |
| `brand`            | `manufacturer` | `brand_name`       | `maker`          |
| `description`      | `details`      | `description_text` | `specifications` |
| `price`            | `cost`         | `amount`           | `listed_price`   |
| `priceCurrency`    | `currency`     | `currency_code`    | `money`          |

The three sources are therefore synthetic source views derived from the same public benchmark.

This design makes it possible to evaluate the complete integration workflow with known ground truth, although it is less complex than integrating three completely independent real-world data providers.

---

## 6. Pipeline A: traditional baseline

The traditional pipeline does not use an LLM for any integration decision.

### 6.1 Schema alignment

Source attributes are mapped to the mediated schema using:

* Attribute-name normalisation.
* Token-based string similarity.
* RapidFuzz token-set similarity.
* Simple deterministic semantic rules.
* A confidence threshold.
* Abstention when the score is below the threshold.

The output is stored in:

```text
llm_bdi_project/results/schema_alignment_baseline.csv
```

### 6.2 Blocking

Candidate record pairs are generated using blocking keys based on:

* Brand.
* Model.
* Title tokens.
* Title bigrams.
* Brand-title combinations.

Pairs belonging to the same source are excluded.

Very large blocks are ignored to prevent an excessive number of candidate comparisons.

The blocking stage reduces the number of record pairs that must be evaluated.

The generated files are:

```text
candidate_pairs_baseline.csv
blocking_stats_baseline.csv
```

### 6.3 Pairwise record matching

Candidate pairs are compared using a weighted similarity score based on:

* Title similarity.
* Brand similarity.
* Model similarity.
* Similarity across mediated attributes.

The final score is computed as:

```text
0.45 × title similarity
+ 0.20 × brand similarity
+ 0.20 × model similarity
+ 0.15 × mediated attribute similarity
```

Pairs with a score greater than or equal to the matching threshold are classified as matches.

The default matching threshold is:

```text
0.40
```

### 6.4 Clustering

Predicted matching pairs are grouped into entities using a Union-Find structure.

Union-Find applies transitive closure. Therefore, if record A matches record B and record B matches record C, the three records are assigned to the same cluster.

The generated clustering file is:

```text
clusters_baseline.csv
```

### 6.5 Data fusion

For each predicted cluster, one canonical value is selected for every mediated attribute.

The baseline uses:

* Majority voting.
* Longest-value tie-breaking when several values have the same frequency.

The final integrated baseline dataset is:

```text
integrated_dataset_baseline.csv
```

---

## 7. Pipeline B: LLM-assisted integration

The LLM-assisted pipeline uses:

```text
Qwen/Qwen2.5-7B-Instruct
```

The model is loaded in 4-bit quantised mode using BitsAndBytes.

Main model configuration:

```text
Model: Qwen/Qwen2.5-7B-Instruct
Quantisation: 4-bit NF4
Compute type: float16
Double quantisation: enabled
Temperature: 0.0
Sampling: disabled
Seed: 42
```

The LLM is used in three stages.

### 7.1 LLM-assisted schema alignment

For each source, the model receives:

* The mediated schema.
* The source name.
* The source attributes.
* Example values.
* Attribute occurrence counts.

The model returns a JSON structure containing:

* Source attribute.
* Predicted mediated attribute.
* Confidence.
* Short explanation.
* Possible abstention.

If the response is invalid, the deterministic baseline mapping is preserved as fallback.

The generated file is:

```text
schema_alignment_llm.csv
```

### 7.2 LLM-assisted record linkage

The traditional similarity model is first applied to all candidate pairs.

The LLM is then used selectively on borderline pairs whose score lies between:

```text
0.35 and 0.50
```

The maximum number of record-linkage LLM calls is:

```text
300
```

For each selected pair, the model receives the two product records and returns:

```json
{
  "match": true,
  "confidence": 0.0,
  "reason": "short explanation"
}
```

The LLM decision replaces the deterministic decision only when a valid response is obtained.

Invalid responses use the original similarity-based decision as fallback.

The generated file is:

```text
record_linkage_llm.csv
```

### 7.3 LLM-assisted data fusion

The LLM is applied selectively to conflicting clusters.

The maximum number of fusion calls is:

```text
50
```

Each prompt contains at most three records from the selected cluster.

The model returns one canonical value for each mediated attribute.

The output must be valid JSON and the model is explicitly instructed to:

* Select values supported by the input records.
* Copy values exactly from the provided records.
* Avoid shortening or combining values.
* Avoid translating values.
* Avoid inferring unsupported information.
* Avoid hallucinating new values.

Each generated value is validated against the original values contained in the cluster.

A value is accepted only if it appears in one of the source records.

If the generated JSON is invalid or the selected value is unsupported, the system uses the traditional majority-voting value as fallback.

The final integrated dataset is:

```text
integrated_dataset_llm.csv
```

The initial LLM pipeline output before generative fusion is also stored in:

```text
integrated_dataset_llm_initial.csv
```

---

## 8. Prompt management

All prompts are versioned and stored in:

```text
llm_bdi_project/prompts/
```

The repository includes:

```text
schema_alignment_prompt.txt
record_linkage_prompt.txt
data_fusion_prompt.txt
```

Every LLM call is also logged in:

```text
llm_bdi_project/results/llm_logs.jsonl
```

Each log contains:

* Integration stage.
* Complete prompt.
* Raw model response.
* Parsed JSON.
* Success or failure status.
* Additional metadata.

No model output is manually corrected during evaluation.

---

## 9. Failure and fallback policy

The project applies a documented fallback policy.

### Schema alignment

If the LLM output is invalid or incomplete:

```text
Use the traditional schema-alignment prediction.
```

### Record linkage

If the LLM output cannot be parsed:

```text
Use the original similarity-based decision.
```

### Data fusion

If the model:

* Returns invalid JSON.
* Omits the expected `fused` object.
* Returns an unsupported value.
* Produces an empty or unusable response.

the system uses:

```text
Majority voting as fallback.
```

Unsupported fusion values are never inserted directly into the final integrated dataset.

---

## 10. Installation

The recommended execution environment is Google Colab with GPU support.

### Option 1: Google Colab

1. Open `3ATICS.ipynb` in Google Colab.
2. Select:

```text
Runtime → Change runtime type → GPU
```

3. Execute all notebook cells in order.

The notebook installs the required libraries automatically.

### Option 2: Install dependencies manually

```bash
pip install -r requirements.txt
```

The main dependencies are:

```text
pandas
numpy
tqdm
rapidfuzz
scikit-learn
networkx
datasets
transformers==4.44.2
accelerate==0.33.0
bitsandbytes==0.45.5
torch
regex
tabulate
```

A CUDA-compatible GPU is recommended because the project loads a 7-billion-parameter language model in 4-bit mode.

---

## 11. Execution

The complete experiment is executed by running all cells in:

```text
3ATICS.ipynb
```

The notebook automatically:

1. Creates `/content/llm_bdi_project`.
2. Downloads the WDC Products dataset.
3. Selects the working subset.
4. Constructs the three source views.
5. Creates the mediated schema.
6. Builds the ground truth.
7. Executes the traditional pipeline.
8. Loads the language model.
9. Executes the LLM-assisted pipeline.
10. Evaluates both pipelines.
11. Stores the prompts.
12. Stores the LLM logs.
13. Generates error examples.
14. Generates the final datasets.
15. Creates `metrics_summary.csv`.
16. Creates a ZIP file containing the complete generated project.

The final ZIP can be downloaded directly from Colab.

---

## 12. Reproducibility

The random seed used by the project is:

```text
42
```

The following random generators are initialised:

```python
random.seed(42)
numpy.random.seed(42)
```

The LLM generation configuration is deterministic:

```text
do_sample = False
temperature = 0.0
```

However, minor differences may still appear depending on:

* GPU architecture.
* CUDA version.
* PyTorch version.
* Transformers version.
* BitsAndBytes implementation.
* Dataset updates.
* Hardware-specific floating-point behaviour.

The main package versions related to the model are fixed in `requirements.txt`.

---

## 13. Evaluation metrics

### Schema alignment

The following metrics are computed:

* Accuracy.
* Precision.
* Recall.
* F1 score.
* True positives.
* False positives.
* False negatives.

### Record linkage

The following metrics are computed:

* Pairwise precision.
* Pairwise recall.
* Pairwise F1.
* Number of candidate pairs.
* Number of predicted matches.
* Number of gold positive pairs.
* True positives.
* False positives.
* False negatives.

### Blocking

The following metrics are reported:

* Candidate pairs after blocking.
* Total possible cross-source pairs.
* Reduction ratio.

### Data fusion

The following metrics are computed:

* Selected-value accuracy.
* Number of evaluated attribute values.
* Number of correct values.

### LLM usage

The project also reports:

* Total number of LLM calls.
* Schema-alignment calls.
* Record-linkage calls.
* Data-fusion calls.

All aggregated metrics are stored in:

```text
llm_bdi_project/results/metrics_summary.csv
```

---

## 14. Results

The complete quantitative results are available in:

```text
llm_bdi_project/results/metrics_summary.csv
```

The main qualitative observations are:

* The LLM is especially useful for schema alignment because it can recognise semantic correspondences between differently named attributes.
* The traditional string-similarity baseline cannot always recognise mappings such as `manufacturer` to `brand` or `cost` to `price`.
* In record linkage, the LLM can reject some false positive matches by reasoning about product models and technical details.
* The LLM may also reject true matches, reducing recall.
* Therefore, better precision does not necessarily imply a better overall F1 score.
* Blocking provides a large reduction in the number of record pairs that must be compared.
* Union-Find clustering can amplify pairwise false positives through transitive closure.
* Data fusion based on an LLM requires strict validation because generated values may be transformed or unsupported.
* Deterministic fallback rules remain useful for reliability and reproducibility.

---

## 15. Error analysis

The repository includes concrete error examples for both pipelines.

### Record-linkage errors

```text
error_examples_linkage_baseline.csv
error_examples_linkage_llm.csv
```

These files contain:

* Error type.
* Record identifiers.
* Similarity score.
* Predicted label.
* Gold label.
* Complete source records.

### Data-fusion errors

```text
error_examples_fusion_baseline.csv
error_examples_fusion_llm.csv
```

These files contain:

* Predicted cluster.
* Gold entity.
* Attribute.
* Predicted value.
* Expected value.
* Correct or incorrect classification.

The technical report discusses at least three representative errors in detail.

---

## 16. Limitations

The project has several limitations.

### Synthetic source construction

The three sources are created from a single public dataset.

Although the attribute names differ, they do not represent three completely independent data providers.

### Controlled entity overlap

The source construction process produces a more controlled overlap pattern than would normally occur in real-world integration systems.

### Small schema-alignment evaluation set

The mediated schema contains five attributes and three sources, resulting in a relatively small number of mapping decisions.

### Selective LLM invocation

The LLM is used only for:

* A maximum of 300 record pairs.
* A maximum of 50 conflicting clusters.

This limits computational cost but means that most integration decisions remain deterministic.

### Transitive clustering errors

Union-Find can create large erroneous clusters when false positive record-linkage decisions connect unrelated entities.

### Silver fusion ground truth

The fusion reference values are built using majority voting over the gold clusters.

This gives a structural advantage to the traditional fusion method, which also uses majority voting.

### Fusion metric interpretation

The baseline and LLM-assisted pipelines may generate different predicted clusters.

Therefore, the reported fusion accuracy also reflects errors from:

* Schema alignment.
* Record linkage.
* Clustering.
* Attribute availability.

It should be interpreted as an integrated or end-to-end value-quality metric rather than a completely isolated comparison of fusion algorithms.

---

## 17. Generated outputs

### Processed data

```text
attribute_stats.csv
canonical_baseline.csv
canonical_gold.csv
canonical_llm.csv
fusion_silver_ground_truth.csv
mediated_schema.csv
schema_ground_truth.csv
work_records.pkl
```

### Schema alignment

```text
schema_alignment_baseline.csv
schema_alignment_llm.csv
```

### Blocking

```text
blocking_stats_baseline.csv
blocking_stats_llm.csv
candidate_pairs_baseline.csv
candidate_pairs_llm.csv
```

### Record linkage

```text
record_linkage_baseline.csv
record_linkage_llm.csv
```

### Clustering

```text
clusters_baseline.csv
clusters_llm.csv
```

### Integrated datasets

```text
integrated_dataset_baseline.csv
integrated_dataset_llm_initial.csv
integrated_dataset_llm.csv
```

### Evaluation

```text
metrics_summary.csv
fusion_eval_baseline.csv
fusion_eval_llm.csv
```

### Error analysis

```text
error_examples_linkage_baseline.csv
error_examples_linkage_llm.csv
error_examples_fusion_baseline.csv
error_examples_fusion_llm.csv
```

### LLM information

```text
llm_logs.jsonl
schema_alignment_prompt.txt
record_linkage_prompt.txt
data_fusion_prompt.txt
```

---

## 18. Notes about the ZIP and consolidated TXT

The notebook contains optional cells for generating:

```text
llm_bdi_project_submission.zip
llm_bdi_project_submission_all_in_one.txt
```

The ZIP contains the complete generated project.

The consolidated TXT contains the readable content of the text-based files, but binary files such as:

```text
work_records.pkl
```

are not embedded in full.

These two files are used only for downloading and reviewing the generated results and are not required inside the GitHub repository when the extracted `llm_bdi_project` directory is already included.

---

## 19. Requirements compliance

The project satisfies the main assignment requirements:

* Three heterogeneous source views.
* More than 1,000 records.
* At least five mediated attributes.
* More than 200 integrated entities.
* More than 300 known positive matching pairs.
* More than 100 conflicting attribute values.
* Complete traditional baseline.
* LLM-assisted pipeline.
* LLM usage in at least two integration stages.
* Structured JSON outputs.
* Versioned prompts.
* Logged LLM responses.
* Documented fallback policy.
* Schema-alignment metrics.
* Record-linkage metrics.
* Blocking statistics.
* Data-fusion evaluation.
* Final integrated datasets.
* Error examples.
* Reproducible notebook.
* Technical report.

---

## 20. Authors

```text
Name: Juan Miguel Pinos
Course: Advanced Topics in Computer Science
Academic year: 2025–2026
```

---

## 21. License

This repository has been created for academic purposes.

The original WDC Products dataset and the Qwen model remain subject to their respective licences and terms of use.
