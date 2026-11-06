# 07 — Yocto and the split: what goes where

## Yocto in one breath (for the non-Yocto audience)
🧠 Yocto is a **build system** (recipes, layers, BitBake) that produces a reproducible,
customized embedded Linux image. You don't ship Yocto; you ship the image Yocto builds.
Reproducibility + layer pinning is exactly what the security/lifecycle story later needs.

## The split (slide 8), as a checklist

### Yocto host image owns
- Boot chain, kernel, BSP
- Drivers, firmware, device rules (udev), DMA-BUF heaps
- **Container engine + update agent** (dockerd/containerd/compose, OTA client)
- Security policy, host services
- Versioned recipes and layers → the reproducible foundation

### OCI application image(s) carry
- Application logic and UI
- Python/C++ dependencies
- Inference framework + model
- Application configuration
- An **independent** application version

> **The separation is not total independence.** If the container needs a driver interface
> or library version the host doesn't provide, it fails. That coupling is the hardware
> contract — which is why you still validate them as one product.

## How containers are actually enabled on Qualcomm Linux  📎
- `qcom-multimedia-image` includes **`packagegroup-container`** from the
  **`meta-virtualization`** layer → Docker **and** Docker Compose on the device.
- Verify on target: `systemctl status docker`.
- Build inside a container yourself (no Yocto deps on your host) with **`kas-container`**,
  which just needs Docker — a reproducible, host-clean build path.

## What changed in Qualcomm Linux 2.0 (migration talking points)  📎
- Container enablement was **streamlined and moved to upstream**: `qcom-multimedia-image`
  now pulls the upstream **`packagegroup-container`** from `meta-virtualization` instead of a
  Qualcomm-defined package group.
- **Kernel deps for Docker are already enabled** in the Qualcomm kernel tree → *"you don't
  need to run compatibility scripts or modify kernel configurations."*
- **Kubernetes was decoupled** from the multimedia image: it used to come from
  `packagegroup-qcom-k8s`; now you build a dedicated **`qcom-container-orchestration-image`**
  (which also uses upstream Kubernetes/orchestration packages from `meta-virtualization`).
- 🧠 The theme — *move off Qualcomm-specific forks toward upstream* — is itself a
  maintainability/lifecycle win worth naming: less delta to carry, easier CVE tracking.

## Orchestration: when do you even need it?
🧠 Most single-box embedded products need **docker-compose**, not Kubernetes. Reach for
k8s / k3s only for fleets of multi-service nodes or when you want declarative scheduling
across devices. Qualcomm Linux supports both, but don't over-engineer a single device.

## Runtime choice (be ready, don't over-claim)
🧠 The talk uses **Docker** as the familiar example. The same model applies to **Podman**
(daemonless/rootless — attractive for security on embedded) and **containerd**. The device
nodes, mounts, CDI and ABI story are identical regardless of engine. Say: *"Docker here as
the lingua franca; Podman and containerd apply the same way."*

## Storage footprint reminder
🧠 Container image layers consume UFS/eMMC. On a space-constrained device, choose minimal
bases, collapse layers, and remember models can be **mounted** (not baked) so they don't
bloat the image or get duplicated on every app update. (Qualcomm's container mounts the
model dir for exactly this reason — `docs/05` block (I).)
