# 04 — Hardware passthrough: the device-node catalog

How you *actually* expose hardware to a container. Two mechanisms, then the node catalog.

## Mechanism 1 — explicit `--device` + bind mounts  📎
Documented in the Qualcomm Linux "Run Docker containers" guide:

```bash
# one or more device nodes
docker run -it --rm --device=<dev-1> --device=<dev-2> <image>

# bind-mount files/dirs from the host (target = path inside container)
docker run -it --rm \
  --mount type=bind,source=<host-path>,target=<container-path> \
  <image>

# graphics example straight from the docs:
docker run -it --rm --device=/dev/kgsl-3d0 <image>
```

Compose form (how Qualcomm ships it):
```yaml
devices:
  - /dev/fastrpc-cdsp
volumes:
  - /usr/share/qcom:/run/host-qcom:ro
```

## Mechanism 2 — Container Device Interface (CDI)  📎
A vendor-neutral standard: describe a "device" (its nodes, mounts, env, hooks) once in a
JSON/YAML spec, then reference it by name. Qualcomm Linux supports it.

```text
/etc/cdi/        # persistent CDI specs
/var/run/cdi/    # dynamic, auto-generated
```
```bash
docker run -it --device <vendor.com/device-kind>=<cdi-device-name> <image> <cmd>
```
🧠 **Why CDI matters / why mention it:** `--device` passes *one node*. Real accelerators
need *a set* — several device nodes **plus** library mounts **plus** environment. CDI
bundles all of that behind one name, so the app author doesn't need to know the plumbing.
It is the clean, declarative answer to "how do I give a container the whole GPU/NPU?" and
it's how this scales beyond a demo. (NVIDIA's container toolkit uses the same CDI standard —
a useful comparison if someone from the GPU world asks.)

## The node catalog (what each `/dev` entry is for)

| Device node | Subsystem | Used for | Source |
|-------------|-----------|----------|--------|
| `/dev/kgsl-3d0` | **KGSL** (Adreno GPU kernel driver) | OpenGL ES / OpenCL / Vulkan compute on the GPU | 📎 docs name this one |
| `/dev/dri/card*`, `/dev/dri/renderD*` | **DRM/KMS** | standard Linux GPU rendering / headless render nodes | 🧠 generic Linux |
| `/dev/fastrpc-cdsp` | **FastRPC → Compute DSP (Hexagon)** | **NPU inference (NPU0)** | 📎 Qualcomm compose file |
| `/dev/fastrpc-cdsp1` | FastRPC → second cDSP | **NPU1** on dual-NPU parts (IQ-9075) | 📎 Qualcomm compose file |
| `/dev/dma_heap/system` | **DMA-BUF heap** | zero-copy shared buffers between CPU and NPU/GPU | 📎 Qualcomm compose file |
| `/dev/dma_heap/qcom,system` | Qualcomm DMA-BUF heap | Qualcomm-specific shared buffers (Ubuntu variant) | 📎 Qualcomm compose file |
| `/dev/video*` | **V4L2** | camera capture / video codec nodes | 🧠 + slide 9 |
| `/dev/media*` | **media controller** | configuring the camera/ISP pipeline topology | 🧠 + slide 9 |

🧠 Note the pattern: **the NPU is not a single magic `/dev/npu`.** It is reached through
**FastRPC channels to the Hexagon DSP** plus **DMA-BUF heaps** for memory. That is the
embedded-specific insight most people don't have. Full explanation in `docs/05`.

## Privileges: do it right, not lazily
- ❌ `--privileged` — gives the container access to *all* devices and most capabilities.
  Fine for a throwaway demo, **wrong** for a product (huge attack surface; the CRA will not
  love it — `docs/09`).
- ✅ Expose **only the nodes you need** with `--device`/CDI.
- ✅ Add **only the capabilities you need**. 📎 Qualcomm's LLM/VLM container adds exactly one:
  `cap_add: SYS_RESOURCE` (to raise `memlock` so it can pin memory for DSP buffers) and sets
  `ulimits: memlock {soft:-1, hard:-1}`. No `--privileged`, no `group_add`.
- ✅ Prefer **read-only** mounts for host-provided libraries (`:ro`).

## The talking point
> "There's a spectrum from `--privileged` (lazy, insecure) to a precise CDI spec
> (declarative, auditable). Qualcomm's own GenAI container sits near the good end: a
> handful of explicit device nodes, one capability, host firmware mounted read-only. That's
> the model to copy."
