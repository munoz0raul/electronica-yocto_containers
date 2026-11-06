# SOURCES — every link, and what it proves

Legend: 📎 = used verbatim / authoritative for a specific fact in this guide.

## Qualcomm Dragonwing docs
- 📎 **Run Docker containers** —
  https://dragonwingdocs.qualcomm.com/Key-Documents/Yocto-Guide/run-docker-containers
  → Docker enabled via `packagegroup-container` (meta-virtualization) in
  `qcom-multimedia-image`; `--device` + `--mount` + CDI mechanisms; `/dev/kgsl-3d0` GPU
  example; `/etc/cdi` & `/var/run/cdi`; `systemctl status docker`; tested arm64 images;
  docker-compose shipped by default.
- 📎 **Running the LLM/VLM container** —
  https://dragonwingdocs.qualcomm.com/AI-Developer-Workflow-Ubuntu/topic/llm-vlm-qcom-setup
  → OpenAI-compatible GenAI container; `docker-compose up`; models at `/mnt/work/models`;
  `genie-w4a16` builds from AI Hub; env `GENAI_PORT`/`MAX_ACTIVE_MODELS`/`GENAI_CONTEXT_CAPPING`;
  "models run entirely on the NPU"; each model holds DSP/NPU resources.
- 📎 **Running on a specific NPU (QCS9075)** —
  https://dragonwingdocs.qualcomm.com/AI-Developer-Workflow-Ubuntu/topic/llm-vlm-qcom-npuconfig
  → `htp_backend_ext_config.json`; dual-NPU IQ-9075; `device_id` 0/1; `soc_model: 43`,
  `dsp_arch: "v73"`, `perf_profile: "burst"`, `weight_sharing_enabled`, `mem_type: "shared_buffer"`.
- 📎 **Migrate features to Qualcomm Linux 2.0 — containers & orchestration** —
  https://dragonwingdocs.qualcomm.com/Key-Documents/Migration-Guide/migrate-features-to-qualcomm-linux-2-0#containers-and-orchestration-updates
  → upstream `packagegroup-container`; kernel deps pre-enabled (no compat scripts); k8s moved
  to dedicated `qcom-container-orchestration-image`.
- **Docs landing / device peripheral pages** — https://dragonwingdocs.qualcomm.com/
  → Ubuntu "native support for containers"; GStreamer camera/audio/video + edge-AI runtime
  integration; per-device Camera/Display/Audio pages (V4L2 context).

## Qualcomm source (authoritative compose files)
- 📎 **LLM/VLM GenAI service — deploy compose files** (CodeLinaro, BSD-3-Clause-Clear) —
  https://git.codelinaro.org/clo/le/solutions-microservices/-/tree/iot-solutions.lnx.1.0/microservices/llm-vlm/chatcompletions/deploy
  → the real `devices:` (`/dev/fastrpc-cdsp`, `/dev/fastrpc-cdsp1`, `/dev/dma_heap/system`,
  `/dev/dma_heap/qcom,system`), `volumes:` (`/usr/share/qcom:…:ro`, model dir,
  device-tree model), `cap_add: SYS_RESOURCE`, `memlock -1`, memory limits, image name.
  Files: `docker-compose-qcs9100-ubuntu.yaml`, `docker-compose-qcs9100.yaml`,
  `-qcs8300-ubuntu.yaml`, `-qcs8300.yaml`.
- **Dragonwing docs repo** (fallback / provenance) —
  https://github.com/dragonwing-docs/dragonwing-docs (content on branches, e.g. `main`;
  `archives/2.0`, `archives/2.1` hold prior QLI versions).
- **AI Hub** — https://aihub.qualcomm.com/ (model compilation/benchmark per chipset).

## Foundries.io (external validation for lifecycle/OTA/TUF)
- **Embedded Linux Security: Protecting Devices Across Their Lifecycle** —
  https://www.foundries.io/insights/blog/embedded-linux-security/
  → FoundriesFactory uses **TUF**; every target signed/verified; SBOM↔reproducible builds;
  CVE triage (exploitability over CVSS); A/B + rollback; secure-OTA properties.
- **OTA Updating ML Models With Compose Apps** —
  https://www.foundries.io/insights/blog/arduino-app-model-update/
  → model as a **bundle artifact** (not baked into image); bind-mounted `./models:/models`;
  new OTA target per model update; model evolves independently of the OS.
- **Compose Apps for IoT / fioup / Using Yocto in Production** — index:
  https://www.foundries.io/insights/blog/
  → Compose Apps = host-OS-independent app packaging; `fioup` OTA client; Yocto-in-production.

## Standards / regulation (verify specifics yourself)
- **TUF** — https://theupdateframework.io/
- **CDI (Container Device Interface)** — https://github.com/cncf-tags/container-device-interface
- **EU Cyber Resilience Act** — official EUR-Lex text (confirm the 11 Sep 2026 / 11 Dec 2027
  dates against the current consolidated regulation before presenting).

## Provenance note
Device nodes, env vars, compose keys, `htp_backend_ext_config.json` fields, and the QLI 2.0
packagegroup changes are **quoted from the sources above** (📎). Deeper framing of FastRPC
internals, remoteproc/firmware loading, libc/ABI mechanics, and the camera stack is
**architecture background** — correct and standard, but phrase as understanding, and verify
any exact filename/version if challenged.
