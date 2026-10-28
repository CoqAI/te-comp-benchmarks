# =============================================================================
# Dockerfile — te-comp memory & accuracy benchmarks artifact
#
# Self-contained environment to reproduce Paper Table 1, Table 2,
# and numerical accuracy verification.
# Uses the pre-built te-comp binary shipped in tools/.
#
# Build context = memory_accuracy_benchmarks/ (self-contained).
#
# Usage:
#   ./docker-build.sh            # build image
#   ./docker-run.sh              # run evaluation
# =============================================================================

FROM ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive

# --- System packages ---------------------------------------------------------
RUN apt-get update && apt-get install -y --no-install-recommends \
        # ARM cross-compiler (cross-compile + arm-none-eabi-size)
        gcc-arm-none-eabi \
        # Python runtime
        python3 python3-pip python3-numpy python3-pil \
        # Host GCC (for verify_accuracy.py: compile engine.c as shared lib)
        gcc \
        # Utilities
        wget curl ca-certificates \
        # The pre-built te-comp binary was built against LLVM with
        # BUILD_SHARED_LIBS=OFF (default), so LLVM/MLIR are linked in.
        # Only standard system libs are needed at runtime.
        libstdc++6 \
    && rm -rf /var/lib/apt/lists/*

# --- Python packages for TFLite/SafeTensors conversion -----------------------
RUN pip3 install --no-cache-dir --break-system-packages \
        tflite==2.10.0 safetensors flatbuffers tflite-runtime

# --- Copy self-contained benchmark directory into /bench/ --------------------
# Build context is memory_accuracy_benchmarks/ (set by docker-build.sh)
COPY . /bench/

# Make the binary executable
RUN chmod +x /bench/tools/te-comp

# Verify the binary runs
RUN /bench/tools/te-comp --help > /dev/null 2>&1 \
    || echo "NOTE: te-comp --help returned non-zero (may need GLIBC >= 2.39)"

WORKDIR /bench
CMD ["./run_evaluation.sh"]
