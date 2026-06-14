# LLM-Assisted Big Data Integration

Implementation and evaluation of a hybrid Big Data Integration system developed for the **Advanced Topics in Computer Science** course at Roma Tre University.

The project compares:

1. A traditional deterministic integration pipeline.
2. An LLM-assisted pipeline using `Qwen/Qwen2.5-7B-Instruct`.

Both pipelines address:

- Schema alignment.
- Record linkage and entity resolution.
- Clustering.
- Data fusion and truth discovery.

The objective is not to demonstrate that Large Language Models are always superior. Instead, the project analyses where semantic reasoning is useful, where deterministic methods remain preferable, and how latency, memory usage, invalid outputs, hallucinations, and reproducibility affect the design of an integration system.

---

## Repository contents

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
```test

The `llm_bdi_project` directory contains the data, prompts, logs, metrics, error examples, and final integrated datasets generated during the final experiment.

---

## Main files

### `3ATICS.ipynb`

This is the main executable version of the project.

It is designed to run in Google Colab and performs the complete workflow:

* Installs the required dependencies.
* Downloads the public dataset.
* Selects the working subset.
* Creates the three heterogeneous source views.
* Builds the mediated schema.
* Generates the ground-truth files.
* Executes the traditional pipeline.
* Loads the language model.
* Executes the LLM-assisted pipeline.
* Evaluates both configurations.
* Stores prompts and LLM responses.
* Generates error examples.
* Produces the final integrated datasets.
* Saves the quantitative metrics.

The recommended method for reproducing the experiment is to execute this notebook from beginning to end in a clean Google Colab GPU runtime.

### `3atics.py`

This file is the Python export generated from the Colab notebook.

It is included mainly for code inspection. Direct execution is not recommended because it contains Colab-specific commands such as:

```python
!pip install ...
```

and imports from:

```python
google.colab
```

### `requirements.txt`

Contains the Python libraries required by the project.

### `ATICS_3_report.pdf`

Contains the complete technical report, including:

* Dataset and integration scenario.
* Traditional and LLM-assisted methodologies.
* Experimental protocol.
* Quantitative comparison.
* Error analysis.
* Discussion and limitations.
* Conclusion.
* Reproduction instructions.

---

## Dataset

The project uses the public Hugging Face dataset:

```text
willcb/wdc-products-multi
```

The dataset is downloaded automatically with the `datasets` library.

It contains product offers and a `cluster_id` identifying records that refer to the same real-world product.

The final experimental subset contains:

| Property                           | Value |
| ---------------------------------- | ----: |
| Product records                    | 1,800 |
| Gold entities                      |   300 |
| Source views                       |     3 |
| Records per source                 |   600 |
| Mediated attributes                |     5 |
| Positive cross-source pairs        | 3,600 |
| Conflicting entity-attribute cases | 1,220 |

Only clusters with at least two records were considered. A maximum of six records was kept for each entity.

---

## Source construction

The Hugging Face version of the dataset is distributed as a unified table.

To create a heterogeneous integration scenario, the records of each entity are assigned in round-robin order to three source views:

```text
source_alpha
source_beta
source_gamma
```

Each source contains the same general type of product information but uses different attribute names.

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

The mediated schema is:

```text
title
brand
description
price
priceCurrency
```

The known schema correspondence is:

| Mediated attribute | Source Alpha   | Source Beta        | Source Gamma     |
| ------------------ | -------------- | ------------------ | ---------------- |
| `title`            | `product_name` | `name`             | `item_title`     |
| `brand`            | `manufacturer` | `brand_name`       | `maker`          |
| `description`      | `details`      | `description_text` | `specifications` |
| `price`            | `cost`         | `amount`           | `listed_price`   |
| `priceCurrency`    | `currency`     | `currency_code`    | `money`          |

The three sources are synthetic views of one public benchmark rather than completely independent data providers.

This allows the complete workflow to be evaluated with known ground truth, although the schema and overlap patterns are more controlled than in a real integration scenario.

---

## Ground truth

Three evaluation references are used.

### Schema ground truth

The correct mapping between each source attribute and the mediated schema is generated from the known renaming rules.

File:

```text
llm_bdi_project/data/processed/schema_ground_truth.csv
```

### Record-linkage ground truth

The original `cluster_id` is preserved as `entity_id`.

Two records form a positive pair when:

* They belong to the same gold entity.
* They come from different source views.

The final subset contains 3,600 known positive cross-source pairs.

### Fusion silver truth

For every gold entity and mediated attribute, the most frequent normalized non-empty value is selected.

When several values have the same frequency, the longest value is used as a tie-breaking rule.

File:

```text
llm_bdi_project/data/processed/fusion_silver_ground_truth.csv
```

This reference is called silver truth because it is generated automatically rather than manually verified.

It also gives a structural advantage to the majority-voting baseline, which follows a similar selection rule.

---

## Pipeline A: traditional baseline

The baseline does not use an LLM for any integration decision.

### Schema alignment

Source attributes are mapped to the mediated schema using:

* Attribute-name normalization.
* Token-based comparison.
* RapidFuzz `token_set_ratio`.
* Simple semantic alias rules.
* A confidence threshold.
* Abstention when no mapping is sufficiently reliable.

Output:

```text
llm_bdi_project/results/schema_alignment_baseline.csv
```

### Canonicalization

Each source record is transformed into the mediated schema.

Text values are normalized by:

* Converting them to lowercase.
* Removing repeated spaces.
* Normalizing missing values.
* Extracting auxiliary title, brand, and model information.

### Blocking

Candidate pairs are generated using blocking keys based on:

* Brand.
* Model.
* Relevant title tokens.
* Title bigrams.
* Brand-title combinations.

Pairs from the same source are excluded.

Blocks containing more than 250 records are ignored because they are too general.

The final experiment contains:

```text
Possible cross-source pairs: 1,080,000
Candidate pairs after blocking: 33,855
Reduction ratio: 96.87%
```

Outputs:

```text
llm_bdi_project/results/candidate_pairs_baseline.csv
llm_bdi_project/results/blocking_stats_baseline.csv
```

### Pairwise record matching

Candidate pairs are evaluated using a weighted similarity score:

```text
0.45 × title similarity
+ 0.20 × brand similarity
+ 0.20 × model similarity
+ 0.15 × mediated-attribute similarity
```

The matching threshold is:

```text
0.40
```

A candidate pair is classified as a match when its score is greater than or equal to the threshold.

Output:

```text
llm_bdi_project/results/record_linkage_baseline.csv
```

### Clustering

Predicted matching pairs are converted into entity clusters using Union-Find.

Union-Find applies transitive closure. Therefore, if record A matches record B and record B matches record C, all three records are assigned to the same predicted cluster.

A limitation is that one incorrect pair can connect entities that should remain separate.

Output:

```text
llm_bdi_project/results/clusters_baseline.csv
```

### Data fusion

For each predicted cluster and mediated attribute:

1. The most frequent non-empty value is selected.
2. If several values have the same frequency, the longest value is selected.

Final output:

```text
llm_bdi_project/results/integrated_dataset_baseline.csv
```

---

## Pipeline B: LLM-assisted integration

The LLM-assisted pipeline uses:

```text
Qwen/Qwen2.5-7B-Instruct
```

The model is loaded in 4-bit quantized mode to reduce GPU memory consumption.

Main configuration:

```text
Model: Qwen/Qwen2.5-7B-Instruct
Quantization: 4-bit NF4
Compute type: float16
Double quantization: enabled
Temperature: 0.0
Sampling: disabled
Seed: 42
```

The LLM is used selectively in three stages.

### LLM-assisted schema alignment

The model receives:

* Mediated schema attributes.
* Source name.
* Source attributes.
* Example values.
* Attribute occurrence information.

The expected JSON response contains:

* Source attribute.
* Mediated attribute.
* Confidence.
* Short explanation.
* Possible abstention.

If the output is invalid or the predicted target is not part of the mediated schema, the deterministic schema mapping is used as fallback.

Output:

```text
llm_bdi_project/results/schema_alignment_llm.csv
```

### LLM-assisted record linkage

The deterministic similarity model is first applied to all candidate pairs.

The predefined borderline interval is:

```text
[0.35, 0.50)
```

Pairs inside this interval are ordered by decreasing deterministic score, and the 300 highest-scoring pairs are sent to the LLM.

In the final execution, all 300 selected pairs were above the deterministic threshold of 0.40. Therefore, the model acted mainly as a verifier of pairs initially classified as matches.

For each selected pair, the model receives the two product records and returns JSON with:

```json
{
  "match": true,
  "confidence": 0.8,
  "reason": "Short explanation"
}
```

When the JSON response is valid, the LLM decision replaces the deterministic decision.

When the response is invalid, the original similarity-based decision is preserved.

Output:

```text
llm_bdi_project/results/record_linkage_llm.csv
```

### LLM-assisted data fusion

The LLM is applied to at most 50 predicted clusters containing conflicting values.

Each prompt contains no more than three records.

The model returns one selected value for each mediated attribute:

```text
title
brand
description
price
priceCurrency
```

The fusion response does not include explicit confidence or evidence fields.

Instead, reliability is controlled through deterministic validation. The prompt instructs the model to:

* Copy values exactly from the provided records.
* Avoid shortening or combining values.
* Avoid translating values.
* Avoid inferring missing information.
* Avoid inventing unsupported values.

Every proposed value is compared with the values present in the corresponding cluster.

A value is accepted only when it is supported by one of the cluster records.

If the response contains invalid JSON or an unsupported value, the majority-voting result is preserved.

Outputs:

```text
llm_bdi_project/results/integrated_dataset_llm_initial.csv
llm_bdi_project/results/integrated_dataset_llm.csv
```

---

## Prompt versions

The final experiment uses version 1 of the following prompt templates:

```text
llm_bdi_project/prompts/schema_alignment_prompt.txt
llm_bdi_project/prompts/record_linkage_prompt.txt
llm_bdi_project/prompts/data_fusion_prompt.txt
```

These files contain the exact prompt templates used in the reported execution.

No prompt or model response was manually modified during the final evaluation.

---

## Structured outputs and logs

All LLM stages use JSON outputs.

Every model call is recorded in:

```text
llm_bdi_project/results/llm_logs.jsonl
```

Each log entry contains information such as:

* Integration stage.
* Complete prompt.
* Raw model response.
* Parsed response.
* Success or failure status.
* Relevant metadata.

The final execution produced:

| Stage            | LLM calls |
| ---------------- | --------: |
| Schema alignment |         3 |
| Record linkage   |       300 |
| Data fusion      |        50 |
| **Total**        |   **353** |

---

## Failure and fallback policy

### Schema alignment

When the response is invalid, incomplete, or contains an unknown mediated attribute:

```text
Use the deterministic schema-alignment result.
```

### Record linkage

When the response cannot be parsed or does not contain the expected fields:

```text
Use the original similarity-based decision.
```

### Data fusion

When the model:

* Returns invalid JSON.
* Omits the expected fused structure.
* Returns an empty value.
* Proposes a value not supported by the cluster records.

the system uses:

```text
Majority voting as fallback.
```

Unsupported generated values are never inserted directly into the final dataset.

No LLM output is manually corrected before evaluation.

---

## Installation

The recommended execution environment is Google Colab with GPU support.

### Google Colab

1. Open `3ATICS.ipynb` in Google Colab.
2. Select:

```text
Runtime → Change runtime type → GPU
```

3. Restart the runtime if the notebook was previously executed.
4. Run all cells in order from beginning to end.

The notebook installs the required libraries automatically.

### Manual dependency installation

```bash
pip install -r requirements.txt
```

Main dependencies include:

```text
pandas
numpy
tqdm
rapidfuzz
scikit-learn
networkx
datasets
transformers
accelerate
bitsandbytes
torch
regex
tabulate
```

A CUDA-compatible GPU is strongly recommended because the project loads a 7-billion-parameter language model.

---

## Reproducing the experiment

Run all cells in:

```text
3ATICS.ipynb
```

The notebook performs the following steps:

1. Installs dependencies.
2. Creates the project directories.
3. Downloads the WDC Products dataset.
4. Selects the working subset.
5. Constructs the three source views.
6. Creates the mediated schema.
7. Builds schema, linkage, and fusion references.
8. Executes the traditional schema alignment.
9. Canonicalizes the source records.
10. Generates blocking candidates.
11. Executes deterministic record linkage.
12. Creates baseline clusters.
13. Produces the baseline integrated dataset.
14. Loads Qwen2.5-7B-Instruct.
15. Executes LLM-assisted schema alignment.
16. Evaluates selected record pairs with the LLM.
17. Creates LLM-assisted clusters.
18. Executes selective LLM-assisted fusion.
19. Applies validation and fallback rules.
20. Computes the evaluation metrics.
21. Generates representative error examples.
22. Saves logs, metrics, clusters, and integrated datasets.

The complete experiment required approximately 30 minutes in the final Google Colab GPU execution.

The exact time may vary depending on the GPU and current Colab environment.

---

## Reproducibility

The main random seed is:

```text
42
```

The experiment initializes:

```python
random.seed(42)
numpy.random.seed(42)
```

LLM generation uses:

```text
do_sample = False
temperature = 0.0
```

This improves stability, but exact reproduction may still depend on:

* GPU architecture.
* CUDA version.
* PyTorch version.
* Transformers version.
* BitsAndBytes version.
* Dataset updates.
* Hardware-specific floating-point operations.

A clean runtime should be used before executing the final experiment. Re-running only selected LLM cells without clearing the runtime may duplicate in-memory logs or produce inconsistent output files.

---

## Evaluation metrics

### Schema alignment

The project computes:

* Accuracy.
* Precision.
* Recall.
* F1 score.
* True positives.
* False positives.
* False negatives.

### Record linkage

The project computes:

* Pairwise precision.
* Pairwise recall.
* Pairwise F1 score.
* Predicted matches.
* Gold positive pairs.
* True positives.
* False positives.
* False negatives.
* Candidate pairs after blocking.

### Blocking

The project reports:

* Total possible cross-source pairs.
* Candidate pairs after blocking.
* Reduction ratio.

### Data fusion

The project computes:

* Integrated-value accuracy.
* Number of evaluated attribute values.
* Number of correctly selected values.

The baseline and LLM-assisted pipelines produce different predicted clusters. Therefore, the number of evaluated fusion values differs between both configurations.

The fusion metric should be interpreted as an end-to-end integrated-value measure because it is affected by:

* Schema alignment.
* Record linkage.
* Clustering.
* Attribute availability.
* Fusion decisions.

### LLM reliability

The project also reports:

* Number of LLM calls.
* Invalid structured responses.
* Unsupported fusion values.
* Use of deterministic fallbacks.

All aggregated metrics are stored in:

```text
llm_bdi_project/results/metrics_summary.csv
```

---

## Main results

| Metric                    | Traditional baseline | LLM-assisted |
| ------------------------- | -------------------: | -----------: |
| Schema accuracy           |                0.400 |        1.000 |
| Schema F1                 |                0.571 |        1.000 |
| Linkage precision         |                0.305 |        0.318 |
| Linkage recall            |                0.655 |        0.578 |
| Linkage F1                |                0.416 |        0.410 |
| Integrated-value accuracy |                0.200 |        0.239 |

Additional results:

```text
Possible cross-source pairs: 1,080,000
Candidate pairs after blocking: 33,855
Reduction ratio: 96.87%
Baseline predicted clusters: 140
LLM-assisted predicted clusters: 197
Gold entities: 300
```

The final execution produced:

```text
3 schema-alignment calls
300 record-linkage calls
50 data-fusion calls
353 total LLM calls
```

Among the 50 fusion calls:

```text
32 responses were fully valid
2 responses contained invalid JSON
16 responses contained at least one unsupported value
22 unsupported attribute values were rejected
```

---

## Interpretation of the results

The clearest improvement appears in schema alignment.

The deterministic baseline depends mainly on lexical similarity and abstains when attribute names are semantically related but lexically different.

The LLM correctly recognizes mappings such as:

```text
manufacturer → brand
details → description
listed_price → price
```

In record linkage, the LLM behaves as a conservative verifier.

It reduces false positives and slightly increases precision, but it also rejects correct pairs. Consequently, recall decreases and the final F1 score is slightly lower than the baseline.

Fusion results must be interpreted carefully because both pipelines produce different predicted clusters.

An incorrect cluster can contain values from several real-world entities. Deterministic validation can reject invented values, but it cannot detect that a supported value came from a record that should not have belonged to the cluster.

Overall, the results support a hybrid design:

* Use LLMs for semantic interpretation.
* Use deterministic methods for blocking, exact identifiers, clustering, validation, and fallback decisions.

---

## Error analysis

The repository contains concrete error examples for both pipelines.

### Record-linkage errors

```text
llm_bdi_project/results/error_examples_linkage_baseline.csv
llm_bdi_project/results/error_examples_linkage_llm.csv
```

These files include:

* Error type.
* Record identifiers.
* Similarity score.
* Predicted label.
* Gold label.
* Source-record contents.

### Data-fusion errors

```text
llm_bdi_project/results/error_examples_fusion_baseline.csv
llm_bdi_project/results/error_examples_fusion_llm.csv
```

These files include:

* Predicted cluster.
* Gold entity.
* Attribute.
* Predicted value.
* Expected value.
* Correct or incorrect decision.

The technical report discusses three representative cases:

1. A false positive between two similar Tissot watches with different model numbers.
2. A false negative caused by multilingual title variation.
3. A fusion error inherited from an incorrect Western Digital product cluster.

---

## Limitations

### Synthetic source views

The three source views are derived from one unified public dataset rather than from three independent providers.

### Controlled schema heterogeneity

The main schema differences are introduced through attribute renaming.

### Controlled entity overlap

The selected entities follow a more regular overlap pattern than would normally occur in an uncontrolled real-world scenario.

### Small schema evaluation set

The mediated schema contains five attributes and three sources, resulting in 15 mapping decisions.

### Selective LLM invocation

The LLM is limited to:

```text
300 record-linkage pairs
50 conflicting clusters
```

Most candidate comparisons therefore remain deterministic.

### Asymmetric linkage selection

The 300 highest-scoring pairs inside the interval `[0.35, 0.50)` were selected.

In the final execution, all selected pairs were above the matching threshold, so the model mainly verified initially positive decisions.

### Transitive clustering errors

Union-Find can propagate a false-positive match and merge several unrelated entities.

### Silver fusion ground truth

The fusion reference is generated using majority voting, which gives an advantage to the deterministic fusion strategy.

### End-to-end fusion metric

The two pipelines produce different clusters and different numbers of evaluated attribute values.

The reported value accuracy therefore reflects the complete integration process rather than only the final fusion rule.

### Computational constraints

Qwen2.5-7B-Instruct was loaded using 4-bit quantization because of Google Colab memory limitations.

The number of calls and prompt sizes were restricted to keep the complete experiment executable.

### Single final execution

The reported values come from one clean final run rather than from several repeated experiments.

---

## Generated outputs

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

### LLM prompts and logs

```text
schema_alignment_prompt.txt
record_linkage_prompt.txt
data_fusion_prompt.txt
llm_logs.jsonl
```

---

## Requirements compliance

The project includes:

* Three heterogeneous source views.
* More than 1,000 records.
* Five mediated attributes.
* More than 200 gold entities.
* More than 300 known positive pairs.
* More than 100 conflicting values.
* A complete traditional pipeline.
* An LLM-assisted pipeline.
* LLM use in three integration stages.
* Structured JSON outputs.
* Version 1 prompt templates.
* Complete LLM logs.
* Documented fallback policies.
* Schema-alignment metrics.
* Record-linkage metrics.
* Blocking statistics.
* Data-fusion evaluation.
* Error examples.
* Final integrated datasets.
* An executable Colab notebook.
* A technical PDF report.

---

## Author

```text
Juan Miguel Pinos Seco
Advanced Topics in Computer Science
Roma Tre University
Academic year 2025–2026
```

---

## License

This repository was created for academic purposes.

The WDC Products dataset and the Qwen model remain subject to their respective licences and terms of use.
