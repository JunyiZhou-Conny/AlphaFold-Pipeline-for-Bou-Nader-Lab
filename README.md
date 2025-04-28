# AlphaPulldown + AlphaFold3 Research Log

This README documents the complete pipeline, setup, troubleshooting, and experimental execution history for running AlphaPulldown and AlphaFold3 workflows for protein complex structure prediction. All content is designed for reproducibility and future reference.

## Table of Contents
- [TODO](#todo)
- [Reference Links](#reference-links)
- [File Transfer & Utilities](#file-transfer--utilities)
- [Expected Data from AlphaFold3](#expected-data-from-alphafold3)
- [AlphaPulldown Overview](#alphapulldown-overview)
- [Environment Setup](#environment-setup)
- [Feature Generation](#feature-generation)
- [Understanding `create_individual_features.py`](#understanding-create_individual_featurespy)
- [Accessing Databases](#accessing-databases)
- [Multimer Job Execution](#multimer-job-execution)
- [Result Analysis](#result-analysis)
- [AlphaFold3 Notes](#alphafold3-notes)
- [Troubleshooting & Exceptions](#troubleshooting--exceptions)

---

## TODO
1. Check `missing_proteins.txt` for Feature Database completeness.
2. Investigate MMseqs2 errors and output inconsistencies.

---

## Reference Links

### UniProt
- Protein sequences: https://www.uniprot.org
- REST API example: https://www.uniprot.org/uniprotkb/Q04740/entry

### AlphaPulldown
- Paper: https://academic.oup.com/bioinformatics/article/39/1/btac749/6839971
- GitHub: https://github.com/KosinskiLab/AlphaPulldown

### AF2-Multimer
- SBGrid example: https://sbgrid.org/wiki/examples/alphafold2
- GitHub: https://github.com/google-deepmind/alphafold

### AlphaFold3
- Server: https://alphafoldserver.com/welcome
- GitHub: https://github.com/google-deepmind/alphafold3

---

## File Transfer & Utilities

```bash
# Transfer files (adjust paths as needed):
scp /local/path/to/file jzho349@kilimanjaro.biochem.emory.edu:/remote/path
scp -r jzho349@kilimanjaro.biochem.emory.edu:/remote/result/path ~/Downloads/

# Split protein FASTA list into 8 chunks:
split -n l/8 --suffix-length=2 --additional-suffix=.txt unique_protein_ids_Jack.txt target_chunk_Jack_

# One-liner tracing number of jobs completed
for d in gpu_*; do echo "$d: $(ls -1 "$d" | wc -l)"; done
```

---

## Expected Data from AlphaFold3
1. **mmCIF**: Predicted complex structure.
2. **pLDDT**: Per-residue confidence score.
3. Possibly more outputs such as ipTM or pairwise confidence (to be explored).

---

## AlphaPulldown Overview
AlphaPulldown consists of two core stages:

1. [`create_individual_features.py`](https://github.com/KosinskiLab/AlphaPulldown#1-compute-multiple-sequence-alignment-msa-and-template-features-cpu-stage):
   - Computes MSAs and finds templates.
   - Stores monomer features as `.pkl` files.

2. [`run_multimer_jobs.py`](https://github.com/KosinskiLab/AlphaPulldown#2-predict-structures-gpu-stage):
   - Predicts structures based on the generated features.

---

## Environment Setup

```bash
# Create environment
conda create -n AlphaPulldown -c omnia -c bioconda -c conda-forge \
  python==3.11 openmm==8.0 pdbfixer==1.9 kalign2 hhsuite hmmer modelcif

# Activate & install JAX for GPU acceleration
conda activate AlphaPulldown
pip install -U "jax[cuda12]"
```

---

## Feature Generation

### MMseqs2 (Sequential Mode)
```bash
NUM_SEQ=$(grep -c "^>" protein_sequences.fasta)
for ((i=0; i<$NUM_SEQ; i++)); do
  echo "Processing index $i"
  create_individual_features.py \
    --fasta_paths=$FASTA_PATH \
    --data_dir=$DATA_DIR \
    --output_dir=$OUTPUT_DIR \
    --skip_existing=True \
    --use_mmseqs2=True \
    --max_template_date="2050-01-01" \
    --seq_index=$i || echo $i >> failed_indexes.txt

done
```

### JackHMMer (Parallelized)
```bash
MAX_JOBS=16
rm -f $FAILED_LOG

for ((i=0; i<$NUM_SEQ; i++)); do
  (
    create_individual_features.py \
      --fasta_paths=$FASTA_PATH \
      --data_dir=$DATA_DIR \
      --output_dir=$OUTPUT_DIR \
      --skip_existing=True \
      --use_mmseqs2=False \
      --max_template_date="2050-01-01" \
      --seq_index=$i || echo $i >> $FAILED_LOG
  ) &

  while (( $(jobs -r | wc -l) >= MAX_JOBS )); do sleep 1; done

done
wait
```

---

## Understanding `create_individual_features.py`
- CPU-only script
- Output format: Pickle files (`.pkl`)
- Requires pre-downloaded **Genetic Database**
- Feature sources:
  1. JackHMMer/HHblits (default)
  2. MMseqs2
  3. Precomputed Feature Pickles

### Resources
- Genetic DB: https://github.com/KosinskiLab/alphafold#genetic-databases
- Feature DB: https://alphapulldown.s3.embl.de/index.html

---

## Accessing Databases

### Genetic Database
```bash
bash download_all_data.sh /data7/Conny/data/AF_GeneticDB
```

### Feature Pickle Database
```bash
# Install MinIO
curl -O https://dl.min.io/client/mc/release/linux-amd64/mc
chmod +x mc && mkdir -p $HOME/bin && mv mc $HOME/bin/
echo 'export PATH=$HOME/bin:$PATH' >> ~/.bashrc && source ~/.bashrc

# Example download
mc cp embl/alphapulldown/input_features/Saccharomyces_cerevisiae/Q01329.pkl.xz /data7/Conny/data/JackFeaturePickleDB

# Batch download
bash download_found.sh | tee download_output.log
xz -dk *.xz  # decompress while keeping original
```

---

## Multimer Job Execution

Parallel execution across 8 GPUs:
```bash
TARGET_FILES=(
  /data7/Conny/data/target_chunk_aa
  /data7/Conny/data/target_chunk_ab
  ...
  /data7/Conny/data/target_chunk_ah
)

for i in ${!TARGET_FILES[@]}; do
  CUDA_VISIBLE_DEVICES=$i run_multimer_jobs.py \
    --mode=pulldown \
    --monomer_objects_dir=$MONOMER_OBJECTS_DIR \
    --protein_lists=$(realpath $BAIT_FILE),${TARGET_FILES[$i]} \
    --output_path=$BASE_OUTPUT/gpu_$i \
    --data_dir=$DATA_DIR \
    --num_cycle=3 \
    --num_predictions_per_model=1 \
    > $BASE_OUTPUT/gpu_$i/run.log 2>&1 &
done

wait
```

---

## Result Analysis

1. Create Jupyter notebooks using:
```bash
create_notebook.py --cutoff=5.0 --output_dir=/path/to/results
```

2. Visualize outputs (e.g., PAE, ipTM, ranked pdbs)
3. For full table generation:
   - Build `apptainer` image with CCP4 libraries
   - Execute:
```bash
apptainer exec --no-home --bind /results:/mnt fold_analysis_final.sif /app/run_get_good_pae.sh --output_dir=/mnt --cutoff=10
```

---

## AlphaFold3 Notes
- GUI: Automated MSA
- Local mode: Accepts MSA input via JSON
- Additional work needed to automate Selenium upload process (autoclicker test in progress)

---

## Troubleshooting & Exceptions

1. **UniRef30 issue** → Fix with: https://github.com/google-deepmind/alphafold/pull/860/files
2. **MMseqs2:** 5 failures → P40449, P23369, Q12754, Q22354, Q12019
3. **JackHMMer:** 3 known failures → P40328, P48570, P32432
4. **FeatureDB:** 20 missing pickles → cd /data7/Conny/data/JackFeaturePickleDB/missing_pickle_proteins.txt
5. **AF3:** 1 missing pickles due to large size → Job_999


---

This document is actively maintained as experimental runs progress.

