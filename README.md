# Build Multi-Arch Docker Action

A reusable GitHub Actions workflow that builds multi-architecture Docker images **natively** on amd64 and arm64 runners (no QEMU emulation), with optional binary extraction and release publishing.

## Features

- **Native multi-arch builds**: Builds on actual amd64 (`ubuntu-26.04`) and arm64 (`ubuntu-26.04-arm`) hardware
- **No QEMU emulation**: True native builds for both architectures
- **Multi-arch manifest**: Creates a single Docker image manifest containing both architectures
- **Binary extraction**: Optionally extracts binaries from built images
- **Release publishing**: Uploads extracted binaries to GitHub releases
- **SLSA attestation**: Creates build provenance attestations for extracted binaries
- **Disk cleanup**: Optional runner disk space optimization for faster builds

## Usage

### Basic: Build and publish multi-arch Docker image

```yaml
name: Build and publish

on:
  release:
    types: [published]
  workflow_dispatch:

jobs:
  build:
    uses: sctg-development/build-multi-arch-docker-action/.github/workflows/build.yml@v1
    with:
      dockerfile: Dockerfile
      image: myorg/myimage
      artifact-prefix: myimage-${{ github.run_id }}
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

### With binary extraction and release upload

```yaml
name: Build, extract and publish

on:
  release:
    types: [published]

permissions:
  contents: write
  id-token: write
  attestations: write

jobs:
  build:
    uses: sctg-development/build-multi-arch-docker-action/.github/workflows/build.yml@v1
    with:
      dockerfile: Dockerfile
      image: myorg/myimage
      artifact-prefix: myimage-${{ github.run_id }}
      extract-binary: true
      binary-path: /usr/local/bin/myapp
      binary-name: myapp
      release-tag: ${{ github.event.release.tag_name }}
      attest: true
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `dockerfile` | No | `Dockerfile` | Path to the Dockerfile to build |
| `image` | Yes | - | Docker Hub image name (e.g., `user/repo`) |
| `context` | No | `.` | Docker build context |
| `tags` | No | `''` | Tags in `docker/metadata-action` format (multiline); empty = defaults |
| `build-args` | No | `''` | Build arguments (multiline `KEY=VALUE`) |
| `artifact-prefix` | No | run id | Unique prefix for artifacts to avoid conflicts |
| `cleanup` | No | `true` | Clean up runner disk space before building (recommended) |
| `extract-binary` | No | `false` | Extract binary from the built image |
| `binary-path` | No | `''` | Path to the binary inside the container |
| `binary-name` | No | `''` | Base name for the extracted binary |
| `release-tag` | No | `''` | GitHub release tag name for upload |
| `attest` | No | `false` | Create SLSA build provenance attestation |

## Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `DOCKER_USERNAME` | Yes | Docker Hub username |
| `DOCKER_PASSWORD` | Yes | Docker Hub password or access token |

## Outputs

| Output | Description |
|--------|-------------|
| `image-digest` | SHA256 digest of the built multi-arch image |

## Architecture

This workflow uses GitHub's native architecture-specific runners:
- **amd64**: `ubuntu-26.04`
- **arm64**: `ubuntu-26.04-arm`

Each architecture is built **natively** (no QEMU emulation), then the digests are merged into a single multi-arch manifest.

## How It Works

1. **Build Phase**: Two parallel jobs build the Docker image on amd64 and arm64 runners
2. **Digest Export**: Each job exports its image digest as an artifact
3. **Merge Phase**: A third job downloads both digests and creates a multi-arch manifest
4. **Binary Extraction** (optional): each native build job extracts its binary; a final job uploads them to the release
5. **Attestation** (optional): Creates SLSA provenance attestations for extracted binaries

## Best Practices

1. **Artifact prefix**: Use a unique prefix for each image to avoid conflicts between concurrent builds
   ```yaml
   artifact-prefix: myapp-${{ github.run_id }}
   ```

2. **Cleanup**: Enable cleanup for faster builds, especially in repositories with large dependencies
   ```yaml
   cleanup: true
   ```

3. **Binary extraction**: Only enable if you need to distribute standalone binaries
   ```yaml
   extract-binary: true
   binary-path: /usr/local/bin/myapp
   binary-name: myapp
   release-tag: ${{ github.event.release.tag_name }}
   ```

4. **Attestation**: Enable for security-critical projects
   ```yaml
   attest: true
   ```

## Complete Example

```yaml
name: CI/CD

on:
  release:
    types: [published]
  workflow_dispatch:

permissions:
  contents: write
  id-token: write
  attestations: write

jobs:
  build-and-publish:
    uses: sctg-development/build-multi-arch-docker-action/.github/workflows/build.yml@v1
    with:
      dockerfile: Dockerfile
      image: myorg/myapp
      artifact-prefix: myapp-${{ github.run_id }}
      cleanup: true
      extract-binary: true
      binary-path: /usr/local/bin/myapp
      binary-name: myapp
      release-tag: ${{ github.event.release.tag_name }}
      attest: true
    secrets:
      DOCKER_USERNAME: ${{ secrets.DOCKER_USERNAME }}
      DOCKER_PASSWORD: ${{ secrets.DOCKER_PASSWORD }}
```

## Requirements

- Docker Hub account with push permissions
- GitHub repository with write permissions for releases (if using binary extraction)
- GitHub token with `id-token: write` permission (if using attestation)

## License

MIT
