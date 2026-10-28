# Benchmark Models

## MCUNet Models (MIT HAN Lab)

10 int8 quantized TFLite models from the MCUNet model zoo.

Source: https://github.com/mit-han-lab/mcunet
Paper: "MCUNet: Tiny Deep Learning on IoT Devices" (NeurIPS 2020)

### Download

```bash
# Download all 10 MCUNet models (with SHA256 verification)
./benchmarks/models/download_mcunet.sh

# Verify existing files only
./benchmarks/models/download_mcunet.sh --verify

# Force re-download
./benchmarks/models/download_mcunet.sh --force
```

Or manually via Python:

```python
from mcunet.model_zoo import download_tflite
tflite_path = download_tflite(net_id="mcunet-in0")
```

### ImageNet Models

| net_id | Filename | MACs | #Params | SRAM | Flash | Res. | Top-1 (fp32/int8) |
|--------|----------|------|---------|------|-------|------|--------------------|
| mcunet-in0 | `mcunet-10fps_imagenet.tflite` | 6.4M | 0.75M | 266kB | 889kB | 48 | 41.5%/40.4% |
| mcunet-in1 | `mcunet-5fps_imagenet.tflite` | 12.8M | 0.64M | 307kB | 992kB | 96 | 51.5%/49.9% |
| mcunet-in2 | `mcunet-256kb-1mb_imagenet.tflite` | 67.3M | 0.73M | 242kB | 878kB | 160 | 60.9%/60.3% |
| mcunet-in3 | `mcunet-320kb-1mb_imagenet.tflite` | 81.8M | 0.74M | 293kB | 897kB | 176 | 62.2%/61.8% |
| mcunet-in4 | `mcunet-512kb-2mb_imagenet.tflite` | 125.9M | 1.73M | 456kB | 1876kB | 160 | 68.4%/68.0% |
| mbv2-w0.35 | `mbv2-w0.35-r144_imagenet.tflite` | 23.5M | 0.75M | 308kB | 862kB | 144 | 49.7%/49.0% |
| proxyless-w0.3 | `proxyless-w0.3-r176_imagenet.tflite` | 38.3M | 0.75M | 292kB | 892kB | 176 | 57.0%/56.2% |

### VWW (Visual Wake Words) Models

| net_id | Filename | MACs | #Params | SRAM | Flash | Res. | Top-1 (fp32/int8) |
|--------|----------|------|---------|------|-------|------|--------------------|
| mcunet-vww0 | `mcunet-10fps_vww.tflite` | 6.0M | 0.37M | 146kB | 617kB | 64 | 87.4%/87.3% |
| mcunet-vww1 | `mcunet-5fps_vww.tflite` | 11.6M | 0.43M | 162kB | 689kB | 80 | 88.9%/88.9% |
| mcunet-vww2 | `mcunet-320kb-1mb_vww.tflite` | 55.8M | 0.64M | 311kB | 897kB | 144 | 91.7%/91.8% |

SRAM/Flash numbers are MCUNet's own profiling with TinyEngine on STM32F746.
Int8 models use post-training quantization (not QAT).

### Name Mapping

The tflite filenames use MCUNet's official `net_name` from `model_zoo.py`.
When compiled with te-comp, hyphens and dots are replaced with underscores
to form valid C identifiers:

| MCUNet net_id | tflite filename | C identifier (output dir) |
|---------------|-----------------|---------------------------|
| mcunet-in0 | `mcunet-10fps_imagenet.tflite` | `mcunet_10fps_imagenet` |
| mcunet-in1 | `mcunet-5fps_imagenet.tflite` | `mcunet_5fps_imagenet` |
| mcunet-in2 | `mcunet-256kb-1mb_imagenet.tflite` | `mcunet_256kb_1mb_imagenet` |
| mcunet-in3 | `mcunet-320kb-1mb_imagenet.tflite` | `mcunet_320kb_1mb_imagenet` |
| mcunet-in4 | `mcunet-512kb-2mb_imagenet.tflite` | `mcunet_512kb_2mb_imagenet` |
| mbv2-w0.35 | `mbv2-w0.35-r144_imagenet.tflite` | `mbv2_w0_35_r144_imagenet` |
| proxyless-w0.3 | `proxyless-w0.3-r176_imagenet.tflite` | `proxyless_w0_3_r176_imagenet` |
| mcunet-vww0 | `mcunet-10fps_vww.tflite` | `mcunet_10fps_vww` |
| mcunet-vww1 | `mcunet-5fps_vww.tflite` | `mcunet_5fps_vww` |
| mcunet-vww2 | `mcunet-320kb-1mb_vww.tflite` | `mcunet_320kb_1mb_vww` |

### Download URL Pattern

```
https://hanlab18.mit.edu/projects/tinyml/mcunet/release/{net_name}.tflite
```

where `net_name` is from the `NET_INFO` dict in
[mcunet/model_zoo.py](https://github.com/mit-han-lab/mcunet/blob/main/mcunet/model_zoo.py).

## Custom Models

6 custom int8 TFLite models for MLPerf Tiny benchmarks, created by
`create_models.py` with post-training quantization on random calibration data.

| # | Filename | Description | Input |
|---|----------|-------------|-------|
| 01 | `01_mnist_cnn.tflite` | MNIST digit classification | 28x28x1 |
| 02 | `02_cifar10_cnn.tflite` | CIFAR-10 image classification | 32x32x3 |
| 03 | `03_mobilenet_v1_025.tflite` | MobileNetV1 0.25 (depthwise sep) | 128x128x3 |
| 04 | `04_kws_ds_cnn.tflite` | Keyword Spotting DS-CNN | 49x10x1 |
| 05 | `05_fc_autoencoder.tflite` | FC Autoencoder (anomaly det) | 640 |
| 06 | `06_person_detection.tflite` | Person Detection (MobileNet) | 96x96x3 |
