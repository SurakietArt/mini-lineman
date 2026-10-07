# Phase 0 — Setup and toolchain

Session log: 2026-10-06, 2026-10-07

---

## 1. Concepts learned

### `GOTOOLCHAIN`, and the two version directives in `go.mod`

`go.mod` carries two separate things: `go 1.25` (the minimum version, and which language
semantics apply) and optionally `toolchain go1.25.3` (which toolchain to actually use). Since
Go 1.21, with `GOTOOLCHAIN=auto` — the default — the `go` command will **download and use** a
newer toolchain if `go.mod` asks for one the machine doesn't have.

The consequence most people miss: a container's Go version is therefore a **floor, not a pin**.
`FROM golang:1.25` plus `go 1.26` in `go.mod` builds with 1.26, not 1.25, and nothing warns you.
It only ever moves up, never down.

**Failure mode:** you believe a pinned image guarantees one compiler across the fleet. It
doesn't. Two repos using the identical image build with different compilers, and the difference
shows up as a runtime or GC behaviour change nobody can trace to a version. `GOTOOLCHAIN=local`
turns the silent resolution into a loud build error.

### `go.mod` can pin tools, not just libraries

Go 1.24 added `tool` directives to `go.mod` (`go get -tool`, then `go tool <name>`). This pins
**Go-based** build tools — `golangci-lint`, `protoc-gen-go`, `protoc-gen-go-grpc`, `mockgen`,
`migrate` — in the same reviewable file as the libraries. It cannot pin non-Go binaries:
`protoc` itself, `kubectl`, `kind`, `helm`.

**Failure mode:** pinning the version in `go.mod` but having the Makefile call bare
`golangci-lint`. The pin is then decorative — the binary that runs is whatever is first on
`$PATH`. The Makefile has to call `go tool golangci-lint` for the pin to mean anything.

### Image tags are a build input

`:latest` makes *the time you ran `docker pull`* part of your build. Two developers a week apart
get different tools; a build that passed last month cannot be reproduced. Immutable tags (a date
or a git SHA) move the upgrade into a reviewable, revertible commit.

**Failure mode, both directions.** `latest`: unannounced breakage, and on a *shared* image one
bad publish breaks every repo at once. Pinned: silent rot — nothing fails, you are simply a year
behind, and the eventual upgrade is a project instead of a commit.

### Build artifacts create repo coupling

If repo A's build needs an image published by repo B, then A depends on B regardless of what the
code imports. This matters here because payment was separated for access control and deploy
cadence — putting the shared images in `mini-lineman` would have made payment's build depend on
the repo it was separated from. A neutral images repo keeps the dependency pointing at something
that isn't a sibling service.

**Failure mode:** declaring services "independent" while every one of them is gated on a build
artifact owned by one team. The independence is on paper only, and you find out during an
incident when that team's pipeline is red.

### Blast radius of shared artifacts

Consolidation and blast radius trade off directly. One shared toolchain image is less to
maintain, and one bad publish breaks everything that pulls it. Per-repo images contain the
damage and multiply the maintenance. There is no free position — only a choice about which
failure you'd rather have.

### OCI naming rules are stricter than GitHub's

Container repository names may not contain uppercase characters. `github.repository_owner`
returns the account name exactly as GitHub stores it, so an account like `SurakietArt` produces
an invalid image reference and buildx rejects the push outright.

**Failure mode:** the error arrives as `invalid tag ...: repository name must be lowercase`
from the build step, which reads like a Dockerfile problem and is actually an identity problem.
Lowercase it in the workflow (`${GITHUB_REPOSITORY_OWNER,,}`) rather than hardcoding the name.

### A workflow pointed at the wrong branch does not fail

`on: push: branches: [main]` in a repo whose default branch is `master` produces no error, no
warning and no run. The pipeline looks configured and is inert.

**Failure mode:** you push, see nothing, and start debugging credentials or permissions — the
loudest-looking part of the system — when the trigger never matched. Check that a run exists
before debugging why a run failed.

