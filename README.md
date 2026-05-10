# PRIME-Py: Data Collection and Feature Extraction Pipeline

[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.20110499.svg)](https://doi.org/10.5281/zenodo.20110499)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.11](https://img.shields.io/badge/python-3.11-blue.svg)](https://www.python.org/downloads/)

This repository contains the complete data collection and feature extraction![Uploading image.png…]()
![Uploading image.png…]()

pipeline used to construct the **PRIME-Py** dataset — a large-scale multimodal
dataset of 1,997,535 Python functions extracted from 2,797 open-source GitHub
repositories.

> **Dataset:** [https://doi.org/10.5281/zenodo.20110499](https://doi.org/10.5281/zenodo.20110499)
>
> **Related article:** Alehaidib R, Ghoneim A, Alrashoud M. 2026.
> Large-Scale Empirical Study of Code Smell and Anti-Pattern Detection
> in Python Open-Source Software. *PeerJ Computer Science.*
> DOI: [to be added on acceptance]

---

## Pipeline Overview

The pipeline runs in six sequential steps, each producing an intermediate
output that feeds into the next step.

```
Step 1                   Step 2                  Step 3
clone_repositories.py -> run_lizard.py --------> run_ast_extractor.py
        |                     |                        |
  Cloned repos          Structural metrics       Lexical features
  (local disk)          (Lizard output)          (AST output)
                              |                        |
                              +----------+-------------+
                                         |
                                  Step 4
                            merge_and_validate.ipynb
                                         |
                              master_dataset.parquet
                         master_dataset_with_lexical.parquet
                                         |
                    +--------------------+--------------------+
                    |                                         |
             Step 5                                    Step 6
  generate_lexical_representation.py      CodeT5 summarisation on Colab A100.ipynb
                    |                                         |
         lexical_representation                         codet5_summary
           final_lexical columns                      (T5 neural summaries)
                    |                                         |
                    +--------------------+--------------------+
                                         |
                          master_dataset_with_codet5.parquet
                          train.parquet / val.parquet / test.parquet
```

---

## Repository Structure

```
PRIME-Py/
├── README.md
├── requirements.txt
├── clone_repositories.py               # Step 1 — Repository collection
├── run_lizard.py                       # Step 2 — Structural metric extraction
├── run_ast_extractor.py                # Step 3 — Lexical feature extraction
├── merge_and_validate.ipynb            # Step 4 — Merge, validate, and split
├── generate_lexical_representation.py  # Step 5 — Lexical text representation
└── CodeT5 summarisation on Colab A100.ipynb  # Step 6 — Neural summarisation
```

---

## Requirements

### Local environment

```bash
conda create -n prime python=3.11
conda activate prime
pip install -r requirements.txt
```

Key dependencies:

| Package | Version | Used in |
|---|---|---|
| pandas | 2.2.2 | All steps |
| pyarrow | 16.1.0 | All steps |
| lizard | 1.17.10 | Step 2 |
| PyGithub | — | Step 1 |
| GitPython | — | Step 1 |
| scipy | 1.17.1 | Step 4 |
| scikit-learn | 1.8.0 | Step 4 |
| tqdm | — | All steps |

### Google Colab (Step 6 only)

Step 6 requires a Google Colab Pro+ session with an A100 GPU.
No local GPU is needed for any other step.

---

## Step-by-Step Usage

### Step 1 — Clone Repositories (`clone_repositories.py`)

Searches GitHub for Python repositories meeting the inclusion criteria
and clones them to a local directory.

**Inclusion criteria:**
- Python as the primary language
- At least 50 GitHub stars
- At least 500 lines of Python code
- Non-forked repository

```bash
conda activate prime
python clone_repositories.py \
  --output_dir /path/to/repos \
  --token YOUR_GITHUB_TOKEN \
  --max_repos 3000
```

**Output:** Cloned repositories saved to `--output_dir`.

> A GitHub personal access token is required.
> Generate one at: https://github.com/settings/tokens

---

### Step 2 — Structural Metric Extraction (`run_lizard.py`)

Runs the Lizard static analyser on all Python files in the cloned
repositories, extracting per-function structural metrics.

**Extracted features:** `nloc`, `cyclomatic_complexity`,
`num_parameter`, `num_token`, `start_line`, `end_line`,
`function_name`, `class_name`, `file_path`, `project_name`

```bash
conda activate prime
python run_lizard.py \
  --repos_dir /path/to/repos \
  --output_file /path/to/output/lizard_output.parquet
```

**Output:** `lizard_output.parquet`

---

### Step 3 — Lexical Feature Extraction (`run_ast_extractor.py`)

Parses all Python source files using Python's built-in `ast` module
to extract per-function lexical and structural call graph features.

**Extracted features:** `outgoing_function_count`,
`outgoing_function_names`, `incoming_function_count`,
`incoming_function_names`, `function_num_variables`,
`function_num_functions`, `function_num_lines`,
`function_body_line_type`, `function_params`,
`function_return_type`, `function_body`, `class_has_bases`,
`class_modifiers`

```bash
conda activate prime
python run_ast_extractor.py \
  --repos_dir /path/to/repos \
  --output_file /path/to/output/ast_output.parquet
```

**Output:** `ast_output.parquet`

---

### Step 4 — Merge, Validate, and Split (`merge_and_validate.ipynb`)

Merges the Lizard and AST outputs into a single dataset, validates
column completeness and data integrity, removes duplicate functions,
and produces the project-level train/validation/test split.

```bash
conda activate prime
jupyter notebook merge_and_validate.ipynb
```

Edit the input paths in the first cell before running:

```python
LIZARD_PATH = "/path/to/output/lizard_output.parquet"
AST_PATH    = "/path/to/output/ast_output.parquet"
OUTPUT_DIR  = "/path/to/output/"
SPLIT_SEED  = 42
```

**Output files:**
- `master_dataset.parquet` — structural features only
- `master_dataset_with_lexical.parquet` — structural + lexical features
- `train.parquet`, `val.parquet`, `test.parquet` — project-level splits

---

### Step 5 — Lexical Representation (`generate_lexical_representation.py`)

Constructs two text representation columns from the function body
and identifier text:

- `lexical_representation` — function name, parameter names, and
  tokenised body identifiers concatenated as a space-separated string
- `final_lexical` — cleaned and normalised version: snake_case tokens
  split into constituent words, camelCase split, numeric literals
  replaced, stop words removed

```bash
conda activate prime
python generate_lexical_representation.py \
  --input_file /path/to/output/master_dataset_with_lexical.parquet \
  --output_file /path/to/output/master_dataset_with_lexical.parquet
```

**Output:** Updates `master_dataset_with_lexical.parquet` in place
with the two new columns.

---

### Step 6 — Neural Code Summarisation (`CodeT5 summarisation on Colab A100.ipynb`)

Generates natural language semantic summaries for each function using
the SEBIS CodeTrans T5 model
(`SEBIS/code-trans-t5-base-code-summarization-python`), fine-tuned
on the CodeSearchNet Python corpus.

**Requirements:**
- Google Colab Pro+ session with A100 GPU (40GB HBM2)
- Google Drive mounted at `/content/gdrive`
- `master_dataset_with_lexical.parquet` uploaded to Google Drive

**Estimated runtime:** ~19.3 hours for the full corpus of 1,997,535
functions (batch size 64, max input 512 tokens, max output 64 tokens).

Upload the notebook to Google Colab and mount your Drive:

```python
from google.colab import drive
drive.mount('/content/gdrive')

INPUT_PATH  = "/content/gdrive/MyDrive/prime/master_dataset_with_lexical.parquet"
OUTPUT_PATH = "/content/gdrive/MyDrive/prime/master_dataset_with_codet5.parquet"
```

**Output:** `master_dataset_with_codet5.parquet` — the complete
feature dataset with the `codet5_summary` column added.

---

## Output Files Summary

| File | Produced in | Rows | Columns | Description |
|---|---|---|---|---|
| `lizard_output.parquet` | Step 2 | 1,997,535 | 10 | Raw Lizard output |
| `ast_output.parquet` | Step 3 | 1,997,535 | 13 | Raw AST output |
| `master_dataset.parquet` | Step 4 | 1,997,535 | 11 | Merged structural dataset |
| `master_dataset_with_lexical.parquet` | Steps 4–5 | 1,997,535 | 26 | + Lexical features and text representations |
| `master_dataset_with_codet5.parquet` | Step 6 | 1,997,535 | 27 | Complete feature dataset |
| `train.parquet` | Step 4 | 1,554,896 | 27 | Training split (2,237 projects) |
| `val.parquet` | Step 4 | 165,226 | 27 | Validation split (280 projects) |
| `test.parquet` | Step 4 | 277,413 | 27 | Test split (280 projects) |

The final six files (`master_dataset.parquet` through `test.parquet`)
are the files deposited at Zenodo
(DOI: [https://doi.org/10.5281/zenodo.20110499](https://doi.org/10.5281/zenodo.20110499)).

---

## Reproducibility Notes

- All random operations use `seed=42` for full reproducibility.
- The project-level split in Step 4 uses `numpy.random.RandomState(42)`.
- Steps 1–5 can be fully reproduced on a standard CPU machine.
- Step 6 requires GPU access; inference results may vary slightly
  across GPU models due to floating-point differences, but summaries
  are functionally equivalent.
- Total estimated runtime for Steps 1–5: 48–72 hours depending on
  network speed (Step 1) and CPU cores (Steps 2–3).

---

## Citation

If you use this pipeline or the PRIME-Py dataset in your research,
please cite:

```bibtex
@article{alehaidib2026primepy,
  title   = {Large-Scale Empirical Study of Code Smell and Anti-Pattern
             Detection in Python Open-Source Software},
  author  = {Alehaidib, Reem and Ghoneim, Ahmed and Alrashoud, Mubarak},
  journal = {PeerJ Computer Science},
  year    = {2026},
  doi     = {[to be added on acceptance]}
}

@dataset{alehaidib2026primepy_data,
  title     = {PRIME-Py: A Large-Scale Multimodal Python Function Dataset},
  author    = {Alehaidib, Reem and Ghoneim, Ahmed and Alrashoud, Mubarak},
  year      = {2026},
  publisher = {Zenodo},
  doi       = {10.5281/zenodo.20110499}
}
```

---

## Authors

**Reem Alehaidib** (Corresponding Author)
Department of Software Engineering,
College of Computer and Information Sciences,
King Saud University, Riyadh, Saudi Arabia.
Email: 444203308@student.ksu.edu.sa

**Ahmed Ghoneim** · **Mubarak Alrashoud**
King Saud University, Riyadh, Saudi Arabia.

---

## License

This repository is released under the
[MIT License](LICENSE).
The dataset is released under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
