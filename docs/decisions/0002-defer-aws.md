# 2. Defer AWS to the final phase

- **Date:** 2026-10-07
- **Status:** accepted
- **Deciders:** ArtZ

## Context

ArtZ raised deploying the whole system on AWS in order to learn AWS. The existing plan targets
Kubernetes on `kind` locally (Phase 4), with GitHub Actions and a local Prometheus/Grafana
stack (Phase 5).

Two facts shaped the choice. First, the target roles named in `CLAUDE.md` list Go, gRPC,
RabbitMQ, Redis, Docker, Kubernetes, CI/CD and Grafana, plus a Python/Django + Vue role — AWS
is not among them, so AWS is an additional goal rather than a better route to the existing one.
Second, a managed AWS stack costs real money continuously: the EKS control plane alone bills
roughly $73/month and a NAT Gateway a further ~$33, both by the hour regardless of use and
neither covered by the free tier. A full stack with RDS, ElastiCache and Amazon MQ lands around
$150–250/month if left running.

The timing question is the actual decision. Phases 1–3 are application work — Go, gRPC,
RabbitMQ, the outbox pattern — and are indifferent to where they run, so nothing built before
Phase 4 is wasted either way.

## Options considered

**A — Keep the plan; add AWS as a final phase after `kind`.**
Kubernetes fundamentals are learned locally at zero cost, and the finished Helm chart is later
ported to AWS as a deliberate exercise. The port itself becomes interview material. Cost: AWS
exposure arrives last, and motivation may not survive that long.

**B — Replace `kind` with EKS at Phase 4.**
Kubernetes is learned on real infrastructure from the start — managed control plane, cloud load
balancers, IAM-to-pod identity — with no local-only habits. Cost: the full monthly bill during
the longest phase, and every mistake becomes a slow apply-and-wait cycle instead of an instant
local one.

**C — `kind` for the dev loop, one timeboxed AWS deployment at Phase 5.**
Keeps the fast local loop, then stands the system up on AWS once, documents it and tears it
down — roughly $20–40 of actual usage. Cost: no day-to-day AWS fluency, and "I deployed it
once" is thinner under questioning.

**D — AWS from now, drop `kind`.**
Maximum AWS depth, one environment, no port to maintain. Cost: highest spend for longest, and
the time goes into VPCs, subnets and IAM rather than the queue and gRPC patterns the target
interviews probe.

## Decision

**Option A.** AWS moves to the end of the plan, after Kubernetes on `kind` is working and the
microservices patterns are built. Phases 0–6 proceed unchanged.

Reasoning as given: "leave AWS as last, continue current work." The supporting argument, which
ArtZ has not yet stated in his own words, is that AWS is not a requirement of the roles being
targeted, so paying its cost before the named skills are built would be optimising for the
wrong interview.

## Consequences

- No AWS spend during the phases that do not need it, which is all of them until Phase 4.
- The `kind` → cloud port becomes a deliberate, documented exercise rather than an assumption,
  and "here is what broke when I moved it" is stronger interview material than having only ever
  deployed to one environment.
- Kubernetes is learned without cloud networking as a confounding variable — faster feedback,
  and failures are attributable to the manifest rather than to IAM or a security group.
- **Risk accepted:** AWS may never happen. A final phase is the one most likely to be dropped,
  and the project would then show Kubernetes experience with no cloud experience at all.
- **Gap accepted:** `kind` teaches Kubernetes concepts but not managed-service operation — no
  cloud load balancer, no IRSA, no managed control plane upgrades. Interviewers for cloud-heavy
  roles do probe this, and the honest answer until the final phase is that it has not been done.
- If a role ArtZ actually applies to lists AWS as a requirement rather than a nice-to-have,
  this decision should be revisited with a new ADR superseding this one.

## Open questions

1. What monthly AWS spend is acceptable when the final phase arrives. Undecided, and it selects
   between EKS and a cheaper substitute such as k3s on a single EC2 instance.
2. Whether the final phase targets EKS specifically or any AWS compute. ECS Fargate would teach
   AWS without re-teaching Kubernetes, and is cheaper, but discards the Helm work.
