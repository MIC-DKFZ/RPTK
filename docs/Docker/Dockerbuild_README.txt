
================================================================================
RPTK — REPRODUCIBLE DOCKER BUILD (OFFLINE / PROXY-AWARE)
================================================================================

This document explains how to rebuild the RPTK Docker image in a controlled,
reproducible way — including strictly pinned dependencies, a prebuilt
PyRadiomics wheel (offline), and optional corporate proxy settings.

It also documents how to update Python, PyRadiomics, or the RPTK repo safely.


-------------------------------------------------------------------------------
1) DIRECTORY LAYOUT (BUILD CONTEXT)
-------------------------------------------------------------------------------

Place the following files in ONE folder (the Docker build context):

  project_root/
  ├─ Dockerfile.local                # reproducible build recipe
  ├─ requirements.txt                # all deps EXCEPT pyradiomics
  ├─ rptk/                           # your RPTK package (source)
  ├─ rptk-run.py                     # convenience runner (optional)
  ├─ docker_entrypoint.py            # entrypoint script
  ├─ pyradiomics-3.0.1-<tag>.whl     # prebuilt PyRadiomics wheel (local)
  └─ .dockerignore                   # (recommended) see section 6

Notes
- requirements.txt MUST NOT list "pyradiomics" (the Dockerfile installs the
  local wheel explicitly; we also filter pyradiomics out as an extra guard).
- The wheel filename will look like:
  pyradiomics-3.0.1-cp310-cp310-linux_x86_64.whl
  (your exact tag may differ depending on Python/ABI).

# Rebuilding docker container:
# Create/activate a clean env (Python 3.10):
conda create -y -n rptk -c conda-forge python=3.10
conda run -n rptk python -m pip install --upgrade pip

# Install all project dependencies and RPTK itself into the env:
# From the repo root where requirements.txt + pyproject.toml exist:
conda run -n rptk python -m pip install -r requirements.txt
conda run -n rptk python -m pip install -e ./rptk

# Ensure native lib for python-magic is inside the env:
conda run -n rptk conda install -y -c conda-forge libmagic

# Install a prebuilt pyradiomics wheel (best). If you don’t have it yet, build once:
# Build on a compatible Linux x86_64 with Python 3.10:
python -m venv /tmp/pyrad-v && . /tmp/pyrad-v/bin/activate
python -m pip install --upgrade pip build wheel
python -m pip wheel --no-build-isolation "pyradiomics==3.0.1"
# This produces: ./pyradiomics-3.0.1-cp310-cp310-linux_x86_64.whl
cp pyradiomics-3.0.1-cp310-cp310-linux_x86_64.whl <repo-root>/

# Quick sanity test inside the env:
conda run -n rptk python - <<'PY'
import sys, pandas, psutil, radiomics, rptk
print(sys.executable)
print('pandas=', pandas.__version__, 'radiomics=', radiomics.__version__, 'has RPTK=', hasattr(rptk,'RPTK'))
PY

# Pack the env:
# Replace the absolute conda-pack path if needed (use `which conda-pack` if available)
conda install -n base -c conda-forge conda-pack -y
/home/<you>/anaconda3/bin/conda-pack -n rptk -o rptk_env.tar.gz

# Verify the tarball contains the critical pieces:
tar tzf rptk_env.tar.gz | grep -E 'site-packages/(pandas|psutil|radiomics|rptk)' | head
tar tzf rptk_env.tar.gz | grep -i 'libmagic.so' | head

# Build Docker container
docker build -f Dockerfile.local -t rptk:offline .

# Smoke tests (inside container)
# Import & version check
docker run --rm --entrypoint "" rptk:offline \
  /opt/conda/envs/rptk/bin/python -c "import pandas, psutil, radiomics, rptk; print('OK:', pandas.__version__, radiomics.__version__, hasattr(rptk,'RPTK'))"

