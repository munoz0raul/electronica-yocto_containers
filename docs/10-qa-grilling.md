# 10 — Q&A grilling: the questions that can embarrass you

Rehearse these out loud. Each answer is tuned to be **crisp, confident, and correct**, with
the honest "I'd verify that" where appropriate. Grouped by theme.

## Containers fundamentals
**Q: Doesn't a VM give better isolation than a container for Edge AI?**
> Yes — a VM has its own kernel and stronger isolation, but it also virtualizes (or has to
> pass through) the accelerators, which is heavy and often unsupported for NPUs. Containers
> share the host kernel, so NPU/GPU/camera access is a device-node mapping, not virtualized
> hardware. For single-purpose edge devices the container tradeoff — light, direct hardware
> access, host-owned drivers — usually wins. Qualcomm Linux does also support a secondary
> VM if you need that boundary.

**Q: Docker or Podman?**
> The architecture is identical — shared kernel, `--device`, CDI, the ABI contract. Podman
> is daemonless and can run rootless, which is attractive for security on embedded. I used
> Docker as the common reference; Qualcomm Linux ships Docker + Compose via
> meta-virtualization. Pick per your security and tooling needs; the model doesn't change.

**Q: Why not just install everything in the Yocto image and skip containers?**
> You can — and for the kernel/driver/firmware you *should*. The reason to containerize the
> app is cadence: the app and model iterate weekly, the OS shouldn't. Without containers,
> every model tweak becomes a full OS rebuild and re-flash. Containers let the two teams
> move at different speeds while still being validated together.

## The NPU / hardware boundary (expect the hardest questions here)
**Q: How does a container actually reach the NPU? Is there a `/dev/npu`?**
> No single node. On Qualcomm the NPU is the Hexagon DSP. The container gets two things from
> the host: the FastRPC channel — `/dev/fastrpc-cdsp` — which is an RPC bridge into the DSP,
> and the DMA-BUF heap — `/dev/dma_heap/system` — for zero-copy shared buffers. The host
> loads the DSP firmware and provides the Qualcomm libraries, which the container mounts
> read-only so versions match. It's all in Qualcomm's own GenAI compose file.

**Q: Do I need `--privileged` for the NPU?**
> No, and you shouldn't. Qualcomm's own LLM/VLM container adds exactly one capability —
> `SYS_RESOURCE`, to raise memlock so it can pin DSP buffers — plus the specific device
> nodes. No `--privileged`, no `group_add`. That's the pattern to copy.

**Q: My container runs on the RB3 but not the IQ-9075 (or vice versa). Why?**
> Almost certainly the model build. A context binary is compiled for a specific `soc_model`
> and Hexagon arch (`dsp_arch`, e.g. v73) — it's not portable across parts. Recompile for
> the target from AI Hub and set `htp_backend_ext_config.json` accordingly. The IQ-9075 also
> has two NPUs; `device_id` 0/1 selects which.

**Q: The device node is mapped but the library fails to load.**
> That's a user-space/kernel or libc ABI mismatch, not a passthrough problem. Two usual
> causes: (1) an alpine/musl base trying to load a glibc host library — use a glibc base;
> (2) the runtime in the container expects a different kernel-driver or firmware version than
> the host provides — pin the (OS commit ↔ runtime) pair and test it. Passing the node is
> necessary but not sufficient.

**Q: Where *should* the AI runtime live — host or container?**
> Package by support boundary, not convenience. Kernel drivers and firmware are host. The
> portable inference framework and the model go in the container. Vendor libs with a tight
> firmware/kernel coupling are mounted read-only from the host — which is what Qualcomm does
> with `/usr/share/qcom`. Then document the contract: nodes, permissions, ABI, firmware,
> tested versions.

**Q: How do you keep a 30fps camera→NPU pipeline real-time in a container?**
> Zero-copy via DMA-BUF. The camera/ISP produces a buffer on the `/dev/dma_heap/system`
> heap and the NPU consumes the same buffer — no CPU copy per frame. That's why a vision
> container needs both the V4L2 nodes and the DMA-BUF heap. I'd also keep the ISP-tuned
> pipeline on the host and hand processed frames in.

**Q: CDI vs `--device` — which and why?**
> `--device` passes one node; fine for simple cases. CDI describes the *whole* device — all
> nodes, library mounts, env — behind one name in `/etc/cdi`, so app authors don't need to
> know the plumbing and it's auditable. It's the same standard NVIDIA's toolkit uses. For a
> product, CDI scales better than a growing list of `--device` flags.

## AI / accelerator choice
**Q: Should everything run on the NPU? It has the most TOPS.**
> No. TOPS is a headline, not a guarantee. Small models are often simpler and plenty fast on
> CPU with easier debugging; GPU suits some parallel workloads; the NPU is most efficient
> for *supported, quantized* models but may need the model optimized/quantized (e.g. W4A16).
> Measure latency, throughput, power and memory on the real target, then choose.

**Q: What tools help prepare models for the NPU?**
> Qualcomm AI Hub to find/compile models for a specific chipset, and QAIRT/QNN as the
> runtime. The GenAI LLMs ship as `genie-w4a16` builds — quantized specifically so the HTP
> can run them efficiently.

## Yocto / build
**Q: Isn't Yocto overkill? Why not Debian/Ubuntu on the device?**
> Ubuntu is great for bring-up and is supported on Dragonwing. Yocto earns its keep for
> *product* lifecycle: pinned layers → reproducible images → trustworthy SBOMs → long-term
> maintenance and CRA processes. It's the difference between a dev board and a shippable,
> maintainable product line.

**Q: What changed for containers in Qualcomm Linux 2.0?**
> Enablement moved to upstream: the multimedia image now pulls the upstream
> `packagegroup-container` from meta-virtualization, kernel deps are already in the Qualcomm
> tree (no compatibility scripts), and Kubernetes was split out into a dedicated
> `qcom-container-orchestration-image`. Net: less Qualcomm-specific delta to maintain.

## Updates / security
**Q: Does TUF let me ship the whole product as one signed file?**
> No. TUF signs and authenticates update *metadata* — target names, hashes, anti-rollback.
> You can put custom fields in Targets metadata to *describe* a compatible release (hardware
> IDs, OS commit, container digests), but that schema is yours, and your OTA client has to
> interpret and apply it. TUF gives trust; the OTA client gives coordination.

**Q: How do you update just the model without touching the OS?**
> Mount the model dir into the container instead of baking it in, then ship a new container
> digest (or, Foundries-style, a new bundle artifact) and restart the one service. No OS
> rebuild, no reboot. Kernel/driver changes are the opposite — new OS artifact, reboot,
> recovery.

**Q: What if an update bricks the device?**
> That's why activation isn't the last step — confirmation is. A/B partitions or OSTree give
> atomic switch with rollback; the OTA client only commits the new version after a health
> check passes, otherwise it rolls back. The container side rolls back by re-pinning the
> previous digest.

**Q: Does using Yocto/containers/TUF make me CRA-compliant?**
> No — compliance is a process, not a toolchain. These are the foundations that make the
> process manageable: reproducible builds, SBOMs, CVE triage, signed updates, recovery. I'm
> careful not to oversell that.

## The "I don't know" move (use it, don't bluff)
> "Good question — the exact [node name / version / date] I'd want to verify against the
> docs rather than guess. The *mechanism* is [X]; let me confirm the specific value and
> follow up." — Confident about the architecture, honest about a specific fact, never bluff
> a version number or a device node you're unsure of.
