# SuperJamm

**Smart coordination for open jam sessions** — helping venue hosts, house
bands, and sit-in musicians run better, more inclusive nights of live music.

Product home (coming soon): [`superjamm.live`](https://superjamm.live).

## The Problem

Open jam sessions are the proving ground of live music: a venue, a house
band, and a rotating cast of sit-in musicians playing through a setlist that
often gets sorted out *during* the show. Week after week, it looks like this:

- The coordinator scribbles on a clipboard or a notes app, trying to remember who can play what.
- Musicians don't know what songs are coming or whether their request will make the set.
- Two guitarists arrive expecting to play the same song; nobody knows.
- The audience has no way to see what's been played or what's coming next.
- Sit-ins with limited experience show up hoping for guidance; the house band has no advance warning.

The information needed to run a great jam exists — it's just scattered across
heads, notebooks, and group chats.

## The Solution

SuperJamm is a web application that captures that information once, organizes
it by venue, and gives everyone the view they need:

- **Coordinators** get a planning surface and a live-jam control panel.
- **Musicians** declare what they can play and what they want to play tonight.
- **The audience** gets a public schedule and an optional archive of past performances.
- **House bands** know in advance who they'll be backing and on what.

The smart part — *SuperJamm Recommendations* — looks across all the sit-in
requests for an upcoming jam and surfaces songs that multiple musicians want
to play on complementary instruments. The coordinator gets a ready-made
candidate set to weave into the night.

## Who It's For

| Audience | What they get |
|---|---|
| **Venue hosts / jam coordinators** | A single place to manage the calendar, the house band, the master song list, and the set list — before, during, and after each jam. |
| **Sit-in musicians** | A profile for instruments and proficiency, practice lists built from a venue's catalog, and per-jam request submission. |
| **House bands** | Per-venue assignments and visibility into what's coming up. |
| **Audience and friends** | A public dashboard per venue, with the night's setlist and optional archived past performances. |

## High-Level Features

### Running the night

- Public and private dashboards per venue, with upcoming jams and live setlist views
- Live jam mode with on-the-fly setlist editing and late-arriving requests
- Kiosk view for the venue: anonymous-by-default setlist display; coordinator can submit requests on behalf of registered musicians
- In-app notifications for schedule changes, role assignments, and request submissions

### Before and after the night

- Song catalog and per-venue master lists (with copy across venues)
- Per-musician practice lists — multiple named lists, with manual entry or CSV import
- Per-jam request lists with optional house-band-guidance flag
- Archived setlists with privacy-respecting attribution

### Smart help

- *SuperJamm Recommendations* — cross-matching sit-in requests to find songs that multiple musicians want to play on complementary instruments
- Set-list time estimation using per-venue setup buffer (overridable per jam)
- Practice-list validation against venue constraints (length, genre, instruments)

### Trust and privacy

- OIDC sign-in via Google, Microsoft, or Apple
- Per-user display anonymization (reversible)
- Hard anonymization of past performances (irreversible, user-initiated)
- All-ages content posture; no hosted copyrighted material

### Multi-venue

- One organization (tenant) can host multiple venues with shared or separate master lists
- Per-venue instrument catalogs and house-band assignments
- A musician can be in the house band at multiple venues

## How It Works in 30 Seconds

A Tuesday-night open mic at *The Backline*:

1. The coordinator schedules next week's jam in SuperJamm. The venue's master list and house band carry over automatically.
2. Sit-ins browse the master list and submit a small request list — "Dale on Lead Guitar for *Dead Flowers*; Bill on Bass for *Dead Flowers*."
3. The night before, the coordinator opens the planning view. SuperJamm flags *Dead Flowers* as a coordinated candidate (two musicians, complementary instruments). One click drops it into the planned setlist.
4. The night of, the coordinator starts the jam. The setlist is editable on the fly; late-arriving sit-ins submit requests from their phones or the venue's kiosk.
5. After the jam, the coordinator archives it. The public dashboard shows the night's setlist — with each musician's anonymization preference respected — for fans to browse.

## Status

SuperJamm is in **early design**. The repository currently contains the
specification and the v1 build plan; implementation begins with the first
slice. See [Project Documents](#project-documents) for the full spec.

## Roadmap

**v1 (in planning):** everything in *High-Level Features* above, for venue
hosts running one or more venues per tenant.

**v2 and beyond (deferred, not lost):**

- Self-serve tenant sign-up (vs. admin-provisioned)
- On-premise venue-appliance deployments
- Email and push notifications (v1 is in-app only)
- Conditional sit-in offers ("I'll play bass if no one else does")
- Auto-resolution of sit-in conflicts based on attendance history and proficiency
- Nashville-numbered charts with lyrics
- Cross-tenant identity (one user, multiple organizations)

## Project Documents

- [Functional Requirements](./docs/requirements.md) — concepts, roles, domain entities, jam lifecycle, privacy. Includes a *Decisions* appendix (`D-R#`).
- [Technical Requirements](./docs/technical-requirements.md) — stack, deployment, multi-tenancy, dev environment. Includes a *Decisions* appendix (`D-T#`).
- [Backlog](./docs/backlog.md) — v1 plan sliced into vertical increments with acceptance criteria.
- [CLAUDE.md](./CLAUDE.md) — working notes and architectural invariants for AI-assisted development.
