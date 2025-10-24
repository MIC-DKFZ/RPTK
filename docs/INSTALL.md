# INSTALLATION GUIDE

## Overview

This document provides instructions on how to install the necessary dependencies for the **Radiomics Processing Toolkit (RPTK)** and set up the Python environment correctly.

## Prerequisites

- **Conda**: It is recommended to use [Miniforge](https://github.com/conda-forge/miniforge) instead of Miniconda or Anaconda, as it provides better compatibility with Conda-Forge packages.
- **Git**: Recommended for cloning the repository.

## Installation Steps

### Docker execution
Alternatively to installing the tool on your machine, we are offering a Docker image to run independently on your machine. For Docker execution, please see `docs/Docker/README.md` for detailed instructions.

### Recommended Installation Steps

Follow these steps for the recommended installation:

```bash
# Create and move into the parent folder
mkdir ~/RPTK
cd ~/RPTK

# Clone the main RPTK repo (this one)
git clone https://github.com/MIC-DKFZ/rptk.git rptk

# create conda env
conda create -y -n rptk -c conda-forge python=3.10

# Activte env
conda activate rptk

# go into the repo folder
cd rptk

# install packages
pip install -r requirements.txt

# install rptk
pip install -e .

```

#### Verify Installation

Run the following command to check that the installation was successful:

```python
python
import sys
from rptk.rptk import RPTK
```
If no errors appear, the setup is complete!

```bash

python -c "import rptk, mirp; print('RPTK version:', getattr(rptk, '__version__', 'dev')); print('MIRP loaded OK')"

```

#### Use RPTK in Script

```bash

chmod +x rptk-run.sh
# usage (from anywhere after activating the env)
~/RPTK/rptk/rptk-run.sh --input-csv data.csv --output-folder out

```

```python
from rptk.rptk import RPTK

rptk = RPTK(path2confCSV= "/path/to/input.csv", 
             n_cpu = 30,  # number of cpu to use
             input_reformat=True,  # use reformat if the ID is not at the beginning of the files
             out_path="/path/to/output/folder/"
           )

rptk.run()
```

#### Use RPTK API

```bash

python run_rptk.py --input-csv data/input.csv --output-folder results/ --num-cpus 8

```

## Additional Notes

For further assistance, open an issue in the repository.

---

**RPTK - Radiomics Processing Toolkit**

