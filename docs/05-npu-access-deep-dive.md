# 05 — How Qualcomm reaches the NPU (the centerpiece) ⭐

If you can walk the audience from "a Python request hits the container" down to "operators
execute on the Hexagon" and back, you own the room. This chapter is that path, grounded in
Qualcomm's **own** published container.

## What the "NPU" actually is
🧠 On these SoCs the NPU is the **Hexagon processor** — specifically the **Compute DSP
(cDSP)** with its matrix (**HMX**) and vector (**HVX**) units. "HTP" (Hexagon Tensor
Processor) is the AI-acceleration personality of that Hexagon. So:
- **NPU ≈ HTP ≈ the AI-accelerating part of the Hexagon cDSP.**
- The IQ-9075 EVK has **two** of them (NPU0, NPU1). 📎

## The call path, top to bottom
```
HTTP request ─► OpenAI-compatible API (Hypercorn)      [in container]
            ─► Genie / QAIRT runtime (libGenie, libQnn*) [in container]
            ─► QNN HTP backend  (libQnnHtp.so + skel)     [in container, compiled for v73]
            ─► FastRPC stub  ── ioctl ──► /dev/fastrpc-cdsp   [host kernel driver]
            ─► FastRPC in kernel ─► Hexagon cDSP running DSP firmware  [host-loaded]
            ─► HMX/HVX execute the model operators
   buffers shared both ways via /dev/dma_heap/system (DMA-BUF, zero-copy)
```

### The pieces, and which side they live on
| Piece | Lives in | Role |
|-------|----------|------|
| OpenAI-compatible server (Hypercorn) | container | HTTP front door |
| Genie / QAIRT / QNN user-space libs | container | build & dispatch the inference graph |
| QNN **HTP backend** + **skel** (`...V73...`) | container | code compiled for Hexagon arch v73 |
| **FastRPC** user-space stub (`libcdsprpc`) | container | marshals calls to the DSP | 🧠 |
| `/dev/fastrpc-cdsp[1]` kernel driver | **host** | the RPC channel to the DSP | 📎 |
| **remoteproc** + **DSP firmware** | **host** | boots/owns the Hexagon image | 🧠 |
| `/dev/dma_heap/*` | **host** | allocates buffers both sides can see | 📎 |

🧠 **FastRPC in one sentence:** it's a remote-procedure-call bridge — the CPU-side *stub*
calls into the DSP-side *skel* as if it were a local function, with the kernel `fastrpc`
driver and shared DMA-BUF memory carrying arguments across. That's why the container needs
*both* the `fastrpc` node (the channel) *and* the `dma_heap` node (the shared buffers).

## The real file — annotated line by line  📎
Source: Qualcomm `solutions-microservices`, `docker-compose-qcs9100-ubuntu.yaml`
(BSD-3-Clause-Clear). This is the authoritative worked example.

```yaml
services:
  genai-llm-vlm-service:
    platform: ${TARGET_ARCH:-linux/arm64}         # (A) arm64 image, cross-built
    image: artifacts.codelinaro.org/iot-solutions-microservices/genai-llm-vlm-service:latest
    working_dir: /mnt/work/models
    ports:
      - "${GENAI_PORT:-9001}:${GENAI_PORT:-9001}"  # OpenAI-compatible API port
    cap_add:
      - SYS_RESOURCE                               # (B) raise memlock — NOT --privileged
    ulimits:
      memlock: { soft: -1, hard: -1 }              # (B) pin DSP buffers, no limit
      nofile:  { soft: 65536, hard: 65536 }
    oom_score_adj: -200                            # (C) don't let the OOM killer pick us first
    deploy:
      resources:
        limits:       { memory: 10G }              # (C) leave ~1G for the host OS
        reservations: { memory: 8G }
    stop_signal: SIGTERM
    stop_grace_period: 20s                         # graceful DSP teardown
    environment:
      - MAX_ACTIVE_MODELS=${MAX_ACTIVE_MODELS:-2}  # (D) each model pins DSP/NPU memory
      - GENAI_CONTEXT_CAPPING=${GENAI_CONTEXT_CAPPING:-2048}
      - DEFAULT_MAX_COMPLETION_TOKENS=${DEFAULT_MAX_COMPLETION_TOKENS:-512}
    devices:
      - /dev/dma_heap/system                       # (E) zero-copy shared buffers
      - /dev/dma_heap/qcom,system                  #     (Ubuntu variant adds this)
      - /dev/fastrpc-cdsp                          # (F) NPU0 channel
      - /dev/fastrpc-cdsp1                          # (F) NPU1 channel (dual-NPU part)
    volumes:
      - /sys/firmware/devicetree/base/model:/run/device-model          # (G) detect board
      - ${QCOM:-/usr/share/qcom}:${HOST_QCOM:-/run/host-qcom}:ro        # (H) host libs/fw RO
      - ${GENAI_MODEL_DIR:-/opt/models/}:/mnt/work/models/             # (I) the models
    command: ["hypercorn", "openapi_server.main:app", "--bind", "0.0.0.0:${GENAI_PORT:-9001}"]
```

