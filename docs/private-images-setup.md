# Setting up private toolchain images on ghcr.io

Implements ADR 0001. Three toolchain images, published from two repos to GitHub Container
Registry as **private** packages, pinned by digest, consumed by CI in four repos.

Status: **steps 1–6 done** on 2026-10-07. Steps 7–9 (access grants, local login, consuming
repo CI) not yet run. Versions and digests verified 2026-10-07.

Published:

```
ghcr.io/surakietart/mini-lineman-go-tools:2026-10-07-6eb6831
  @sha256:2ed93679668750c246221f3a63d037982383cac2e14806698c2eba4d8486ca01
ghcr.io/surakietart/mini-lineman-py-tools:2026-10-07-6eb6831
  @sha256:eef5bd78092eb650c82fc047ba3883c9f927dee8c1271165380fb0b6191fcb59
```

---

## What you are building

| Image | Built from | Contains |
|---|---|---|
| `mini-lineman-go-tools` | `mini-lineman-images` | Go toolchain, `golangci-lint` |
| `mini-lineman-py-tools` | `mini-lineman-images` | Python, Django tooling |
| `mini-lineman-proto-tools` | `mini-lineman-proto` | `protoc`, `protoc-gen-go`, `protoc-gen-go-grpc` |

Consumers: `mini-lineman`, `mini-lineman-order` (Go), `mini-lineman-payment` (Python),
`mini-lineman-proto` (proto).

---

## Versions — resolved 2026-10-07

Latest stable of each, looked up from the authoritative source on that date.

| Tool | Version | Source | Previous stable |
|---|---|---|---|
| Go | `1.27.1` | `go.dev/dl/?mode=json` | `1.26.8` |
| `golangci-lint` | `v2.14.0` | GitHub releases, 2026-09-24 | — |
| Python | `3.14.8` | endoflife.date (3.14 EOL 2030-10-31) | `3.13.16` |
| `protoc` | `v36.2` | GitHub releases, 2026-09-17 | — |
| `protoc-gen-go` | `v1.36.12` | `protocolbuffers/protobuf-go`, 2026-08-10 | — |
| `protoc-gen-go-grpc` | `v1.6.2` | `grpc/grpc-go`, tag `cmd/protoc-gen-go-grpc/v1.6.2` | — |

Base image digests, resolved the same day:

```
golang:1.27.1   sha256:162be5298a40ed317005c8339c6de4d10d3eef336d66dc8e9259b03ab9d3a6d2
python:3.14.8   sha256:1eb6b7d4b76454b1de8317863ac3213b678c337b27e604a4e3fb70bddbb2bad7
```

**These digests are already going stale.** Re-resolve them with the commands in step 2 before
the first build; the numbers here are a snapshot, not a guarantee.

One toolchain-compatibility note, since "always latest" can break lint: Go 1.27.0 shipped
2026-08-19 and `golangci-lint` v2.14.0 shipped 2026-09-24, so the linter postdates the compiler
and should handle Go 1.27 syntax. That ordering is worth checking on every future Go bump — a
linter older than the Go release it is asked to parse is the usual way this combination fails.

Still yours to fill in:

- [ ] Your GitHub account or org name (used as `<owner>` throughout)

---

## Step 1 — Create the images repo

```bash
cd /home/artz/mini-lineman
gh repo create mini-lineman-images --private --clone
cd mini-lineman-images
mkdir -p go-tools py-tools .github/workflows
```

Private repo, because the packages will be private and the Actions-access grants are simpler
when ownership is consistent.

## Step 2 — Resolve the base image digests

A tag is a pointer and GitHub or Docker can move it. `golang:1.27.1` gets rebuilt and re-pushed
when the underlying Debian is patched, so the same tag is different bytes a month later. A
digest is content-addressed and cannot move.

Re-run these at build time rather than trusting the values recorded above:

```bash
docker buildx imagetools inspect golang:1.27.1 | awk '/^Digest:/{print $2; exit}'
docker buildx imagetools inspect python:3.14.8 | awk '/^Digest:/{print $2; exit}'
```

Three things about that command:

- No `docker pull` is needed; `imagetools` queries the registry directly.
- Use `imagetools`, not `docker inspect`. `imagetools` returns the **manifest-list** digest,
  which covers every architecture; a digest read off a pulled single-arch image will fail to
  pull on a machine with a different CPU.
