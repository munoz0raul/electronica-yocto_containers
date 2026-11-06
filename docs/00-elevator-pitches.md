# 00 — Elevator pitches

Three lengths. Memorize the 10-second one word-for-word; it is your fallback for any
"so what is this really about?" moment.

## 10 seconds
> "An embedded AI product is not four problems — Linux, containers, AI, security. It's
> one system with contracts between the layers. Yocto builds the foundation, containers
> carry the app, and the hardware boundary is a tested compatibility contract."

## 30 seconds
> "Teams build embedded Edge AI products by gluing together a Linux BSP, Docker, an AI
> runtime and an update system — and then every release becomes a painful field
> integration. My argument is to assign clear ownership: the **Yocto host** owns the
> kernel, drivers and firmware; the **container** carries the application, its
> dependencies and the model; and the **hardware boundary** between them — device nodes,
> permissions, and the runtime-to-driver ABI — is an explicit contract you test in CI.
> Build the pieces independently, but validate and release them together."

## 60 seconds
> "Everyone can run `docker run` and everyone can run a model. The hard part of embedded
> Edge AI isn't either of those — it's the **boundaries**. Who owns the NPU driver? Does
> the AI runtime inside the container match the firmware on the host? If I pass a device
> node but the ABI is wrong, it fails at runtime, in the field, not in the build.
>
> So I propose one product with clear ownership at every layer. Yocto gives you a
> reproducible Linux foundation — kernel, drivers, firmware, the container engine and the
> update agent. Containers give the application team independence — they ship their code,
> their Python/C++ deps, the inference framework and the model without rebuilding the OS.
> The hardware boundary is a contract: expose only the device nodes you need, never
> `--privileged` by default, and pin the runtime-driver versions you've actually tested.
> Then CI/CD builds the two pipelines separately but converges at an integration gate,
> TUF describes a compatible release, and OTA makes it real — including a health check
> and rollback. That's how Yocto, containers and Edge AI become one maintainable system."

## The one-liner that is the spine of everything
> **"A container shares the host kernel — not the devices, drivers or firmware. Hardware
> acceleration is a host–container contract."**
