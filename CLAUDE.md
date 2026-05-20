# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

*SuperJamm* is a web app for coordinating open jam sessions at venues that host a
house band and rotating sit-in musicians. It surfaces candidate songs based on a
per-venue master list and per-musician practice/request lists, and helps the Jam
Coordinator plan and run the jam.

The repository is currently **documentation-only** — implementation has not started.
The v1 plan is sliced into vertical increments in [`docs/backlog.md`](docs/backlog.md).
The first slice (S0 — walking skeleton) introduces the Next.js app and the
kind / Kustomize / Helm bootstrap.

## Authoritative Documents

Read in order; this is the source of truth for product and tech decisions:

1. [`docs/requirements.md`](docs/requirements.md) — functional requirements, roles, domain entities, jam lifecycle, anonymization, kiosk, notifications. Carries a *Decisions* appendix (`D-R#`) with rationale for product-level choices.
2. [`docs/technical-requirements.md`](docs/technical-requirements.md) — stack, deployment modes, multi-tenancy strategy, dev environment, data model sketch, security baseline. Carries a *Decisions* appendix (`D-T#`) with rationale for technical choices.
3. [`docs/backlog.md`](docs/backlog.md) — v1 slices (S0–S10) with per-story acceptance criteria.

When in doubt, prefer the doc closer to the top of the list. When discussing
or proposing a change to a decision, reference it by its `D-R#` / `D-T#` tag.

## Reference Repository