# Save DOCKER
 docker save -o rptk_010_docker.tar rptk:offline

# RUN DOCKER
docker run --rm \
  -u $(id -u):$(id -g) \
  --mount type=tmpfs,destination=/workspace/input,tmpfs-mode=1777 \
  --mount type=tmpfs,destination=/workspace/tmp,tmpfs-mode=1777 \
  -v /path/to/data/WORC/CRLM/CRLM:/data \
  rptk:offline \
  --input_csv /data/CRLM_docker_test.csv \
  --output_folder /data/out \
  --num_cpus 8
  
  
# Check
docker run --rm rptk:offline --help
  
  
-------------------------------------------------------------------------------
2) PREREQUISITES
-------------------------------------------------------------------------------

- Docker (root or a compatible setup)
- Internet access for:
  - pulling base image: mambaorg/micromamba:1.5.7
  - or via proxy (see build args below)
- A prebuilt PyRadiomics wheel, created on a compatible host:

  # Example: build a PyRadiomics 3.0.1 wheel for Python 3.10
  python3.10 -m pip install --user virtualenv
  python3.10 -m virtualenv /tmp/pyrad-v --always-copy
  . /tmp/pyrad-v/bin/activate
  python -m pip install --upgrade pip wheel "setuptools<74" "numpy==1.26.4" "versioneer>=0.28"
  python -m pip wheel --no-build-isolation "pyradiomics==3.0.1"
  # Resulting wheel is in the current directory (copy it into project_root/)
  deactivate


-------------------------------------------------------------------------------
3) BUILD THE IMAGE
-------------------------------------------------------------------------------

If you are behind a proxy (DKFZ example), pass proxy build-args:

  sudo docker build \
    -f Dockerfile.local \
    --build-arg BASE_IMAGE=mambaorg/micromamba:1.5.7 \
    -t rptk:local .

If you do NOT need a proxy, omit the --build-arg lines:

  sudo docker build -f Dockerfile.local -t rptk:local .


-------------------------------------------------------------------------------
4) ENTRYPOINT & RUNTIME USAGE
-------------------------------------------------------------------------------

The image is configured to run the entrypoint:

  micromamba run -n rptk python /usr/local/bin/docker_entrypoint.py

Therefore, any arguments you pass to "docker run" are forwarded to
docker_entrypoint.py directly.

Examples:

  # Show help
  docker run --rm rptk:local --help

  # Real run (mount input & output)
  docker run --rm \
    -v /path/to/data:/data \
    -v /path/to/output:/out \
    rptk:local \
    --input_csv /data/patients.csv \
    --output_folder /out \
    --num_cpus 8 \
    --config /data/config.yaml

Temporarily override ENTRYPOINT (for ad-hoc tests inside the env):

  docker run --rm --entrypoint "" rptk:local \
    micromamba run -n rptk python -c "import radiomics; print(radiomics.__version__)"


-------------------------------------------------------------------------------
5) SMOKE TESTS
-------------------------------------------------------------------------------

Quick check the PyRadiomics version:

  docker run --rm --entrypoint "" rptk:local \
    micromamba run -n rptk python -c "import radiomics; print(radiomics.__version__)"
  # Expected: 3.0.1

Check that rptk imports:

  docker run --rm --entrypoint "" rptk:local \
    micromamba run -n rptk python -c "import rptk, importlib; print('rptk OK:', hasattr(rptk, 'RPTK'))"


-------------------------------------------------------------------------------
6) RECOMMENDED .dockerignore
-------------------------------------------------------------------------------

To reduce build context size and speed up builds:

  .git/
  __pycache__/
  **/__pycache__/
  .ipynb_checkpoints/
  **/.ipynb_checkpoints/
  *.pyc
  .DS_Store
  Thumbs.db
  build/
  dist/
  *.egg-info/
  .eggs/
  archive/
  data/
  datasets/
  wandb/
  mlruns/
  # IMPORTANT: keep wheels in context
  !*.whl


