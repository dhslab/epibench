# EpiBench

EpiBench is a software tool designed for predicting DNA methylation levels using genomic sequence data and histone modification marks. It employs a multi-branch Convolutional Neural Network (CNN) architecture (`SeqCNNRegressor`) specifically tailored for integrating these data types to achieve high prediction accuracy.

## Table of Contents

- [Overview](#overview)
  - [Model Input](#model-input)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Basic Usage Examples](#basic-usage-examples)
- [Configuration](#configuration)
- [Orchestration](#orchestration)
- [Environment Validation](#environment-validation)
- [Contributing](#contributing)
- [License](#license)
- [Logging, Run Tracking, and Analysis (New in v1.2)](#logging-run-tracking-and-analysis-new-in-v12)

## Overview

The tool encompasses a pipeline for:
- **Data Processing:** Converting raw genomic data (bed, bigwig) into model-ready matrices.
- **Model Training:** Training the `SeqCNNRegressor` model.
- **Hyperparameter Optimization:** Using Optuna to find optimal model parameters.
- **Evaluation:** Assessing model performance using various regression metrics.
- **Prediction:** Generating methylation predictions on new data.
- **Interpretation:** Understanding model predictions using Integrated Gradients.
- **Comparative Analysis:** Comparing models trained/evaluated on different sample groups.

It is primarily operated via a Command-Line Interface (CLI) (`epibench`).

### Model Input

The `SeqCNNRegressor` model expects input data in HDF5 format, generated by the `epibench process-data` command. Each sample, corresponding to a specific genomic region defined in the input BED file, is represented as a matrix.

- **Shape:** `(sequence_length, num_channels)`
  - `sequence_length`: Typically 10,000 base pairs.
  - `num_channels`: Usually 11 (4 for DNA sequence + 6 for histone marks + 1 for mask), but can vary based on the number of input histone BigWig files.
- **Channels:**
  - **0-3:** One-hot encoded DNA sequence (A, C, G, T).
  - **4-9 (Example):** Normalized histone modification signals (e.g., H3K4me3, H3K27ac). The exact number and order depend on the BigWig files provided during processing.
  - **Last Channel:** A binary mask indicating the boundaries of the original BED region within the fixed-size window (1 inside the region, 0 outside).

The corresponding target methylation value (e.g., beta value) for each region is stored separately within the HDF5 file.

## Installation

1.  **Clone the repository:**
    ```bash
    git clone https://github.com/Bonney96/epibench.git
    cd epibench
    ```
2.  **Set up a Python virtual environment (Recommended):**
    ```bash
    python3 -m venv .venv
    source .venv/bin/activate  # Linux/macOS
    # or ".venv\Scripts\activate" on Windows
    ```
3.  **Install dependencies:**
    ```bash
    pip install -r requirements.txt
    ```
4.  **Install EpiBench in development mode:**
    ```bash
    pip install -e .
    ```
    This allows you to modify the code and have the changes reflected immediately when running the `epibench` command.

## Quick Start

This guide helps you get EpiBench running quickly.

1.  **Install EpiBench:** Follow the steps in the [Installation](#installation) section.

2.  **Verify Installation:** Check that the `epibench` command is available:
    ```bash
    epibench --version
    epibench --help
    ```

3.  **Explore Commands:** Get help for specific subcommands:
    ```bash
    epibench process-data --help
    epibench train --help
    # ... and so on for evaluate, predict, interpret, compare
    ```

4.  **Review Example Configurations:** Examine the files in the `config/` directory (e.g., `config/process_config_example.yaml`, `config/train_config_example.yaml`) to understand the required parameters and structure for different pipeline steps.

5.  **Run a Basic Workflow (Tutorial):** Follow the steps outlined in the [Training and Evaluating a Model Tutorial](docs/tutorial_train_evaluate.md). You'll use the example configuration files, replacing placeholder paths with the actual paths to your data files.

6.  **Run the Orchestration Script:** For a more automated run (after setting up configuration files), use the script:
    ```bash
    # Example for a single sample defined in args
    python scripts/run_full_pipeline.py --output-dir ./quickstart_out --single-sample-name test_sample --process-data-config config/process_config_example.yaml --train-config config/train_config_example.yaml

    # Example using a multi-sample config file
    python scripts/run_full_pipeline.py --output-dir ./quickstart_out --samples-config config/samples_config_example.yaml --max-workers 2
    ```
    *(Remember to replace placeholder paths in the config files with actual paths to your data before running)*.

## Basic Usage Examples

The `epibench` tool utilizes a command-line interface structured around several subcommands. Below are examples for common tasks:

*   **Process Data:** Convert raw data (BED, BigWig, FASTA) into the HDF5 format required by the model.
    ```bash
    epibench process-data --config config/process_config.yaml -o output/processed_data
    ```

*   **Train Model:** Train the `SeqCNNRegressor` using processed data.
    ```bash
    epibench train --config config/train_config.yaml --output-dir output/training_run_01
    ```

*   **Evaluate Model:** Assess the performance of a trained model on test data.
    ```bash
    epibench evaluate --config config/train_config.yaml --checkpoint output/training_run_01/best_model.pth --test-data output/processed_data/test.h5 -o output/evaluation_results
    ```

*   **Generate Predictions:** Use a trained model to predict methylation levels for new input data.
    ```bash
    epibench predict --config config/train_config.yaml --checkpoint output/training_run_01/best_model.pth --input-data data/new_samples.h5 -o output/predictions
    ```

*   **Interpret Model:** Calculate feature attributions (e.g., using Integrated Gradients) to understand which input features (sequence bases, histone marks) contribute most to the model's predictions for specific regions. See the [Interpretation Tutorial](docs/tutorial_interpret.md) for detailed instructions.
    ```bash
    epibench interpret --config config/interpret_config.yaml --checkpoint output/training_run_01/best_model.pth --input-data output/processed_data/interpret_subset.h5 -o output/interpretation_results
    ```
    Example visualization of feature attributions:
    ![Feature Attribution Visualization](docs/images/test_interpret_config_aml_263578_viz_sample_5.png)

*   **Compare Models/Groups:** Perform comparative analyses, such as evaluating model performance differences across various sample groups defined in the configuration.
    ```bash
    epibench compare --config config/compare_config.yaml -o output/comparative_analysis
    ```

## Logging, Run Tracking, and Analysis (New in v1.2)

EpiBench now includes a comprehensive logging system for experiment tracking, reproducibility, and analysis. All pipeline runs are logged with detailed metadata, and you can manage and analyze logs via the CLI:

### List Logs
```bash
epibench logs list --log-dir logs/ --status completed --format table
```

### Show Log Details
```bash
epibench logs show <execution_id> --log-dir logs/ --section all --format rich
```

### Search Logs
```bash
epibench logs search --metric "r_squared>0.9" --config "model.name=SeqCNNRegressor" --format table
```

### Compare Logs
```bash
epibench logs compare <run1> <run2> --focus metrics --format table
```

### Export Logs
```bash
epibench logs export --format csv --output logs_export.csv --fields execution_id mse r_squared
```

### Analyze Logs
```bash
epibench logs analyze --analysis-type summary --metric r_squared --plot
```

See [docs/logging.md](docs/logging.md) for full documentation, schema details, and advanced examples.

## Configuration

EpiBench relies heavily on configuration files (YAML format preferred, JSON also supported) to define parameters for data processing, model architecture, training settings, evaluation metrics, interpretation methods, and comparison setups.

Example configuration files demonstrating required parameters and structure can be found in the `config/` directory.

## Orchestration

For running common end-to-end workflows (e.g., processing, training, evaluating, and predicting sequentially), you can use the provided orchestration scripts located in the `scripts/` directory.

Example:
```bash
python scripts/run_full_pipeline.py --output-dir ./pipeline_runs --samples-config config/samples_to_run.yaml --max-workers 4
```
See `scripts/README.md` for detailed usage of the orchestration scripts.

## Environment Validation

Before running complex pipelines, it's crucial to ensure your environment (Python packages, external tools, environment variables) is set up correctly. EpiBench includes a validation script for this purpose:

```bash
python scripts/check_environment.py
```

This script checks:
- **Python Packages:** Verifies that all packages listed in `requirements.txt` are installed and meet the specified version constraints.
- **External Tools:** Checks for the presence (and optionally, minimum versions) of required external command-line tools in your system's PATH.
- **Environment Variables:** Ensures that necessary environment variables are set.

The script provides detailed error messages and suggestions for fixing any detected issues.

The main orchestration script (`scripts/run_full_pipeline.py`) automatically runs this validation at the beginning. If you need to bypass this check (e.g., in a tightly controlled environment where you are certain of the setup), you can use the `--skip-validation` flag:

```bash
python scripts/run_full_pipeline.py --skip-validation ... [other arguments]
```

## Contributing

Contributions are welcome! Please refer to the contributing guidelines for details on how to submit pull requests, report issues, or suggest enhancements.

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.
