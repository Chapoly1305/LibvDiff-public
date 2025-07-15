# LibvDiff RTX 4090 Setup Guide

This document provides the correct workflow for setting up and running LibvDiff with RTX 4090 GPU support.

## Prerequisites

- Docker with GPU support (nvidia-docker2)
- NVIDIA RTX 4090 (or compatible GPU)
- NVIDIA drivers 525.60.11 or later
- At least 16GB RAM
- 50GB free disk space
- Internet connection for dataset download

## Quick Start

### Step 1: Build Docker Image
Build the optimized Docker image for RTX 4090:

```bash
docker build -f Dockerfile.rtx4090 -t libvdiff:rtx4090 .
```

**Expected Output:**
- Build completes in ~10-15 minutes
- Final image size: ~8GB
- No build errors

### Step 2: Run Container with Bind Mount
Run the container with GPU support and bind mount for file exchange:

```bash
docker run --gpus all -v "$(pwd):/root/LibvDiff" libvdiff:rtx4090 /bin/bash -c "source /root/miniconda3/bin/activate libvdiff && python -c \"import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0)}')\" && python libvdiff.py -o freetype --cvf --apf -e co"
```

**Expected Output:**
```
==========
== CUDA ==
==========

CUDA Version 12.4.0

Container image Copyright (c) 2016-2023, NVIDIA CORPORATION & AFFILIATES. All rights reserved.

PyTorch: 2.4.1
CUDA: True
GPU: NVIDIA GeForce RTX 4090
load model from /root/LibvDiff/saved/models/Asteria/crossarch_train_100000_1659022264.018625.pt, device is cuda:0
oss:freetype, lib: libfreetype, cvf: True, apf: True, exp:cross_optim

[+] source bin option ARM-O0, target bin option: ARM-O1, true: VER-2-4-0, predict: VER-2-4-0
identify VER-2-4-0-ARM-O0 is VER-2-4-0-ARM-O1, 1.000:   0%|          | 0/20 [00:00<?, ?it/s]

[-] source bin option ARM-O0, target bin option: ARM-O1, true: VER-2-4-1, predict: VER-2-4-0
identify VER-2-4-1-ARM-O0 is VER-2-4-0-ARM-O1, 0.500:   0%|          | 0/20 [00:00<?, ?it/s]

[+] source bin option ARM-O0, target bin option: ARM-O1, true: VER-2-4-2, predict: VER-2-4-2
identify VER-2-4-2-ARM-O0 is VER-2-4-2-ARM-O1, 0.667:   0%|          | 0/20 [00:00<?, ?it/s]

... (continues with version identification results)

identify VER-2-8-1-ARM-O3 is VER-2-8-1-ARM-O2, 0.825: 100%|██████████| 20/20 [00:00<00:00, 61.09it/s]
Finished, precision:0.825
```

### Step 3: Interactive Container Usage
For interactive use with persistent container:

```bash
# Start interactive container
docker run --gpus all -it -v "$(pwd):/root/LibvDiff" --name libvdiff-interactive libvdiff:rtx4090 /bin/bash

# Inside container, activate environment
source /root/miniconda3/bin/activate libvdiff

# Verify setup
python -c "import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0)}')"

# Run LibvDiff tests
python libvdiff.py -o freetype --cvf --apf -e co
```

## Key Configuration Details

### Dockerfile.rtx4090 Specifications
- **Base Image:** `nvidia/cuda:12.4.0-devel-ubuntu20.04`
- **PyTorch:** 2.4.1 with CUDA 12.4 support
- **Python:** 3.8 in conda environment
- **Key Dependencies:** Fixed charset-normalizer==3.3.2 for compatibility

### GPU Requirements
- **CUDA Version:** 12.4.0
- **PyTorch CUDA:** 12.4 (compatible with RTX 4090)
- **Memory:** Utilizes GPU memory efficiently
- **Performance:** Achieves ~80-85% precision on freetype dataset

## LibvDiff Command Options

### Basic Usage
```bash
python libvdiff.py -o <OSS> --cvf --apf -e <EXPERIMENT>
```

### Parameters
- `-o, --oss`: OSS to test (freetype, openssl, zlib, etc.)
- `--cvf`: Enable Candidate Version Filter
- `--apf`: Enable Anchor Path Filter  
- `-e, --exp`: Experiment type
  - `co`: Cross-optimization
  - `ca`: Cross-architecture
  - `cb`: Cross-both

### Example Commands
```bash
# Cross-optimization test with freetype
python libvdiff.py -o freetype --cvf --apf -e co

# Cross-architecture test with openssl
python libvdiff.py -o openssl --cvf --apf -e ca

# Cross-both test with zlib
python libvdiff.py -o zlib --cvf --apf -e cb
```

## Expected Performance Results

### Freetype Cross-Optimization (co)
- **Final Precision:** ~82.5%
- **Processing Time:** ~2-3 minutes
- **GPU Utilization:** High during inference
- **Memory Usage:** <4GB GPU memory

### Output Interpretation
- **[+]**: Correct version prediction
- **[-]**: Incorrect version prediction
- **Progress bars:** Show processing across version pairs
- **Final precision:** Overall accuracy percentage

## Troubleshooting

### Common Issues

**1. charset-normalizer Import Error**
- **Solution:** Already fixed in Dockerfile.rtx4090 with version 3.3.2

**2. PyTorch CUDA Version Mismatch**
- **Issue:** Using wrong CUDA version for PyTorch
- **Solution:** Dockerfile uses consistent CUDA 12.4 throughout

**3. Container Permission Issues**
- **Solution:** Use bind mount with `$(pwd):/root/LibvDiff`

**4. GPU Not Detected**
```bash
# Verify NVIDIA Docker support
docker run --rm --gpus all nvidia/cuda:12.4-base nvidia-smi
```

**5. Out of Memory Errors**
- **Solution:** RTX 4090 has sufficient memory; check other GPU processes

## Container Management

### Cleanup Commands
```bash
# Remove containers
docker rm libvdiff-interactive libvdiff-test

# Remove image (if needed)
docker rmi libvdiff:rtx4090

# System cleanup
docker system prune -f
```

### Persistent Data
- All project files are bind-mounted from host
- Model files and datasets persist in host directory
- Container can be removed/recreated without data loss

## Validation Steps

### 1. Verify PyTorch and CUDA
Expected output when running the verification command:
```
PyTorch: 2.4.1
CUDA: True
GPU: NVIDIA GeForce RTX 4090
```

### 2. Verify LibvDiff Model Loading
Expected output from model loading:
```
load model from /root/LibvDiff/saved/models/Asteria/crossarch_train_100000_1659022264.018625.pt, device is cuda:0
```

### 3. Verify Precision Results
Expected final precision for freetype cross-optimization: **0.825 (82.5%)**

This indicates LibvDiff is correctly:
- Loading the pre-trained Asteria model
- Using GPU acceleration (cuda:0) 
- Processing binary features with CVF and APF filters
- Achieving research-grade accuracy on version identification

## File Exchange with Bind Mount

The bind mount setup allows seamless file exchange between host and container:

### Host to Container
- Edit files on host system with your preferred editor
- Changes are immediately visible inside container
- No need to rebuild container for code changes

### Container to Host  
- Results and logs created in container appear on host
- Model outputs and analysis persist after container stops
- Easy backup and version control of all work

### Example Usage
```bash
# On host: Edit LibvDiff source code
vim libvdiff.py

# In container: Changes are immediately available
python libvdiff.py -o freetype --cvf --apf -e co

# Results save to host automatically
ls -la saved/results/
```