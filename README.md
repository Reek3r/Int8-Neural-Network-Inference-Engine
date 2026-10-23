# Int8-Neural-Network-Inference-Engine
 a heap-free C++ int8 inference engine on an STM32F401 (Cortex-M4), targeting bit-exact agreement with PyTorch and benchmarking against CMSIS-NN kernels



A heap-free, zero-dependency C++17 INT8 inference engine implemented from scratch on the STM32F401 (ARM Cortex-M4). This project focuses on low-level microarchitectural optimizations, hardware-software co-design, and achieving 100% bit-exact numerical parity with PyTorch's quantization framework (`torch.ao`).

# int8-engine

A from-scratch int8 neural network inference engine in C++ for the Arm Cortex-M4F, benchmarked honestly against Arm's [CMSIS-NN](https://github.com/ARM-software/CMSIS-NN) on identical hardware.

> **Status: Phase 0 (infrastructure). Nothing is benchmarked yet.** This README states what the project will do and how results will be measured. Results get added only when they exist.

## Goal

Build a correct, tested int8 inference engine that runs a small convolutional network on an STM32, then measure where and why Arm's production library outperforms a careful first-principles implementation.

**The goal is not to beat CMSIS-NN.** It is to understand the problem well enough to explain each performance gap precisely: which instruction, which memory access pattern, which layout decision.

## Target hardware

| | |
|---|---|
| Board | STM32 Nucleo-F401RE |
| Core | Arm Cortex-M4F, 84 MHz |
| Memory | 512 KB flash, 96 KB SRAM |
| Why this core | The DSP extension (`SMLAD`, `SMUAD`, `SXTB16`) lets one instruction do two 16-bit multiply-accumulates, which is what makes the optimisation phase meaningful. Cortex-M0/M3 cores do not have it. |

## Design decisions

- **Hardcoded model topology.** No model parser or interpreter. Weights are exported from PyTorch as C headers (int8 arrays plus scales and zero-points). This keeps the engine small and the memory layout visible.
- **Quantisation scheme:** per-tensor, symmetric int8 weights, asymmetric int8 activations, int32 accumulation, requantisation back to int8 with a fixed-point multiplier and shift (no division, no float on the hot path).
- **Correctness is bit-exact.** Reference input/output pairs are generated on the host and every operator is tested against them, both on the host (CI) and on the board.
- **Measurement uses the core's own cycle counter** (`DWT->CYCCNT`), not wall-clock time, so results do not depend on timer setup.

## Models

| Model | Purpose |
|---|---|
| Small CNN on a standard benchmark task (MNIST-scale, or the MLPerf Tiny keyword-spotting model) | Comparison against CMSIS-NN, using a model an outsider can reproduce |
| IMU gesture classifier (MPU-6050) | Live end-to-end demo on the same engine |

## Operators

`conv2d`, `depthwise_conv2d`, `fully_connected`, `relu` (clamp), `max_pool2d`, `softmax`.

## Roadmap

Each phase ends in something that stands on its own.

- [ ] **Phase 0: Infrastructure.** CMake build with `arm-none-eabi-gcc`, CI on GitHub Actions, UART output, cycle counter, one fully-connected layer running on the board and printing a cycle count.
- [ ] **Phase 1: Correctness.** Plain C++ reference operators, validated bit-exact against PyTorch outputs on hardware.
- [ ] **Phase 2: Optimisation.** `SMLAD`-based MAC loops, `im2col`, memory layout and placement. Each change is measured and recorded, including the ones that did not help.
- [ ] **Phase 3: Benchmark against CMSIS-NN.** Per-operator latency, peak SRAM, flash footprint, and a written explanation of each gap.
- [ ] **Phase 4: Upstream.** Any documentation, test or correctness issues found in CMSIS-NN while benchmarking are reported or fixed upstream.

## Benchmark methodology (fixed in advance)

- Same board, same compiler version, same optimisation flags for both implementations.
- Same model weights and same inputs.
- Cycles per operator and per full inference, taken from `DWT->CYCCNT`, median over repeated runs.
- Peak SRAM and flash footprint reported from the linker map, not estimated.
- Outputs of both implementations compared before any timing is trusted.

## Explicitly out of scope (v1)

Training infrastructure, float32/float16 support, architectures other than Cortex-M4/M7, a model parser, arbitrary model support, any GUI or web interface, and beating CMSIS-NN.

## Planned repository layout

```
include/    engine headers
src/        operator implementations
tests/      host-side unit tests (run in CI)
firmware/   board bring-up, UART, cycle counter, linker script
models/     PyTorch training/quantisation scripts, exported weights
docs/       write-ups: quantisation maths, optimisation log, benchmark results
```

## Build

Build instructions will be added when Phase 0 is complete. The build uses CMake and Ninja with `arm-none-eabi-gcc`, and CI runs the same build on every push.

## Notes

This is a learning project. The write-ups in `docs/` aim to explain the reasoning behind each decision, not only record the outcome.

## Licence

To be decided before the first release.
