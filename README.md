# Electric Network Frequency Signatures for Semiconductor Chip Geolocation Open Science Artifact

## 1. Overview

This repository accompanies the paper titled **Dude, Where's My Chip? Electric Network Frequency Signatures for Semiconductor Chip Geolocation**, prepared for IEEE Symposium on Security and Privacy (S&P) 2027. It studies power-grid-level chip geolocation through Electric Network Frequency (ENF) signatures sensed from the ambient electromagnetic (EM) emissions of DC-powered boards and compared with mains references.

This repository provides the losslessly compressed experimental data, analysis scripts, and ZedBoard prototype implementation supporting our study of ENF-based chip geolocation. The following sections describe the experimental evaluations, data restoration and reproduction procedures, and corresponding outputs.

## 2. Repository Structure

```text
opensci_repo_enf_chip_geoloc/
    README.md
    exp_validation/
        exp_inputs/
            # 146 losslessly compressed FLAC recordings
        exp_scripts/
        exp_logs/
        exp_results/
    exp_evaluation/
        exp_inputs/
            # 441 losslessly compressed FLAC recordings
        exp_scripts/
        exp_logs/
        exp_results/
    exp_prototype/
        README.md
        LICENSE
        zedboard/
            hardware/
            secure_world/
            normal_world/
            platform/
        reference/
            mains/
            pico/
        verifier/
            scripts/
            keys/
        data.7z.001
        ...
        data.7z.015
        # Created by extracting the split archive:
        data/
            paper_runs/
            expected_correlations.csv
            offline_reproduction_metadata.csv
        reproduce/
            reproduce_paper_results.py
            verification.py
```

- [`exp_validation/`](exp_validation/) contains the losslessly compressed experimental recordings and MATLAB scripts used to **validate the ENF leakage hypothesis** (Section 4). It includes ambient EM measurements from the Sakura-G and ChipWhisperer-305 (CW305) FPGA boards, AC–DC adapter and power-rail measurements, and experiments characterizing ENF recovery near active PCBs. The ENF extraction pipeline is provided through `exp_scripts/proc_enf_analysis.p`.

- [`exp_evaluation/`](exp_evaluation/) contains the losslessly compressed datasets and six MATLAB scripts supporting the **temporal reliability, robustness, grid-level discrimination, and security evaluation** (Section 6). The evaluations examine ENF recovery across different recording times, sensing conditions in a high-density computing environment, discrimination between power grids, and resilience against replay, relay, and synthetic ENF forgery attacks.

- [`exp_prototype/`](exp_prototype/) contains the **end-to-end ENF-based location attestation prototype** implemented on a Zynq-7000 ZedBoard (Section 5). It includes the hardware and TrustZone software implementation for trusted ENF sensing, the reference acquisition services, the verifier, and a split 7-Zip archive containing the recorded prototype dataset. Offline reproduction scripts support verification of saved attestation evidence and reproduction of the reported ENF correlations after the archive is extracted. See the [prototype README](exp_prototype/README.md) for setup and reproduction instructions.

Losslessly compressed input recordings for the MATLAB experiments are organized under each experiment's `exp_inputs/` directory, with the corresponding analysis scripts in `exp_scripts/`. Restore them to WAV before running the scripts, as described below. The `exp_logs/` and `exp_results/` directories contain generated logs and results and may be empty in a source-only release. The cached relay-analysis mode requires previously generated raw-result CSV files.

**Note:** Our core analysis function, originally developed under authors' university policy that restricts source release, is provided here as a compiled MATLAB function (`proc_enf_analysis.p`) rather than in source form. The top-level scripts in this repository call that compiled function so that all analyses in the paper can be reproduced without exposing proprietary internals.

## 3. Software Requirements

### MATLAB Analysis

The MATLAB experiments use **MATLAB R2018b (9.5)**, matching the supplied `proc_enf_analysis.p`. Required toolboxes depend on the experiment:

- **Signal Processing Toolbox:** ENF processing, resampling, and filtering.
- **Image Processing Toolbox:** Spectrum combining and high-resolution ENF extraction.
- **Statistics and Machine Learning Toolbox:** Statistical analyses and hypothesis testing.
- **Parallel Computing Toolbox:** Parallel relay and synthetic-forgery evaluation (serial execution is also supported).

MATLAB requires read/write access to the experiment directories. The original source of `proc_enf_analysis.p` is not required.

### Prototype Offline Reproduction