### `GITHUB_TOKEN` is scoped to its own repository

The token minted for a workflow run can push packages owned by *that* repo and nothing else. A
package published from one repo is invisible to another repo's token until access is granted
explicitly.

**Failure mode:** CI in a consuming repo fails with `denied` or `manifest unknown`, which looks
like a registry or auth outage and is actually a missing per-repo grant. This is the hidden cost
of private packages across a multi-repo layout.

---

## 2. Decisions made

### [ADR 0001 — Toolchain install and pinning](../decisions/0001-toolchain-install.md)

Three toolchain images split by **ecosystem** (Go, Python, proto) rather than per repo. Proto
image lives in the proto repo; shared images live in a dedicated images repo. `GOTOOLCHAIN=local`
makes the image authoritative over `go.mod` for the Go version. Cluster tools documented, not
containerised. Local Docker optional, CI as the enforcement point. Immutable pinned tags.

**How to say this in an interview:**
"We split build images by ecosystem rather than per service — the Go and Python toolchains have
nothing in common, so one image for both means every build carries weight it never uses. We set
`GOTOOLCHAIN=local` so the image, not each repo's `go.mod`, decides the Go version; we traded
per-service autonomy for a guarantee that CI and developers compile with the same toolchain."

**And the honest second half, which is what makes it credible:**
"The cost is that upgrading Go became a cross-repo operation — image PR, publish, then a tag
bump everywhere. On a four-repo project that's more ceremony than the problem deserves; I chose
it to rehearse how a larger team would handle it, and I'd probably let `go.mod` decide on a
smaller codebase."

### [ADR 0002 — Defer AWS to the final phase](../decisions/0002-defer-aws.md)

AWS moves to the end of the plan. Kubernetes is learned on `kind` at zero cost first; phases 0–6
proceed unchanged.

**How to say this in an interview:**
"AWS wasn't a requirement for the roles I was targeting, and EKS plus a NAT gateway costs around
$100 a month before you deploy anything, so I learned Kubernetes locally on kind first and left
the cloud port as a deliberate final exercise."

**And the cost, which is the half that matters:**
"The risk I took is that a final phase is the phase most likely to get dropped — so until I do
it, I have Kubernetes experience and no managed-service experience. kind doesn't teach you cloud
load balancers, IAM-to-pod identity or control-plane upgrades, and I'd say that plainly rather
than imply the two are equivalent."

---

## 3. Questions & model answers

### Q1 — "Explain why `GOTOOLCHAIN=local`, and what you gave up."

**ArtZ's answer:** "Is single source of truth to let go version manage by image."

**Feedback:** Core is right but too thin to survive a follow-up. "Single source of truth" is a
phrase, not a mechanism — it doesn't show you know what the default does, so it can't show what
you turned off. And the question asked what you gave up; that half wasn't answered. In
interviews the cost side is where the signal is: anyone can state a benefit.

**Model answer:**
"By default Go reads `go.mod` and, if it asks for a newer toolchain than the container has,
downloads and uses it. That makes the image's Go version a floor rather than a pin — two repos
on the same image can end up building with different compilers. `GOTOOLCHAIN=local` disables
that: the image's Go is the only Go, and a repo requiring newer fails with a clear error instead
of silently resolving. What I gave up is per-service Go autonomy — upgrading now means changing
the image and bumping the tag in every repo, so services move in lockstep. I took that trade
because consistency between CI and developer machines mattered more to me here than letting each
service pick its own compiler."

### Q2 — "You've pinned your build tools to immutable tags. Six months later, what's gone wrong?"

**ArtZ's answer:** "Actually it can get latest tag but it possible to get bug the whole system
from 1 image update. So, let each team manage tags or let devops manage it on all services."

**Feedback:** This answered a different question — it gave the cost of `latest`, the option
*not* chosen, rather than the cost of pinning. Worth catching as a habit: asked for the downside
of his own decision, he defended the decision. Interviewers read that as a design that hasn't
been stress-tested, and it's the single most costly reflex to carry into an interview.

