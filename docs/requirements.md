# SuperJamm — Features and Requirements

This document captures the functional requirements for *SuperJamm*. Technical and
solution details live in [`technical-requirements.md`](technical-requirements.md).
A *Decisions* appendix at the end captures the meaningful product-level choices
with rationale, indexed as `D-R#`.

## Glossary

- **Tenant / Account** — top-level scope. One tenant has one Account Holder, one or more Jam Coordinators, many Musicians, and one or more Venues.
- **Account Holder** — the user who owns the tenant. Always exactly one; a Backup Account Holder may be designated.
- **Jam Coordinator** — admin of the tenant; manages venues, master lists, schedules and runs jams.
- **Musician** — base role. Every signed-up user is a Musician by default. Carries instrument proficiencies (1–5) independent of any song.
- **Sit-In** — a Musician not on the House Band who plays a jam.
- **House Band** — Musicians assigned to back the standard set at a specific venue. Per-venue assignment; a Musician may be a House Band member at multiple venues.
- **Venue** — a location where jams are held. Has constraints (song length, genres, instruments) and a default Setup Buffer.
- **Jam (Session)** — an event at a venue on a specific date; public or private; has a Set List.
- **Song Catalog** — tenant-scoped catalog of songs.
- **Master List** — per-venue curated list, drawn from the Song Catalog, that the Jam Coordinator endorses for that venue.
- **Practice List** — per-musician list of songs they want to play; a musician may have multiple named lists.
- **Request List** — per-musician, per-jam list of songs they want to play at that jam.
- **Set List** — the songs forecasted or performed at a jam.

## Roles and Permissions

### Account Holder

Bootstrap role for a tenant. Inherits all Jam Coordinator capabilities. May:

- Designate a Backup Account Holder
- Reassign Account Holder to another active user (cannot orphan the role)

### Jam Coordinator

Tenant admin. Inherits all Musician capabilities. May:

- Create/update/archive/restore venues, venue constraints, and venue ↔ Master List assignments
- Maintain the Song Catalog and Master Lists
- Schedule, plan, run, and archive jams
- Create or maintain practice lists on behalf of other Musicians
- Promote a Musician to House Band (at a specific venue) or Jam Coordinator
- Resolve sit-in conflicts during planning
- Impersonate a registered Musician in Kiosk mode to submit a Request on their behalf

### Musician (default role)

Every signed-up user. May:

- View the jam schedule (public and private dashboards for tenants they belong to)
- Maintain personal instrument proficiencies
- Maintain one or more practice lists
- Submit a Request List for a scheduled or active jam
- Opt in/out of display anonymization at any time
- Trigger hard-anonymization of past performances (irreversible)

### House Band (at Venue X)

An assignment, not a separate role. Musicians with this assignment back the venue's
default-backed instruments unless a Sit-In displaces them. v1 grants no special
functionality beyond the Musician/Sit-In capabilities; the assignment is recorded
for planning and visibility.

## Account & Tenant Lifecycle

