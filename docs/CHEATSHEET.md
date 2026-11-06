# CHEATSHEET — one page, night-before recall

## The spine
> A container shares the host kernel — **not** devices, drivers or firmware.
> **Hardware acceleration is a host–container contract.**
> Build independently · validate & release together.

## Four contracts (the takeaway)
1. **Host** — kernel, BSP, drivers, firmware, device policy
2. **Container** — app, dependencies, model, runtime
3. **Hardware** — device nodes, permissions, ABI, performance
4. **Release** — CI/CD, TUF, OTA, health & rollback

## Who owns what
- Host (Yocto image): kernel, drivers, firmware, device nodes, container engine, OTA client
- Container: app code, Python/C++ deps, inference framework, model
- Yocto = **build system** that makes the host image — NOT a runtime layer

## 4 compatibility surfaces (why a working container fails elsewhere)
1. Kernel ⇄ user-space driver **ioctl ABI**
2. **Firmware ⇄ runtime** ABI (QAIRT build ↔ DSP firmware)
3. **libc**: glibc vs musl (don't mount glibc host libs into alpine)
4. **arch**: arm64 image + Hexagon **dsp_arch v73** / **soc_model** (binaries not portable)

## NPU access = 4 things from the host
- `/dev/fastrpc-cdsp` (+`cdsp1`) → FastRPC channel to Hexagon (NPU0/NPU1)
- `/dev/dma_heap/system` → zero-copy shared buffers
- host DSP **firmware** (loaded via remoteproc)
- host Qualcomm libs mounted **`:ro`** (`/usr/share/qcom → /run/host-qcom`)
- **No `--privileged`** — just `cap_add: SYS_RESOURCE` + unlimited `memlock`

## Device-node catalog
| node | what |
|------|------|
| `/dev/kgsl-3d0` | Adreno **GPU** |
| `/dev/fastrpc-cdsp[1]` | **NPU** (Hexagon cDSP) |
| `/dev/dma_heap/system` | zero-copy buffers |
| `/dev/video*` / `/dev/media*` | **camera** V4L2 / pipeline |

## htp_backend_ext_config.json
`device_id` 0/1 (which NPU) · `soc_model` · `dsp_arch:"v73"` · `perf_profile:"burst"` ·
`weight_sharing_enabled` · `mem_type:"shared_buffer"`

## Yocto enablement
`qcom-multimedia-image` → `packagegroup-container` (from **meta-virtualization**) → Docker +
Compose. QLI 2.0: upstream packagegroup, no compat scripts, k8s → `qcom-container-orchestration-image`.

## Accelerator choice
Don't chase TOPS. CPU (small/simple/debug) · GPU (parallel) · NPU/HTP (efficient, needs
quantization e.g. **W4A16**). Measure latency/throughput/power/memory **on target**. Tools: **AI Hub + QAIRT**.

## CI/CD · TUF · OTA
Two pipelines (OS + app) → **integration gate on real hardware** → TUF signs a release
description (custom fields = illustrative) → OTA: SELECT·VERIFY·DOWNLOAD·ACTIVATE·**CONFIRM/rollback**.
"A successful download is not a successful product update."

## CRA (verify dates!)
**11 Sep 2026** reporting · **11 Dec 2027** main requirements. SBOM + CVE triage + signed
OTA + reproducible builds. Foundations, **not** automatic compliance.

## If stuck
State the mechanism confidently; verify the specific version/node/date rather than bluff.
