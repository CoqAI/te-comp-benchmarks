# Memory & Accuracy Benchmarks: te-comp CNN Evaluation

Artifact for reproducing the CNN memory evaluation results in:

> **"te-comp: A Complete MLIR Compiler Generating Self-Contained C Inference
> Engines for Microcontrollers"**

## Contents

```
memory_accuracy_benchmarks/              # <-- self-contained, this is the repo root
├── tools/
│   ├── te-comp                          # Pre-built compiler binary (x86-64 Linux)
│   ├── conversion/
│   │   ├── tflite_to_tosa.py            # TFLite -> TOSA MLIR converter
│   │   └── safetensors_to_tosa.py       # SafeTensors -> TOSA MLIR converter
│   └── codegen/
│       ├── generate_rtos.py             # RTOS glue code generator
│       ├── manifest_schema.py           # Manifest JSON schema
│       └── templates/                   # FreeRTOS and bare-metal templates
├── scripts/
│   ├── generate_outputs.sh              # Run te-comp pipeline on all models
│   ├── suggest_optimizations.sh         # Run auto-optimization analysis
│   └── create_models.py                 # Create custom CNN models (TF/Keras)
├── harness/
│   ├── build_benchmarks.sh              # Generate benchmark firmware for boards
│   ├── generate_test_data.py            # Generate test input data arrays
│   ├── benchmark_main.c.tpl             # Benchmark main template (DWT cycle counter)
│   ├── linker/
│   │   ├── stm32g474re.ld               # NUCLEO-G474RE linker script
│   │   └── s32k344.ld                   # S32K3X4EVB-T172 linker script
│   └── startup/
│       └── startup_cm.c                 # Minimal Cortex-M startup code
├── models/
│   ├── download_mcunet.sh               # Download all 10 MCUNet TFLite models
│   └── README.md                        # Model descriptions and name mapping
├── test_samples/                        # Pre-saved real test inputs + ground-truth labels
│   ├── {model}.npy                      # INT8 input arrays (N, H, W, C)
│   └── {model}_labels.npy               # Ground-truth class labels
├── verify_accuracy.py                   # Numerical verification: te-comp C vs TFLite
├── run_evaluation.sh                    # Master script: reproduce all paper results
├── Dockerfile                           # Docker environment (Ubuntu 22.04 + ARM toolchain)
├── docker-build.sh                      # Build Docker image
├── docker-run.sh                        # Run evaluation in Docker
├── COMPARISON.md                        # Reference comparison data
└── README.md                            # This file
```

## Prerequisites

- **Linux x86-64** (tested on Ubuntu 24.04)
- **arm-none-eabi-gcc** >= 10.0: `sudo apt install gcc-arm-none-eabi`
- **Python 3.6+** with numpy: `pip install numpy`
- **tensorflow** or **tflite-runtime** (for accuracy verification): `pip install tflite-runtime`
- **wget** or **curl** (for downloading MCUNet models)

## Quick Start: Docker (Recommended)

The easiest way to reproduce results is with Docker — it sets up Ubuntu 24.04
with `gcc-arm-none-eabi` and all Python dependencies, using the pre-built
`te-comp` binary shipped in `bin/`:

```bash
cd memory_accuracy_benchmarks

# Build the image (< 5 min, no compilation needed)
./docker-build.sh

# Run the full evaluation
./docker-run.sh

# Or: interactive shell inside the container
./docker-run.sh --shell

# Or: save results to host
./docker-run.sh --output ./results
```

## Quick Start: Native

```bash
cd memory_accuracy_benchmarks
./run_evaluation.sh
```

This will:
1. Download all 10 MCUNet TFLite models (with SHA-256 verification)
2. Run the te-comp pipeline on all models (TOSA -> te -> optimizations -> C)
3. Cross-compile generated C with arm-none-eabi-gcc
4. Print the SRAM/Flash comparison table (Paper Table 1)
5. Print the board fit analysis (Paper Table 2)
6. Run the auto-optimization suggestion (Paper Section 4.6)
7. Verify numerical accuracy of generated C vs TFLite interpreter
8. Generate benchmark test data for on-device measurement

## Accuracy Verification

The `verify_accuracy.py` script compiles the generated C engine as a shared
library on the host, feeds the same inputs to both TFLite and the engine, and
compares output tensors element-by-element:

```bash
# Verify all models (random inputs)
python3 verify_accuracy.py

# Single model, verbose per-input output
python3 verify_accuracy.py --model mcunet_10fps_imagenet -v

# Real ImageNet validation images
python3 verify_accuracy.py --imagenet-dir /path/to/ILSVRC2012_img_val/

# Allow ±1 LSB tolerance (INT8 rounding)
python3 verify_accuracy.py --atol 1

# Write JSON report
python3 verify_accuracy.py --json results.json
```

Metrics reported per model:
- **Max Absolute Error (LSB)**: worst-case element-wise int8 deviation
- **RMSE (int8 / float)**: RMS of raw and dequantized differences
- **Top-1 Match Rate**: fraction of inputs where argmax agrees
- **Element Error Rate**: fraction of output elements that differ

## Step-by-Step Reproduction

### 1. Download Models

```bash
# Download all 10 MCUNet models
bash models/download_mcunet.sh

# Verify checksums
bash models/download_mcunet.sh --verify
```

Models are downloaded from https://hanlab18.mit.edu/projects/tinyml/mcunet/release/
(official MIT HAN Lab release, same files used in the MCUNet paper).

