# Exercise 03: SLSA L2 Supply Chain

Set up CI to:
- Build + push image
- Generate SBOM via syft
- Sign with cosign keyless
- Attest SBOM
- Verify in cluster via Kyverno admission policy

Demonstrate: unsigned image is rejected at admission.

Companion: engineer-solutions/mod-103 ex-10.
