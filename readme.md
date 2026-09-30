# Repository-Level Code Merge Conflict Resolution Based on Historical Information

> Repository-Level Code Merge Conflict Resolution Based on Historical Resolution Retrieval and Large Language Models

## Table of Contents

- [Project Overview](#project-overview)
- [Method Overview](#method-overview)
- [Repository Structure](#repository-structure)
- [Environment and Dependencies](#environment-and-dependencies)
- [Configuration](#configuration)
- [Data](#data)
- [Reproduction Workflow](#reproduction-workflow)
- [Modules and APIs](#modules-and-apis)
- [Security and Secret Management](#security-and-secret-management)
- [References and Acknowledgments](#references-and-acknowledgments)

---

## Project Overview

During software development, parallel changes to the same codebase by multiple developers can cause merge conflicts when branches are merged. Resolving conflicts manually is time-consuming and error-prone. This project proposes a **repository-level automatic merge conflict resolution method based on historical resolutions**:

1. For each code repository, collect all of its historical merge conflicts and their manually created resolutions to build a repository-specific **retrieval source** (using the Milvus cloud vector database).
2. For a merge conflict to be resolved, use **hybrid retrieval combining sparse retrieval (BM25 / term inner product) and dense retrieval (CodeT5 / CodeBERT encoding), followed by RRF / weighted reranking** to find similar historical conflict-resolution examples.
3. Combine the retrieved examples with the current conflict block to construct an augmented prompt, then call a **large language model** (OpenAI GPT / DeepSeek / Alibaba Cloud Model Studio Qwen) to generate a resolution consistent with the repository's style.

The overall method corresponds to Algorithm 4.1 in the paper. Its two core components are **retrieval of similar historical conflict-resolution records** (retrieval-source construction and the hybrid retrieval module) and **LLM-driven generation of conflict-resolution proposals**.

---

## Method Overview

```text
Input:
    history_tuple   — all historical conflict tuples in the repository (conflict block, resolution, globally unique ID)
    conflict        — the merge conflict block to resolve
Output:
    resolution_code — the complete resolved code

1. IF history_tuple or conflict is empty THEN RETURN fault
2. id_list, conflict_list, resolution_list = preprocess(history_tuple)
3. source = create_retrieval_source(id_list, conflict_list, resolution_list, BM25, CodeT5)  # Build historical retrieval source
4. history_conflict_pair = RRF(conflict, CodeT5, source)                                    # Hybrid retrieval
5. prompt = prepare(conflict, history_conflict_pair)                                         # Populate prompt
6. resolution = LLM(prompt)                                                                  # Call the large language model
7. RETURN postprocess(resolution)
```

### 1. Building the Retrieval Source

- Group merge conflicts from the selected dataset by **repository**, and extract each repository's conflict tuples (conflict block `chunk_content`, resolution `chunk_resolution`, and globally unique ID `global_id`).
- Define the vector database collection schema with fields for the original conflict-block content, its resolution, sparse and dense embedding vectors (768 dimensions, encoded by CodeT5), and the globally unique ID.
- When inserting data, encode conflict blocks with the pretrained code model **CodeT5** to generate dense vectors, and use **BM25** to generate sparse vectors. Create indexes for efficient retrieval.

### 2. Hybrid Text and Vector Retrieval

The method uses a **dual-channel retrieval and reranking** strategy:

- **Sparse retrieval**: Use BM25 / term inner product to measure relevance based on keyword frequency and distribution.
- **Dense retrieval**: Encode conflicts as dense vectors with CodeT5 / CodeBERT to measure semantic similarity.
- **Fusion and reranking**: Combine and rank results from both channels with **RRF (Reciprocal Rank Fusion)** or **WeightedRanker**, returning historical resolution records sorted by similarity.

### 3. Prompt Design and LLM Resolution

The prompt has three parts:

1. **Task Instructions**;
2. **Reference Example**: the most similar historical conflict-resolution cases retrieved (the number, n, is determined experimentally);
3. **Merge Conflict to be Resolved Next**.

After calling the large language model API to generate a resolution, a regular expression extracts the last code block enclosed in triple backticks and trims leading and trailing whitespace to obtain the resolved code.

---

## Repository Structure

```
repoMerge/
├── README.md                           # This document
├── requirements.txt                    # Dependency list
├── .env.example                        # Environment variable template (secret placeholders)
├── .gitignore                          # Ignore rules
│
├── utils.py                            # Utilities: Git commands, diff-marker handling, result extraction, prompt templates
├── openai_api.py                       # OpenAI (GPT-3.5 / GPT-4o) API
├── deepseek_api.py                     # DeepSeek API
├── alibaba_api.py                      # Alibaba Cloud Model Studio (Qwen) API
│
├── database_api_T5_BM25_L2.py          # Retrieval API: CodeT5 + BM25 (sparse) + L2 (dense) + RRF
├── database_api_T5_BM25_L2_threshold.py    # Variant: T5-BM25-L2 with quality-threshold filtering
├── database_api_T5_BM25_L2_dense_only.py   # Variant: T5-BM25-L2 with dense-only retrieval (ablation)
├── database_api_T5_IP_COSINE.py        # Retrieval API: CodeT5 + BM25 (sparse) + COSINE (dense) + WeightedRanker
├── database_api_T5_IP_COSINE_dense_only.py # Variant: T5-IP-COSINE with dense-only retrieval (WeightedRanker(0,1))
├── database_api_T5_IP_COSINE_sparse_only.py # Variant: T5-IP-COSINE with sparse-only retrieval (WeightedRanker(1,0))
├── database_api_CB_IP_COSINE.py        # Retrieval API: CodeBERT + BM25 (sparse) + COSINE (dense) + WeightedRanker
│
├── data-process.ipynb                  # ① Dataset processing (grouping, cleaning, splitting)
├── vector-database-T5-BM25-L2.ipynb    # ② Build vector retrieval source (T5-BM25-L2 variant)
├── vector-database-T5-IP-COSINE.ipynb  # ② Build vector retrieval source (T5-IP-COSINE variant)
├── vector-database-CB-IP-COSINE.ipynb  # ② Build vector retrieval source (CB-IP-COSINE variant)
├── reranker.ipynb                      # ③ Main workflow (retrieval + LLM resolution) and reranking experiments
├── reranker_all.ipynb                  # ③ Full reranking workflow experiments
│
├── dataset/                            # Raw datasets (git-ignored; sources: MergeBERT / 50-repo)
├── dataset_all/                        # Repository-level merge-conflict data (all, sorted, top/bottom slices)
├── dataset_split/                      # Split data (train / test / vector)
└── dataset_rerank/                     # Reranking experiment data (train / test / vector)
```

> **Note**: `dataset/` contains downloaded raw data and is large, so it is excluded by `.gitignore` by default. Model weights (`codeT5/`, `codebert/`) are downloaded automatically on first use or loaded from the local cache, and are also not checked into the repository.

---

## Environment and Dependencies

- Python 3.9+
- Install dependencies:

```bash
pip install -r requirements.txt
```

Main dependencies:

| Dependency | Purpose |
| --- | --- |
| `openai` | Call GPT / DeepSeek / Qwen through OpenAI-compatible APIs |
| `pymilvus` | Connect to the Milvus / Zilliz Cloud vector database for hybrid retrieval |
| `transformers` | Load CodeT5 / CodeBERT encoding models |
| `torch` | Deep learning inference framework |
| `pandas` / `numpy` | Data processing |
| `colorama` / `tqdm` | Terminal output and progress display |

---

## Configuration

All secrets and connection details are read from **environment variables**. Copy `.env.example` to `.env` and fill in the real values, or export them in the terminal before running:

```bash
# Large language models
export OPENAI_API_KEY="..."
export OPENAI_BASE_URL="https://api.openai-proxy.org/v1"
export DEEPSEEK_API_KEY="..."
export DEEPSEEK_BASE_URL="https://api.deepseek.com"
export DASHSCOPE_API_KEY="..."
export DASHSCOPE_BASE_URL="https://dashscope.aliyuncs.com/compatible-mode/v1"

# Vector database
export MILVUS_CLUSTER_ENDPOINT="..."
export MILVUS_TOKEN="..."
```

Environment variables read by each code module:

| File | Environment variables read | Default values |
| --- | --- | --- |
| `openai_api.py` | `OPENAI_API_KEY` / `OPENAI_BASE_URL` | Placeholder / `https://api.openai-proxy.org/v1` |
| `deepseek_api.py` | `DEEPSEEK_API_KEY` / `DEEPSEEK_BASE_URL` | Placeholder / `https://api.deepseek.com` |
| `alibaba_api.py` | `DASHSCOPE_API_KEY` / `DASHSCOPE_BASE_URL` | Placeholder / `https://dashscope.aliyuncs.com/compatible-mode/v1` |

> The notebook variables `CLUSTER_ENDPOINT` and `TOKEN` have been replaced with the placeholders `YOUR_MILVUS_ENDPOINT` and `YOUR_MILVUS_TOKEN`. Replace them with real values before running (or inject them through environment variables).

---

## Data

This project uses publicly available code merge-conflict datasets (**MergeBERT** and **50-repo**), reorganized by repository. The data is stored in four directories:

- **`dataset/` (raw data)**: Raw merge-conflict data organized by language/source. Includes `mergebert/` (`json`, `json_cs`, `json_js`, `json_ts`, `vector`) and `50repo/` (`json`, `vector`).
- **`dataset_all/` (all repository-level data)**: One JSON file per repository (e.g., `spring_time.json`, `orientdb_time.json`), along with aggregated slices such as `merged_sorted.json`, `merged_top80.json`, and `merged_bottom20.json`.
- **`dataset_split/` (split data)**: `mergebert_20/40/60/80.json` files under `train/` and `test/`, plus `*_Conflicts.txt` files under `vector/` (tab-separated conflict text used to fit BM25).
- **`dataset_rerank/` (reranking experiment data)**: `merged_top20/40/60/80.json` files under `train/`, `merged_bottom20/40/60/80.json` files under `test/`, and `vector/`.

The core fields in each conflict record include the conflict-block content `chunk_content`, resolution `chunk_resolution`, globally unique ID `global_id`, timestamp `chunk_timestamp`, and group/repository ID `group_id`.

---

## Reproduction Workflow

The full experimental workflow consists of three stages, corresponding to the numbered notebooks:

### ① Data Preprocessing — `data-process.ipynb`

- Extract merge-conflict tuples from the raw data by repository.
- Clean conflict blocks (for example, normalize diff markers `<<<<<<<` / `=======` / `>>>>>>>`).
- Generate datasets such as `dataset_all` and `dataset_split` (train/test/vector).

### ② Build the Vector Retrieval Source — `vector-database-*.ipynb`

- Connect to Milvus, create the collection schema (with sparse and dense vectors, global IDs, and other fields), and create indexes.
- Encode conflict blocks with CodeT5 / CodeBERT and insert the vectors into the database.
- Three retrieval variants are provided: `T5-BM25-L2`, `T5-IP-COSINE`, and `CB-IP-COSINE`.

### ③ Main Workflow and Reranking Experiments — `reranker.ipynb` / `reranker_all.ipynb`

- Call `query_similar` / `query_similar_inAll` in `database_api_*.py` to retrieve similar historical resolution records.
- Build augmented prompts using the prompt templates in `utils.py`.
- Call `openai_api.py` / `deepseek_api.py` / `alibaba_api.py` to generate resolved code and extract the final result.
- Rerank, compare, and evaluate retrieval results (supports resuming from checkpoints and reusing existing outputs).

> When running a notebook, start Jupyter from the repository root so that modules such as `utils`, `database_api_*`, and `openai_api` can be imported correctly.

---

## Modules and APIs

### `utils.py`

| Function | Purpose |
| --- | --- |
| `run_git_command` / `get_parent_hashes` / `get_merge_base` / `get_commit_message` / `get_commit_time` | Extract metadata from Git merge commits |
| `execute` / `execute_with_info` | Analyze merge commits and return parent and common-ancestor messages and timestamps |
| `rewrite_DiffMarks` | Normalize diff markers in results to a form without file paths |
| `extract_resolved_code` | Extract the last code block from generated text |
| `simplePrompt` / `prompt_message` / `prompt1` ~ `prompt5` | Prompt template variants |

### Retrieval Modules

| File | Sparse retrieval | Dense retrieval | Reranking |
| --- | --- | --- | --- |
| `database_api_T5_BM25_L2.py` | BM25 | CodeT5 + L2 | RRF |
| `database_api_T5_BM25_L2_threshold.py` | BM25 | CodeT5 + L2 | RRF + quality-threshold filtering |
| `database_api_T5_BM25_L2_dense_only.py` | — (dense only) | CodeT5 + L2 | WeightedRanker(1) |
| `database_api_T5_IP_COSINE.py` | BM25 (term inner product, IP) | CodeT5 + COSINE | WeightedRanker |
| `database_api_T5_IP_COSINE_dense_only.py` | — (dense only) | CodeT5 + COSINE | WeightedRanker(0,1) |
| `database_api_T5_IP_COSINE_sparse_only.py` | BM25 (term inner product, IP) | — (sparse only) | WeightedRanker(1,0) |
| `database_api_CB_IP_COSINE.py` | BM25 (term inner product, IP) | CodeBERT + COSINE | WeightedRanker |

Main functions:

- `query_similar(CLUSTER_ENDPOINT, TOKEN, vector_database, query, limit[, current_ts])`: Retrieve similar conflicts within a repository, with optional upper-bound time filtering.
- `query_similar_inAll(CLUSTER_ENDPOINT, TOKEN, vector_database, query, limit, group_id)`: Retrieve across repositories, filtering out conflicts from the current group.

### LLM API Modules

| File | Model | Function |
| --- | --- | --- |
| `openai_api.py` | `gpt-3.5-turbo` / `gpt-4o` | `gpt_35` / `gpt_4o` |
| `deepseek_api.py` | `deepseek-v4-pro` | `deepSeek` |
| `alibaba_api.py` | `qwen-turbo` | `qwen` |

All three use the same interface: `func(question, number, truth)`, which returns a list of resolved code.

---

## Security and Secret Management

- **Never hard-code** API keys or database tokens in source code. Inject them through environment variables; see `.env.example`.
- The `.env` file is excluded by `.gitignore`; do not commit it to a public repository.
- `YOUR_*` strings in the repository are placeholders and safe to publish.

---

## References and Acknowledgments

This work uses the following public resources:

- Datasets: MergeBERT (a merge-conflict dataset) and 50-repo.
- Pretrained models: [Salesforce/CodeT5](https://huggingface.co/Salesforce/codet5-base), [microsoft/CodeBERT](https://huggingface.co/microsoft/codebert-base).
- Vector database: [Milvus](https://milvus.io/) / Zilliz Cloud.
- Large language models: OpenAI GPT, DeepSeek, and Alibaba Cloud Model Studio Qwen.

> If you use this project, cite the relevant data sources and follow their respective model and data licenses.
