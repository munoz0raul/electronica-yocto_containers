# 09 — Security is a lifecycle requirement (CRA, SBOM, CVE)

You're presenting in Munich → the **EU Cyber Resilience Act (CRA)** makes this concrete and
local. Keep the framing modest: these are *engineering foundations*, not automatic
compliance.

## The CRA in two dates (verify before you go on stage)
- **11 September 2026** — manufacturer **reporting obligations** apply (actively exploited
  vulnerabilities and severe incidents must be reported).
- **11 December 2027** — the **main requirements** become applicable.

⚠️ 🧠 **Double-check these dates against the current official text before Munich.** Neither
Foundries post nor the Dragonwing docs restated them; they are yours to defend. (The CRA
entered into force in 2024 with a staggered application timeline — confirm the exact days.)

## What the CRA pushes you to have (map each to this talk)
| CRA-driven need | The engineering practice | Where this talk already delivers it |
|-----------------|--------------------------|-------------------------------------|
| Know what's in your product | **SBOM** (Software Bill of Materials) | Yocto + container scans produce it per pipeline (`docs/08`) |
| Handle vulnerabilities | **CVE monitoring + triage** | SBOM-driven CVE feeds; patch host *and* app |
| Deliver security updates | **Secure OTA** | TUF-signed targets, A/B/OSTree rollback (`docs/08`) |
| Prove what you shipped | **Reproducible builds** | Yocto pinned layers/recipes; image digests |
| Protect the update chain | **Signing + verification** | TUF metadata, digest pinning |
| Support & end-of-life | **Defined maintenance + recovery** | release contract, health check, rollback |

## SBOM — the detail that sounds expert
🧠 An SBOM is only as good as its **reproducibility**. If you can't rebuild the exact image,
your SBOM "drifts" from reality. Yocto's pinned layers + container digests are what make the
SBOM trustworthy. Regenerate and archive it **with every release**, host and app separately.
Embedded makes this harder (custom Yocto builds, stripped cross-compiled metadata, mixed
open-source/proprietary) — which is exactly why you bake SBOM generation into CI, not bolt
it on later. 📎 (This framing aligns with Foundries' "Embedded Linux Security" guidance.)

## CVE triage — avoid the rookie answer
🧠 Don't say "we patch every CVE." Say: **prioritize exploitability over raw CVSS** (a high
score may need conditions your device never meets), automate CVE feeds mapped against the
SBOM to avoid alert fatigue, and weigh patch risk — **kernel updates are riskier than
user-space**, backport vs upstream-bump is a real tradeoff. This nuance reads as someone who
has actually maintained a fleet.

## Security ties back to containers (don't lose the thread)
- **Least privilege is a CRA-friendly default:** no `--privileged`, minimal `cap_add`,
  read-only mounts, non-root — exactly what Qualcomm's own container does (`docs/04`, `05`).
- **Attack surface = image contents:** minimal base images, no secrets in layers, pinned
  bases → smaller SBOM, fewer CVEs, faster triage.
- **The host/app split helps maintenance:** you can patch an app CVE by shipping a new
  container digest without touching the certified OS, and vice versa.

## The honest closing line
> "I'm not claiming Yocto, containers, CI/CD or TUF make you CRA-compliant. Compliance is a
> process. But these are the **engineering foundations** that make that process *manageable*
> instead of heroic — you can't do SBOMs, CVE triage, signed updates and recovery by hand on
> a fleet."
