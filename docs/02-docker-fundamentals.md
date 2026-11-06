# 02 — Docker fundamentals (the parts that matter for embedded)

## Dockerfile → Image → Container

| Concept | What it is | Key property |
|---------|-----------|--------------|
| **Dockerfile** | The recipe | base image, deps, files, startup command |
| **Image** | The built artifact | versioned, layered, portable as an **OCI** artifact, pin by **SHA-256 digest** |
| **Container** | The running process | created from an image, **shares the host kernel**, uses mounts + devices, disposable |

## The embedded truth, stated twice in the talk on purpose
🧠 **A container does not contain a kernel.** It is a set of isolated *user-space*
processes (namespaces + cgroups) running on the **host's** kernel. Consequences:
- The container cannot "bring its own driver." A kernel driver is loaded by the **host**.
- `uname -r` inside the container reports the **host** kernel.
- A container *can* bring its own **user-space** libraries and a different libc/distro —
  that is exactly why we use it to decouple the app from the OS.
- But the moment the container's user space talks to hardware, it depends on the host's
  kernel driver + firmware ABI → that is the whole subject of `docs/03`.

## Build for the target architecture
Your laptop is x86; the device is arm64. You must cross-build:

```bash
# Build an arm64 image from an x86 workstation and push it
docker buildx build \
  --platform linux/arm64 \
  -t registry/edge-ai:1.0 \
  --push .
```

📎 Qualcomm's own compose files pin `platform: ${TARGET_ARCH:-linux/arm64}` for exactly
this reason.

## Tag vs digest (say which you'd ship)
```
registry/edge-ai:1.0                      # convenient for dev; MUTABLE — do not ship
registry/edge-ai@sha256:9f86d0818…        # immutable; THIS is what a release pins
```
- A tag can be re-pushed to point at different bytes. A digest cannot.
- A controlled release references the **digest**, so "which exact software" is unambiguous.

## Production Dockerfile hygiene (one line each if asked)
- **Pin the base image** by digest, not `:latest`.
- **Scan dependencies** (produces part of your SBOM — see `docs/09`).
- **No credentials in the image** — they end up in a layer forever.
- **Smallest base that works** — flash/UFS space is finite on embedded (`-slim`, distroless, alpine *if* libc allows — see `docs/03`).
- **Run as non-root** where possible; drop capabilities.

## Where the container engine comes from on a Qualcomm device
📎 It is **not** hand-installed. On Qualcomm Linux the engine is built into the Yocto
image via the `meta-virtualization` layer:
- `qcom-multimedia-image` pulls in `packagegroup-container` (from `meta-virtualization`).
- Docker **and** Docker Compose are present after flashing.
- Verify on device: `systemctl status docker`.

This is the credibility point for slide 8: *"the Yocto host owns the container engine."*
