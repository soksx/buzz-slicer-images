# Buzz Slicer runner images

Public OCI images for the `buzz-backend-slicer` provider. The initial runner
is `ghcr.io/soksx/buzz-slicer-codex` for `linux/arm64` Slicer for Mac hosts.

The image provides Bun, `buzz`, `buzz-acp`, `buzz-relay`,
`git-credential-nostr`, Codex, Codex ACP, RTK, and an unprivileged `buzz`
account. It contains no Buzz relay credentials, Codex login, API tokens, or
Nostr identity.

## Publishing

Only a push to `main` or a manually dispatched workflow can publish. Pull
requests receive a build-only job with a read-only token; they cannot log in to
GHCR or publish an image. Publishing uses GitHub's short-lived
`GITHUB_TOKEN`, not a personal access token. The workflow pins every action by
commit SHA and attaches provenance and an SBOM to the published image.

The Dockerfile pins the Slicer base image, Buzz relay distribution, Node, Bun,
and Rust inputs by digest. It also checks RTK's release archive SHA-256.

## Use with Slicer

After a successful publish, copy the *manifest digest* from the workflow and
pin Slicer's image field to it. Do not use a mutable tag for production:

```yaml
config:
  image: "ghcr.io/soksx/buzz-slicer-codex@sha256:<manifest-digest>"
```

Restart the Slicer daemon after changing its image. In Buzz Desktop, choose
`host_group: sbox`, leave `template_commit` empty, and choose `runner: codex`.
Each new agent then gets an ephemeral VM from the digest-pinned root image.

The GHCR package is public, so Slicer can pull it without a registry credential.
Never embed registry tokens in this repository, a Dockerfile, or Buzz provider
settings. If you publish a private derivative, provide its credential to each
Slicer daemon through the daemon's secret configuration instead.
