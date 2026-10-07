# 1. Toolchain install and pinning

- **Date:** 2026-10-06
- **Status:** accepted; private-package consequences superseded by
  [ADR 0003](0003-public-repos-and-packages.md)
- **Deciders:** ArtZ

## Context

mini-lineman spans four repos (`mini-lineman`, `-order`, `-payment`, `-proto`) and two language
ecosystems: Go for most services, Python/Django for payment. The build needs a Go toolchain,
`golangci-lint`, `protoc` plus `protoc-gen-go` and `protoc-gen-go-grpc`, and — for cluster work
— `kubectl`, `kind`, `helm` and `k9s`.

Nothing is pinned yet. Left alone, each developer and the CI runner install whatever version
their package manager offers that week, and the symptom is the one every team hits: a build or
a lint run that passes locally and fails in CI, with no way to tell which of the two is right.

Two constraints shaped the choice. Payment was deliberately separated into its own repo with
restricted access and an independent deploy cadence, so anything that makes payment's build
depend on a sibling service repo erodes a boundary that was drawn on purpose. And since Go 1.21
the Go toolchain resolves its own version from `go.mod`, which means the container's Go version
is not automatically the Go version that builds the code — that behaviour has to be chosen
explicitly rather than inherited.

## Options considered

**A — Install everything on the host, document versions in a README.**
Zero infrastructure, works offline, no Docker needed to start. But nothing enforces the
documented versions, so the README drifts from reality and the local/CI gap stays open. Teams
pick this for small single-developer projects where the toolchain rarely changes.

**B — One shared toolchain image for every repo.**
A single `docker pull` and every tool is present at a known version; one place to upgrade. But
one image means one Go version for all services, and an image carrying both the Go and the
Python/Django toolchains is large and mostly unused by any given build.

**C — One pinned toolchain image per service/repo.**
Each repo gets exactly the tools it needs, and repos upgrade independently. But the number of
images tracks the number of repos, every one needs building and refreshing, and common tooling
is duplicated across them.

**D — Devcontainer.**
Editor-integrated, one-click onboarding, and the same definition drives CI. But it binds the
workflow to editors that support it, and ArtZ wanted the toolchain usable from a bare terminal.

**E — Host version manager (asdf / mise).**
Pins versions per directory without containers and stays fast, with no image builds at all. But
it needs the manager installed everywhere including CI, and it cannot pin non-Go,
non-language-runtime binaries like `protoc` cleanly.

## Decision

A middle ground between B and C: **images split by ecosystem rather than by repo, with the
image — not `go.mod` — as the authority on the Go version.**

**1. Three toolchain images, split by ecosystem, not per service.**
One Go image (Go toolchain plus `golangci-lint`), one Python image for payment, and one proto
image for `protoc` and the codegen plugins. Rationale: an image per repo duplicates identical
Go tooling across repos, while a single image for everything forces the Go and Python
ecosystems into one artifact that every build pays for and no build fully uses. The ecosystem
is the boundary at which the tool set actually differs.

**2. The proto image lives in the proto repo; shared images live in a dedicated images repo.**
The proto image is specific to one repo's codegen step, so it stays with that repo. The Go and
Python images are consumed by several repos, so they live in a repo that exists only to build
and publish images.

Rationale for a dedicated repo rather than `mini-lineman`: `mini-lineman` is a service repo.
Putting shared images there would mean payment cannot build without an artifact produced by the
repo it was deliberately separated from, which partly undoes that separation. A dedicated
images repo is neutral — no service repo depends on a sibling service repo.

**3. `GOTOOLCHAIN=local` in the Go image. The image decides the Go version.**
Go's default (`GOTOOLCHAIN=auto`) downloads and uses whatever toolchain `go.mod` asks for when
it is newer than the local one, which makes the image's Go version a floor rather than a pin.
Setting `local` disables that: if a service's `go.mod` requires a newer Go than the image
provides, the build fails loudly instead of silently resolving. Upgrading Go is then an explicit
request to whoever owns the images repo, and every repo moves to the new version together.

**4. Cluster tools (`kubectl`, `kind`, `helm`, `k9s`) are documented, not containerised.**
They run against a cluster from the host, not inside a build. The README lists them with
required versions; nothing enforces it.

**5. Local Docker is optional. CI is the enforcement point.**
Developers may work on the host or in the images, their choice. CI runs the same pinned images
and fails the build on any mismatch, so a wrong local version is caught before merge rather
than prevented up front.

