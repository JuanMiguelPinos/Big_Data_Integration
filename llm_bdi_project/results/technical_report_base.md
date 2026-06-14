
# LLM-Assisted Big Data Integration - Experimental Summary

## Dataset

Dataset: WDC Products Multi from Hugging Face.

The experiment uses product offers from the WDC Products benchmark. The Hugging Face version provides a unified table with product attributes and a cluster identifier. Since the assignment requires three heterogeneous sources, I construct three source views by assigning offers to source_alpha, source_beta, and source_gamma, and by renaming attributes differently in each source.
The sources are synthetically constructed from a single public dataset by distributing bids using a round-robin method. This simplifies the integration pipeline because the underlying values ​​are consistent, and it can underestimate the real difficulties of integrating with truly heterogeneous sources.

The working subset contains:

- Records: 1800
- Sources: 3
- Gold entities: 300
- Gold positive matching pairs: 3600
- Mediated schema: ['title', 'brand', 'description', 'price', 'priceCurrency']
- Fusion conflicts detected in silver truth: 1220

## Pipelines

### Pipeline A - Traditional Baseline

- Schema alignment based on string similarity and simple attribute-name rules.
- Blocking based on brand, model, and title tokens.
- Pairwise matching using weighted similarity over title, brand, model, and mediated attributes.
- Clustering using Union-Find.
- Data fusion using majority voting and longest-value tie-breaking.

### Pipeline B - LLM-Assisted Integration
The LLM-assisted pipeline uses Qwen/Qwen2.5-7B-Instruct in three stages:

1. Schema alignment for source-to-mediated attribute mappings.
2. Record linkage for borderline candidate pairs.
3. Data fusion on a small selected subset of conflicting clusters.

For computational reasons on Colab, LLM-assisted fusion is limited to MAX_LLM_FUSION_CALLS = 50 conflicting clusters, using short prompts with at most 3 records per cluster and max_new_tokens = 384.
The model is prompted to return JSON only. Invalid JSON, missing fields, empty responses, and unsupported values are logged and treated as failures or fallback cases.

## Results

| component        | pipeline     |   accuracy |   precision |     recall |         f1 |   n_eval |   tp |   fp |   fn |   candidate_pairs |   predicted_matches |   gold_positive_pairs |   n_eval_values |   correct_values |   all_possible_cross_source_pairs |   reduction_ratio |   total_llm_calls |   schema_alignment_calls |   record_linkage_calls |   data_fusion_calls |
|:-----------------|:-------------|-----------:|------------:|-----------:|-----------:|---------:|-----:|-----:|-----:|------------------:|--------------------:|----------------------:|----------------:|-----------------:|----------------------------------:|------------------:|------------------:|-------------------------:|-----------------------:|--------------------:|
| schema_alignment | baseline     |   0.4      |    1        |   0.4      |   0.571429 |       15 |    6 |    0 |    9 |               nan |                 nan |                   nan |             nan |              nan |                        nan        |        nan        |               nan |                      nan |                    nan |                 nan |
| schema_alignment | llm_assisted |   1        |    1        |   1        |   1        |       15 |   15 |    0 |    0 |               nan |                 nan |                   nan |             nan |              nan |                        nan        |        nan        |               nan |                      nan |                    nan |                 nan |
| record_linkage   | baseline     | nan        |    0.304601 |   0.654722 |   0.41577  |      nan | 2357 | 5381 | 1243 |             33855 |                7738 |                  3600 |             nan |              nan |                        nan        |        nan        |               nan |                      nan |                    nan |                 nan |
| record_linkage   | llm_assisted | nan        |    0.31755  |   0.5775   |   0.409776 |      nan | 2079 | 4468 | 1521 |             33855 |                6547 |                  3600 |             nan |              nan |                        nan        |        nan        |               nan |                      nan |                    nan |                 nan |
| data_fusion      | baseline     |   0.200295 |  nan        | nan        | nan        |      nan |  nan |  nan |  nan |               nan |                 nan |                   nan |             679 |              136 |                        nan        |        nan        |               nan |                      nan |                    nan |                 nan |
| data_fusion      | llm_assisted |   0.23904  |  nan        | nan        | nan        |      nan |  nan |  nan |  nan |               nan |                 nan |                   nan |             958 |              229 |                        nan        |        nan        |               nan |                      nan |                    nan |                 nan |
| blocking         | baseline     | nan        |  nan        | nan        | nan        |      nan |  nan |  nan |  nan |             33855 |                 nan |                   nan |             nan |              nan |                          1.08e+06 |          0.968653 |               nan |                      nan |                    nan |                 nan |
| blocking         | llm_assisted | nan        |  nan        | nan        | nan        |      nan |  nan |  nan |  nan |             33855 |                 nan |                   nan |             nan |              nan |                          1.08e+06 |          0.968653 |               nan |                      nan |                    nan |                 nan |
| llm_usage        | llm_assisted | nan        |  nan        | nan        | nan        |      nan |  nan |  nan |  nan |               nan |                 nan |                   nan |             nan |              nan |                        nan        |        nan        |               353 |                        3 |                    300 |                  50 |

## Output files

The notebook generates:

- schema_alignment_baseline.csv
- schema_alignment_llm.csv
- candidate_pairs_baseline.csv
- candidate_pairs_llm.csv
- record_linkage_baseline.csv
- record_linkage_llm.csv
- clusters_baseline.csv
- clusters_llm.csv
- integrated_dataset_baseline.csv
- integrated_dataset_llm.csv
- fusion_eval_baseline.csv
- fusion_eval_llm.csv
- metrics_summary.csv
- llm_logs.jsonl

## Notes
The silver truth of the merger is constructed using majority voting on the values ​​of the gold cluster, which introduces a structural advantage for the baseline that also uses majority voting. The merger results should be interpreted with this limitation in mind.
