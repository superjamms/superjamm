# Technical Requirements

This document captures technology choices, architecture, and non-functional
constraints for *SuperJamm*. Functional requirements live in
[`requirements.md`](requirements.md). A *Decisions* appendix at the end
captures the meaningful technical choices with rationale, indexed as
`D-T#`.

## Non-functional requirements

- **Multi-tenant.** v1 isolates tenants at the application layer via a `tenant_id` column on every domain table, enforced in a repository wrapper. Namespace-per-tenant isolation is a v2+ option.
- **Easy to iterate.** Single language (TypeScript), single repository, one command to bring up dev.
- **Secure by default.** OIDC-only auth, mainstream libraries, real TLS in dev, no rolled-our-own crypto.
- **OSS-friendly.** Mainstream language and tools; no obscure middleware.
- **Runs on a developer workstation** via a kind cluster.
- **Same artifact at every deploy size** — kind (dev), k3s (appliance, v2+), managed K8s (SaaS) — via Kustomize overlays for the app plus a shared platform bootstrap that installs the supporting Helm charts.
- **No realtime infrastructure.** Live-jam updates use HTTP polling (5–10s interval). Avoids WebSocket/SSE plumbing in v1.

## Stack

| Layer | Choice |
|---|---|
| Language | TypeScript |
| Web framework | Next.js (App Router) |
| UI | Tailwind CSS + shadcn/ui |
| Auth (in-app) | Auth.js (NextAuth v5) |
| Identity provider | Authentik (dev + prod) |
| Federation (prod) | Google, Microsoft, Apple, via Authentik |
| Database | PostgreSQL 16+ |
| ORM | Prisma |
| Validation | Zod + React Hook Form |
| CSV parsing | papaparse + Zod schema |
| Tests | Vitest (unit/component), Playwright (e2e) |
| Lint / format | ESLint, Prettier |
| Package manager | pnpm |
| Container build | `docker buildx` |
| Orchestration (dev) | kind |
| Orchestration (appliance, v2+) | k3s |
| Orchestration (SaaS prod) | Managed K8s |
| App deploy artifact | Kustomize manifests under `k8s/` (base + components + overlays) |
| Platform deploy artifact | Helm charts installed by a bootstrap script (pinned versions) |
| Inner-loop dev | Tilt |
| Gateway / L7 | Project Contour (HTTPProxy CRDs) |
| Load balancer (dev) | cloud-provider-kind — assigns LB IPs to Envoy Service inside kind |
| Load balancer (SaaS prod) | IaaS provider LB (e.g., NLB/ALB on EKS, GCLB on GKE) via Service type `LoadBalancer` |
| Load balancer (appliance, v2+) | metal-lb |
| Cert mgmt | cert-manager + step-ca (dev), cert-manager + Let's Encrypt (prod) |
| In-cluster CA trust | trust-manager (distributes step-ca root bundle into pods) |
| DNS (dev) | CoreDNS with etcd backend + external-dns auto-registering `*.superjamm.test` |
| Logging | pino (structured JSON) |
| Background work (v1) | None. Hooks present for `node-cron` later. |

## Repository layout

