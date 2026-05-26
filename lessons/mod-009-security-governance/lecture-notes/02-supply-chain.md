# Lecture 02: Supply Chain Security

## The chain

```
Code → CI → Image → Registry → K8s admission → Pod → Pod talks to Vault
```

Every link is an attack surface.

## SLSA framework

[SLSA](https://slsa.dev/) defines 4 levels. For an ML platform, target SLSA L2:
- Provenance generated automatically (GitHub Actions can produce this)
- Provenance signed (cosign keyless)
- Hosted build service
- Source integrity (every build references an immutable commit)

## Image signing + verification

```bash
# Sign in CI:
cosign sign --yes ghcr.io/me/iris-api:0.6

# Verify at admission via Kyverno:
- imageReferences: ["ghcr.io/me/*"]
  attestors:
    - keyless:
        subject: "https://github.com/me/*/.github/workflows/*"
        issuer:  "https://token.actions.githubusercontent.com"
```

## Model artifacts

Model files are also part of the supply chain:
- Use safetensors (avoids the unsafe-deserialization risk of older formats)
- Sign model artifacts when published to registry
- Verify hash at load time

## Companion

[engineer-solutions/mod-103 ex-10 (sbom-and-supply-chain)](https://github.com/ai-infra-curriculum/ai-infra-engineer-solutions/tree/main/modules/mod-103-containerization/exercise-10-sbom-and-supply-chain).
