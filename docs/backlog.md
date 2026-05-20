# SuperJamm v1 Backlog

A first-cut backlog organized as **vertical slices**. Each slice is a coherent
demoable increment; each story inside a slice is sized to roughly 1–3 days.
Sizing and ordering are starting points — treat this as a planning aid, not a
contract.

Conventions:

- Stories are prefixed `SX.Y` where `X` is the slice and `Y` is sequence in slice.
- *AC* = acceptance criteria.
- Stories with **[infra]** are platform/plumbing; **[feature]** is user-visible behavior.

---

## Slice 0 — Walking skeleton

Goal: a deployable empty app with auth, behind real TLS, with CI in place.

### S0.1 Repo scaffold **[infra]**

Next.js (App Router) + TS + Tailwind + shadcn/ui + ESLint + Prettier + pnpm +
Vitest + Playwright in a single repo.

*AC:* `pnpm install && pnpm dev` runs a placeholder homepage; `pnpm test` runs
an empty Vitest suite; `pnpm e2e` runs an empty Playwright suite.

### S0.2 Kustomize app manifests skeleton **[infra]**

`k8s/base/` (Deployment, Service, HTTPProxy stub) + `k8s/overlays/dev/` and
`k8s/overlays/saas/`. Reusable bits live under `k8s/components/`. Modeled on
`spinguard/forgejo-deployment-k8s`.

*AC:* `kubectl kustomize k8s/overlays/dev` produces valid manifests; `kustomize
build` is wired into CI.

### S0.3 kind + cloud-provider-kind + Tilt bootstrap **[infra]**

`kind/cluster.yaml`, `Tiltfile`, `scripts/bootstrap.sh`. The bootstrap script
creates the kind cluster and verifies that `cloud-provider-kind` is installed
and running on the host (used to assign LoadBalancer IPs to in-cluster
Services like Envoy).

*AC:* `scripts/bootstrap.sh` succeeds end-to-end on a clean host; `tilt up`
then produces a green UI; Envoy's Service has a LoadBalancer IP.

### S0.4 Contour + cert-manager + step-ca + trust-manager **[infra]**

Installed via Helm with pinned chart versions from `scripts/bootstrap.sh`.
HTTPProxy CRD replaces Ingress. step-ca is the in-cluster CA; cert-manager
issues per-HTTPProxy certs; trust-manager publishes the root bundle as a
ConfigMap for pods that need to trust it.

*AC:* `curl https://app.superjamm.test/` returns 200 with a cert chained to
step-ca's root, after `scripts/dev-setup.sh` has been run once.

### S0.5 CoreDNS-etcd + external-dns + `scripts/dev-setup.sh` **[infra]**

CoreDNS with etcd backend deployed in cluster. external-dns watches HTTPProxy
resources and writes A records to etcd, pointing `*.superjamm.test` at Envoy's
LoadBalancer IP. `scripts/dev-setup.sh` configures the host resolver to use
the in-cluster CoreDNS for the `.superjamm.test` zone, and trusts step-ca's
root in the OS trust store.

*AC:* Adding a new HTTPProxy in the dev overlay automatically resolves to
Envoy's IP without editing `/etc/hosts`; the browser accepts the cert without
a TLS warning.

### S0.6 Postgres in cluster + Prisma scaffold **[infra]**

Postgres StatefulSet (dev sizing). Prisma installed; first migration creates an
empty `tenant` table.

*AC:* `pnpm prisma migrate dev` runs cleanly against the in-cluster Postgres.

### S0.7 Authentik in cluster **[infra]**

Authentik server + worker + redis, pre-seeded with a dev realm, an OIDC
application named `superjamm`, and a single admin user.

*AC:* Authentik UI is reachable at `https://auth.superjamm.test/` and the
application client credentials are available to the app via Kubernetes Secret.

### S0.8 Auth.js wired to Authentik **[feature]**

Login, logout, session, callback. The home page is gated behind sign-in.

*AC:* Visiting `/` while signed out redirects to Authentik; signing in returns
to a "Hello, {first name}" placeholder.

### S0.9 Empty authenticated dashboard placeholder **[feature]**

A simple `/dashboard` page showing the logged-in user's name and a stub list of
"upcoming jams."

*AC:* `/dashboard` renders only when authenticated.

### S0.10 CI pipeline **[infra]**

GitHub Actions: lint, typecheck, unit tests, build container image, render
Kustomize overlays (`kustomize build`). Playwright e2e against ephemeral kind is wired in (may be empty).

*AC:* PRs run the pipeline; main builds an image tagged `:main-<sha>`.

---

## Slice 1 — Tenant & user foundation

Goal: an operator can provision a tenant; users can sign in and exist as
Musicians in that tenant.