The blast-radius observation inside it was genuinely strong, though — "one image update can bug
the whole system" is precisely the argument for pinning a *shared* artifact, and it's a better
defence of the decision than anything in the original reasoning. It just wasn't the answer to
this question. He also raised an unasked and real question — who owns the tag bump — which went
into ADR 0001 as open question 7.

**Model answer:**
"Nothing has failed, which is the problem. We're six months behind: the Go version may be out of
security support, `golangci-lint` is missing rules we assume we have, and the first person who
needs a newer Go is blocked on an image upgrade. The upgrade itself has become expensive —
six months of changes arrive at once, so a routine bump turns into a project with its own
rollback risk. The fix is to make the bump scheduled rather than event-driven: automated PRs
against the pinned tag, or a standing cadence, so the pin moves in small reviewable steps. The
point of pinning was never to stop upgrading, it was to make upgrading deliberate."

---

## 4. Mistakes and what they taught

### Pinning the image while letting its base float

ArtZ decided the image would be authoritative over the Go version, then suggested it be built
from the latest official base image — a reasonable small-project instinct, applied one layer
too low. If the image is the authority but rebuilds from a moving base, the authority changes on
every rebuild.

Worse, the failure is silent rather than loud: a moving base only advances, so `go.mod`'s minimum
stays satisfied and the `GOTOOLCHAIN=local` error never fires. He withdrew it once the
interaction was worked through.

**The lesson, and the interview story:** pinning is only as strong as the weakest unpinned layer
beneath it. "We pinned the image" means nothing if the Dockerfile says `FROM golang:latest`. This
is a good answer to "tell me about a design flaw you caught before it shipped."

### The first CI run failed on an uppercase letter

The publish workflow was correct in every way that felt important — digest-pinned base, scoped
token, matrix build, immutable tag — and failed because the GitHub account name contains capital
letters and OCI repository names may not. Both matrix legs failed identically.

**The lesson, and the interview story:** the thing that breaks a pipeline first is almost never
the part you designed carefully. It is a naming or encoding rule in a layer you did not think
about. Good answer to "tell me about a time something failed for a reason you didn't expect."

### Three reversals in one decision

The trajectory was per-repo images → one shared image → per-ecosystem images, plus one
superseded decision (`go.mod` owns the Go version, killed by `GOTOOLCHAIN=local`). The ADR
records all of it on purpose.

**The lesson:** the reversals *are* the interview material. "Why not one image per repo?" is only
answerable because that option was held and abandoned for a reason. A decision that arrives
fully-formed has no defensible reasoning attached to it.

---

## 5. Still shaky — re-ask next session

1. **Stating the cost of his own decisions.** Both answers this session named benefits and
   skipped trade-offs, and Q2 substituted the alternative's cost for his own. Re-test directly:
   pick any decision in ADR 0001 and ask only "what does this cost you?"
2. **Why per-ecosystem beat per-repo.** Still rests on tooling duplication alone. The
   payment-isolation argument carries decision 2 well; decision 1 has no equivalent. Flagged in
   the ADR's "Reasoning not yet captured".
3. **The six open questions in ADR 0001** — notably what pins the custom images' own base, and
   what stops a pinned base from going stale. Q2's model answer suggests a direction but ArtZ
   has not chosen one.
4. **Registry, authentication and credential handling.** Entirely undiscussed, and decision 6
   depends on it.
5. **`go tool` directives.** Taught this session but never acted on — no `go.mod` exists yet, so
   nothing has been pinned in practice.
6. **Unanswered question from 2026-10-07:** "You pin base images by digest. Walk me through what
   happens when a CVE lands in the base OS." Asked, not answered. This is the same staleness
   problem as ADR 0001 open question 1, approached from the security side — if he can answer it
   he has closed that question himself.
7. **Credential blast radius.** Also asked and unanswered: where CI's registry credentials come
   from and what leaking them would cost. He has used `GITHUB_TOKEN` correctly without yet
   explaining why it is safer than a stored PAT.