- Read the top-level `Digest:` line. The tidier-looking `--format '{{.Manifest.Digest}}'`
  returned a per-architecture entry when actually run, not the list digest.

## Step 3 — Write the Dockerfiles

`go-tools/Dockerfile`:

```dockerfile
# golang:1.27.1 — resolved 2026-10-07. Bump via the digest, never the tag.
FROM golang@sha256:162be5298a40ed317005c8339c6de4d10d3eef336d66dc8e9259b03ab9d3a6d2

# Install tools BEFORE locking the toolchain: some tools' own go.mod
# requires a newer Go, and GOTOOLCHAIN=local would refuse to build them.
RUN go install github.com/golangci/golangci-lint/v2/cmd/golangci-lint@v2.14.0

# ADR 0001 decision 3: the image decides the Go version, not each service's go.mod.
# A repo requiring newer Go now fails loudly instead of silently downloading it.
ENV GOTOOLCHAIN=local

WORKDIR /src
```

The `/v2/` in that import path is required and verified: `v2.14.0` is published under
`github.com/golangci/golangci-lint/v2` in the module proxy. The pre-v2 path will not resolve
this version.

Always keep that first comment. A bare 71-character hash tells the next reader nothing about
which version they are on.

`py-tools/Dockerfile` follows the same shape:

```dockerfile
# python:3.14.8 — resolved 2026-10-07
FROM python@sha256:1eb6b7d4b76454b1de8317863ac3213b678c337b27e604a4e3fb70bddbb2bad7
WORKDIR /src
```

`proto-tools/Dockerfile` lives in `mini-lineman-proto`, not here, and pins
`protoc` v36.2, `protoc-gen-go` v1.36.12 and `protoc-gen-go-grpc` v1.6.2. `protoc` is a
release-archive download rather than a `go install`, so pin its checksum as well as its
version.

## Step 4 — Fix the tag scheme before the first push

**ghcr does not enforce tag immutability.** Nothing stops a later push from overwriting
`:2026-10-07`. ADR 0001's "immutable tags" is a convention you have to keep, so put the commit
SHA in the tag and never reuse one:

```
ghcr.io/<owner>/mini-lineman-go-tools:2026-10-07-a1b2c3d
```

## Step 5 — Publish workflow

`.github/workflows/publish.yml`:

```yaml
name: publish images
on:
  push:
    branches: [main]

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write          # lets GITHUB_TOKEN push to ghcr
    strategy:
      matrix:
        image: [go-tools, py-tools]
    steps:
      - uses: actions/checkout@v7

      - uses: docker/setup-buildx-action@v4

      - uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      # Owner is lowercased: OCI repository names may not contain uppercase.
      - name: Compute image reference
        id: ref
        run: |
          echo "owner=${GITHUB_REPOSITORY_OWNER,,}" >> "$GITHUB_OUTPUT"
          echo "tag=$(date -u +%Y-%m-%d)-${GITHUB_SHA::7}" >> "$GITHUB_OUTPUT"

      - uses: docker/build-push-action@v7
        with:
          context: ${{ matrix.image }}
          push: true
          # provenance off: buildx otherwise pushes an extra attestation manifest
          # that appears as an "unknown/unknown" version and muddies digest checks.
          provenance: false
          tags: ghcr.io/${{ steps.ref.outputs.owner }}/mini-lineman-${{ matrix.image }}:${{ steps.ref.outputs.tag }}
```

**The lowercase step is not optional.** `github.repository_owner` is `SurakietArt`, and the
first run failed with `invalid tag ...: repository name must be lowercase`. Any GitHub account
with a capital letter in it hits this.

Note the trigger is `branches: [master]` — this repo's default branch is `master`, not `main`.
A workflow pointed at the wrong branch does not error, it simply never runs.

`GITHUB_TOKEN` is minted per run and needs no stored secret. This only works for pushing to
packages owned by *this* repo — which is why the consuming side needs step 7.

## Step 6 — First publish

```bash
git add . && git commit -m "feat(images): add Go and Python toolchain images"
git push -u origin main
gh run watch
```

Then confirm the packages exist:

```bash
gh api /user/packages?package_type=container --jq '.[].name'
```

## Step 7 — Grant each consuming repo access (the private-specific part)

A package published from `mini-lineman-images` is owned by that repo. Another repo's
`GITHUB_TOKEN` is scoped to its own repo and **cannot pull a private package it was not granted
access to**. Without this step, CI in every consuming repo fails with `denied` or
`manifest unknown`.

For each image, in the browser:

1. `https://github.com/users/<owner>/packages/container/mini-lineman-go-tools/settings`
2. **Manage Actions access** → **Add repository**
3. Add `mini-lineman`, `mini-lineman-order` — role **Read**
4. Repeat on `mini-lineman-py-tools` for `mini-lineman-payment`
5. Repeat on `mini-lineman-proto-tools` for every repo that runs codegen

Optionally use **Connect repository** to link the package to its source repo, which only affects
where the package appears in the UI.

This is a manual grant per image per repo, repeated whenever you add either. It is the
operational cost of choosing private, and it is the step people forget — then spend an hour
debugging a CI auth failure that is actually a missing checkbox.

## Step 8 — Your own login, for pulling locally

CI uses `GITHUB_TOKEN`. You cannot; `gh auth login` does not authenticate `docker`.

1. Create a **classic** PAT with scope `read:packages`
   (`https://github.com/settings/tokens`)
2. ```bash
   echo "$GHCR_TOKEN" | docker login ghcr.io -u <owner> --password-stdin
   ```

Do not commit the token. ADR 0001 decision 5 makes local Docker optional, so this step is
optional too — skip it if you work on the host and let CI be the judge.

## Step 9 — Consume the image in a service repo

This is where decision 5 becomes real: CI runs *inside* the pinned image, so a version mismatch
is impossible in CI regardless of what is installed on anyone's laptop.

`.github/workflows/ci.yml` in `mini-lineman-order`:

```yaml
name: ci
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    container:
      image: ghcr.io/<owner>/mini-lineman-go-tools:2026-10-07-a1b2c3d
      credentials:
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}
    steps:
      - uses: actions/checkout@v4
      - run: go build ./...
      - run: go test ./...
      - run: golangci-lint run
```

The tag is pinned here, in the consuming repo. Upgrading is a one-line reviewable commit — which
is the whole point of decision 6.

---

## Verification checklist

- [ ] `gh api /user/packages?package_type=container` lists all three images
- [ ] Each package's visibility reads **private**
- [ ] Each consuming repo appears under that package's Actions access
- [ ] A CI run in `mini-lineman-order` pulls the image and passes
- [ ] `docker pull` of the image works locally after step 8
- [ ] A repo whose `go.mod` demands a newer Go than the image **fails** — proves
      `GOTOOLCHAIN=local` is actually in effect, rather than assumed

That last one is the only check that proves decision 3 works. Test it deliberately once.

---

## Costs this setup carries

**Private package storage is metered.** Public packages are free; private ones consume a quota
(GitHub Free includes roughly 500 MB storage and 1 GB/month data transfer — verify against
current billing docs before relying on it). A Go toolchain image is a few hundred MB compressed,
and every immutable tag is a new stored version.

This collides directly with decision 6. "A build from six months ago is reproducible" requires
keeping old versions; the quota pushes you to delete them. One of the two has to give, and that
choice has not been made. Data transfer via `GITHUB_TOKEN` inside Actions does not count, so CI
pulls are free — it is storage and local pulls that accumulate.

**Tag immutability is yours to maintain**, not the registry's. One careless re-push and the
guarantee is gone with no error.

**The PAT in step 8 is a long-lived credential** on your machine with no rotation story.

**Every new image or repo means new manual grants** in step 7.

---

## Still undecided (ADR 0001 open questions)

1. What moves the digest. "Devops manages it" is an owner, not a mechanism — nothing prompts
   the bump, so digest pinning currently guarantees staleness rather than preventing it.
   Renovate or Dependabot watching the digest would close this.
2. Whether old image versions are retained or pruned — see the storage quota above.
3. Who bumps the pinned tag in each consuming repo: each repo, or centrally.
4. Whether services commit generated `*.pb.go` or generate at build time. This decides whether
   the proto image needs Actions access in every repo, or only in `mini-lineman-proto`.