Deployment topology is modeled on
[`spinguard/forgejo-deployment-k8s`](https://github.com/spinguard/forgejo-deployment-k8s).
Before inventing a new manifest, script, or convention, check that repo for an
existing pattern.

## Tech Stack (summary; details in `docs/technical-requirements.md`)

| Layer | Choice |
|---|---|
| Language | TypeScript |
| Framework | Next.js (App Router) |
| Database | PostgreSQL + Prisma |
| Auth | Auth.js → Authentik (OIDC), federating Google/Microsoft/Apple in prod |
| Gateway | Project Contour (HTTPProxy CRDs) |
| Dev LB | `cloud-provider-kind` |
| Prod LB | IaaS-provided LB via `Service type: LoadBalancer` |
| Appliance LB (v2+) | metal-lb |
| Cert mgmt | cert-manager + step-ca (dev) / Let's Encrypt (prod) + trust-manager |
| Dev DNS | CoreDNS-etcd + external-dns auto-registering `*.superjamm.test` |
| Orchestration | kind (dev), Managed K8s (SaaS), k3s (appliance, v2+) |
| App manifests | Kustomize under `k8s/` (base + components + overlays) |
| Platform infra | Helm via `scripts/bootstrap.sh` (pinned versions) |
| Inner-loop dev | Tilt |
| Tests | Vitest (unit/component) + Playwright (e2e) |

## Repository Layout (target, post-S0)

```text
superjamm/
├── apps/web/                # Next.js app source
├── k8s/
│   ├── base/                # Kustomize base manifests
│   ├── components/          # Reusable bits (authentik, trust-manager, coredns-etcd)
│   └── overlays/{dev,saas}/ # Env-specific patches and HTTPProxy resources
├── kind/cluster.yaml        # kind cluster config
├── prisma/                  # schema and migrations
├── scripts/                 # bootstrap, teardown, dev-setup, seeds
├── docs/
└── Tiltfile
```

## Common Commands (will populate as S0 lands)

```bash
# One-time host setup: trust step-ca root, point host resolver at in-cluster
# CoreDNS for the *.superjamm.test zone, verify cloud-provider-kind installed
scripts/dev-setup.sh

# Bootstrap: kind cluster + platform Helm charts (Contour, cert-manager,
# step-ca, trust-manager, CoreDNS-etcd, external-dns, Authentik, Postgres)
scripts/bootstrap.sh

# Inner-loop dev (after bootstrap.sh)
tilt up

# App manifest render check
kubectl kustomize k8s/overlays/dev

# Tear it all down
scripts/teardown.sh
```

Per-subsystem checks (mirrors the reference repo's conventions):

```bash
# Contour / Envoy
kubectl get po -n projectcontour
kubectl get httpproxy -A
kubectl get svc -n projectcontour contour-envoy \
  -o jsonpath='{.status.loadBalancer.ingress[0].ip}'

# cert-manager
kubectl get certificate,certificaterequest,order,challenge -A

# external-dns
kubectl logs -n external-dns -l app.kubernetes.io/name=external-dns -f
```

## Architectural Invariants

Document so future sessions don't re-litigate.

### Multi-tenancy at the application layer

Every domain table carries `tenant_id`. All Prisma access goes through a thin
repository layer that injects the tenant filter automatically. **Direct Prisma
client usage outside the repository layer is forbidden**, enforced by an
ESLint rule. Tenant context is pinned in the request scope by Next.js
middleware. Namespace-per-tenant isolation is a v2+ option, deferred.

### Roles live in SuperJamm's database, not Authentik

Authentik authenticates; SuperJamm authorizes. Role changes happen in
SuperJamm's UI and are audit-logged. Do not derive roles from Authentik
groups.

### v1 uses polling, not realtime push

The live-jam dashboard polls `GET /api/jams/:id/state` every 5 seconds with
ETag / `If-None-Match`. No WebSocket / SSE plumbing in v1.

### Dev/prod parity for TLS, DNS, and identity

Dev uses real TLS via step-ca, real DNS via in-cluster CoreDNS-etcd +
external-dns, and real OIDC via Authentik. Avoid env-only shortcuts
(`http://localhost`, self-signed-without-trust) that let bugs hide until prod.

### Soft-delete + restore

Every domain table has `deleted_at`. Restore is a Jam Coordinator action. No
hard deletes from the UI.

### Anonymization is two-tier

- **Display anonymization** is reversible — data intact; rendered Set List shows the literal word "Anonymous."
- **Hard anonymization** is irreversible — user references on past `set_list_entry` rows are overwritten. UI must surface this with an explicit confirm dialog warning of irreversibility.

### Variant instruments are first-class

`Lead Guitar`, `Rhythm Guitar`, and `Guitar` are distinct instrument records,
not a sub-attribute of a generic "Guitar." This is intentional and drives
request / set-list mechanics.

### Two distinct proficiency relationships

- **Musician × Instrument** — overall comfort, drives "needs guidance" logic.
- **Practice List Entry × Instrument** — comfort with a specific song on a specific instrument.

Do not collapse these into one table.

### Audio never transits SuperJamm

**SuperJamm stores no audio, ever.** No upload endpoints, no file or blob
columns on any domain table, no attachment storage, no proxying or embedding of
a vendor's stream. Backing tracks and stems are played by whoever holds the
rights to them, from their own device. SuperJamm records only *metadata about
instrument coverage*.

This is the invariant that keeps SuperJamm out of the copyright analysis
entirely, and it is what makes the "no hosted copyrighted material" posture in
`docs/requirements.md` structurally true rather than a policy promise. Erosion
happens one convenience feature at a time — "just let the coordinator upload
the track so it's ready" — so treat any proposal that puts audio bytes in the
system as a change to this invariant, requiring an explicit decision, not an
implementation detail.

Corollaries:

- **No transfer between parties.** Vendor-purchased tracks stay with the
  purchaser (venue, house band, or coordinator) and are never distributed to
  sit-ins. Sit-ins' own stems stay on their own devices and are never sent to
  the venue. This removes distribution from the model structurally — the one
  exposure that would otherwise grow with every musician on the list.
- **No provenance enum for derived material.** The schema has no value
  describing a track separated from a commercial master. This is deliberate:
  a field that records the act also normalizes it, and a documented expectation
  is evidence of intent. A schema that cannot express it cannot endorse it.
- **The gap closes to live-only.** When no licensed track exists and the song
  is not the performer's own composition, the correct product answer is that
  the song is performed live or not tonight — *never* "the musician supplies
  stems." Do not reintroduce that phrasing as a convenience.

- **Lyrics text is the same category of problem.** Lyrics are copyrighted
  separately from recordings. Metadata *about* lyrics is fine and an outbound
  **link** to an external lyrics site is the sanctioned pattern; storing,
  caching, scraping, framing, or displaying the text is hosting copyrighted
  material. Never fetch lyric text server-side "for performance" — a cached
  copy is a stored copy. This constrains the v2 Nashville-charts feature — see
  *Copyright* in `docs/requirements.md`.

Instrument coverage (which parts come from the house band, which from sit-ins,
which from a backing track someone else holds) is an input to set-list
feasibility, not a media feature. That distinction is the whole of SuperJamm's
involvement with backing tracks.

Full rationale in `D-R21` (`docs/requirements.md`). The underlying legal
analysis lives in the sibling `song-track-tools` project's `features.md` §3 and
§8 — a private, local-only toolset that is **not** a SuperJamm dependency and
whose code will never be integrated here.

## Conventions

### Dev TLD

`*.superjamm.test` (RFC 2606-reserved). Dev FQDNs:

- `app.superjamm.test` — SuperJamm web app
- `auth.superjamm.test` — Authentik

### Prod TLD

`*.superjamm.live` (product domain, not yet active). Prod FQDNs:

- `superjamm.live` (apex) — marketing and user docs
- `app.superjamm.live` — SuperJamm web app (public + private dashboards)
- `auth.superjamm.live` — Authentik

Public dashboard URL: `app.superjamm.live/public/<tenant-slug>/<venue-slug>` (see `D-T13`).

### Secrets

Never commit. `scripts/bootstrap.sh` reads values from env vars (with
documented local-dev defaults) and materializes Kubernetes Secrets via
`kubectl create secret … --dry-run=client -o yaml | kubectl apply -f -`. The
prod overlay references External Secrets Operator.

### Pinned versions

Platform Helm chart versions are pinned in `scripts/bootstrap.sh`. Bump
deliberately; never `helm upgrade --version ""`.

### Audit log

Role assignment / removal, Account Holder reassignment, and impersonated
Kiosk requests must write `audit_log` entries with `actor_user_id`,
`subject_user_id`, `action`, `tenant_id`, and timestamp.

### Authorization policies live in per-resource modules

`policies/jam.ts`, `policies/venue.ts`, etc. Repositories call into policies;
the UI never trusts client-side flags.

## What Is Explicitly Out of v1

(Pulled from `docs/requirements.md` and `docs/technical-requirements.md`.)

- Self-serve tenant sign-up (admin-provisioned only)
- k3s appliance release packaging
- Email / push notifications (in-app only in v1)
- Conditional sit-in offers ("I'll play bass if no one else does")
- Auto conflict resolution
- Nashville-numbered charts with lyrics
- Cross-tenant identity
- Realtime push (WebSocket / SSE)
- metal-lb (post-v1, for appliance mode)
- Tenant-per-namespace isolation
- Multi-cluster federation
- Metrics / tracing backend (hooks present, backend unwired)

When tempted to pull one of these in, push back to the user — these are
deliberate v1 exclusions documented in the requirements.
