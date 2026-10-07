# mini-lineman

Shared infrastructure and low-churn services for the mini-lineman system: api-gateway,
restaurant-service, rider-service, notification-service, back-office and `deploy/`.

A self-study microservices system (food ordering: order → payment → rider dispatch →
notification). **The goal is learning, not shipping.**

Status: **Phase 0.** No service code yet.

## Repos

| Repo | Contents |
|---|---|
| `mini-lineman` (this one) | gateway, restaurant, rider, notification, backoffice, deploy/ |
| `mini-lineman-order` | order-service (Go) |
| `mini-lineman-payment` | payment-service (Python/Django) |
| `mini-lineman-proto` | shared `.proto` contracts, versioned by git tag |
| `mini-lineman-images` | shared toolchain images (Go, Python) |

## Docs

- `docs/decisions/` — ADRs, one per decision
- `docs/interview-notes/` — study notes per phase; start at `docs/interview-notes/README.md`

## Branches

`main` is the production release branch. `develop` is the integration branch; work branches off
`develop`.
