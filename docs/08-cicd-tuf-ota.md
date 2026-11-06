# 08 — Build separately, validate together: CI/CD, TUF, OTA

## Two pipelines, one release
```
Platform pipeline:    git + Yocto recipes ──► OS image   ──► build, test, scan, SBOM
Application pipeline:  git + Dockerfile    ──► OCI image  ──► build, test, scan, publish digest
                                   │                │
                                   └──── INTEGRATION GATE ────┘
          (correct OS + correct containers + REAL-hardware tests)
```
- Each pipeline runs independently with its **own** tests, security scans, and
  software-component inventory (SBOM).
- They converge at an **integration gate** before release. Questions the gate answers:
  - Does this OS expose the expected driver/firmware?
  - Can the container actually reach the camera / NPU on real silicon?
  - Does inference run on the chosen accelerator at the expected latency?
  - Can the device recover from a failed update?
- Slogan: **"Passing a container build is not proof that an AI device is ready to ship."**

🧠 This gate is literally where the `docs/03` compatibility surfaces get tested: it is the
only place the (OS commit ↔ runtime ↔ model) triple is exercised on hardware.

## TUF: a trusted description of a compatible release
**TUF = The Update Framework.** It is a security framework for **signed update metadata**:
it authenticates *target* file names, sizes and hashes, and protects against rollback and
several registry/repo attacks. It does **not** itself define what a "product" is.

🧠 The useful trick: TUF **Targets** metadata allows **custom fields**. You can describe a
compatible release in them:

```json
"iq9-release-42.json": {
  "length": 214,
  "hashes": { "sha256": "…" },
  "custom": {
    "hardwareIds": ["iq9"],
    "os_commit":   "abc123…",
    "containers":  { "vision": "registry/vision@sha256:…" }
  }
}
```

Be honest on stage (this is a credibility move):
- This schema is **illustrative**, *not* a standard TUF definition of `os_commit`/`containers`.
- TUF signs and authenticates; the **OTA client** must know how to *interpret* these custom
  fields, verify compatibility, and apply the artifacts.
- So TUF doesn't "pack a product into one file" — it gives you a **trusted release
  description** an OTA system can coordinate against.

📎 Real-world anchor: **FoundriesFactory uses TUF** so that *"every update bundle and target
image is cryptographically signed and verified."* Good external validation to cite.

## The OTA flow (slide 14) — five steps, and the one everyone forgets
```
1 SELECT    choose a compatible, signed target
2 VERIFY    check metadata signatures + artifact digests
3 DOWNLOAD  retrieve the OS / OCI artifacts
4 ACTIVATE  restart app  (model-only)  OR  switch OS + reboot  (kernel/driver)
5 CONFIRM   health check + report version  — else ROLL BACK
```
- **Activation depends on what changed:** a model-only update may just replace a container
  image and restart one service; a kernel/driver update needs a new OS artifact, a reboot,
  and recovery support.
- 🧠 **Step 5 is the point of the slide:** *"A successful download is not a successful product
  update."* Shipping = device is **healthy on the new version**, or has safely rolled back.

🧠 Mechanism to name if asked how activation/rollback works safely: **A/B partitions** or
**OSTree**-style atomic updates with rollback. (Qualcomm Linux documents capsule + OSTree
update mechanisms; Foundries' LmP uses OSTree.) The container half rolls back simply by
re-pinning the previous image **digest**.

📎 Foundries pattern worth citing — **models as separate Compose App bundle artifacts**
(not baked into the image): FoundriesFactory publishes a new OTA target referencing the
updated model, the model dir is bind-mounted into the container, and the model *"evolves
independently from the base OS image."* That is exactly `docs/05` block (I) at fleet scale.

## The sentence
> "Build the pieces independently. Converge at an integration gate on real hardware. Let
> TUF describe and sign the compatible release, let OTA apply it — and don't call it done
> until the device reports healthy or rolls back."
