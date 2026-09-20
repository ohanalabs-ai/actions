# docker/slsa-report

Verify a container image's **build provenance** and emit a Markdown report (step
summary, PR comment, release notes) with the exact command a consumer runs to
re-verify. Two mechanisms are supported because GitHub's own one does not cover
every repository:

| Mechanism | Produced by | Verified with | Available for |
|---|---|---|---|
| `github` — GitHub Artifact Attestations | `actions/attest-build-provenance` (+ `attest-sbom`) | `gh attestation verify oci://<image>@<digest> --repo <owner/repo> --signer-workflow <reusable-workflow>` | **public** repos on any plan; **private** repos only on **GitHub Enterprise Cloud** |
| `cosign` — Sigstore keyless | `cosign sign` + `cosign attest --type slsaprovenance1` in the build job (OIDC identity = the reusable workflow) | `cosign verify --certificate-identity-regexp '^https://github.com/<signer-workflow>@' --certificate-oidc-issuer https://token.actions.githubusercontent.com <image>@<digest>` | any repo, any plan |

## Why the private-repo case exists

A private repository that is not on GitHub Enterprise Cloud **cannot** store
GitHub artifact attestations — `actions/attest-build-provenance` cannot push and
`gh attestation verify` answers `HTTP 404` (e.g. `ohanalabs-ai/vault-mcp-server`
run 35250882375, while the identical pipeline passed on the public
`marcellodesales/mcp-brasil`, run 35491108003). There is **no org-level setting
to flip**; the only GitHub-native fix is the Enterprise plan. The platform
requirement that every `ohanalabs-ai` / `planeodev` image carries verifiable
provenance is therefore met with **cosign keyless** on private repos.

## Status semantics

| Status | Meaning | Step result |
|---|---|---|
| ✅ `verified` | an attestation exists and matches the signer identity (`attested_by=github` or `cosign`) | success |
| ❕ `not-attested` | nothing to verify **by design**: `attestations-pushed: "false"` and cosign not configured. Informational — the build itself is fine, the image just has no provenance yet | success (exit 0, `verified=false`) |
| ❌ `failed` | an attestation was expected (pushed, or cosign configured) but does **not** verify | failure when `fail-on-unverified: "true"` (default) |

`❌` is the only red case. A private repo that has not adopted cosign yet gets `❕`, never `❌`.

## Inputs (new ones in bold)

| Input | Default | Purpose |
|---|---|---|
| `subject` | — | `image@sha256:…` |
| `signer-workflow` | — | reusable workflow identity, e.g. `ohanalabs-ai/github-platform/.github/workflows/docker-multiarch-cicd.yaml` |
| **`mechanism`** | `auto` | `github` \| `cosign` \| `auto` (GitHub first; on 404/none fall back to cosign when `cosign-identity-regexp` is set) |
| **`attestations-pushed`** | `"true"` | pass the build's `PUSH_ATTESTATIONS`; `"false"` turns a missing GitHub attestation into ❕ |
| **`cosign-identity-regexp`** | `""` | e.g. `^https://github.com/ohanalabs-ai/github-platform/.github/workflows/docker-multiarch-cicd.yaml@` — empty disables cosign |
| **`cosign-oidc-issuer`** | `https://token.actions.githubusercontent.com` | Fulcio OIDC issuer |
| `fail-on-unverified` | `"true"` | only affects ❌ |
| `slsa-build-level`, `repo`, `gh-token`, `ref-name`, `title`, `image-base`, `digest` | as before | unchanged |

`cosign` is installed (pinned `sigstore/cosign-installer`) only when the cosign path can run.

## Outputs

Backward compatible: `verified`, `image`, `digest`, `subject`, `signer_workflow`,
`declared_slsa_build_level`, `summary_markdown`, `verification_command` (now the
command for the mechanism that verified). New: **`attested_by`**
(`github|cosign|none`) and **`status`** (`verified|not-attested|failed`).

## Usage

```yaml
- uses: ohanalabs-ai/actions/docker/slsa-report@main
  with:
    subject: ${{ needs.build.outputs.image_subject }}
    signer-workflow: ohanalabs-ai/github-platform/.github/workflows/docker-multiarch-cicd.yaml
    mechanism: auto
    attestations-pushed: ${{ env.PUSH_ATTESTATIONS }}
    cosign-identity-regexp: '^https://github.com/ohanalabs-ai/github-platform/.github/workflows/docker-multiarch-cicd.yaml@'
```

The consumer-side commands the report prints:

```bash
# public repo (GitHub attestations)
gh attestation verify oci://ghcr.io/<owner>/<image>@sha256:<digest> \
  --repo <owner>/<repo> \
  --signer-workflow ohanalabs-ai/github-platform/.github/workflows/docker-multiarch-cicd.yaml

# private repo (cosign keyless)
cosign verify \
  --certificate-identity-regexp '^https://github.com/ohanalabs-ai/github-platform/.github/workflows/docker-multiarch-cicd.yaml@' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  ghcr.io/<owner>/<image>@sha256:<digest>
```