Requires **Python 3.14.0** and packages listed in [`exp_prototype/requirements.txt`](exp_prototype/requirements.txt). After restoring the archived prototype data, offline reproduction uses NumPy, SciPy, Matplotlib, and cryptography without requiring physical hardware or reference services. See the [prototype README](exp_prototype/README.md#3-dependencies).

### FPGA and Firmware Builds

Rebuilding the ZedBoard implementation requires **Vivado/Vitis 2025.2**, Zynq-7000 support, and the Cortex-A9 toolchain/platform components. See the [prototype build instructions](exp_prototype/README.md#7-vivadovitis-build-instructions). These tools are not required for MATLAB experiments or offline prototype reproduction.

## 4. Restoring Compressed Experimental Data

The evaluation and validation inputs are distributed as losslessly compressed FLAC files. Before running the MATLAB scripts, reviewers must restore the original WAV files. The original directory hierarchy and filenames are preserved, except that the distributed recordings use the `.flac` extension. Restoring the WAV recordings alongside the FLAC files allows the existing MATLAB scripts to run without modification.

Install the FLAC command-line tool and make `flac` available on `PATH`, then run the following from the repository root in Windows PowerShell:

```php
foreach ($dataset in @("exp_validation", "exp_evaluation")) {
    Get-ChildItem ".\$dataset\exp_inputs" -Recurse -File -Filter *.flac |
    ForEach-Object {
        $wav = [IO.Path]::ChangeExtension($_.FullName, ".wav")

        if (-not (Test-Path -LiteralPath $wav)) {
            & flac -d --keep-foreign-metadata --silent `
                "--output-name=$wav" $_.FullName

            if ($LASTEXITCODE -ne 0) {
                throw "FLAC restoration failed: $($_.FullName)"
            }
        }
    }
}
```

FLAC 1.5.0 with `--keep-foreign-metadata` was used and verified for this release. All 587 restored WAV files passed byte-for-byte SHA-256 comparisons against the preserved originals.

The prototype `data/` directory is distributed as the split archive `exp_prototype/data.7z.001` through `exp_prototype/data.7z.015`. All 15 parts must be present in the same directory. From the repository root, extract the archive with 7-Zip (tested with version 24.08):

```arduino
& "C:\Program Files\7-Zip\7z.exe" x `
    ".\exp_prototype\data.7z.001" `
    "-o.\exp_prototype" -y