### S1.1 Tenant model + repository tenant-scoping **[infra]**

`tenant` table; repository wrapper that injects `tenant_id` filter. ESLint rule
forbidding direct Prisma access outside the repository layer.

*AC:* A unit test demonstrates that a repository query for tenant A returns no
rows belonging to tenant B even when the SQL would otherwise match.

### S1.2 User model + first-sign-in attachment **[feature]**

`user` table. First OIDC sign-in attaches the Authentik subject to the
pre-created user record (by email). New unattached subjects are rejected.

*AC:* Pre-creating a user with `email = "x@y.com"` and signing in via Authentik
as that user attaches them; signing in as an unknown email is rejected with a
clear error.

### S1.3 Provisioning script: `pnpm provision-tenant` **[infra]**

CLI script: creates a tenant, the first Account Holder user record, and seeds
the instrument + genre defaults.

*AC:* `pnpm provision-tenant --name "Foo" --account-holder-email "a@b.com"`
produces a working tenant whose first sign-in is the Account Holder.

### S1.4 User profile page **[feature]**

Read-only first/last/email from OIDC claims; display anonymization preferences
will be added in S7.6.

*AC:* `/me` renders the user's claims-derived profile.

### S1.5 Account Holder management UI **[feature]**

Designate Backup Account Holder; reassign Account Holder. Cannot orphan.

*AC:* Trying to unassign yourself as Account Holder without designating a
successor is blocked with a clear error.

### S1.6 Jam Coordinator role assignment UI **[feature]**

Promote/demote Musicians to Jam Coordinator. House Band assignment is per-venue,
deferred to Slice 4.

*AC:* A Musician promoted to Jam Coordinator sees admin nav items on next page
load; demoted, they no longer see them.

### S1.7 Audit log scaffold **[infra]**

`audit_log` table + helper. Role changes write an entry.

*AC:* Promoting a user produces a row in `audit_log` with `actor_user_id`,
`subject_user_id`, `action`, `tenant_id`, and timestamp.

### S1.8 Invite-by-email flow **[feature]**

Account Holder / Jam Coordinator adds a Musician by email; the record is
pending until that user's first sign-in.

*AC:* The invitee can sign in via Authentik and is attached to the pending
record; the invitee's roster row flips from "pending" to "active."

---

## Slice 2 — Vocabularies + Musician profile

### S2.1 Instrument catalog **[feature]**

Tenant-scoped CRUD seeded with `Lead Guitar`, `Rhythm Guitar`, `Bass`, `Drums`,
`Keys`, `Vocals`.

*AC:* Jam Coordinator can add/rename/archive instruments; deleted instruments
remain referenced by existing data (soft-delete).

### S2.2 Genre catalog **[feature]**

Tenant-scoped CRUD.

*AC:* Jam Coordinator can manage the genre tag list.

### S2.3 Musician instrument proficiency **[feature]**

`user_instrument_proficiency` table; UI on `/me` to declare 1–5 per instrument.

*AC:* The Musician can set proficiency for any instrument in the catalog;
removing an instrument from the catalog hides (but does not delete) the
proficiency record.

---

## Slice 3 — Songs and Master List

### S3.1 Song Catalog CRUD **[feature]**

`song` table + Coordinator-only CRUD. Name, artist, length.

*AC:* Coordinator can create, edit, soft-delete, and restore songs.

### S3.2 Song instrumentation and genres **[feature]**

Song editor exposes required-instrument multi-select and genre multi-select.

*AC:* A song's required instruments and genres persist and round-trip in the
editor.

### S3.3 Song external links **[feature]**

Multi-row link editor (YouTube, charts, notation, tab) per song.

*AC:* Links persist; opening one opens the external URL.

### S3.4 Master List CRUD **[feature]**

Per-tenant Master Lists; add/remove song entries; copy entries between Master
Lists.

*AC:* A Coordinator can build two Master Lists and copy a song from one to the
other in one action.

### S3.5 Soft-delete + restore UI **[feature]**

Restored items rejoin lists they belonged to.

*AC:* A deleted song restored within the UI appears again in any Master List
that referenced it.

---

## Slice 4 — Venues

### S4.1 Venue CRUD **[feature]**

Name, description, address (optional), public/private flag.

*AC:* Coordinator creates and edits venues; soft-deleted venues are hidden from
dashboards but restorable.

### S4.2 Venue constraints — song length + genres **[feature]**

Max song length (soft warning), genre inclusions, genre exclusions.

*AC:* A practice list / request entry referencing a song exceeding the venue's
max length surfaces a warning, not a block.

### S4.3 Venue supported instruments **[feature]**

Multi-select from instrument catalog.

*AC:* A Sit-In may still request to play an unsupported instrument; the
request is flagged as unsupported on the Coordinator's planning view.