### 2. Run te-comp Pipeline

```bash
# Single model
tools/te-comp \
    --convert-tosa-to-te \
    --te-flash-weights \
    --te-in-place-depthwise \
    --te-operator-specialize \
    --te-memory-plan \
    --convert-te-to-loops \
    --convert-arm-dsp-to-llvm \
    --emit-c --rtos baremetal \
    -o output/mcunet_10fps_imagenet/ \
    models/mcunet-10fps_imagenet.tflite

# All models at once
bash scripts/generate_outputs.sh
```

Output for each model:
- `{model}_engine.c` — complete inference engine (kernels + weights)
- `{model}_engine.h` — public header
- `inference_task.c` — RTOS integration (bare-metal or FreeRTOS)
- `inference_api.h` — 3-function API
- `CMakeLists.txt` — drop-in CMake library
- `manifest.json` — machine-readable metadata

### 3. Cross-Compile and Measure Sizes (Table 1)

```bash
# Cross-compile a single model
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfloat-abi=soft -Os \
    -ffunction-sections -fdata-sections \
    -c output/mcunet_10fps_imagenet/mcunet_10fps_imagenet_engine.c \
    -o /tmp/model.o -I output/mcunet_10fps_imagenet/

# Read sizes
arm-none-eabi-size /tmp/model.o
#   text    data     bss     dec     hex filename
# 842372       0   46080  888452   d8f04 /tmp/model.o
#
# .text = Flash (code + weights in .rodata) = 823 kB
# .bss  = SRAM (arena)                      =  45 kB
```

### 4. Auto-Optimization Analysis (Section 4.6)

```bash
tools/te-comp --te-suggest-optimizations \
    --sram-limit 131072 --flash-limit 524288 \
    models/mcunet-10fps_imagenet.tflite
```

### 5. On-Device Benchmark (requires hardware)

```bash
# Generate test data + benchmark mains
bash harness/build_benchmarks.sh

# This produces:
#   harness/test_data/test_data_{model}.h    — test inputs
#   harness/generated/benchmark_main_{model}_{board}.c

# To run on NUCLEO-G474RE:
arm-none-eabi-gcc -mcpu=cortex-m4 -mthumb -mfloat-abi=soft -Os \
    -T harness/linker/stm32g474re.ld -specs=nosys.specs \
    harness/generated/benchmark_main_mcunet_10fps_vww_g474re.c \
    output/mcunet_10fps_vww/mcunet_10fps_vww_engine.c \
    harness/startup/startup_cm.c \
    -I output/mcunet_10fps_vww/ -I harness/test_data/ \
    -o firmware.elf -lm -lnosys

# Flash and read UART for DWT cycle counts
```

### Boards

| Board | MCU | Core | SRAM | Flash | Use |
|-------|-----|------|------|-------|-----|
| STM32 NUCLEO-G474RE | STM32G474RE | Cortex-M4F @ 170 MHz | 128 KB | 512 KB | CNN |
| NXP S32K3X4EVB-T172 | S32K344 | Cortex-M7 @ 160 MHz | 512 KB | 4 MB | LLM |

## Paper Claims and How to Verify

| Claim | How to verify | Script |
|-------|---------------|--------|
| Beats MCUNet on all 10 models (SRAM) | Compare .bss vs MCUNet reference | `run_evaluation.sh` Step 3 |
| Beats MCUNet on all 10 models (Flash) | Compare .text vs MCUNet reference | `run_evaluation.sh` Step 3 |
| Self-contained C code, zero deps | Inspect `output/{model}/` — no external includes | Manual inspection |
| Bit-exact or near-exact output vs TFLite | Compare C engine output vs TFLite interpreter | `verify_accuracy.py` |
| 6/16 models fit G474RE | Cross-compile with linker script | `run_evaluation.sh` Step 4 |
| All 16 models fit S32K344 | Cross-compile with linker script | `run_evaluation.sh` Step 4 |
| LLM on S32K344 | See `../tinystories_v2/` S32DS project | Build with S32DS IDE |

## Model Name Mapping

The MCUNet TFLite files use official names from `mcunet.model_zoo`. When
compiled, hyphens/dots become underscores for valid C identifiers:

| MCUNet net_id | tflite filename | C identifier |
|---------------|-----------------|--------------|
| mcunet-in0 | mcunet-10fps_imagenet.tflite | mcunet_10fps_imagenet |
| mcunet-in1 | mcunet-5fps_imagenet.tflite | mcunet_5fps_imagenet |
| mcunet-in2 | mcunet-256kb-1mb_imagenet.tflite | mcunet_256kb_1mb_imagenet |
| mcunet-in3 | mcunet-320kb-1mb_imagenet.tflite | mcunet_320kb_1mb_imagenet |
| mcunet-in4 | mcunet-512kb-2mb_imagenet.tflite | mcunet_512kb_2mb_imagenet |
| mbv2-w0.35 | mbv2-w0.35-r144_imagenet.tflite | mbv2_w0_35_r144_imagenet |
| proxyless-w0.3 | proxyless-w0.3-r176_imagenet.tflite | proxyless_w0_3_r176_imagenet |
| mcunet-vww0 | mcunet-10fps_vww.tflite | mcunet_10fps_vww |
| mcunet-vww1 | mcunet-5fps_vww.tflite | mcunet_5fps_vww |
| mcunet-vww2 | mcunet-320kb-1mb_vww.tflite | mcunet_320kb_1mb_vww |