- **v1: admin-provisioned.** New tenants are created by a SuperJamm platform operator (CLI script or admin UI). Self-serve sign-up is a v2 goal.
- **Standalone mode** must run on a developer workstation in v1. The same code path will inform v2+ on-prem venue-appliance deployments (e.g., k3s on dedicated hardware); the appliance itself is not a v1 deliverable.
- A user belongs to exactly one tenant in v1. Cross-tenant identity (one user, multiple tenants) is out of scope.
- The Account Holder may invite Musicians to the tenant by email; on the invitee's first OIDC sign-in, their session is attached to the pre-created user record.
- **Product domain.** The SaaS deployment is hosted at [`superjamm.live`](https://superjamm.live) — marketing and user docs at the apex, the web app at `app.superjamm.live`, Authentik at `auth.superjamm.live`. The domain is not yet active.

## Identity & Privacy

### PII

Only the following PII is stored per user:

- First name
- Last name
- Email (used as the OIDC principal)

### Authentication

OIDC-only. Authentik is the application's identity provider in both dev and prod.
In prod, Authentik federates Google, Microsoft, and Apple. The public dashboard is
viewable by anonymous, stateless visitors — no cookie or session is set for them.

### Anonymization

Two distinct mechanisms:

1. **Display anonymization** (reversible). A Musician opts out of display on the public dashboard, the private dashboard, or both. The system continues to store their performance data; the rendered Set List shows the literal word *"Anonymous"* where their name would appear.
2. **Hard anonymization** (irreversible). The Musician triggers permanent anonymization of their historical performance records. The user link on past Set List entries is overwritten. Cannot be undone, even if the user later opts back into display. The UI surfaces this with an explicit confirm dialog that warns of irreversibility.

A Musician's display anonymization preference does not hide their *own* name from
*themselves* on the private dashboard during a live jam — they need to see their
place in the order.

### Age and Content

SuperJamm targets all ages. The platform hosts no DMCA-protected or
age-restricted content. Song records may *link to* external content (YouTube
performances, charts, tabs) but do not host it. Jam Coordinators are responsible
for keeping venue/jam/song names age-appropriate.

### Copyright

No hosted copyrighted material in v1. Rudimentary Nashville-numbered charts with
lyrics are a v2 candidate — see the caution under *Lyrics and charts* below.

#### SuperJamm handles no audio

**SuperJamm stores, processes, transmits, and plays back no audio, in v1 or
after.** This is a permanent product boundary, not a deferred feature. See
`D-R21` for the full rationale.

Backing tracks are a real part of how jams run, and SuperJamm supports them —
but only as *metadata*. It records that a track exists, which instruments it
covers, and who will play it. It never touches the file.

| SuperJamm does | SuperJamm does not |
|---|---|
| Record which instruments a song needs | Store audio files of any kind |
| Record which parts the house band covers | Accept audio uploads |
| Record which parts sit-ins cover | Separate, mix, or render audio |
| Record that a backing track covers the remainder, and who holds it | Host, stream, proxy, or embed a backing track |
| Flag a song as live-only when no track is available | Instruct users on how to obtain or create one |

The operative rule is **no transfer**: every copy stays with whoever holds the
right to it. A venue, house band, or coordinator that purchases a licensed
backing track plays it from their own device and does not distribute it to
sit-ins. A sit-in who brings their own material plays it from their own device
and does not hand it to the venue. SuperJamm mediates the *plan*, never the
media.

#### Lyrics and charts

Lyrics are copyrighted separately from recordings, and reproducing or
displaying them is its own licensing regime — the same shape of problem as
audio, not a lesser one. The v2 charts feature must therefore distinguish:

- **Lyrics metadata** — that a song has lyrics, where a performer's own copy
  lives, which key a chart is in. Clean; this is bookkeeping.
- **Lyrics text stored or displayed by SuperJamm** — hosting copyrighted
  material, contrary to the posture above, and requiring a licence.

Treat any proposal to store lyric text as a licensing decision requiring
explicit sign-off, not a feature increment.

**The sanctioned pattern is an outbound link.** A song record may carry a URL
to an external lyrics site, exactly as *Age and Content* already permits for
YouTube performances, charts, and tabs. A link is not reproduction,
distribution, or display — the party serving the bytes performs the act
(*Perfect 10 v. Amazon*, 9th Cir. 2007). Four rules keep it that way:

1. **Never fetch, scrape, or cache the text.** Retrieving lyrics to render a
   preview or to speed up display puts the text in our database, which is the
   thing being avoided. The link stays a link.
2. **Link out; do not frame or embed.** An iframe or in-app reader is closer to
   display than a plain outbound link and forfeits the distinction above.
3. **Prefer licensed providers** (Musixmatch, Genius, LyricFind and similar
   operate under publisher agreements). Curating links to sites known to be
   unlicensed is where the residual risk sits.
4. **If links are user-submitted, register a DMCA designated agent.** §512(d)
   safe harbor covers "information location tools," but only with a designated
   agent and a working takedown process. Cheap, and applies to any
   user-submitted content, not just lyrics.

Note for non-US deployment: the server test is US law. *GS Media v. Sanoma*
(CJEU 2016) holds that for-profit linking to known-infringing content can
itself be a communication to the public in the EU.

## Domain Entities and Vocabularies

### Instruments

Tenant-configurable catalog. Defaults: `Lead Guitar`, `Rhythm Guitar`, `Bass`,
`Drums`, `Keys`, `Vocals`. Variant instruments (e.g., Lead vs. Rhythm Guitar) are
first-class — they are distinct instrument records, not a sub-attribute of a
generic "Guitar."

### Genres

Tenant-configurable controlled tag list. Used on song records and in venue
inclusion/exclusion constraints.

### Proficiency

Numeric scale 1–5, used in two distinct relationships:

- **Musician × Instrument** — a Musician's overall comfort with an instrument, independent of any song. Drives "needs guidance" decisions.
- **Practice List Entry × Instrument** — a Musician's comfort with a specific song on a specific instrument, scoped to a practice list entry.

### Songs

A song record in the Song Catalog has:

- name
- artist
- length (seconds)
- genres (multiple tags)
- required instruments (multiple, drawn from the instrument catalog)
- external links (YouTube, charts, notation, tab) — optional

The Song Catalog is tenant-scoped. A song is one record regardless of how many
Master Lists reference it.

### Master List

A per-venue list referencing songs from the Song Catalog. A Jam Coordinator
managing multiple venues may share a Master List across venues or maintain
separate ones; the system provides copy operations to move entries between lists.

### Practice List

A per-musician list. A Musician may have multiple practice lists by name. Each
entry references a song (from the Song Catalog, if matched) and carries:

- Instrument(s) the Musician intends to play on the song
- Song-specific proficiency (1–5) per instrument
- Optional house-band-guidance request flag

Entry sources:

1. **Copy from Master List** — for tech-shy musicians; the Jam Coordinator may also generate on a musician's behalf.
2. **Manual entry** through the SuperJamm UI.
3. **CSV upload** (CSV; Google Sheets/Excel via CSV export). Upload flow flags any song not matching the Song Catalog and any song violating venue constraints. Matching is **fuzzy on `(name, artist)`** with a coordinator/musician-confirm fallback when fuzzy match is ambiguous. Unmatched songs are staged for Jam Coordinator review.

### Request List

A per-musician, per-jam list. Each entry references a song and one instrument
from the venue's instrument vocabulary. Optional flag for house-band guidance.
v1 does not support conditional offers ("I'll play bass if no one else does");
that is a v2 candidate.

### Set List

A per-jam ordered list of songs with musician/instrument assignments. Compiled
by the Jam Coordinator from Request Lists, with assistance from SuperJamm
recommendations (below).

## Venue Constraints

Per-venue, all optional:

- **Max song length** — exceeding it produces a soft warning, never a block.
- **Genre inclusions** — genres explicitly allowed.
- **Genre exclusions** — genres explicitly disallowed.
- **Supported instruments** — the instruments the House Band can back / support sit-ins on. Sit-ins are *not* restricted to these; an unsupported instrument simply means no house-band backing is available for it.
- **Setup Buffer (default)** — default time between songs for setup, used in time estimates. Overridable per jam.

## Dashboards

Two dashboards per venue:

- **Public dashboard** — visible to anonymous, stateless viewers. Shows public venue details, scheduled public jams, and (optionally) archived public Set Lists.
- **Private dashboard** — visible to logged-in tenant users. Shows all jams (public and private), forecasted and past Set Lists, and live-jam state.

**URL shape:** path-based under the app subdomain.

- Public dashboard: `app.superjamm.live/public/<tenant-slug>/<venue-slug>`
- Private dashboard: `app.superjamm.live/dashboard` (post-auth, tenant inferred from session)

During an active jam, a Musician who has opted out of display anonymization may
still see their own name on the **private** dashboard. They do not see their name
on the public dashboard.

## Kiosk Mode

A dedicated per-jam URL renders an anonymous-by-default view of the active jam's
Set List. From the kiosk:

- A Musician may sign in to submit/edit their own Request List.
- A Jam Coordinator may impersonate a registered Musician (chosen from the tenant's roster) to submit a Request on their behalf. Free-form names are not allowed — the target Musician must exist as a tenant user.

## Notifications

In-app only in v1. Notifications fire on:

- Role assignment / removal
- Jam scheduled / rescheduled / cancelled
- (Coordinator) new Request List submitted for an upcoming jam
- (Coordinator) practice list flagged because of unmatched songs or constraint violations

Email and push notifications are out of scope for v1.

## SuperJamm Recommendations

When a Jam Coordinator is pre-planning a jam, SuperJamm:

1. **Cross-matches Request Lists** — finds songs requested by two or more musicians on complementary instruments. Example: Dale requests *Dead Flowers* on Lead Guitar, Bill requests it on Bass — surface the song as a coordinated set candidate.
2. **Estimates Set List runtime** — sums song lengths plus the venue's Setup Buffer (or the jam's override).
3. **Suggests an order** — initial ordering optimizing for variety and coverage. The Jam Coordinator may reorder, add, or remove anything; SuperJamm never makes the final call.

Auto-resolution of conflicts (two musicians want the same instrument lead on the
same song) is **out of scope for v1**. Conflicts are flagged; the Jam Coordinator
decides. v2 may add auto-resolution informed by attendance history and proficiency.

## Jam Lifecycle

1. **Scheduled** — Jam Coordinator creates the jam at a venue with a date/time. Inherits venue visibility; overridable.
2. **Planning** — Musicians submit/edit Request Lists. The Jam Coordinator reviews recommendations and builds a forecasted Set List, visible on the private dashboard.
3. **Live** — Jam Coordinator starts the jam. The Set List is editable on the fly. Musicians may submit/update Requests during the jam (latecomers, no-shows). The private dashboard reflects current state via polling.
4. **Archived** — Jam Coordinator archives the completed jam. The Set List freezes. The jam falls off forecasted schedule views. If the venue's public archive is enabled, the public dashboard shows the final Set List with anonymization applied per musician preference.

## Maintenance Operations

CRUD with soft-delete and restore for:

- Songs in the Song Catalog
- Master Lists and Master List entries
- Practice Lists and entries
- Venues and venue constraints
- Jams
- Users (deactivation, not deletion)

Restore is performed by a Jam Coordinator.

## Time Zones

All times stored in UTC, rendered in the viewer's browser-local time zone.

## Out of v1 (recorded so we don't forget)

- Self-serve tenant sign-up
- On-prem venue-appliance release target (the code path is shared, but the appliance is not a v1 deliverable)
- Email / push notifications
- Conditional sit-in offers
- Auto conflict resolution
- Nashville-numbered charts with lyrics (subject to the licensing caution under *Copyright*)
- Cross-tenant identity
- Multi-cluster federation
- Realtime push (WebSocket/SSE) — v1 uses polling

## Decisions

This appendix captures the meaningful product-level decisions made during
requirements work, with brief rationale. New decisions are appended;
superseded entries are marked as such, not deleted. Reference entries by
their `D-R#` tag.

### D-R1 — Tenant = Account = one Account Holder

A SuperJamm tenant has exactly one Account Holder. A Backup Account Holder may
be designated; reassignment is supported; orphaning is not.

*Why:* keeps role bootstrap unambiguous; defers cross-tenant identity to v2+.

### D-R2 — v1 is admin-provisioned

New tenants are created by a SuperJamm platform operator (CLI script or admin
UI). Self-serve sign-up is a v2 goal. Standalone mode must run on a developer
workstation in v1; on-prem appliance deployments are v2+.

*Why:* removes self-service sign-up surface, abuse handling, and billing
infrastructure from v1 scope.

### D-R3 — A user belongs to exactly one tenant in v1

Cross-tenant identity is out of scope.

*Why:* removes federation-of-self complexity; matches the user expectation of
"I belong to my venue's organization."

### D-R4 — Song Catalog is tenant-scoped; one record per song

A song appears once in the Song Catalog regardless of how many Master Lists
reference it. Catalog is scoped to the tenant.

*Why:* normalizes data; makes cross-match and constraint validation tractable.

### D-R5 — Catalog matching is fuzzy on `(name, artist)`, with confirm fallback

Practice list and request list entries are matched against the catalog via
fuzzy match on song name and artist; ambiguous matches prompt the user to
confirm or override.

*Why:* musicians enter data inconsistently ("The Rolling Stones" vs. "Rolling
Stones"); strict match would frustrate them and bloat the catalog with
duplicates.

### D-R6 — Proficiency lives in two distinct relationships

- **Musician × Instrument** drives "needs guidance" decisions independent of song.
- **Practice List Entry × Instrument** captures comfort with a specific song.

*Why:* an advanced player may have low proficiency with a *specific song* but
still not need guidance; conflating the two would over-flag.

### D-R7 — Instrument variants are first-class

`Lead Guitar`, `Rhythm Guitar`, and `Guitar` are distinct instrument records
in the tenant catalog, not sub-attributes of a generic "Guitar."

*Why:* request and set-list semantics rely on this distinction; flattening
loses meaningful planning information.

### D-R8 — Instruments and genres are tenant-configurable controlled vocabularies

Instrument defaults are seeded (Lead Guitar, Rhythm Guitar, Bass, Drums, Keys,
Vocals); tenants may edit. Genres are controlled tags, not free text.

*Why:* controlled vocabularies make venue constraints and cross-matching
reliable.

### D-R9 — Venue max-song-length is a soft warning, not a block

Exceeding the limit warns the user; does not block submission.

*Why:* jams sometimes intentionally exceed the limit; warnings beat false
blocks.

### D-R10 — Venue genre constraints are dual lists (inclusion AND exclusion)

A venue may specify both genres it explicitly allows and genres it explicitly
disallows.

*Why:* "rock and country only, but no metal" is a real-world configuration
that single-direction lists can't express cleanly.

### D-R11 — "Supported instruments" at a venue means HB-backable, not restrictive

Sit-ins may request to play any instrument; an unsupported instrument simply
means no house-band backing is available for it.

*Why:* restricting sit-ins by instrument would close the door on unusual but
desirable acts.

### D-R12 — House Band is a per-venue assignment; a Musician may be HB at multiple venues

"House Band" is not a global role flag; it is a relationship between a user
and a specific venue.

*Why:* musicians often play in multiple house bands; modeling it per-venue
captures reality and supports the multi-venue tenant case.

### D-R13 — Requests are unconditional in v1

"I'll play bass if no one else needs it" is not supported in v1.

*Why:* conditional offers add significant conflict-resolution complexity;
deferred to v2+.

### D-R14 — The Jam Coordinator resolves request conflicts manually in v1

SuperJamm flags conflicts (two musicians want the same instrument lead on the
same song) but does not auto-resolve.

*Why:* trust-building feature in v1; algorithmic resolution arrives in v2+
informed by attendance history and proficiency.

### D-R15 — Display anonymization renders the literal word "Anonymous"

The rendered Set List shows *"Anonymous"* in place of the musician's name;
the entry is not hidden.

*Why:* the audience still sees that the song was performed; only the
identity is withheld.

### D-R16 — Hard anonymization is irreversible and requires explicit confirm

A musician may permanently anonymize their past performances. User
references on past Set List entries are overwritten. UI surfaces an explicit
confirm dialog warning of irreversibility.

*Why:* honors the right to be forgotten; the warning prevents accidental
loss.

### D-R17 — Notifications are in-app only in v1

Email and push notifications are out of scope for v1.

*Why:* email/push add deliverability, templates, suppression list, and opt-in
management — significant infrastructure not worth v1 scope.

### D-R18 — Kiosk is anonymous-by-default; coordinator impersonation requires a registered Musician

The kiosk URL is readable without authentication. Coordinator impersonation
picks from registered tenant Musicians, not free-form names.

*Why:* prevents creation of "ghost" musicians and preserves anonymization
integrity.

### D-R19 — Songs in the catalog carry their required instrument list

A song record includes the instruments it requires (e.g., *Dead Flowers*
requires Lead Guitar, Rhythm Guitar, Bass, Drums, Vocals).

*Why:* cross-match recommendations need a concrete instrument list to reason
about; deriving the list from performance history is unreliable in early
data.

### D-R20 — Times stored in UTC; rendered in the viewer's browser local time zone

*Why:* avoids per-venue timezone configuration; users get a familiar display.

### D-R21 — SuperJamm handles no audio, permanently

Backing tracks are tracked as metadata only: that one exists, which instruments
it covers, and who holds and plays it. SuperJamm never stores, processes,
transmits, or plays back audio. This is a permanent boundary, not a v1 deferral.

*Why:* three reasons, in order of weight.

**1. It keeps SuperJamm out of the copyright analysis entirely.** Backing tracks
derived from commercial recordings are reproductions and derivative works of the
sound recording (17 U.S.C. §106(1) and §106(2)). Once audio bytes enter the
system, SuperJamm has to reason about the provenance of every one of them.
Holding no audio means the question never arises. The venue's PRO blanket
licence already covers public performance of the *compositions*, which is what
makes live covers at a jam legal in the first place — SuperJamm needs nothing
further to support the jam itself.

**2. Facilitating at scale is a different posture than doing it once.** An
individual musician making a practice mix from a record they own faces
negligible risk. A *platform* that organizes, encourages, or stores that
activity across many venues invites contributory and inducement liability,
where intent expressed in how a tool is promoted was decisive in
*MGM v. Grokster* (2005). SuperJamm's leverage is coordination; that leverage
cuts both ways.

**3. The audio problem is someone else's, and better solved.** Licensed
multitrack vendors already sell exactly what a jam needs — per-instrument stems,
cleared for use, because they licence the composition and record their own
masters. There is no product gap for SuperJamm to fill, only liability to
absorb.

*Consequences:*

- No upload endpoints, no file or blob columns on any domain table, no
  attachment storage, no proxying or embedding of third-party streams.
- The schema carries **no value describing a track derived from a commercial
  master**. This omission is deliberate: a field that records the act also
  normalizes it, and a documented expectation is evidence of intent. A schema
  that cannot express it cannot endorse it.
- When no licensed track exists and the song is not the performer's own
  composition, the correct product answer is **live-only** — never "the
  musician supplies stems." Do not reintroduce that phrasing as a convenience.
- Erosion happens one feature at a time ("just let the coordinator upload the
  track so it's ready on the night"). Treat any proposal that puts audio bytes
  in the system as a change to this decision, requiring explicit sign-off.

*Related:* the full legal analysis behind this decision — fair use factors,
the composition/sound-recording split, public-domain terms, licensed vendor
structure, and why cloud separation services do not change the position — lives
in the sibling `song-track-tools` project's `features.md` §3 and §8. It is
deliberately kept out of this repository; that project is a private, local-only
toolset and is not a SuperJamm dependency.
