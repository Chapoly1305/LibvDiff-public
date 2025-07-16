# LibvDiff RTX 4090 Setup and Usage Guide

This guide provides a complete workflow for setting up LibvDiff with RTX 4090 GPU support and processing projects including the Matter dataset.

## Prerequisites

- Docker with GPU support (nvidia-docker2)
- NVIDIA RTX 4090 (or compatible GPU)
- NVIDIA drivers 525.60.11 or later
- At least 16GB RAM
- 50GB free disk space
- Internet connection for dataset download

## Environment Setup

### Step 1: Build Docker Image

Build the optimized Docker image for RTX 4090:

```bash
docker build -f Dockerfile.rtx4090 -t libvdiff:rtx4090 .
```

**Expected Output:**
- Build completes in ~10-15 minutes
- Final image size: ~8GB
- No build errors

### Step 2: Quick Test Run

Verify the environment with a quick test:

```bash
docker run --gpus all -v "$(pwd):/root/LibvDiff" libvdiff:rtx4090 /bin/bash -c "source /root/miniconda3/bin/activate libvdiff && python -c \"import torch; print(f'PyTorch: {torch.__version__}'); print(f'CUDA: {torch.cuda.is_available()}'); print(f'GPU: {torch.cuda.get_device_name(0)}')\" && python libvdiff.py -o freetype --cvf --apf -e co"
```

**Expected Output:**
```
PyTorch: 2.4.1
CUDA: True
GPU: NVIDIA GeForce RTX 4090
load model from /root/LibvDiff/saved/models/Asteria/crossarch_train_100000_1659022264.018625.pt, device is cuda:0
...
Finished, precision: 0.825
```

### Step 3: Interactive Container Setup

For development and processing work:

```bash
# Start interactive container
docker run --rm --gpus all -it -v "$(pwd):/root/LibvDiff" --name libvdiff-interactive libvdiff:rtx4090 /bin/bash

# Inside container, activate environment
source /root/miniconda3/bin/activate libvdiff

# Configure environment variables
export IDA_HOME=/root/LibvDiff/ida-classroom-free-9.1
export IDAUSR=/root/LibvDiff/.idapro
```

## General LibvDiff Usage

### Basic Commands

```bash
# Cross-optimization test
python libvdiff.py -o <project> --cvf --apf -e co

# Cross-architecture test  
python libvdiff.py -o <project> --cvf --apf -e ca

# Cross-both test
python libvdiff.py -o <project> --cvf --apf -e cb
```

### Available Projects
- `freetype` - Font rendering library
- `openssl` - Cryptography library
- `zlib` - Compression library
- `matter` - IoT connectivity standard (requires special processing)

### Command Parameters
- `-o, --oss`: Project to test
- `--cvf`: Enable Candidate Version Filter
- `--apf`: Enable Anchor Path Filter
- `-e, --exp`: Experiment type (co/ca/cb)

## Matter Project Processing

The Matter project requires special preprocessing due to its complex structure with 2,740+ object files.

### Directory Structure Required

```
LibvDiff-public/
├── data_process/
│   ├── dataset/matter/matter/ARM/
│   │   ├── O2/
│   │   │   ├── v1.0.0/ (273 .o files)
│   │   │   ├── v1.1.0/ (257 .o files)
│   │   │   ├── v1.2.0/ (269 .o files)
│   │   │   ├── v1.3.0/ (280 .o files)
│   │   │   └── v1.4.0/ (291 .o files)
│   │   └── Os/
│   │       ├── v1.0.0/ through v1.4.0/
│   └── features/matter/matter-code/ (git repository)
└── ida-classroom-free-9.1/
```

### Processing Steps

#### Step 1: Combine Object Files (Host System)

```bash
cd ~/LibvDiff-public/data_process

# Combine all individual .o files into single combined .o files per version
python combine_object_files.py

# Verify combined files were created (should show 10)
find dataset/matter -name "*combined.o" | wc -l
```

#### Step 2: Docker Environment Setup

```bash
# Inside Docker container
cd /root/LibvDiff/data_process

# Fix git repository ownership
git config --global --add safe.directory /root/LibvDiff/data_process/features/matter/matter-code
```

#### Step 3: Clean Previous Attempts

```bash
# Remove any old feature files to avoid conflicts
find dataset/matter/matter/ARM -name "*.json" -delete
find dataset/matter -name "*.pkl" -delete
find dataset/matter -name "*.i64" -delete
```

#### Step 4: Generate Binary Features

```bash
# Extract binary features using IDA Pro (processes 10 combined files)
python feature_generator.py -o matter
```

