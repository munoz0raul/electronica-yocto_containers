# 06 — Cameras in containers

The camera is the second-hardest device after the NPU, and the one most Edge-AI demos need
(vision models). The structure mirrors `docs/05`: nodes on the host, pipeline in the app.

## The Linux camera stack (bottom → top)
🧠
```
sensor ─► MIPI CSI ─► ISP (image signal processor) ─► kernel camera driver (CAMSS)
      ─► V4L2 + media-controller device nodes ─► user-space (GStreamer / libcamera)
      ─► frames (often NV12) ─► your model
```
- **V4L2** (Video4Linux2) is the standard kernel API for video capture. Each capture/codec
  node appears as `/dev/video*`.
- The **media controller** (`/dev/media*`) configures the *topology* of a complex pipeline
  (sensor → ISP → output) — necessary on SoCs where the camera isn't a simple webcam.
- Frames commonly come out as **NV12** (semi-planar YUV 4:2:0) — the native format for
  hardware codecs and ISPs; you usually convert to RGB for the model.

## What you expose to the container
🧠 (same two-mechanism pattern as `docs/04`)
```bash
# USB UVC webcam — the easy case
docker run -it --rm --device=/dev/video0 <image>

# SoC camera pipeline — you typically need the media node and several video nodes,
# plus the DMA-BUF heap for zero-copy buffers:
docker run -it --rm \
  --device=/dev/video0 --device=/dev/video1 \
  --device=/dev/media0 \
  --device=/dev/dma_heap/system \
  <image>
```
Permissions: the device nodes are usually owned by the `video` group → add the container
user to it (`group_add: [video]` in compose) instead of going `--privileged`.

## The Qualcomm specifics (have these ready)
🧠
- The Qualcomm camera subsystem is **CAMSS** in the kernel; user space is served by a
  camera service / HAL and exposed to apps through **GStreamer** (e.g. a `qtiqmmfsrc`-style
  source) or **libcamera**.
- 📎 The Dragonwing docs describe accelerating **"GStreamer-based pipelines for video, audio
  and camera feeds with direct edge-AI runtime integrations"** — i.e. the intended path is
  *camera → GStreamer → model*, not raw V4L2 by hand.
- The ISP does heavy lifting (debayer, 3A, HDR, LDC/EIS). 🧠 Decide early: does the **host**
  own the tuned camera pipeline (common, because ISP tuning is BSP-coupled) and hand
  processed frames to the container, or does the container drive V4L2 directly? That choice
  *is* a host–container contract, exactly like the NPU one.

## The compatibility trap specific to cameras
🧠 Camera user-space (libcamera/GStreamer plugins, the vendor camera HAL) is often **tightly
coupled to the BSP** — tuning files, ISP firmware, kernel driver version. This makes the
camera a strong candidate for the **"keep it on the host, pass processed frames to the
container"** pattern, rather than rebuilding the whole pipeline inside the image. When in
doubt, treat the camera HAL like firmware: a host contract.

## Zero-copy is the performance story
🧠 For real-time vision you don't want to copy every frame CPU→NPU. The same **DMA-BUF**
mechanism from `docs/05` lets a frame buffer produced by the camera/ISP be handed to the
NPU without a copy. That's why a camera+NPU container often needs **both** `/dev/video*`
**and** `/dev/dma_heap/system`. If someone asks "how do you keep 30fps vision real-time in a
container?" the answer is **DMA-BUF zero-copy across camera → NPU, same heap for both.**

## 20-second spoken version
> "Cameras follow the same rule as the NPU. The sensor, ISP and CAMSS driver are the host's;
> the container gets V4L2 and media-controller nodes plus the DMA-BUF heap, and usually
> talks to the camera through GStreamer rather than raw V4L2. Because ISP tuning is
> BSP-coupled, I often keep the camera pipeline on the host and pass already-processed,
> zero-copy frames into the container. Device nodes in, tuned pipeline stays home."
