# Interview notes — index

One page to skim the whole project the night before an interview. Every concept, decision and
question, with links to the detail.

**Current phase: 0** · last updated 2026-10-07

---

## Phases

| Phase | Topic | Notes | Status |
|---|---|---|---|
| 0 | Setup: toolchain, pinning, repo layout | [phase-0.md](phase-0.md) | in progress |
| 1 | order-service alone: Go + Postgres, TDD, Docker, compose | — | not started |
| 2 | restaurant-service + gateway, gRPC, proto split, timeouts | — | not started |
| 3 | RabbitMQ: routing, at-least-once, idempotency, retry/DLQ, outbox | — | not started |
| 4 | Kubernetes on kind: probes, limits, HPA, Ingress, Helm | — | not started |
| 5 | CI/CD, Prometheus/Grafana, structured logs | — | not started |
| 6 | Vue back-office, load test, architecture docs | — | not started |

---

## Concepts covered so far

### Phase 0 — toolchain and build reproducibility

| Concept | One-line version | Detail |
|---|---|---|
| `GOTOOLCHAIN` / `go` vs `toolchain` directives | A container's Go version is a **floor, not a pin** — `go.mod` can pull a newer toolchain silently | [phase-0](phase-0.md#gotoolchain-and-the-two-version-directives-in-gomod) |
| `go.mod` `tool` directives (Go 1.24+) | Pins Go-based tools in the same reviewable file; cannot pin `protoc`, `kubectl`, `helm` | [phase-0](phase-0.md#gomod-can-pin-tools-not-just-libraries) |
| Image tags as build inputs | `:latest` makes pull *time* part of the build; pinning trades surprise breakage for silent rot | [phase-0](phase-0.md#image-tags-are-a-build-input) |
| Build artifacts create repo coupling | If A's build needs B's image, A depends on B no matter what the code imports | [phase-0](phase-0.md#build-artifacts-create-repo-coupling) |
| Blast radius vs consolidation | One shared artifact = less maintenance, one bad publish breaks everything | [phase-0](phase-0.md#blast-radius-of-shared-artifacts) |
| OCI naming rules | Container repo names cannot contain uppercase; `github.repository_owner` can | [phase-0](phase-0.md#oci-naming-rules-are-stricter-than-githubs) |
| Inert workflow triggers | A workflow on the wrong branch produces no error and no run | [phase-0](phase-0.md#a-workflow-pointed-at-the-wrong-branch-does-not-fail) |
| `GITHUB_TOKEN` scope | Scoped to its own repo; cross-repo package pulls need an explicit grant | [phase-0](phase-0.md#github_token-is-scoped-to-its-own-repository) |

---

## Decisions (ADRs)

| ADR | Decision | Status |
|---|---|---|
| [0001](../decisions/0001-toolchain-install.md) | Toolchain install and pinning — per-ecosystem images, `GOTOOLCHAIN=local`, dedicated images repo, pinned tags | accepted, 7 open questions |
| [0002](../decisions/0002-defer-aws.md) | Defer AWS to the final phase — learn Kubernetes on `kind` first | accepted, 2 open questions |

---

## Every interview question asked

Answers, feedback and model answers in the linked phase notes.

| # | Question | Phase |
|---|---|---|
| 1 | Explain why `GOTOOLCHAIN=local`, and what you gave up. | [0](phase-0.md#q1--explain-why-gotoolchainlocal-and-what-you-gave-up) |
| 2 | You've pinned your build tools to immutable tags. Six months later, what's gone wrong? | [0](phase-0.md#q2--youve-pinned-your-build-tools-to-immutable-tags-six-months-later-whats-gone-wrong) |

**Asked but not yet answered:**

- A developer says the build works on their machine but fails in CI. Walk me through how your
  setup makes that diagnosable. *(Phase 0)*
- Your CI pushes images to a registry. Where do the credentials come from, and what is the blast
  radius if they leak? *(Phase 0)*
- You pin base images by digest. Walk me through what happens when a CVE lands in the base OS.
  *(Phase 0)*

---

## Stories ready to tell

Short, concrete, with a decision and a consequence — the ones to reach for when asked about
design judgement.

- **"Tell me about a design flaw you caught before it shipped."** The image was made
  authoritative over the Go version, then nearly built from a floating base — which would have
  changed the authority on every rebuild, silently, because a moving base only advances so the
  guard never fires. Pinning is only as strong as the weakest unpinned layer beneath it.
  [phase-0](phase-0.md#pinning-the-image-while-letting-its-base-float)
- **"Tell me about a time something failed for a reason you didn't expect."** The first publish
  run failed because the GitHub account name has capital letters and OCI repository names may
  not — not because of anything in the design that had been thought about carefully.
  [phase-0](phase-0.md#the-first-ci-run-failed-on-an-uppercase-letter)
- **"Tell me about a decision you changed your mind on."** Toolchain images went per-repo → one
  shared → per-ecosystem, plus one decision superseded outright. The reversals are what make
  "why not one image per repo?" answerable at all.
  [phase-0](phase-0.md#three-reversals-in-one-decision)

---

## Weak spots to fix

Live list — these get re-asked until they're clean.

- Stating the **cost** of his own decisions, not just the benefit. Current habit: naming the
  alternative's downside instead of his own choice's.
- Why per-ecosystem images beat per-repo images — still rests on tooling duplication alone.
- Registry, authentication and credential handling: entirely undiscussed.
- `go tool` directives taught but never used in practice.

Full list with context: [phase-0 § Still shaky](phase-0.md#5-still-shaky--re-ask-next-session).