**6. Images are pulled by immutable pinned tags, never `:latest`.**
Each consuming repo pins an exact tag. Upgrading is a reviewable one-line commit that can be
reverted, and a build from six months ago can be reproduced. `:latest` would make `docker pull`
timing a hidden build input and reintroduce precisely the drift these images exist to remove.

## Consequences

### What this buys

- One place to change each ecosystem's toolchain, and the change arrives as a reviewable diff.
- A version mismatch between a developer and CI becomes a build failure with a clear cause,
  rather than a confusing difference in behaviour.
- Payment's build depends on nothing owned by a sibling service, so its access and cadence
  separation holds.
- Old builds are reproducible, because every input is pinned by an immutable tag.

### What it costs

- **Decision 3 overrides an earlier decision in this discussion** that each service's Go version
  would come from its own `go.mod`. Services no longer choose their Go version; the image does.
  A service that needs a newer Go is blocked until the image ships it.
- **Upgrading Go is now a multi-step, cross-repo operation:** a PR to the images repo, a
  publish, then a tag bump in every consuming repo. On a project where "devops" and "developer"
  are the same person, this is real friction bought for reproducibility that a solo project does
  not yet need. It is a deliberate rehearsal of how a larger team would do it.
- **Five repos, where the project documents four.** The images repo is a new thing to own, keep
  building, and keep credentials for.
- **A registry is now a dependency.** The images must be published somewhere both developers and
  CI can authenticate to, and an outage there blocks builds.
- **Pinned tags rot silently.** The failure mode inverts: instead of unexpected upgrades, the
  risk becomes sitting on a stale toolchain for a year because nothing forces the bump.
- **Decision 5 knowingly allows local/CI drift.** A developer who skips Docker can write code
  that passes locally and fails in CI. This is accepted: the cost is a failed CI run, and the
  alternative — mandating containers locally — was judged more friction than the failures are
  worth.
- Three images instead of one is more to build than option B, and the Go image is still a shared
  bottleneck across every Go repo.

### Reversals during this decision

Recorded because the reasoning matters more than the endpoint:

1. Started at **C** (one image per repo), accepting image proliferation as the cost.
2. Moved to **B** (one shared image), on grounds of simplicity for the people pulling it.
3. Landed between the two: **per-ecosystem images**, once it became clear that one image cannot
   serve both a Go and a Python toolchain without carrying dead weight for every build, and that
   payment's Python toolchain has nothing in common with the Go one.

An earlier intuition — that a small project should just track the latest official base image —
was withdrawn once the interaction with decision 3 was worked through: if the image is the
authority on the Go version but rebuilds from a moving base, the authority changes on every
rebuild, and because a moving base only advances, `go.mod`'s minimum stays satisfied and
`GOTOOLCHAIN=local` never fires. The drift would be silent rather than loud, which is worse
than the problem being solved.

## Open questions

Not yet decided. Each needs an answer before Phase 0 can be called done.

1. **What principle assigns an image to a repo?** Decision 2 states the outcome — proto image in
   the proto repo, shared images in the images repo — but not the rule that produced it. The
   working principle appears to be "an image used by one repo lives with it; an image used by
   several lives in the images repo", but ArtZ has not confirmed that wording, and the next image
   added will need it.
2. **What pins the custom images' own base?** The images are built `FROM` something. If that
   something is a moving tag, the silent drift described above returns one level down.
3. **What stops a pinned base from going stale?** Dropping the moving-base idea removed the one
   thing keeping tooling current. Nothing yet replaces it.
4. **Do services commit generated `*.pb.go`, or generate at build time?** Decision 1 gives
   `protoc` to the proto repo only, which implies services consume generated code rather than
   producing it — but this has not been decided explicitly.
5. **Are cluster tool versions verified or merely listed?** Decision 4 documents them. Whether
   a `make check-tools` target enforces the documented versions is undecided.
6. **Which registry, and how do developers and CI authenticate to it?** Not discussed.
7. **Who bumps the pinned tags — each repo, or the images repo owner centrally?** Raised by
   ArtZ while working through the staleness problem, undecided. Central bumping keeps the fleet
   consistent but makes one owner a bottleneck on every repo's tooling; per-repo bumping gives
   each service autonomy but lets the fleet diverge, which weakens the "one known environment"
   argument behind decision 1.

## Reasoning not yet captured

Honest gap, flagged rather than filled: the move from C to B (step 2 of the reversals) was made
without a stated reason beyond simplicity. The final per-ecosystem shape is closer to C than to
B, so the trajectory reads as a refinement rather than a reversal — but if asked "why not one
image per repo?", the answer currently rests on tooling duplication alone, which is a weaker
argument than the payment-isolation reasoning carrying decision 2.
