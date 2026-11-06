# 01 — The mental model: one product, clear ownership at every layer

## The stack (bottom → top)

```
┌─────────────────────────────────────────────────────────────┐
│  EDGE AI APPLICATION + MODEL                                  │  ← container
│  voice/vision logic, model assets, configuration              │
├─────────────────────────────────────────────────────────────┤
│  USER-SPACE AI RUNTIME                                         │  ← container (usually)
│  ONNX Runtime, QAIRT / QNN / Genie, framework libs            │
├─────────────────────────────────────────────────────────────┤
│  CONTAINER RUNTIME + DEVICE ACCESS                            │  ← host provides engine;
│  OCI image, mounts, --device, permissions, orchestration      │    app runs on top
├─────────────────────────────────────────────────────────────┤
│  LINUX HOST / BSP                                             │  ← Yocto-built host image
│  kernel, firmware, drivers (fastrpc, kgsl, camss…), dev nodes │
├─────────────────────────────────────────────────────────────┤
│  HARDWARE:  Camera · CPU · GPU · NPU/HTP · MCU                │
└─────────────────────────────────────────────────────────────┘
```

## Who owns what (say this exactly)

| Layer | Owner | Examples |
|-------|-------|----------|
| Hardware | SoC / board | Camera, CPU, Adreno GPU, Hexagon NPU/HTP |
| Kernel, drivers, firmware, device nodes | **Host (Yocto image)** | `fastrpc`, `kgsl`, `remoteproc`, DSP firmware, DMA-BUF heaps |
| Container engine + update agent | **Host (Yocto image)** | dockerd, containerd, docker-compose, OTA client |
| User-space AI runtime | **Usually the container** | QAIRT/QNN/Genie, ONNX Runtime, TFLite/LiteRT |
| Application + model | **Container** | your code, deps, the `.bin`/`genie` model |

## The single clarification that prevents confusion
🧠 **Yocto is not a runtime layer.** It is not something that executes between Linux and
Docker at runtime. **Yocto is the *build system* that produces the Linux host image.** At
runtime there is no "Yocto." Say this explicitly on slide 5 — it is the most common
misunderstanding from newcomers.

## Why separate platform from application
- The **platform** (kernel/BSP/firmware) changes *slowly*, is *risky* to change, and is
  owned by the platform team. It wants reproducibility and long-term maintenance.
- The **application + model** change *fast*, iterate *weekly*, and are owned by the app/AI
  team. They want independence from OS rebuilds.
- Containers are the seam that lets those two cadences coexist **without** pretending they
  are fully independent — because the hardware boundary still couples them (see `docs/03`).

## The four contracts (this is the takeaway slide, memorize the four)
1. **Host contract** — kernel, BSP, drivers, firmware, device policy.
2. **Container contract** — application, dependencies, model, runtime.
3. **Hardware contract** — device nodes, permissions, ABI, performance.
4. **Release contract** — CI/CD, TUF metadata, OTA, health & rollback.