**Expected Output:**
```
Start feature generation
Generating software level features...
gen gen_sl_feature: 100%|██████████| 10/10 [XX:XX<XX:XX, X.XXit/s]
Generating asteria features...
gen gen_asteria_feature: 100%|██████████| 10/10 [XX:XX<XX:XX, X.XXit/s]
...
Feature generation finished
```

#### Step 5: Generate Function Embeddings

```bash
# Generate Asteria embeddings using GPU
python feat_encoding.py -o matter
```

#### Step 6: Generate Version Differences

```bash
# Generate version differences from source code analysis
python vdcs_generator.py -o matter
```

#### Step 7: Generate Version Coordinate System

```bash
# Generate version coordinates
python vct_generator.py -o matter
```

#### Step 8: Test Matter Performance

```bash
# Test cross-optimization version identification
cd /root/LibvDiff
python libvdiff.py -o matter --cvf --apf -e co
```

## Performance Expectations

### General Projects (freetype, openssl, zlib)
- **Processing Time:** 2-5 minutes
- **Precision:** 80-85%
- **GPU Memory:** <4GB

### Matter Project
- **Initial Processing:** 30-45 minutes (one-time setup)
- **Testing:** 5-10 minutes
- **Expected Precision:** >60% (improvement from 20% with individual files)
- **File Reduction:** 2,740 → 10 files (~95% reduction)

## Troubleshooting

### Environment Issues

**1. charset-normalizer Import Error**
- **Solution:** Fixed in Dockerfile.rtx4090 with version 3.3.2

**2. PyTorch CUDA Version Mismatch**
- **Verification:** 
  ```bash
  python -c "import torch; print(torch.cuda.is_available(), torch.version.cuda)"
  ```

**3. GPU Not Detected**
```bash
# Verify NVIDIA Docker support
docker run --rm --gpus all nvidia/cuda:12.4-base nvidia-smi
```

### Matter-Specific Issues

**4. ARM Linker Not Found**
```bash
# Install ARM toolchain (in container)
apt-get update && apt-get install -y gcc-arm-none-eabi
```

**5. IDA Pro Python Errors**
```bash
# Reconfigure Python for IDA Pro
cd /root/LibvDiff/ida-classroom-free-9.1
./idapyswitch --force-path /root/miniconda3/envs/libvdiff/lib/libpython3.8.so
```

**6. Git Repository Ownership**
```bash
git config --global --add safe.directory /root/LibvDiff/data_process/features/matter/matter-code
```

### Container Management

**7. Container Cleanup**
```bash
# Remove containers
docker rm libvdiff-interactive

# Remove image (if needed)  
docker rmi libvdiff:rtx4090

# System cleanup
docker system prune -f
```

## File Exchange and Persistence

The bind mount setup (`-v "$(pwd):/root/LibvDiff"`) enables:

- **Host to Container:** Edit files on host, changes visible immediately in container
- **Container to Host:** Results and logs persist on host after container stops
- **No Rebuild Needed:** Code changes don't require container rebuilding

### Example Workflow
```bash
# On host: Edit source code
vim libvdiff.py

# In container: Run immediately with changes
python libvdiff.py -o freetype --cvf --apf -e co

# Results automatically save to host
ls -la saved/results/
```

## Key Configuration Details

### Docker Specifications
- **Base Image:** `nvidia/cuda:12.4.0-devel-ubuntu20.04`
- **PyTorch:** 2.4.1 with CUDA 12.4 support
- **Python:** 3.8 in conda environment
- **IDA Pro:** Free version 9.1 for binary analysis

### Hardware Requirements
- **CUDA Version:** 12.4.0
- **GPU Memory:** RTX 4090 (24GB) recommended
- **System RAM:** 16GB minimum
- **Storage:** 50GB for datasets and models

## Quick Reference

### Essential Commands
```bash
# Build environment
docker build -f Dockerfile.rtx4090 -t libvdiff:rtx4090 .

# Start interactive session
docker run --rm --gpus all -it -v "$(pwd):/root/LibvDiff" libvdiff:rtx4090 /bin/bash
source /root/miniconda3/bin/activate libvdiff

# Test general project
python libvdiff.py -o freetype --cvf --apf -e co

# Process Matter (one-time setup)
cd data_process
python combine_object_files.py
python feature_generator.py -o matter
python feat_encoding.py -o matter
python vdcs_generator.py -o matter
python vct_generator.py -o matter

# Test Matter
cd /root/LibvDiff
python libvdiff.py -o matter --cvf --apf -e co
```

This unified approach provides both quick testing capabilities for standard projects and comprehensive processing workflows for complex datasets like Matter.
