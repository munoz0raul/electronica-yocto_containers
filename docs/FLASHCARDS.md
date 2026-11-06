# FLASHCARDS — question → answer drill

Cover the right column. Say the answer out loud before revealing.

| Prompt | Answer |
|--------|--------|
| What does a container share with the host? | The **kernel** (not drivers, firmware, or devices). |
| Is Yocto a runtime layer? | No — it's the **build system** that produces the host image. |
| The one-sentence spine of the talk? | A container shares the host kernel, not the devices; hardware acceleration is a host–container contract. |
| The four contracts? | Host, Container, Hardware, Release. |
| Four compatibility surfaces? | kernel↔UMD ioctl ABI; firmware↔runtime ABI; libc (glibc/musl); CPU arch + Hexagon dsp_arch/soc_model. |
| Which node is the Adreno GPU? | `/dev/kgsl-3d0`. |
| Which nodes reach the NPU? | `/dev/fastrpc-cdsp` (+`cdsp1`) + `/dev/dma_heap/system`. |
| Is there a `/dev/npu`? | No. NPU = Hexagon cDSP, reached via FastRPC + DMA-BUF. |
| What is FastRPC? | An RPC bridge: CPU-side stub calls DSP-side skel via the kernel fastrpc driver + shared memory. |
| Why does the NPU container need a DMA-BUF heap? | Zero-copy shared buffers between CPU and DSP (`mem_type: shared_buffer`). |
| Does Qualcomm's NPU container use `--privileged`? | No — only `cap_add: SYS_RESOURCE` + unlimited `memlock`. |
| Why mount `/usr/share/qcom` read-only? | To use host-matched Qualcomm libs/firmware → firmware↔runtime ABI match. |
| Why bind-mount `/sys/firmware/devicetree/base/model`? | So the container can detect which board it's running on. |
| How do you pick NPU0 vs NPU1? | `device_id` 0/1 in `htp_backend_ext_config.json` (IQ-9075 has two NPUs). |
| What is `dsp_arch: "v73"`? | The Hexagon architecture version; context binaries built for it aren't portable across arch/SoC. |
| What does W4A16 mean? | 4-bit weights, 16-bit activations — quantization so the HTP runs LLMs efficiently. |
| Container works on RB3, fails on IQ-9075 — likely cause? | Model compiled for wrong soc_model/dsp_arch — recompile for target. |
| Device node mapped but `dlopen` fails — likely cause? | libc mismatch (musl base + glibc host lib) or runtime↔driver version mismatch. |
| Camera nodes to expose? | `/dev/video*` (V4L2) + `/dev/media*` (pipeline) + `/dev/dma_heap/system`. |
| Preferred camera path on Qualcomm? | Through GStreamer (ISP-tuned pipeline), often kept on host, pass frames in. |
| How is Docker enabled in Qualcomm Linux? | `qcom-multimedia-image` pulls `packagegroup-container` from `meta-virtualization`. |
| Verify Docker on device? | `systemctl status docker`. |
| QLI 2.0 container change? | Upstream packagegroup; no compat scripts; k8s → `qcom-container-orchestration-image`. |
| Build Yocto without host deps? | `kas-container` (needs only Docker). |
| Tag vs digest for release? | Ship the immutable **SHA-256 digest**, not a mutable tag. |
| Build image for the device from x86? | `docker buildx build --platform linux/arm64 … --push`. |
| What is CDI and why use it? | Container Device Interface: describe all nodes+mounts+env behind one name in `/etc/cdi`; scales past `--device`. |
| The integration gate checks what? | Correct OS + correct containers + real-hardware tests (camera? NPU? latency? recovery?). |
| What is TUF? | The Update Framework — signs/authenticates update metadata (names, hashes, anti-rollback). |
| Can TUF pack a whole product into one file? | No — it signs metadata; custom Target fields *describe* a release; the OTA client interprets them. |
| Five OTA steps? | Select, Verify, Download, Activate, **Confirm/rollback**. |
| The OTA step people forget? | **Confirm** — health check; a download is not a successful update. |
| Safe activation/rollback mechanism? | A/B partitions or OSTree; containers roll back by re-pinning previous digest. |
| Update model only — how? | Mount model dir, ship new container digest, restart one service; no OS rebuild. |
| CRA dates? | 11 Sep 2026 (reporting) · 11 Dec 2027 (main) — **verify before stage**. |
| Does Yocto/containers/TUF = CRA compliance? | No — foundations that make the compliance *process* manageable. |
| Accelerator choice rule? | Measure latency/throughput/power/memory on target; don't choose on TOPS. |
| Model-prep tools? | AI Hub (find/compile per chipset) + QAIRT/QNN runtime. |
| Docker vs Podman for the architecture? | Identical — shared kernel, `--device`, CDI, ABI. Podman daemonless/rootless. |
