# GLOSSARY — acronyms & terms

## Containers / build
- **OCI** — Open Container Initiative; the standard image/runtime format (Docker images are OCI artifacts).
- **CDI** — Container Device Interface; vendor-neutral spec to expose a whole device (nodes + mounts + env) by name.
- **Yocto** — build system (recipes/layers/BitBake) that produces a reproducible embedded Linux image.
- **meta-virtualization** — the Yocto layer that provides Docker/Podman/containerd/k8s packages.
- **kas-container** — wrapper to run a Yocto build inside Docker (no Yocto deps on the host).
- **BSP** — Board Support Package; kernel + drivers + firmware for a specific board.
- **ABI** — Application Binary Interface; the binary contract between compiled components (e.g. a lib and a kernel ioctl).
- **UMD** — user-mode driver (the user-space half of a graphics/compute driver).
- **glibc / musl** — the two common C libraries; not ABI-compatible with each other.

## Qualcomm hardware / AI
- **Dragonwing** — Qualcomm's brand for industrial/embedded/IoT platforms.
- **SoC / SoM / EVK** — System on Chip / System on Module / Evaluation Kit.
- **NPU** — Neural Processing Unit; on these SoCs, the AI-accelerating part of the Hexagon.
- **HTP** — Hexagon Tensor Processor; the NPU personality of the Hexagon DSP.
- **HVX / HMX** — Hexagon Vector / Matrix eXtensions (vector & matrix compute units).
- **cDSP** — Compute DSP; the Hexagon core used for AI compute.
- **FastRPC** — RPC mechanism bridging CPU (stub) and DSP (skel); kernel driver at `/dev/fastrpc-cdsp`.
- **remoteproc** — Linux framework the host uses to boot/manage DSP firmware.
- **DMA-BUF / dma_heap** — kernel buffer-sharing mechanism for zero-copy between CPU/GPU/NPU.
- **KGSL** — the Adreno GPU kernel driver; node `/dev/kgsl-3d0`.
- **CAMSS** — Qualcomm camera subsystem in the kernel.
- **ISP** — Image Signal Processor (debayer, 3A, HDR, LDC/EIS).
- **Adreno / Hexagon / Kryo** — Qualcomm GPU / DSP / CPU family names.

## AI runtime / models
- **QAIRT** — Qualcomm AI Runtime SDK (contains QNN).
- **QNN** — Qualcomm Neural Network backend API (HTP backend runs on the NPU).
- **Genie** — the GenAI/LLM runtime format & engine within QAIRT (`genie-w4a16` models).
- **AI Hub** — Qualcomm's portal to find/compile models for a specific chipset.
- **ONNX Runtime / LiteRT (TFLite)** — portable inference frameworks that can live in the container.
- **W4A16 / W8A8** — quantization schemes (4-bit weights 16-bit activations / 8-bit both).
- **soc_model / dsp_arch** — fields identifying the target SoC and Hexagon arch (e.g. v73) for a model build.
- **TOPS** — Tera Operations Per Second; a peak-throughput headline number, not a latency guarantee.
- **LLM / VLM** — Large Language Model / Vision-Language Model.
- **KV cache** — the per-session key/value memory an LLM holds; a major NPU memory consumer.

## Camera / media
- **V4L2** — Video4Linux2; kernel video capture API; `/dev/video*`.
- **media controller** — configures pipeline topology; `/dev/media*`.
- **GStreamer** — user-space media pipeline framework; the intended camera path on Dragonwing.
- **NV12 / YUV** — common hardware pixel formats.
- **MIPI CSI** — camera serial interface from sensor to SoC.

## Lifecycle / security
- **TUF** — The Update Framework; signs & authenticates update metadata (names, hashes, anti-rollback).
- **OTA** — Over-the-Air update.
- **OSTree** — atomic, git-like OS update/rollback system.
- **A/B partitions** — two system slots for safe update + rollback.
- **SBOM** — Software Bill of Materials; inventory of all software components.
- **CVE** — Common Vulnerabilities and Exposures; a tracked vulnerability.
- **CRA** — EU Cyber Resilience Act.
- **Compose App** — Foundries' unit: container image(s) + a bundle artifact (compose file + assets like a model).
- **FoundriesFactory / LmP** — Foundries.io's platform / Linux microPlatform (uses OSTree + TUF).
