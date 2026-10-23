# Int8-Neural-Network-Inference-Engine

A from-scratch int8 neural network inference engine in C++ for Arm Cortex-M, built and benchmarked on an STM32 Nucleo-F401RE and measured against Arm's [CMSIS-NN](https://github.com/ARM-software/CMSIS-NN) on identical hardware.

The goal is not to beat CMSIS-NN. It is to build a correct, tested first-principles implementation, then measure where and why a production library outperforms it, and document the reasons.

## Status

| Phase | Scope | State |
|---|---|---|
| 0 | Infrastructure: CMake build, CI, cycle counter, UART output, one fully-connected layer on hardware | In progress |
| 1 | Correctness: reference operators, bit-exact against PyTorch | Planned |
| 2 | Optimisation: Cortex-M4 DSP instructions, im2col, memory layout | Planned |
| 3 | Benchmark against CMSIS-NN: latency, SRAM, flash | Planned |
| 4 | Upstream contributions to CMSIS-NN found during benchmarking | Planned |

Each phase ends in something publishable. Results are added to this README only once the corresponding phase is complete.

## Why this project

Running a neural network on a microcontroller is a memory and cycle budgeting problem. Weights live in read-only flash, intermediate activations live in a small RAM, and every operator competes for both. Int8 quantisation cuts weight storage by 4x against float32 and lets the CPU do more multiply-accumulates per instruction, at the cost of precision that has to be managed carefully.

This repository works through that problem end to end: define a quantisation scheme, implement the operators, prove they are correct, optimise them for the target CPU, and compare the result against the library the industry uses.

## Hardware

| | |
|---|---|
| Board | STM32 Nucleo-F401RE |
| Core | Arm Cortex-M4F, 84 MHz |
| Memory | 512 KB flash, 96 KB SRAM |
| Notable features | DSP extension (SMLAD, SMUAD, SXTB16), single-precision FPU, no cache |

The DSP extension is what makes Phase 2 meaningful. It performs two 16-bit multiply-accumulates in a single instruction.

## Toolchain

- `arm-none-eabi-gcc` (Arm GNU Toolchain)
- CMake and Ninja
- OpenOCD or ST-Link tools for flashing
- GitHub Actions for CI (firmware build and host-side unit tests)
- PyTorch on the host for model definition, training, quantisation and reference output generation

The build is CMake-based rather than IDE-based so that the same build runs locally and in CI.

## Models

| Model | Purpose |
|---|---|
| Small CNN, MNIST-scale or the MLPerf Tiny keyword-spotting model | Benchmark model. Standard models allow comparison against published CMSIS-NN figures. |
| IMU gesture classifier on an MPU-6050 | Demo model. Same engine with live sensor input. |

Models are trained and quantised in PyTorch, then exported as C headers containing int8 weight arrays with per-tensor scale and zero-point values. Reference input/output pairs are exported alongside them for bit-exactness testing.

## Quantisation scheme

- Per-tensor quantisation
- Symmetric int8 weights
- Asymmetric int8 activations
- int32 accumulation
- Requantisation from int32 to int8 using a fixed-point multiplier and shift, avoiding division and floating point at inference time

## Operators

Implemented in Phase 1 as clear reference code, then optimised in Phase 2:

- `conv2d`
- `depthwise_conv2d`
- `fully_connected`
- `relu` (clamp)
- `max_pool2d`
- `softmax`

## Methodology

Correctness and performance claims in this repository follow these rules.

- **Bit-exactness.** Every operator is tested against PyTorch reference outputs. A test passes only if the int8 outputs match exactly.
- **Cycle counts.** Latency is measured in CPU cycles using the DWT cycle counter (`DWT->CYCCNT`), not wall-clock time.
- **Like-for-like comparison.** The engine and CMSIS-NN are built with the same compiler, flags and optimisation level, run on the same board at the same clock, using the same model and weights.
- **Memory.** Peak SRAM and flash footprint are reported separately.
- **Reproducibility.** Build flags, compiler version and board configuration are recorded alongside every published result.

## Scope

In scope:

- Cortex-M4 on the STM32F401RE
- Hard-coded model topologies with generated weight headers
- Int8 inference only

Out of scope for v1:

- Training infrastructure
- Float32 or float16 inference
- A model parser or interpreter
- Support for arbitrary models
- Other architectures
- Beating CMSIS-NN

## Repository layout

Planned structure, populated phase by phase.

```
.
├── CMakeLists.txt
├── cmake/              toolchain file for arm-none-eabi
├── include/            public headers
├── src/                operator implementations
├── board/              startup code, linker script, UART, cycle counter
├── tests/              host-side unit tests and reference vectors
├── models/             PyTorch scripts, exported weight headers
├── bench/              benchmark harness and recorded results
├── docs/               design notes and write-ups
└── .github/workflows/  CI
```

## Build

Build instructions will be added at the end of Phase 0.

## Results

Results will be published here after Phase 3, with the full methodology and raw numbers.

## References

- [CMSIS-NN](https://github.com/ARM-software/CMSIS-NN), Arm's optimised neural network kernels for Cortex-M
- [MLPerf Tiny](https://github.com/mlcommons/tiny), benchmark suite for ultra-low-power machine learning
- Jacob et al., *Quantization and Training of Neural Networks for Efficient Integer-Arithmetic-Only Inference*, CVPR 2018

## License

To be added.