```

Extraction recreates `exp_prototype/data/`, including the saved prototype runs, expected correlations, and offline metadata required by the offline reproduction script. Restoring the WAV datasets and extracting the prototype archive require additional local disk space; the compressed files remain in place unless removed separately.

## 5. Reproducing the Experiments

Sections 6.1–6.2 provide reproduction instructions for ENF sensing validation (`exp_validation/`), while Sections 6.3–6.8 cover temporal reliability, server-room robustness, grid-level discrimination, and attack evaluation (`exp_evaluation/`). The end-to-end ZedBoard prototype is introduced in Section 7, with detailed reproduction instructions in the [prototype README](exp_prototype/README.md).

Run the MATLAB entrypoints from the repository root using MATLAB R2018b.

## 6. Experimental Reproduction

The repository provides eight MATLAB entrypoints for reproducing the experimental validation and evaluation results. Open MATLAB R2018b, set the Current Folder to the repository root, and run the corresponding commands below. Evaluation figures and numerical output files are saved under exp_evaluation/exp_results/, while execution logs are saved under each experiment's exp_logs/ directory.

### 6.1 Pairwise ENF Analysis

**Script:** [`enf_analysis_top_pair_wise.m`](exp_validation/exp_scripts/enf_analysis_top_pair_wise.m)  
**Paper:** Sections 4.3–4.4

Compares two recordings and visualizes their spectrograms, extracted ENF traces, and Pearson correlation. In `STEP_1`, select the input paths and filenames (without `.wav`) from `exp_validation/exp_inputs/`. The default configuration uses the Sakura-G ambient-EM recordings.

```matlab
run(fullfile(pwd, 'exp_validation', 'exp_scripts', 'enf_analysis_top_pair_wise.m'));
```

**Outputs:** Pearson correlation, interactive figures, and an execution log. Optional chunk processing can be enabled through `SCRP_CFG_DO_CHUNK_PROCESSING`.

### 6.2 Consolidated Validation and Sensing Characterization

**Script:** [`enf_analysis_top_validation_summary.m`](exp_validation/exp_scripts/enf_analysis_top_validation_summary.m)  
**Paper:** Sections 4.3–4.4 and Additional Artifacts A–D/F

Processes the supplied 77 recording comparisons covering ambient-EM sensing, power-rail measurements, adapters, shielding, and probe characterization. The input inventory and ENF extraction configurations are predefined.

```matlab
run(fullfile(pwd, 'exp_validation', 'exp_scripts', 'enf_analysis_top_validation_summary.m'));
```

**Outputs:** Per-comparison correlations and summary statistics in the console and execution log.

### 6.3 Temporal Reliability

**Script:** [`enf_analysis_top_reliability_stat.m`](exp_evaluation/exp_scripts/enf_analysis_top_reliability_stat.m)  
**Paper:** Section 6.1

Analyzes 160 recording pairs from `exp_evaluation/exp_inputs/TREND/` across recording times, days, and weeks. The supplied dataset requires no configuration changes.

```matlab
run(fullfile(pwd, 'exp_evaluation', 'exp_scripts', 'enf_analysis_top_reliability_stat.m'));
```

**Outputs:** Correlation statistics, repeated-measures analyses, and the inferential figure under `exp_results/enf_temporal_reliability/`.

### 6.4 Server-Room Robustness

**Script:** [`enf_analysis_top_reliability_stat_server.m`](exp_evaluation/exp_scripts/enf_analysis_top_reliability_stat_server.m)  
**Paper:** Section 6.2

Evaluates ENF sensing across two sensor placements and recording days using 20 pairs from `exp_evaluation/exp_inputs/SRV_L/`. The supplied dataset requires no configuration changes.

```matlab
run(fullfile(pwd, 'exp_evaluation', 'exp_scripts', 'enf_analysis_top_reliability_stat_server.m'));
```

**Outputs:** Correlation statistics, repeated-measures analyses, and the inferential figure under `exp_results/srv_room_robustness/`.

### 6.5 Multi-Location and Cross-Grid Discrimination

**Script:** [`enf_analysis_top_multi_loc_stat.m`](exp_evaluation/exp_scripts/enf_analysis_top_multi_loc_stat.m)  
**Paper:** Section 6.3

Evaluates intra-grid and inter-grid ENF correlations using the `MULTI/` dataset, containing US (60 Hz) and German (50 Hz) recordings. The script automatically selects the corresponding ENF configurations.

```matlab
run(fullfile(pwd, 'exp_evaluation', 'exp_scripts', 'enf_analysis_top_multi_loc_stat.m'));
```

**Outputs:** Correlation distributions, statistical tests, EER estimates, and figures under `exp_results/multiloc_validation/`.

### 6.6 Replay Attacks

**Script:** [`enf_analysis_top_replay_attacks.m`](exp_evaluation/exp_scripts/enf_analysis_top_replay_attacks.m)  
**Paper:** Section 6.4

Evaluates replay attacks using the `TREND/` dataset with eight authentic targets and 124 replay comparisons. The target selection and detection threshold are predefined.

```matlab
run(fullfile(pwd, 'exp_evaluation', 'exp_scripts', 'enf_analysis_top_replay_attacks.m'));
```

**Outputs:** Replay correlation statistics, detection results, CSV files, and the ENF overlay figure under `exp_results/replay_attack_resilience/`.

### 6.7 Relay Attacks

**Script:** [`enf_analysis_top_relay_attacks.m`](exp_evaluation/exp_scripts/enf_analysis_top_relay_attacks.m)  
**Paper:** Section 6.4

Evaluates relay-induced timing offsets using high-resolution ENF extraction over 160 recording pairs. For full reproduction, retain `config.RUN_ENF_EXTRACTION=true` and `config.RUN_BASELINE=false`. Parallel processing can be disabled with `config.USE_PARALLEL=false`.

```matlab
run(fullfile(pwd, 'exp_evaluation', 'exp_scripts', 'enf_analysis_top_relay_attacks.m'));
```

**Outputs:** Delay estimates, detection thresholds, TPR results, CSV files, and the relay-delay figure under `exp_results/relay_attacks/`.

Setting `config.RUN_ENF_EXTRACTION=false` enables downstream analysis using previously generated raw-result CSV files.

### 6.8 Synthetic ENF Forgery

**Script:** [`enf_analysis_top_syn_forgery_attacks.m`](exp_evaluation/exp_scripts/enf_analysis_top_syn_forgery_attacks.m)  
**Paper:** Section 6.4

Evaluates synthetic ENF forgery against 160 authentic recordings from the `TREND/` dataset. The attack-generation parameters and detection threshold are predefined. Set `config.USE_PARALLEL=false` for serial execution.

```matlab
run(fullfile(pwd, 'exp_evaluation', 'exp_scripts', 'enf_analysis_top_syn_forgery_attacks.m'));
```

**Outputs:** Authentic/forged correlation statistics, detection results, CSV files, and the ENF overlay figure under `exp_results/syn_forgery_attacks/`.

## 7. End-to-End Prototype

The [`exp_prototype/`](exp_prototype/) directory contains the ZedBoard TrustZone implementation, reference acquisition services, verifier, and a split archive of the 15-run experimental dataset. After extraction, its offline reproduction scripts verify saved attestation evidence and reproduce the ENF correlation results.

See the [prototype README](exp_prototype/README.md) for software dependencies, build instructions, and offline reproduction.

## 8. Licensing

The prototype source code and dataset are distributed under the [`exp_prototype/LICENSE`](exp_prototype/LICENSE) MIT License. Bundled third-party components retain their respective licenses and attribution notices, as documented in the [prototype README](exp_prototype/README.md#11-licensing).
