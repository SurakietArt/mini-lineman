# 3. Public repositories and public packages

- **Date:** 2026-10-07
- **Status:** accepted
- **Supersedes:** the private-package assumptions in [ADR 0001](0001-toolchain-install.md)
  (decisions 2 and 6 stand; their private-visibility consequences do not)
- **Deciders:** ArtZ

## Context

ADR 0001 chose private toolchain packages on ghcr.io. Three problems surfaced once it was
implemented.

**Branch protection was unreachable.** Making `main` a protected release branch — requiring a
pull request, blocking force pushes — returns `403 Upgrade to GitHub Pro or make this repository
public` on a free plan, for both classic branch protection and repository rulesets. The
branching model ArtZ wanted could not be enforced at all while the repos stayed private.

**Private package storage is metered** at roughly 500 MB on GitHub Free, while ADR 0001 decision
6 requires a new immutable tag per publish and never reusing one. A Go toolchain image is a few
hundred MB compressed, so reproducibility ("a build from six months ago can be rebuilt") and the
quota were in direct conflict within a handful of publishes.

**Cross-repo access needed manual grants.** A package published from one repo is unreachable by
another repo's `GITHUB_TOKEN` until access is granted explicitly, per image per repo, forever.

The decisive factor was none of those. ArtZ's reason: **"it is required to show or attach on
email"** — the repos are portfolio material for job applications, so they have to be readable by
someone who is not him.

## Options considered

**A — Make the repositories and packages public.** Branch protection free; no package storage
quota; no per-repo access grants; no local PAT; and the repo doubles as a portfolio an
interviewer can read. Cost: `payment`'s stated reason for being a separate repo —
"sensitive; restricted access" — is contradicted outright, and every mistake is permanently
visible.

**B — GitHub Pro, roughly $4/month.** Keeps every ADR 0001 decision intact and rehearses a real
company's setup, where repos genuinely are private and grants genuinely are manual. Cost: a
subscription for a learning project, and the storage quota still binds.

**C — No branch protection; rely on discipline.** Free, no work. Cost: `main` and `develop`
become two names with no enforced difference, so the release flow cannot be practised or
demonstrated.

**D — Public code repos, private `payment`.** Protection and free packages where it matters,
isolation preserved where it was argued for. Cost: a mixed model to explain, and `payment` alone
still cannot have protection without Pro.

## Decision

**Option A.** All five repositories and both container packages are public.

Reasoning as given: the repos must be showable to employers. The supporting arguments —
protection, quota and grants all resolving at once — were confirmed but were not what decided it.

## Consequences

- **Branch protection works.** `main` now requires a pull request, blocks force pushes and
  blocks deletion, with admin enforcement on, in all five repos. Verified: a direct push to
  `main` is rejected with `protected branch hook declined`.
- **The storage/reproducibility conflict is gone**, not traded. Public packages have no storage
  quota, so immutable never-reused tags cost nothing to keep.
- **Step 7 of the setup guide is obsolete.** No per-repo access grants; any CI job can pull
  without configuration.
- **No local credential needed.** `docker pull` works unauthenticated; the classic PAT with
  `read:packages` is no longer required.
- **The `dev-` tag cleanup changes motivation.** It was designed to protect a quota that no
  longer exists; it now exists to keep the version list readable. Still worth having, on weaker
  grounds.
- **`payment`'s documented reason is now false.** `CLAUDE.md` says it is separate because it is
  "sensitive; restricted access". A public repo disproves that. Its independent deploy cadence
  still justifies the split, but that has to become the stated reason or the repo table
  contradicts the setup.
- **Mistakes are permanent and visible.** Commit history, failed CI runs and bad early designs
  are all public. Mitigated once: every commit on every branch of all five repos was scanned for
  tokens, keys, `.env` files and credential patterns before the switch, and was clean.
- **Secrets discipline is now load-bearing rather than advisory.** A committed credential in a
  private repo is a mistake; in a public one it is an incident.

## Open questions

1. The replacement reason for `payment` being a separate repo, and the `CLAUDE.md` amendment
   that records it.
2. Whether `enforce_admins` stays on. It is the only thing making protection real on a
   single-maintainer project, and it also means ArtZ cannot bypass his own gate in a hurry.
