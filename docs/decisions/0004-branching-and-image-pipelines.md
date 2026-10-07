# 4. Branching model and image pipelines

- **Date:** 2026-10-07
- **Status:** accepted
- **Deciders:** ArtZ

## Context

Five repositories existed with inconsistent default branches — three on `main`, two on `master`
— and no CI beyond a single publish job. The inconsistency was not cosmetic: the images repo's
workflow triggered on `branches: [master]`, and a workflow pointed at a branch that does not
exist produces no error, no warning and no run. The pipeline looks configured and is inert.

Separately, the publish pipeline pushed an image and kept it without ever running it, while the
first real failure in that pipeline had been in the tag computation — a class of bug that only
a push can reveal.

## Options considered

### Branch naming

**A — standardise on `main`.** Four of five repos already matched; GitHub's default; copied
workflow snippets work unmodified. Cost: rename the images repo and move its trigger.
**B — standardise on `master`.** Matched the local `git init` default, so new repos would not
drift. Cost: three renames, and against every tool default.
**C — leave it split.** No work, permanent per-repo footgun where failure is silent.

### What a develop build does with its image

**A — build without pushing.** Free, proves the Dockerfile compiles. Cost: never exercises the
tag computation, the registry auth or the push — which is where the real failure was.
**B — build, push a throwaway, verify it, delete it.** Exercises the entire path. Cost: a delete
step that must not be skipped on failure or cancellation.
**C — publish from develop and keep it.** Simplest. Cost: accumulates versions nobody will use.

## Decision

**`main` is the production release branch; `develop` is the integration branch.** All five repos
renamed and standardised, `git config --global init.defaultBranch main` set so new repos do not
drift back.

**`main` is protected** in all five repos: pull request required (0 approvals, so a single
maintainer is not deadlocked), force pushes blocked, deletion blocked, **admin enforcement on**.

**Two pipelines, differing only in what happens to the artifact.**

| Branch | Build | Push | Smoke test | Delete |
|---|---|---|---|---|
| `develop` | yes | `dev-<date>-<sha>` | yes | yes, `if: always()` |
| `main` | yes | `<date>-<sha>` | yes | no |

The `dev-` prefix makes a throwaway distinguishable from a release at a glance. The smoke test
pulls the image back **from the registry** rather than using the local build cache, and runs the
tools inside it, so it proves the pushed artifact is present and runnable rather than proving the
build succeeded. Cleanup runs under `if: always()` so a cancelled or failed job cannot orphan a
throwaway.

Reasoning as given, on the develop pipeline: build and destroy, "to just check and make sure it
work as expected."

## Consequences

- A broken Dockerfile, a broken tag or broken registry auth is caught on `develop`, before
  anything reaches `main`.
- The kept artifact is now verified. Previously `main` published an image it had never executed.
- The release path is enforced, not conventional: a direct push to `main` is rejected with
  `protected branch hook declined`. Verified by attempting one.
- **`enforce_admins: true` applies to ArtZ.** With it off, GitHub prints "Changes must be made
  through a pull request" as a notice and *accepts the push anyway* — protection with admin
  bypass is decorative on a single-maintainer repo. This was discovered the hard way: a test
  push succeeded and left a stray commit on a public `main` that took temporarily re-enabling
  force push to remove.
- Every change to `main` now costs a PR, even a one-line fix. That is the point, and it is also
  friction on a solo project.
- The delete step depends on `GITHUB_TOKEN` being able to remove package versions. It can:
  `DELETE /users/{owner}/packages/container/{pkg}/versions/{id}` works with `packages: write`,
  confirmed in run 37589157208. No PAT required.
- Two workflows now duplicate the build, login and tag-computation steps. If a third appears,
  this should become a reusable workflow rather than a third copy.
- Service repos have no CI yet. This ADR covers the images repo only; the same two-branch shape
  will need applying to `order`, `payment` and `proto` when they have code to test.

## Open questions

1. Whether the rest of git-flow — release branches, hotfix branches, tags on `main` — is adopted,
   or whether `develop → main` with a tag on merge is enough.
2. Whether `main` should require the develop pipeline to have passed (a required status check),
   which is configurable now that protection works but is currently not set.
3. What the service repos' pipelines run, and whether they run inside the toolchain image
   (ADR 0001 decision 5's enforcement point) or install tooling per job.