-------------------------------------------------------------------------------
7) HOW TO UPDATE SAFELY
-------------------------------------------------------------------------------

A) Update the RPTK source code
   - Replace the "rptk/" folder with new code
   - Rebuild:  sudo docker build -f Dockerfile.local -t rptk:local .

B) Update Python version
   - Edit in Dockerfile.local:  ARG PYVER=3.10  -> e.g., 3.11
   - Rebuild a matching PyRadiomics wheel (same Python version)
   - Replace the wheel file in project_root/
   - Rebuild the image

C) Update PyRadiomics version
   - Build a new wheel on a compatible host:
       python -m pip wheel --no-build-isolation "pyradiomics==<NEW_VERSION>"
   - Replace the wheel file in project_root/
   - Ensure requirements.txt does NOT include "pyradiomics"
   - Rebuild the image

D) Update other dependencies
   - Edit requirements.txt (EXCLUDING pyradiomics)
   - Rebuild the image

Tip: Keep a CHANGELOG note with:
   - PYVER (e.g., 3.10)
   - PyRadiomics version and wheel filename
   - Git commit hash of RPTK used for the image


-------------------------------------------------------------------------------
8) DOCKERFILE NOTES (WHAT IT DOES)
-------------------------------------------------------------------------------

- Base image: mambaorg/micromamba:1.5.7
- Creates env "rptk" with Python ${PYVER}
- Installs libmagic via micromamba (no apt-get needed)
- Pre-installs Numpy (ABI dependency for compiled wheels)
- Installs local PyRadiomics wheel from /tmp (copied in build)
- Installs remaining requirements (pyradiomics line filtered out)
- Installs your package in editable mode (-e /opt/rptk)
- ENTRYPOINT runs docker_entrypoint.py under the "rptk" env

Final ENTRYPOINT in Dockerfile.local:
  ENTRYPOINT ["micromamba","run","-n","rptk","python","/usr/local/bin/docker_entrypoint.py"]


-------------------------------------------------------------------------------
9) TROUBLESHOOTING
-------------------------------------------------------------------------------

A) "Invalid wheel filename (wrong number of parts): 'pyradiomics'"
   - Cause: pip did not see a .whl file (wrong name or empty file).
   - Fix:
       * Ensure the wheel exists in build context:
         ls -l pyradiomics-3.0.1-*.whl
       * Ensure .dockerignore does NOT ignore *.whl
       * In Dockerfile, we now install via the exact glob:
         /tmp/pyradiomics-3.0.1-*.whl
       * Build without cache if necessary: --no-cache

B) Entrypoint vs. ad-hoc commands
   - The entrypoint always runs docker_entrypoint.py.
   - For quick Python one-liners, override ENTRYPOINT:
       docker run --rm --entrypoint "" rptk:local micromamba run -n rptk python -c "..."

C) Proxy issues
   - Verify proxy args; you can also set them in the shell:
       export http_proxy=http://www-int2:3128
       export https_proxy=http://www-int2:3128
   - Use --network=host only if your policy allows.

D) Changing Python or PyRadiomics
   - Always rebuild the PyRadiomics wheel to match the Python version.
   - Replace the wheel in project_root/, then rebuild.


-------------------------------------------------------------------------------
10) VERSION PINNING (REPRODUCIBILITY)
-------------------------------------------------------------------------------

- Python version is pinned via ARG PYVER in Dockerfile.local.
- Numpy is pinned (1.26.4).
- PyRadiomics is pinned by the wheel filename.
- Keep requirements.txt pinned wherever feasible (==).
- Record git commit SHA of the "rptk" folder used for a given image.


-------------------------------------------------------------------------------
11) LICENSE & CREDITS
-------------------------------------------------------------------------------

- Base image: mambaorg/micromamba
- PyRadiomics: Aerts et al., Harvard/MGH

================================================================================
END
================================================================================