Modeled on [spinguard/forgejo-deployment-k8s](https://github.com/spinguard/forgejo-deployment-k8s)
— Kustomize for the application manifests, Helm for the platform infrastructure
installed by a bootstrap script.

```text
superjamm/
├── apps/
│   └── web/                  # Next.js app source
├── k8s/
│   ├── base/                 # Kustomize base: Deployment, Service, HTTPProxy, Postgres
│   ├── components/           # Reusable bits: authentik, trust-manager bundle, coredns-etcd
│   └── overlays/
│       ├── dev/              # dev-specific patches + HTTPProxy with *.superjamm.test FQDNs
│       └── saas/             # prod patches + cert-manager Let's Encrypt issuer + IaaS LB hints
├── kind/
│   └── cluster.yaml          # kind cluster config
├── prisma/                   # schema, migrations
├── scripts/
│   ├── bootstrap.sh          # one-shot installer: kind + Contour + cert-manager + step-ca + ...
│   ├── teardown.sh
│   ├── dev-setup.sh          # trust step-ca root on host; verify cloud-provider-kind installed
│   ├── seed-authentik-oidc.sh
│   └── seed-test-data.sh
├── docs/
└── Tiltfile
```

## Deployment modes

| Mode | Target | LoadBalancer | DNS | Status |
|---|---|---|---|---|
| Dev | kind on a developer workstation | cloud-provider-kind | external-dns → CoreDNS-etcd (`*.superjamm.test`) | v1 |
| SaaS | Managed K8s (provider TBD) | IaaS LB (cloud-provider integrated) | external-dns → cloud DNS, or operator-managed records | v1 |
| Appliance | k3s on dedicated hardware | metal-lb | external-dns → local DNS, or static records | v2+ (code path shared; not a v1 release target) |

## Multi-tenancy

**v1 strategy: application-level scoping.**

- Every domain row carries `tenant_id`.
- Tenant context is derived from the authenticated session, pinned in the request scope (Next.js middleware).
- Prisma access is wrapped in a thin repository layer that injects the tenant filter automatically. Direct Prisma client usage outside the repository is forbidden by an ESLint rule.
- One shared Postgres database, one logical schema, all tenants co-resident.

**Appliance mode** is the degenerate case: one tenant per deployment, same schema, same scoping.

Namespace-per-tenant isolation (separate Postgres per tenant) is deferred to v2+.

## Identity & authorization

- Auth.js handles the OIDC dance against Authentik.
- Authentik is the sole upstream IdP from the application's perspective. In prod it federates Google, Microsoft, and Apple.
- **Roles are application-level**, stored in SuperJamm's database, not derived from Authentik groups. Role changes happen in SuperJamm's UI and are audit-logged.
- Authorization policies live in per-resource policy modules (e.g., `policies/jam.ts`). Repositories call into policies; the UI never trusts client-side flags.
- New Musicians are pre-created by an Account Holder or Jam Coordinator via email; on the user's first OIDC sign-in the session is attached to the existing record.

## Dev environment

A two-step bring-up:

1. `scripts/bootstrap.sh` provisions the kind cluster and installs the platform Helm charts (pinned versions): Contour, cert-manager, step-ca, trust-manager, CoreDNS-etcd, external-dns, Authentik. Requires `cloud-provider-kind` running on the host so Envoy's Service gets a LoadBalancer IP. Modeled on the equivalent script in `spinguard/forgejo-deployment-k8s`.
2. `tilt up` watches `apps/web/` and live-updates the SuperJamm Deployment in cluster.

In-cluster components after bootstrap:

- **Contour** (`projectcontour` namespace) — Envoy gateway. HTTPProxy CRDs route per-FQDN traffic.
- **cloud-provider-kind** (host process) — gives Envoy's `Service` a routable LoadBalancer IP.
- **cert-manager + step-ca + trust-manager** — step-ca is the in-cluster root/intermediate CA; cert-manager issues per-HTTPProxy certs; trust-manager publishes the root bundle as a `ConfigMap` consumed by pods that need to trust the CA.
- **CoreDNS-etcd + external-dns** — external-dns watches HTTPProxy resources and writes A records to etcd; CoreDNS resolves `*.superjamm.test` to Envoy's LoadBalancer IP. Adding a new HTTPProxy auto-creates its DNS record — no manual `/etc/hosts` edits.
- **Postgres** (single instance, dev volume).
- **Authentik** (server + worker + redis), pre-seeded with a dev realm, OIDC application, and an admin user.
- **SuperJamm app** — Tilt live-update on source change.

One-time host setup, codified in `scripts/dev-setup.sh`:

- Verify `cloud-provider-kind` is installed and instruct the user to run it (`sudo cloud-provider-kind`).
- Trust step-ca's root certificate in the OS trust store so the browser accepts dev certs.
- Point the host's resolver at the in-cluster CoreDNS for `*.superjamm.test` (typically a `/etc/resolver/superjamm.test` file on macOS, or systemd-resolved config on Linux).

This gives dev/prod parity for cookies, CORS, SameSite, and TLS — issues that bite later if only one environment uses HTTPS — and removes the per-service `/etc/hosts` edits.

Dev FQDNs (defined as HTTPProxy resources in `k8s/overlays/dev/`):

- `app.superjamm.test` — SuperJamm web app
- `auth.superjamm.test` — Authentik

Prod FQDNs (under the product domain [`superjamm.live`](https://superjamm.live), not yet active):

- `superjamm.live` (apex) — marketing site and user docs
- `app.superjamm.live` — SuperJamm web app (public + private dashboards live here)
- `auth.superjamm.live` — Authentik

See `D-T13` for the rationale and v2+ considerations.

## Data model (top-level tables, sketch)

```text
tenant
user                          -- belongs to tenant
user_instrument_proficiency   -- user × instrument → 1..5
venue                         -- belongs to tenant
venue_constraint              -- belongs to venue (max_song_length, buffer, etc.)
venue_genre_include           -- venue × genre
venue_genre_exclude           -- venue × genre
venue_supported_instrument    -- venue × instrument
venue_house_band_default      -- venue × instrument (default-backed)
venue_house_band_member       -- venue × user
instrument                    -- tenant-scoped catalog
genre                         -- tenant-scoped catalog
song                          -- tenant-scoped catalog
song_instrument               -- song × instrument (required)
song_genre                    -- song × genre
song_link                     -- song × external link
master_list                   -- belongs to tenant
master_list_song              -- master_list × song
venue_master_list             -- venue × master_list assignment
practice_list                 -- belongs to user
practice_list_entry           -- practice_list × song
practice_list_entry_part      -- entry × instrument + proficiency
jam                           -- belongs to venue
jam_setup_buffer_override     -- belongs to jam
request_list_entry            -- user × jam × song × instrument
set_list_entry                -- jam × song × user × instrument (ordered)
display_anonymization_pref    -- user × scope (public, private)
notification                  -- user × payload (in-app)
audit_log                     -- generic write log
```

All tables carry `created_at`, `updated_at`, and `deleted_at` (soft-delete).
Restore = clear `deleted_at`. Restore is a Jam Coordinator action.

## Live jam — polling design

The private dashboard "live jam" view polls `GET /api/jams/:id/state` every
5 seconds. The endpoint returns the current Set List + metadata with an `ETag`.
The client sends `If-None-Match` to short-circuit unchanged responses to a
304. No WebSocket/SSE plumbing in v1.

## Testing

- **Unit (Vitest)** for policy modules, repository tenant scoping, CSV parser, and the SuperJamm cross-match algorithm.
- **Component (Vitest + React Testing Library)** for forms and dashboards.
- **E2E (Playwright)** for critical paths: sign-in via Authentik, submit a Request, coordinator builds a Set List, run a jam end-to-end, archive.
- E2E runs against a Tilt-managed kind cluster in CI.

## Observability (v1 minimum)

- Structured JSON logs (pino) with `tenant_id`, `user_id`, `request_id`.
- Health endpoint (`/api/health`) returning DB and Authentik reachability.
- No metrics/tracing stack in v1; OpenTelemetry hooks added in code so it can be turned on without refactor.

## CI/CD

- GitHub Actions: lint, typecheck, unit tests, build container image, run Playwright against an ephemeral kind cluster, push image, render+lint Kustomize overlays (`kustomize build`).
- Deploy to prod is **manual `kubectl apply -k k8s/overlays/saas`** in v1.

## Security baseline

- No secrets in repo. Local secrets via 1Password CLI or a dev `.env` outside the source tree.
- The prod overlay references External Secrets Operator for prod secrets; dev secrets are Kubernetes Secrets seeded by `scripts/bootstrap.sh` from env vars (no secrets committed).
- CSRF protection via Auth.js defaults.
- All mutating endpoints require an authenticated session, except the stateless public dashboard reads (which never mutate).
- DB connections use TLS in prod.

## Out of v1 (technical)

- Realtime push (WebSocket/SSE)
- Email transport / templates
- Push notifications
- Background job framework (Bull, Temporal, etc.)
- Multi-cluster federation (Cluster API)
- Tenant-per-namespace isolation
- Multi-region deploy
- Metrics / tracing backend (hooks present, backend not wired)
- Appliance release packaging (k3s artifact)

## Decisions

This appendix captures the meaningful technical decisions made during
planning, with brief rationale. New decisions are appended; superseded
entries are marked as such, not deleted. Reference entries by their `D-T#`
tag.

### D-T1 — Authentik is the IdP in dev and prod

Auth.js connects to Authentik exclusively; in prod Authentik federates
Google, Microsoft, and Apple as upstream providers.

*Why:* one integration in the application; identity federation logic and
audit lives in one place (Authentik). Dev and prod use the same IdP shape,
which keeps cookie / session behavior consistent.

### D-T2 — Public dashboard reads are stateless

The public dashboard route sets no cookies and creates no session.

*Why:* lower attack surface, simpler caching, no PII risk for anonymous
visitors.

### D-T3 — Multi-tenancy is application-level in v1

Every domain table carries `tenant_id`. All Prisma access goes through a
repository wrapper that injects the tenant filter. ESLint forbids direct
Prisma client usage outside the repository layer.

*Why:* lowest-cost tenancy model; runs co-resident in one Postgres; appliance
mode is just "one tenant per deployment." Namespace-per-tenant deferred to
v2+ enterprise tier.

### D-T4 — Live-jam updates use HTTP polling, not realtime push

The private dashboard polls `GET /api/jams/:id/state` every 5 seconds with
ETag / `If-None-Match`.

*Why:* avoids WebSocket / SSE infrastructure; 5-second freshness is more
than enough for a jam setlist.

### D-T5 — Kubernetes is the orchestration target at every scale

Dev = kind, SaaS prod = managed K8s, appliance (v2+) = k3s. One set of
conventions across scales.

*Why:* avoids stack divergence between dev and prod; aligns with existing
operational expertise.

### D-T6 — App manifests are Kustomize; platform infra is Helm via bootstrap script

Application Deployment / Service / HTTPProxy live as Kustomize bases +
overlays. Platform components (Contour, cert-manager, step-ca, trust-manager,
CoreDNS-etcd, external-dns, Authentik) install via pinned Helm charts from
`scripts/bootstrap.sh`.

*Why:* Kustomize is light and overlay-friendly for app-level env differences;
Helm is the upstream distribution mechanism for the platform components.
Modeled on
[spinguard/forgejo-deployment-k8s](https://github.com/spinguard/forgejo-deployment-k8s).

### D-T7 — Project Contour (HTTPProxy CRDs) is the gateway

Not ingress-nginx.

*Why:* matches existing organizational standard; HTTPProxy is more expressive
than Ingress for the routing patterns SuperJamm needs (per-FQDN TLS,
external-dns annotations).

### D-T8 — cloud-provider-kind for dev LB; IaaS LB in prod; metal-lb deferred to v2+

Dev uses `cloud-provider-kind` (host process) to assign real `LoadBalancer`
IPs to Envoy. SaaS prod uses the cloud provider's LB integration via
`Service type: LoadBalancer`. metal-lb is reserved for on-prem appliance,
post-v1.

*Why:* lets `Service type: LoadBalancer` be the consistent abstraction across
modes; no NodePort + extraPortMappings hacks.

### D-T9 — TLS uses cert-manager + step-ca (dev) and cert-manager + Let's Encrypt (prod)

step-ca runs in-cluster in dev with a root + intermediate; trust-manager
publishes the root bundle as a `ConfigMap` for in-cluster pods that need to
trust it. Dev hosts trust the step-ca root once via `scripts/dev-setup.sh`.

*Why:* gives dev real TLS so cookies, CORS, and SameSite behave identically
to prod. Avoids the "works on localhost, breaks under HTTPS" class of bugs.

### D-T10 — DNS in dev uses CoreDNS-etcd + external-dns auto-registering `*.superjamm.test`

external-dns watches HTTPProxy resources and writes A records to an
etcd-backed CoreDNS instance. Dev hosts point `.superjamm.test` resolution at
the in-cluster CoreDNS.

*Why:* eliminates per-service `/etc/hosts` edits; adding a new HTTPProxy is
enough to get the FQDN routable.

### D-T11 — Inner-loop dev uses Tilt

`tilt up` watches `apps/web/` and live-updates the SuperJamm Deployment.

*Why:* hot-reload ergonomics inside a kind cluster without rebuilding and
redeploying images by hand.

### D-T12 — Reference architecture is `spinguard/forgejo-deployment-k8s`

When designing a new manifest, script, or convention, check that repo for an
existing pattern before inventing one.

*Why:* consistency across related projects; well-trodden patterns reduce
design risk.

### D-T13 — Production FQDN layout

The product domain is `superjamm.live`. The v1 SaaS deployment uses:

- `superjamm.live` (apex) for the marketing site and user docs
- `app.superjamm.live` for the web app — both public and private dashboards
- `auth.superjamm.live` for Authentik

The public dashboard URL is **path-based** under the app subdomain:
`app.superjamm.live/public/<tenant-slug>/<venue-slug>`.

*Why:* mirrors the dev FQDN layout (`*.superjamm.test`); keeps the apex
clean for SEO and stakeholder sharing; couples app and public dashboards on
one cert and one origin, which keeps cookie/CORS behavior straightforward.

Per-tenant subdomains (`<tenant>.superjamm.live`) are deferred to v2+. Adding
them later would require revisiting Auth.js cookie scope, Authentik client
configuration, and wildcard TLS — non-trivial changes that don't earn their
keep in v1.