### S4.4 House Band assignment **[feature]**

Coordinator assigns Musicians as House Band members at a venue. A Musician may
be House Band at multiple venues.

*AC:* Bill is House Band at Venue X and a Sit-In at Venue Y; both states are
visible on the user's profile and on each venue's roster.

### S4.5 Default-backed instruments per venue **[feature]**

Per-venue list of instruments the House Band backs by default.

*AC:* Coordinator can set per-venue defaults; surfaced on the planning view.

### S4.6 Venue ↔ Master List assignment **[feature]**

Assign one Master List to a venue.

*AC:* Reassigning a Master List to another venue does not duplicate songs.

### S4.7 Default Setup Buffer per venue **[feature]**

Default seconds between songs, used for time estimates.

*AC:* Buffer change reflects in the time estimate on next render.

---

## Slice 5 — Practice Lists

### S5.1 Practice List CRUD **[feature]**

Per-user, multi-list with names. Add/edit/remove entries.

*AC:* A user can have multiple named lists; switching list shows different
entries.

### S5.2 "Copy from Master List" flow **[feature]**

Select-and-copy entries from a venue's Master List into a Practice List.

*AC:* Copied entries arrive with default instrument(s) but no proficiency until
the user sets it.

### S5.3 Manual song entry **[feature]**

User searches the Song Catalog and adds an entry. If no match, the entry is
staged for Coordinator review.

*AC:* A staged song appears in the Coordinator's review queue.

### S5.4 CSV upload **[feature]**

Upload CSV with a column-mapping UI. Errors surfaced per row.

*AC:* A malformed row reports a clear error and does not block valid rows.

### S5.5 Fuzzy matching **[feature]**

`(name, artist)` fuzzy match against the Song Catalog (Postgres `pg_trgm`).
Ambiguous matches prompt confirm/override.

*AC:* Uploading "Deadflowers / Rolling Stones" matches "Dead Flowers / The
Rolling Stones" and prompts the user to confirm.

### S5.6 Constraint validation warnings **[feature]**

Practice list entry warns on max-length or genre exclusion based on the venue
the practice list is intended for (user-selectable).

*AC:* Warnings appear inline; the entry still saves.

### S5.7 Coordinator-on-behalf practice list **[feature]**

Coordinator creates/edits a Practice List for another Musician.

*AC:* The owning Musician sees the list as theirs; the Coordinator's actions
are audit-logged.

---

## Slice 6 — Jam scheduling and dashboards

### S6.1 Jam CRUD **[feature]**

Schedule a jam at a venue (UTC stored). Cancel (soft) and reschedule.

*AC:* A cancelled jam disappears from forecasted views but is restorable by a
Coordinator.

### S6.2 Visibility inheritance **[feature]**

Jam inherits venue public/private, overridable.

*AC:* Toggling a jam to public on a private venue still keeps the *venue*
private on the public dashboard.

### S6.3 Public dashboard route **[feature]**

Stateless `/public/venues/:id` route; no cookies set.

*AC:* The route responds with no Set-Cookie header and shows only public
content.

### S6.4 Private dashboard route **[feature]**

Authed `/dashboard` shows all tenant venues and jams.

*AC:* Switching between forecasted and archived views works.

### S6.5 In-app notification framework **[infra]**

`notification` table; UI dropdown; subscriptions for role changes and jam
events.

*AC:* Scheduling a jam fires notifications to relevant users; opening the
notification marks it read.

---

## Slice 7 — Request List + Set List

### S7.1 Request List CRUD **[feature]**

Per musician per jam: song + one instrument; optional house-band-guidance flag.

*AC:* A Musician can submit, edit, and withdraw requests for an upcoming jam.

### S7.2 Coordinator planning view **[feature]**

Per-jam aggregate of all requests; flags conflicts and unsupported instruments.

*AC:* Two requests for the same song on the same instrument show as a flagged
conflict with a one-click "pick this one" resolution.

### S7.3 Set List build **[feature]**

Coordinator assembles ordered Set List from requests and/or Master List entries.

*AC:* Drag-and-drop reordering persists.

### S7.4 Set List time estimate **[feature]**

Sum of song lengths + venue/jam Setup Buffer.

*AC:* Adding a song updates the total estimate; jam buffer override is
respected.

### S7.5 Display anonymization preference **[feature]**

User × scope (public, private). Set List rendering honors preference.

*AC:* A user opted out of public display renders as "Anonymous" on the public
dashboard but their name on the private dashboard.

---

## Slice 8 — SuperJamm recommendations

### S8.1 Cross-match algorithm **[feature]**

Identify songs requested by ≥2 musicians on complementary (non-conflicting)
instruments.

