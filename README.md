# Synthetic Fine-Tuning for Multilingual Biomedical Named Entity Recognition

This repository implements a synthetic-data generation and training pipeline for biomedical named entity recognition (BioNER) in English, Spanish, German, and French. The pipeline samples disease terms from biomedical terminologies and uses a decoder language model to generate and annotate synthetic text. The synthetic examples are used to pretrain a smaller BERT-based BioNER model with a bidirectional recurrent layer, multi-head attention (MHA), and a conditional random field (CRF). The model is then fine-tuned on 1%, 5%, 10%, 20%, 50%, or 100% of a gold-labelled downstream dataset.

TODO: The paper associated with this code [`paper`](https://TBD) TBD.
Synthetic data available at [`ieee-dataport`](https://dx.doi.org/10.21227/jq10-2p73).

## Main results
TODO: plots and images 
> **TODO — main-results figure:** Add the paper figure containing the complete F1 results, including the 20%, 50%, and 100% gold-data settings.

## Method overview

1. Download the gold corpora and biomedical terminology resources.
2. Convert each corpus to the repository's sentence-level JSON format and create the 1%, 5%, 10%, 20%, and 50% training subsets.
3. Sample disease terms from SNOMED CT, ICD-10, and the Disease Ontology.
4. Generate zero-shot or three-shot synthetic sentences with a decoder language model served by llama.cpp. 
5. Ask the language model to check or correct the entity annotations and apply language-specific spaCy processing.
6. Pretrain the BioNER model on the corrected synthetic data.
7. Fine-tune the model on a selected percentage of gold data.
8. Evaluate the model on the unchanged gold test set.

> **TODO — pipeline figure:** Add the methodology figure from the paper here.

## Repository contents

```text
synth_bioNER/
├── train.py                       # BioNER training and evaluation entry point
├── models.py                      # BiRNN, MHA, BERT, character CNN, and CRF model
├── datasets.py                    # Processed dataset loader
├── preprocessing.py               # Transformer tokenization and embeddings
├── evaluation.py                  # seqeval precision, recall, and F1 evaluation
├── hyper_param_tuning.py          # Optuna hyperparameter search
├── kshot_synthetic_generation.py  # Synthetic generation and correction pipeline
├── data_wranglig/                 # Corpus-specific preprocessing scripts
├── src_generate/                  # Prompt construction and generation utilities
├── experiments/
│   ├── data/                      # Raw-data preprocessing configurations
│   ├── ncbi/                      # English experiments
│   ├── distemist/                 # Spanish experiments
│   ├── bronco150/                 # German experiments
│   ├── quaero/                    # French experiments
│   ├── generate_kshot4B.sh        # 4B generation SLURM template
│   ├── generate_kshot27B.sh       # 27B generation SLURM template
│   └── preprocess.sh              # Preprocessing SLURM template
├── utils/                         # Trainers, logging, and output analysis
├── start-bioner-docker/           # Container recipe and Python requirements
└── settings.template.py           # Local path configuration template
```

Experiment directories contain three configuration families:

- `baseline/*.yml`: train only on a gold-data subset.
- `0shot/pretrain.yml` and `0shot/ft_*.yml`: pretrain on zero-shot synthetic data, then fine-tune on gold data.
- `3shot_<percentage>/pretrain.yml` and `3shot_<percentage>/ft_*.yml`: pretrain on three-shot synthetic data generated using examples from the indicated percentage of the gold training set, then fine-tune on gold data.
- `data/<dataset>/0shot_generation.yml` and `data/<dataset>/3shot_generation.yml`: generate synthetic data. 

## Requirements

The supplied research workflow targets a Linux server or HPC cluster. NVIDIA CUDA GPU recommended for training; generation scripts request two GPUs a CUDA llama.cpp server image, using GGUF model weights.

### Language models and encoders

The experiments expect local copies of the following encoder families beneath `EMBEDDINGS_PATH`:

| Experiment name | Expected local directory |
|---|---|
| `bioBERT` | `bioBERT_setup/` |
| `roBERTa_clinical_es` | `roBERTa_clinical_es_setup/` |
| `drBERT_fr` | `drBERT_fr_setup/` |
| `medBERT_ger` | `medBERT_ger_setup/` |


The preprocessing and generation configurations additionally use these spaCy pipelines:
```text
en_core_web_trf, es_dep_news_trf, de_dep_news_trf, fr_dep_news_trf
```

## Installation and environment setup

Clone the repository:
```bash
git clone --branch paper https://github.com/VidasMarta/synth_bioNER.git
cd synth_bioNER
```

Create a Python environment and install the current dependency list:

```bash
python3 -m venv .venv
source .venv/bin/activate

python -m pip install --upgrade pip
python -m pip install -r start-bioner-docker/requirements_base.txt
```

Create the local settings file:

```bash
cp settings.template.py settings.py
```

Edit `settings.py` and configure all paths:

```python
MODEL_PATH = "/absolute/path/to/model-checkpoints"
DATA_PATH = "/absolute/path/to/processed-data"
OUTPUT_PATH = "/absolute/path/to/output"
EXPERIMENTS_PATH = "/absolute/path/to/synth_bioNER/experiments"
LOG_PATH = "/absolute/path/to/logs"
EMBEDDINGS_PATH = "/absolute/path/to/local-transformer-models"
```

Create the model and log directories before training. `settings.py` is ignored by Git so machine-specific paths are not committed.

### Container setup

The intended image is based on:

```text
pytorch/pytorch:2.4.1-cuda12.4-cudnn9-runtime
```

The intended build interface is:

```bash
docker build -t syn-bioner -f start-bioner-docker/Dockerfile .
```

- Document `docker run` with NVIDIA GPU support.

## Data

### Synthetic-data archive
- DOI Link: https://dx.doi.org/10.21227/jq10-2p73 

### Gold NER corpora

Gold corpora and terminology resources are not tracked in Git. Obtain each resource from its provider and comply with its individual terms. [NCBI Disease Corpus](https://www.ncbi.nlm.nih.gov/CBBresearch/Dogan/DISEASE/), [Zenodo record 7614764](https://zenodo.org/records/7614764), [BRONCO project page](https://www2.informatik.hu-berlin.de/~leser/bronco/index.html), [DrBenchmark/QUAERO](https://huggingface.co/datasets/DrBenchmark/QUAERO)

### Knowledge bases
Synthetic prompts sample disease terminology from:

- SNOMED CT [Athena OHDSI](https://athena.ohdsi.org/);
- language-specific ICD-10 resources [FR](https://www.bfs.admin.ch/asset/fr/20384008) and [DE](https://athena.ohdsi.org/);
- [Disease Ontology](https://github.com/DiseaseOntology) snapshot obtained in August 2025.

## Runtime and computational cost

A full reproduction is expected to take approximately **seven days on a server with two NVIDIA A100 GPUs**, although runtime depends strongly on generator size, GPU memory, llama.cpp build, batch size, storage performance, and queue time.

## Citation

```bibtex
@article{TODO,
  title   = {Multi-head Attention based Synthetic Data Generation and BioNER},
  author  = {MArta Vidas, Miha Keber, Anja Barešić, Jelena Bozek},
  journal = {TODO: Venue},
  year    = {TODO: Year},
  doi     = {TODO: DOI},
  url     = {TODO: Publication URL}
}
```
When using an individual gold corpus, terminology, language model, or pretrained encoder, cite that resource separately according to its provider's instructions.