**Why each block exists — the talking points:**
- **(A)** arm64, cross-built — the container-must-match-arch rule (`docs/03` surface 4).
- **(B)** The *only* privilege it takes is `SYS_RESOURCE` + unlimited `memlock`. Inference
  pins large buffers that must not be swapped. This is the textbook "least privilege"
  alternative to `--privileged`.
- **(C)** Memory governance: cap at 10G, reserve 8G, and bias the OOM killer away — because
  the NPU buffers and KV-cache are large and you don't want the host killing the service.
- **(D)** `MAX_ACTIVE_MODELS` — each loaded model *holds DSP/NPU resources and memory*; beyond
  the limit the least-recently-used model is unloaded. Real NPU memory pressure, in config.
- **(E)** DMA-BUF heaps = the shared-memory half of FastRPC. No heap → no zero-copy → no NPU.
- **(F)** FastRPC channels = the control half. Two nodes because the IQ-9075 has two NPUs.
- **(G)** Bind-mounting the device-tree `model` node lets the container *discover which
  board it's on* — a neat trick: the container adapts to the host, not the reverse.
- **(H)** The host's Qualcomm components are mounted **read-only**. This is the
  firmware/runtime-ABI contract in YAML: the container deliberately uses host-matched pieces
  instead of shipping its own (`docs/03` surface 2).
- **(I)** The model directory is a mount, so **updating the model never rebuilds the image**
  — the clean separation the whole talk argues for.

### Yocto vs Ubuntu variant (a sharp detail to have ready)  📎
The Qualcomm-Linux (`docker-compose-qcs9100.yaml`) variant is nearly identical but:
- drops `/dev/dma_heap/qcom,system` (only `/dev/dma_heap/system`),
- defaults the model dir to `/etc/models/` instead of `/opt/models/`.

🧠 Takeaway: *the device nodes you pass are host-OS-specific.* Same model, same NPU, slightly
different plumbing between Ubuntu and Qualcomm Linux — proof that the hardware contract is
real and must be validated per host.

## Choosing which NPU: `htp_backend_ext_config.json`  📎
This file (inside each model dir) is the host–container contract *in a file*. Default runs
on **NPU0**; add `device_id` to target **NPU1**:

```json
{
  "devices": [
    {
      "device_id": 1,            // 0 = NPU0 (default), 1 = NPU1  (IQ-9075 only)
      "soc_model": 43,           // identifies the SoC — model must match the target
      "dsp_arch": "v73",         // Hexagon arch — a binary built for v73 is NOT portable
      "cores": [
        { "core_id": 0, "perf_profile": "burst", "rpc_control_latency": 100 }
      ]
    }
  ],
  "memory":  { "mem_type": "shared_buffer" },      // uses the DMA-BUF heaps above
  "context": { "weight_sharing_enabled": true }    // share weights across contexts → save NPU RAM
}
```
Talking points:
- `soc_model` + `dsp_arch` **are** compatibility surface 4 from `docs/03`, made concrete.
- `perf_profile: burst` = run the DSP clocks high for lowest latency (vs power-saving).
- `weight_sharing_enabled` lets multiple model contexts share one copy of weights — directly
  relevant to the `MAX_ACTIVE_MODELS` memory story.
- `mem_type: shared_buffer` ties back to the `/dev/dma_heap/*` nodes.

## The models themselves  📎
- Delivered as `...-genie-w4a16-qualcomm_qcs9075.zip` from AI Hub / Hugging Face.
- **W4A16** = 4-bit weights, 16-bit activations — quantized specifically so the HTP can run
  a 4–8B LLM efficiently. (The acronym itself is your slide-11 "optimize/quantize as needed"
  point.)
- `genie` = the GenAI runtime format within QAIRT.
- Mounted at `/mnt/work/models/`; the container exposes an **OpenAI-compatible** API, so any
  LangChain/OpenWebUI client just works.

## 30-second spoken version (rehearse this)
> "People picture a `/dev/npu` you hand to the container. It's not like that. The NPU is the
> Hexagon DSP. The container holds the QAIRT runtime and the quantized model, but to reach
> the silicon it needs two things from the host: the **FastRPC channel** — `/dev/fastrpc-cdsp`
> — which is an RPC bridge into the DSP, and the **DMA-BUF heap** — `/dev/dma_heap/system` —
> for zero-copy shared buffers. The host also loads the DSP firmware and provides its
> Qualcomm libraries, which the container mounts **read-only** so the versions match. And a
> little JSON, `htp_backend_ext_config.json`, picks which of the two NPUs you run on. No
> `--privileged` anywhere. That one compose file *is* the host–container contract."
