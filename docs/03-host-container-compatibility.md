# 03 — File & library compatibility between host and container  ⭐ core chapter

This is the chapter that makes you the expert. Everyone understands "a container packages
the app." Almost nobody can explain **why a container that runs perfectly on one device
fails on another even though the device node is present.** The answer is always one of the
compatibility surfaces below.

## The mental model: three things cross the boundary

When a container uses hardware, three categories of "file" cross the host↔container line:

1. **Device nodes** (`/dev/...`) — the kernel's doorways to hardware. Owned by the host
   kernel; *exposed* to the container with `--device` or a CDI entry.
2. **Shared libraries** (`.so`) — user-space code. Either **baked into the image** (travel
   with the app) or **bind-mounted from the host** (`-v`). This is where ABI breaks happen.
3. **Firmware / config / model files** — data the hardware or runtime needs. Firmware is
   loaded by the **host kernel**; models travel with the container.

The golden rule:

> **The kernel driver + firmware live on the host. The user-space runtime can live in the
> container — but only if its ABI matches the host's kernel driver and firmware.**
> Passing the device node is necessary but *not sufficient*.

## The four compatibility surfaces (know all four cold)

### 1. Kernel ⇄ user-space driver ABI (the one that bites)
The container's user-space library (e.g. the Adreno GL driver, or the FastRPC/QAIRT stack)
talks to the host **kernel driver** through `ioctl`s on a `/dev` node. That ioctl interface
is a contract.
- If the host kernel driver is **older/newer** than the user-space lib expects → the ioctl
  layout mismatches → failures ranging from "device busy" to silent wrong results.
- 🧠 This is why you cannot just copy a working `libQnnHtp*.so` onto an arbitrary kernel and
  expect the NPU to work. The **kernel driver version** is set by the host (Yocto) image.
- **Design rule:** the kernel driver is a *host contract*. Pin the host image (OS commit)
  and the runtime version together, and test the pair.

### 2. Firmware ⇄ runtime ABI (the embedded-specific one)
On Qualcomm the NPU is the **Hexagon DSP**. The model doesn't run "on a chip" directly — a
**firmware image is loaded onto the DSP by the host kernel** (via `remoteproc`), and your
model's compiled operators run against *that firmware's* interface.
- 🧠 The QAIRT/QNN version used to **compile the model** must be compatible with the DSP
  firmware shipped on the host. Mismatch → the context binary refuses to load, or you get a
  version/skel error at init.
- This is why Qualcomm's container mounts the **host's** Qualcomm components read-only
  (`/usr/share/qcom` → `/run/host-qcom:ro`) instead of baking its own: it deliberately uses
  the host-provided, host-matched pieces. 📎 (see the compose file in `docs/05`)
- **Design rule:** firmware is a *host contract*; the compiled model/runtime is a
  *container contract*; the tested (firmware ↔ QAIRT) pair is the *hardware contract*.

### 3. libc & the C/C++ runtime (glibc vs musl)
A container brings its **own** libc, so normally the app is independent of the host libc.
The trap appears when you **bind-mount a host `.so` into the container**:
- A host library built against **glibc** will **not** load into an **alpine/musl** container
  (and vice versa). Symbol/ABI mismatch, cryptic `dlopen` errors.
- 🧠 Rule of thumb: if you intend to mount host vendor libraries (GPU userspace, Qualcomm
  components), make the **container base image match the host libc** — on Qualcomm Linux /
  Ubuntu that means a **glibc** base (debian/ubuntu/`python:3.x-slim`), not alpine.
- Also watch the **glibc version floor**: a library built against glibc 2.35 won't load on
  a container whose glibc is 2.31. Prefer a base ≥ the host's glibc when mounting host libs.

### 4. CPU architecture & instruction level
- The image must be **arm64** (`linux/arm64`). An amd64 image won't run (or runs slowly via
  emulation that doesn't exist on the device).
- 🧠 For the NPU specifically there is a *second* architecture dimension: the **Hexagon DSP
  architecture version** (e.g. `dsp_arch: "v73"`). A model/context binary compiled for one
  Hexagon arch (and one `soc_model`) is **not portable** to another. 📎 This value is set in
  `htp_backend_ext_config.json` (see `docs/05`).

## Decision table — bake into the image, or mount from the host?

| Component | Where it should live | Why |
|-----------|---------------------|-----|
| Kernel driver (`fastrpc`, `kgsl`, camss) | **Host only** | kernel-space; container can't provide it |
| DSP / GPU firmware | **Host only** | loaded by host kernel via remoteproc |
| Device nodes `/dev/*` | **Host**, exposed via `--device`/CDI | kernel-owned doorways |
| Vendor user-space libs tied to the BSP (GPU UMD, some Qualcomm components) | **Host, mounted `:ro`** when ABI is tight | guarantees firmware/kernel match |
| Portable inference runtime (ONNX Runtime, TFLite) | **Container** | self-contained, host-interface-agnostic |
| QAIRT/QNN/Genie app runtime | **Container**, *using* host components | app iterates fast; still mounts host `:ro` pieces |
| The model / context binary | **Container** (or a mounted model dir) | app artifact; versioned with the app |
| Application code + Python/C++ deps | **Container** | the whole point of containerizing |

🧠 **Heuristic:** *package by support boundary, not by convenience.* If a library's correct
version is dictated by the kernel/firmware, treat it as a host component. If it is portable
and only depends on a documented host interface, put it in the container.

## "The contract" — what you must document for every accelerated container
This is the slide-10 checklist, now operational. For every container that touches hardware,
write down:
- [ ] **Device nodes** exposed (`/dev/fastrpc-cdsp`, `/dev/dma_heap/system`, `/dev/video0`…)
- [ ] **Permissions / groups / capabilities** (e.g. `cap_add: SYS_RESOURCE`, video group)
- [ ] **User-space ↔ driver ABI**: which kernel driver version this runtime needs
- [ ] **Required firmware** and the host OS version that provides it
- [ ] **Host libraries mounted** and their path (`/usr/share/qcom` → `/run/host-qcom:ro`)
- [ ] **Tested versions**: the exact (OS commit, runtime version, model build) triple
- [ ] **Validated on the real target**, not only in CI emulation

## Three failure stories you can tell (each maps to a surface above)
1. 🧠 *"Works on my RB3, fails on the IQ-9075."* → model compiled for the wrong `soc_model`/
   Hexagon arch (surface 4). Fix: recompile for the target, set `htp_backend_ext_config.json`.
2. 🧠 *"Device node is there, but `dlopen` fails."* → alpine/musl base mounting a glibc host
   lib (surface 3). Fix: switch to a glibc base image.
3. 🧠 *"Ran fine, then an OS update broke inference."* → host firmware/kernel moved under a
   pinned runtime (surfaces 1–2). Fix: treat (firmware ↔ runtime) as one tested pair, gate
   it in CI, ship them together via a TUF target (`docs/08`).

## The one sentence to repeat
> **"Passing a device node is necessary but not sufficient — the runtime in the container
> and the driver + firmware on the host must be ABI-compatible, and that pair is something
> you *test*, not something you *assume*."**
