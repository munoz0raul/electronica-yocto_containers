# Yocto, Containers & Edge AI — Speaker Study Guide

> Personal preparation material for the electronica 2026 technical session
> **"Yocto, Containers and Edge AI: Making All the Pieces Work Together"**
> (Munich · 10 November 2026 · 16:20–16:45 · Raul Muñoz, Qualcomm Developer Relations)

This repository is **study material to dominate the talk** — not the slides themselves.
It goes one or two levels deeper than the deck so that any question from the audience
lands inside something you already know cold.

The guiding principle of the whole talk, and of this guide:

> **Build the pieces independently. Validate and release them together.**
> A container shares the host kernel — it does **not** bring drivers, firmware, or
> automatic hardware access. Hardware acceleration is a **host–container contract**.

---

## How to use this

1. Read `docs/01` → `docs/09` once, slowly. That is the full argument of the talk, deepened.
2. Drill `docs/10-qa-grilling.md` — these are the questions that can embarrass you on stage.
3. The night before: read **`docs/CHEATSHEET.md`** only. One page, rapid recall.
4. On the train: run through **`docs/FLASHCARDS.md`** (question → answer).

Everything with a 📎 is **backed by an official source** (Dragonwing docs or Qualcomm's
own compose files). Everything marked 🧠 is **architecture background** — true and useful,
but conceptual framing rather than a verbatim doc quote, so phrase it as understanding,
not as a citation.

---

## Index

| # | File | What it gives you |
|---|------|-------------------|
| 00 | [`docs/00-elevator-pitches.md`](docs/00-elevator-pitches.md) | 10-, 30- and 60-second answers to "what's the talk about" |
| 01 | [`docs/01-mental-model.md`](docs/01-mental-model.md) | The layered architecture and who owns each layer |
| 02 | [`docs/02-docker-fundamentals.md`](docs/02-docker-fundamentals.md) | Dockerfile → image → container, shared-kernel truth |
| 03 | [`docs/03-host-container-compatibility.md`](docs/03-host-container-compatibility.md) | **The core chapter** — file/library/ABI compatibility between host and container |
| 04 | [`docs/04-hardware-passthrough.md`](docs/04-hardware-passthrough.md) | The device-node catalog: how you actually expose camera/GPU/NPU |
| 05 | [`docs/05-npu-access-deep-dive.md`](docs/05-npu-access-deep-dive.md) | **How Qualcomm reaches the NPU** — FastRPC, QAIRT/Genie, dual-NPU, real compose file |
| 06 | [`docs/06-camera-in-containers.md`](docs/06-camera-in-containers.md) | Cameras: V4L2, media controller, GStreamer, the Qualcomm pipeline |
| 07 | [`docs/07-yocto-and-the-split.md`](docs/07-yocto-and-the-split.md) | What belongs in the Yocto host vs the container; QLI 2.0 changes |
| 08 | [`docs/08-cicd-tuf-ota.md`](docs/08-cicd-tuf-ota.md) | Build separately, validate together; TUF targets; OTA flow |
| 09 | [`docs/09-security-cra.md`](docs/09-security-cra.md) | Cyber Resilience Act, SBOM, CVE, signing |
| 10 | [`docs/10-qa-grilling.md`](docs/10-qa-grilling.md) | Anticipated hard questions + crisp answers |
| — | [`docs/CHEATSHEET.md`](docs/CHEATSHEET.md) | One-page rapid recall |
| — | [`docs/FLASHCARDS.md`](docs/FLASHCARDS.md) | Question/answer drill set |
| — | [`docs/GLOSSARY.md`](docs/GLOSSARY.md) | Acronyms & terms |
| — | [`docs/SOURCES.md`](docs/SOURCES.md) | Every link used, with what it proves |

---

## The single most important example in the whole guide

Qualcomm's **own** LLM/VLM container is the perfect worked example of the host–container
contract. This is what "give the container access to the NPU" actually looks like — note
there is **no `--privileged`**:

```yaml
# docker-compose-qcs9100-ubuntu.yaml  (Qualcomm, BSD-3-Clause-Clear)
services:
  genai-llm-vlm-service:
    image: artifacts.codelinaro.org/iot-solutions-microservices/genai-llm-vlm-service:latest
    cap_add:
      - SYS_RESOURCE          # raise memlock, not full privilege
    ulimits:
      memlock: { soft: -1, hard: -1 }   # pin memory for DSP buffers
    devices:
      - /dev/dma_heap/system        # shared (zero-copy) buffers CPU<->NPU
      - /dev/dma_heap/qcom,system
      - /dev/fastrpc-cdsp           # FastRPC channel to NPU0 (Hexagon cDSP)
      - /dev/fastrpc-cdsp1          # FastRPC channel to NPU1 (IQ-9075 has two)
    volumes:
      - /sys/firmware/devicetree/base/model:/run/device-model  # detect the board
      - ${QCOM:-/usr/share/qcom}:${HOST_QCOM:-/run/host-qcom}:ro # host Qualcomm libs/fw (READ-ONLY)
      - ${GENAI_MODEL_DIR:-/opt/models/}:/mnt/work/models/       # the model(s)
```

If you understand every line of that file, you understand the talk.
`docs/05` explains each line.