*AC:* Given Dale → Lead Guitar on *Dead Flowers* and Bill → Bass on *Dead
Flowers*, the planning view surfaces the song as a "coordinated candidate."

### S8.2 Recommendation surface in planning view **[feature]**

Recommendations appear in a side panel; one-click "add to Set List."

*AC:* Adding a recommended song populates the Set List entry with the
musicians and instruments inferred from the requests.

### S8.3 Suggested ordering heuristic **[feature]**

Order recommendations by variety (instrument coverage) and time-fit against
the venue's typical jam window.

*AC:* The suggested ordering is deterministic and documented in a code
comment.

---

## Slice 9 — Live jam + Kiosk

### S9.1 Jam state machine **[infra]**

`scheduled → planning → live → archived`. Transitions guarded.

*AC:* Attempting an invalid transition (e.g., `archived → live`) is rejected
with a clear error.

### S9.2 Start / end jam actions **[feature]**

Coordinator can start (transition to live) and end (transition to archived).

*AC:* Start emits a state change; the live dashboard becomes available.

### S9.3 Polling endpoint **[infra]**

`GET /api/jams/:id/state` with ETag / `If-None-Match`. 5-second client poll.

*AC:* Repeated polls without changes return 304.

### S9.4 On-the-fly Set List edits **[feature]**

Coordinator can reorder, add, remove during live mode; changes appear within
one poll interval on the private dashboard.

*AC:* Edit persists and propagates to all connected viewers within 5 seconds.

### S9.5 Live Request submission **[feature]**

Musicians can submit/edit requests during a live jam.

*AC:* A request submitted during live appears in the Coordinator's planning
panel within the next poll.

### S9.6 Kiosk URL **[feature]**

`/kiosk/jams/:id` is anonymous-by-default, scoped to one jam, shows current
Set List.

*AC:* The kiosk route requires no auth to read; refreshes via the same polling
mechanism.

### S9.7 Kiosk: Musician self sign-in **[feature]**

A "sign in" affordance on the kiosk lets a Sit-In submit their own request
without leaving the kiosk URL.

*AC:* Sign-in completes via Authentik and returns to the kiosk view with the
request form available.

### S9.8 Kiosk: Coordinator impersonation **[feature]**

A signed-in Jam Coordinator can pick a registered Musician from the tenant's
roster and submit a Request on their behalf.

*AC:* Free-form names are not accepted; the dropdown is restricted to tenant
users. The submitted request is attributed to the impersonated user with an
`actor` audit-log entry naming the Coordinator.

### S9.9 Own-name visibility during live jam **[feature]**

A user opted out of private-dashboard display still sees their own name on the
private dashboard during live mode.

*AC:* Other tenant users do not see this user's name; the user themselves does.

---

## Slice 10 — Archive + hard-anonymization

### S10.1 Archive jam **[feature]**

Coordinator archives a completed jam; Set List freezes; jam drops from
forecasted views.

*AC:* Attempting to edit an archived jam's Set List is blocked.

### S10.2 Public archive view **[feature]**

Per-venue setting toggles whether archived Set Lists appear on the public
dashboard.

*AC:* When enabled, archived jams appear on the public route with anonymization
applied.

### S10.3 Anonymous rendering everywhere **[feature]**

Set List entries linked to display-anonymized or hard-anonymized users render
as literal "Anonymous."

*AC:* Snapshot tests cover public dashboard, private dashboard, and kiosk
renderings.

### S10.4 Hard-anonymize flow **[feature]**

Musician-initiated permanent anonymization of past performances. Explicit
confirm dialog warning of irreversibility.

*AC:* After confirm, all `set_list_entry.user_id` references for this user are
overwritten; the user record remains; the user cannot un-anonymize.

### S10.5 Anonymization test coverage **[infra]**

Test matrix: {display, hard} × {public, private, kiosk} × {live, archived}.

*AC:* All combinations covered with deterministic snapshot or assertion tests.

---

## Out of scope for v1 (do not pull into a slice)

- Self-serve tenant sign-up
- k3s appliance release packaging
- Email / push notifications
- Conditional sit-in offers
- Auto conflict resolution
- Nashville-numbered charts with lyrics
- Cross-tenant identity
- Realtime push (WebSocket/SSE)
- Multi-region deploy
- Metrics / tracing backend

## Suggested order of attack

Slices are roughly dependency-ordered. Slice 0 must come first; Slice 1 next.
After Slice 1, Slices 2–4 can be partially parallelized if more than one
person is working. Slices 5–10 depend on the data shapes from 2–4.

A first demoable internal milestone is **Slice 6** (auth + venues + master
lists + jams visible on a dashboard) — enough to show the shape of the product
without yet supporting the live-jam workflow.
