# AlphaFold-Pipeline-for-Bou-Nader-Lab
# AlphaPulldown and AlphaFold3 Protein Interaction Workflow

## Overview
This README documents a complete pipeline for predicting protein–protein interactions (PPI) using AlphaPulldown (built on AlphaFold-Multimer) and AlphaFold3. It includes setup instructions, sequence preparation, feature extraction strategies (HHBlits, MMseqs2), parallelization strategies for large-scale prediction, and downstream analysis including PAE and structure visualization.

The project was executed on a Linux HPC system with Conda-based environments and GPU-based parallel prediction.

---

## Resources and References
- **UniProt Sequences:** https://www.uniprot.org
- **AlphaPulldown:**
  - Paper: https://academic.oup.com/bioinformatics/article/39/1/btac749/6839971
  - Repo: https://github.com/KosinskiLab/AlphaPulldown
- **AlphaFold2 Multimer:**
  - https://github.com/google-deepmind/alphafold
  - https://sbgrid.org/wiki/examples/alphafold2
- **AlphaFold3:**
  - https://github.com/google-deepmind/alphafold3
  - Server: https://alphafoldserver.com/welcome

## 1. Environment Setup
- Create Conda environment with required dependencies:
```bash
conda create -n AlphaPulldown -c omnia -c bioconda -c conda-forge \
  python==3.11 openmm==8.0 pdbfixer==1.9 kalign2 hhsuite hmmer modelcif
source activate AlphaPulldown
```
- Install JAX for GPU acceleration:
```bash
pip install -U "jax[cuda12]"
```

## 2. Genetic Database Download (Required for feature extraction)
Use AlphaFold-provided script:
```bash
bash download_all_data.sh /data7/Conny/alphafold_genetic_db
```
- Troubleshooting:
  - `download_uniref30.sh` sometimes causes incompatibility with AlphaPulldown. Use updated versions: https://github.com/google-deepmind/alphafold/pull/860/files

## 3. Feature Extraction
### Option A: HHBlits (slower)
```bash
create_individual_features.py \
  --fasta_paths=$FASTA_PATH \
  --data_dir=$DATA_DIR \
  --output_dir=$OUTPUT_DIR \
  --use_mmseqs2=False \
  --skip_existing=True \
  --max_template_date=$MAX_TEMPLATE_DATE
```

### Option B: MMseqs2 (faster)
```bash
create_individual_features.py \
  --fasta_paths=$FASTA_PATH \
  --data_dir=$DATA_DIR \
  --output_dir=$OUTPUT_DIR \
  --use_mmseqs2=True \
  --skip_existing=True \
  --max_template_date=$MAX_TEMPLATE_DATE
```
- If file errors (e.g., corrupted sequences like P40449, P23369) occur, remove them from input fasta.

## 4. Parallelized Feature Extraction (JackHMMer or MMseqs2)
```bash
MAX_JOBS=16
for ((i=0; i<$NUM_SEQ; i++)); do
  (
    create_individual_features.py \
      --seq_index=$i \
      ... # same args as above
  ) &
  while (( $(jobs -r | wc -l) >= MAX_JOBS )); do sleep 1; done
done
wait
```
- Controls job concurrency to avoid I/O bottlenecks.

## 5. Using Feature Pickle Database
- Use `check_download_feature_db.py` to check which proteins already exist in EMBL's pickle repo
- Use `mc` (MinIO Client) to download:
```bash
mc cp embl/alphapulldown/input_features/ORG/PROTEIN.pkl.xz /data7/Conny/data/FeaturePickleDB/
xz -dk *.xz  # Decompress while keeping originals
```

## 6. Running Multimer Prediction (Pulldown Mode)
### Single GPU or Small Batch
```bash
run_multimer_jobs.py \
  --mode=pulldown \
  --monomer_objects_dir=... \
  --protein_lists=bait.txt,target.txt \
  --output_path=... \
  --data_dir=... \
  --num_cycle=3 --num_predictions_per_model=1
```

### Parallel (8 GPUs)
```bash
for TARGET_FILE in chunk_aa chunk_ab ...; do
  GPU_ID=... # set per file
  CUDA_VISIBLE_DEVICES=$GPU_ID run_multimer_jobs.py ... &
done
wait
```

## 7. AlphaFold3: Advanced Prediction and Visualization
- Run AF3 locally for MSA feature support: https://github.com/google-deepmind/alphafold3/blob/main/docs/input.md
- Server GUI only supports sequence input
- AF3 Output Files (per complex):
  - `ranked_*.pdb`, `*_PAE_plot.png`, `confidence_model.json`, `timings.json`, etc.

## 8. Postprocessing and Notebook Generation
```bash
create_notebook.py --cutoff=5.0 --output_dir=/data7/Conny/result_mmseqs2/pulldown_results/gpu_0
jupyter-lab output.ipynb
```

## 9. Fold Analysis (Apptainer Sandbox)
- Install Apptainer:
```bash
wget ...
./mconfig --prefix=$HOME/opt/apptainer
make -j 4 && make install
```
- Add CCP4 binaries and run:
```bash
apptainer exec --bind /data:/mnt fold_analysis_final.sif run_get_good_pae.sh --output_dir=/mnt --cutoff=10
```
- Must install `squashfuse` or use sandbox extraction mode

---

## Timeline (Simplified Log)
- **March 24:** Setup MMseqs2 pipeline, identified failing sequences
- **March 25–29:** Parallelized feature extraction across 16 CPUs, managed I/O bottlenecks
- **April 1–3:** Ran multimer jobs on 8 GPUs, analyzed failures, mapped proteins to GPU logs
- **April 6–10:** Filtered broken chains, improved parallel reruns
- **April 12:** Built sandbox Apptainer image for visualization (fold_analysis_final.sif)
- **April 14:** Began visualization metrics on ipTM, pTM, using Jupyter Notebooks

---

## Retrospective Notes
- Parallelism at controlled job slots (not all 1700 at once) greatly reduced system crashes
- MMseqs2 is substantially faster but may silently fail for rare sequences
- Maintaining a `failed_indexes.txt` log and trimming input fasta is essential for rerun stability
- Relying on feature pickle DB (via EMBL MinIO) can skip ~50% of precomputation if used wisely
- AF3 shows promise but has limited flexibility unless run locally with MSA input support

---

## Future Improvements
- Automate the rerun of failed sequences based on `failed_indexes.txt`
- Build dynamic combination generation based on completed results
- Create dashboard summary of pTM/ipTM vs bait–target combinations for ranking
- Develop file consistency checker post-run to catch incomplete output early

---

_End of Readme._

