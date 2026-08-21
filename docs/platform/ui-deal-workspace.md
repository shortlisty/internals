# VenueMi — UI: Deal Workspace

> **Audience:** Frontend engineers, designers.
> **Purpose:** Component structure, data model, interaction patterns, and file placement for the
> Deal Workspace — the shared planner–client space where a venue proposal is assembled, reviewed,
> and approved.

---

## Related Documents

- [ui-venue-management.md](ui-venue-management.md) — Venue catalog UI, form spec, AnyVenue abstraction
- [data-model.md](data-model.md) — venue, venue_annotations, canonical metadata
- [api.md](api.md) — REST endpoints
- [architecture-overview.md](architecture-overview.md) — FSD layers, addon pattern, plan entitlements
- [events.md](events.md) — plan entitlement codes (`white_label`, `api_access`)
- [ui-shared-packages.md](ui-shared-packages.md) — shared workspace packages, drift prevention rules
- [business/digital-sales-room-for-events/product.md](../business/digital-sales-room-for-events/product.md) — product layer 2 (Digital Sales Room)

---

## 1. Concept

The Deal Workspace is the **shared space** between a planner and a client, built around a
`Proposal` — a curated selection of venues assembled from the planner's catalog.

```
Planner's catalog  ──select venues──►  Proposal  ──share link──►  Client view
                                           │
                                   immutable history
                                   confidence score
                                   AI assistance
                                   bilateral control
                                           │
                                       Approve  ──►  Snapshot (locked, timestamped)
```

Key properties:

- **One link, no client login.** Client opens a branded URL — no account, no app.
- **Bilateral control.** Planner controls what the client sees. Client controls their responses and
  the final Approve.
- **Immutable history.** Every significant action is logged. Nothing can be edited retroactively.
  Trust is built through transparency, not promises.
- **Progressive:** planner sends a draft link; client comments; planner refines; client approves.
  Neither side is forced into a single-shot workflow.

---

## 2. Core Entities

### `Proposal`

```typescript
export type ProposalStatus =
  | "DRAFT" // planner is assembling, not yet sent
  | "SHARED" // link sent to client, awaiting interaction
  | "IN_REVIEW" // client has opened and is actively commenting
  | "APPROVED" // client clicked Approve — snapshot locked
  | "ARCHIVED"; // closed without approval
```

> `PROPOSAL_STATUSES`, `PROPOSAL_EVENT_TYPES`, `CLIENT_PREFERENCES`, and `CONFIDENCE_THRESHOLDS`
> are defined in `@venuemi/ui-types` and re-exported from `src/addons/deal-workspace/index.ts`.
> They are not redefined in the addon — the addon imports and re-exports them.

export interface Proposal {
  id: string;
  tenantId: string;
  title: string;
  clientName: string;
  clientEmail: string | null;
  status: ProposalStatus;
  brandingEnabled: boolean; // plan gate: white_label
  shareToken: string; // opaque token — part of public URL
  shareUrl: string; // full URL sent to client
  eventDate: string | null; // ISO date — drives retention policy
  /**
   * IAM user id of the agency member who owns this proposal.
   * Mandatory — every proposal must have exactly one owner on the agency side.
   * Shown to client as "Your contact", receives all client activity notifications.
   * Can be reassigned by TENANT_OWNER but never unset.
   */
  ownerId: string;
  ownerName: string; // display name snapshot — kept in sync on read
  ownerEmail: string; // for client-facing "contact your planner" link
  createdBy: string; // IAM user id — may differ from ownerId if reassigned
  approvedAt: string | null;
  approvedByClientName: string | null;
  snapshotId: string | null; // set on APPROVED
  createdAt: string;
  updatedAt: string;
}

export interface ProposalSummary extends Pick<
  Proposal,
  | "id"
  | "title"
  | "clientName"
  | "status"
  | "shareUrl"
  | "eventDate"
  | "approvedAt"
  | "ownerId"
  | "ownerName"
  | "createdAt"
  | "updatedAt"
> {
  venueCount: number;
  lastClientActivityAt: string | null;
}

export interface ProposalListResponse {
  items: ProposalSummary[];
  totalElements: number;
}
```

### `ProposalVenue`

A venue as it appears inside a specific proposal — a point-in-time snapshot of fields the planner
chose to expose, plus the client's response.

```typescript
export type ClientPreference = "SHORTLISTED" | "CONSIDERING" | "DECLINED" | null;

export interface ProposalVenue {
  id: string;
  proposalId: string;
  venueId: string; // FK → tenant venues
  order: number; // display order in the board
  // Planner-curated view (subset of full metadata)
  exposedFields: string[]; // list of metadata field keys visible to client
  plannerNote: string | null; // planner's public comment for this venue
  // Client response
  clientPreference: ClientPreference;
  clientNote: string | null;
  // Snapshot of venue data at time of last board publish
  venueSnapshot: ProposalVenueSnapshot;
  createdAt: string;
  updatedAt: string;
}

/** Immutable copy of venue data taken when proposal is published/approved. */
export interface ProposalVenueSnapshot {
  name: string;
  displayName: string | null;
  city: string | null;
  address: string | null;
  websiteUrl: string | null;
  primaryPhotoUrl: string | null;
  metadata: VenueMetadata; // canonical shape from ui-venue-management addon
  snapshotAt: string;
}
```

### `ProposalEvent` — immutable history

Append-only log. Nothing is ever deleted or updated in this table.

```typescript
export type ProposalEventType =
  | "PROPOSAL_CREATED"
  | "PROPOSAL_SHARED"
  | "PROPOSAL_VENUE_ADDED"
  | "PROPOSAL_VENUE_REMOVED"
  | "PROPOSAL_VENUE_REORDERED"
  | "PROPOSAL_FIELD_VISIBILITY_CHANGED"
  | "CLIENT_OPENED"
  | "CLIENT_PREFERENCE_SET" // SHORTLISTED / CONSIDERING / DECLINED
  | "CLIENT_NOTE_ADDED"
  | "CLIENT_NOTE_UPDATED"
  | "PLANNER_NOTE_ADDED"
  | "PLANNER_NOTE_UPDATED"
  | "PLANNER_RESPONSE_ADDED" // planner replies to client question
  | "AI_RESPONSE_SUGGESTED" // AI drafted a response from catalog data
  | "PROPOSAL_APPROVED" // client clicked Approve
  | "SNAPSHOT_CREATED"; // immutable snapshot locked

export interface ProposalEvent {
  id: string;
  proposalId: string;
  eventType: ProposalEventType;
  actorType: "PLANNER" | "CLIENT" | "SYSTEM" | "AI";
  actorId: string | null; // IAM user id for PLANNER; null for CLIENT/SYSTEM
  actorName: string | null; // display name at time of event
  payload: Record<string, unknown>; // event-specific details
  occurredAt: string;
}

> `ProposalEventType` and `PROPOSAL_EVENT_TYPES` are defined in `@venuemi/ui-types`.
> Re-exported from `src/addons/deal-workspace/index.ts`.
```

### `ProposalSnapshot`

Locked on `APPROVED`. Never mutated.

```typescript
export interface ProposalSnapshot {
  id: string;
  proposalId: string;
  venues: ProposalVenueSnapshot[];
  clientName: string;
  approvedAt: string;
  /** Owner at the time of approval — immutable part of the record. */
  ownerName: string;
  ownerEmail: string;
  tenantName: string;
  /** All ProposalEvents up to and including PROPOSAL_APPROVED */
  eventLog: ProposalEvent[];
  createdAt: string;
}
```

### Colored Labels (dynamic select)

Planner-defined labels scoped to a proposal — not global tags. Used for quick visual communication
on the client board.

```typescript
export interface ProposalLabel {
  id: string;
  proposalId: string;
  key: string; // e.g. 'recommended', 'needs_budget_check'
  displayText: string; // e.g. 'Recommended'
  colorHex: string; // e.g. '#22C55E'
}

export interface ProposalVenueLabel {
  proposalVenueId: string;
  labelId: string;
}
```

---

## 3. Confidence Score

Each `ProposalVenue` carries a computed `dataConfidence` — a UI-level indicator of how complete
and reliable the venue data is. Not stored; derived client-side from `venueSnapshot.metadata` and
`metadataSources`.

```typescript
export type DataConfidenceLevel = "HIGH" | "MEDIUM" | "LOW" | "INSUFFICIENT";

export interface DataConfidence {
  level: DataConfidenceLevel;
  score: number; // 0.0–1.0 weighted average across key fields
  missingKeyFields: string[]; // field keys that are null but important for this proposal
  lowConfidenceFields: string[];
}
```

> `CONFIDENCE_THRESHOLDS` (`{ HIGH: 0.8, MEDIUM: 0.55, LOW: 0.3 }`) and
> `computeDataConfidence()` are defined in `@venuemi/ui-types` and re-exported from
> `src/addons/deal-workspace/index.ts`.

Displayed as a small indicator on each venue card — both in planner view (with detail on hover)
and in client view (simplified: just HIGH/MEDIUM, never LOW shown to client).

---

## 4. App Placement

### `foundation-ui-app` (tenant — planner side)

```
src/
├── addons/
│   └── deal-workspace/            ← isolated addon, parallel to venue-management
│       ├── types.ts               ← all types from §2–3 above
│       ├── constants.ts           ← PROPOSAL_STATUSES, PROPOSAL_EVENT_TYPES, etc.
│       ├── confidence.ts          ← computeDataConfidence(), CONFIDENCE_THRESHOLDS
│       └── index.ts               ← public barrel
├── entities/
│   └── proposal/
│       ├── index.ts
│       └── types.ts               ← re-exports from addon + list response types
├── features/
│   ├── create-proposal/           ← modal: title, client name/email, select venues
│   ├── edit-proposal-venues/      ← add/remove/reorder venues, set exposedFields
│   ├── share-proposal/            ← generate/copy link, set branding options
│   └── proposal-history/          ← ProposalEvent timeline panel
├── widgets/
│   ├── proposal-list/             ← /proposals list with status filter
│   ├── proposal-board/            ← planner view of the assembled board
│   │   ├── proposal-board.tsx
│   │   ├── proposal-venue-card.tsx
│   │   └── proposal-labels-legend.tsx
│   └── proposal-collaboration/    ← comment threads, client activity feed
└── pages/
    ├── proposals/                 ← /proposals
    └── proposals_.$id/            ← /proposals/:id  (planner view)
```

### Public client route (no auth)

```
src/pages/
└── share_.$token/                 ← /share/:token  (client view, no auth guard)
    └── index.tsx                  ← ClientBoardPage
```

---

## 5. Component Structure

### Planner side

```
ProposalBoardPage  (/proposals/:id)
│
├── ProposalBoardHeader
│     title, clientName, status badge, share button, event date
│     confidence score summary across all venues
│
├── ProposalVenueGrid              ← ordered venue cards
│   └── ProposalVenueCard
│         primaryPhoto, name, city
│         colored labels (planner-defined)
│         client preference indicator (SHORTLISTED / CONSIDERING / DECLINED)
│         dataConfidence chip
│         plannerNote (editable inline)
│         "View details" → side panel with exposed fields from VenueMetadata
│
├── ProposalCollaborationPanel     ← right sidebar (collapsible)
│   ├── ClientActivityFeed         ← real-time: opened, preference set, note added
│   └── CommentThreads             ← per-venue Q&A between planner and client
│
└── ProposalHistoryPanel           ← bottom drawer (collapsed by default)
      ProposalEvent timeline — append-only, no edit/delete controls
      Each event: icon, actorType badge, actorName, occurredAt, payload summary
```

### Client side

```
ClientBoardPage  (/share/:token)
│
├── ClientBoardHeader
│     agency branding (logo, name — if brandingEnabled)
│     proposal title, planner name
│
├── ClientVenueGrid
│   └── ClientVenueCard
│         photo gallery, name, city
│         colored labels + legend
│         key metadata (only exposedFields)
│         preference selector  ← dynamic select from proposal context
│           options: SHORTLISTED / CONSIDERING / DECLINED
│         "Ask a question" inline → creates ClientNote + notifies planner
│         "View details" → sheet/drawer with full exposed venue profile
│
├── ClientApproveBar               ← sticky bottom bar (visible once ≥1 venue SHORTLISTED)
│     "Approve [venue name]" → confirmation → PROPOSAL_APPROVED event + snapshot
│
└── ClientBoardFooter
      "Powered by VenueMi" (free/pro tier)   ← removed on white_label plan
```

---

## 6. Bilateral Control Model

```
Planner controls:
  - which venues appear (ProposalVenue.order)
  - which fields are visible (ProposalVenue.exposedFields)
  - colored labels and their meaning
  - plannerNote per venue
  - when to share (DRAFT → SHARED transition)
  - whether to show branding

Client controls:
  - their preference per venue (ClientPreference)
  - their notes / questions
  - the final Approve action

Neither side can:
  - edit or delete past ProposalEvents
  - change the other side's data
  - modify the snapshot after APPROVED
```

The `exposedFields` array is the privacy boundary — metadata fields not in this list are never
sent to the client board API response. The server enforces this, not just the UI.

---

## 7. Immutable History UX

The `ProposalHistoryPanel` is a read-only timeline. Design rules:

- No edit or delete affordances — not even for `PLANNER` events
- Events rendered oldest-first (append visual metaphor)
- `CLIENT_OPENED` event is shown to planner, never back to client
- `AI_RESPONSE_SUGGESTED` shown with a distinct AI badge; planner must explicitly send before it
  becomes `PLANNER_RESPONSE_ADDED`
- `SNAPSHOT_CREATED` event anchors the end of the timeline with a visual "locked" marker

---

## 8. AI Assistance

The planner can ask questions about catalog data from within the proposal board:

```
"What does the catalog say about loading access at Grand Hotel Ballroom?"
→ pulls from venue_assets extracted_text + metadata
→ returns cited answer with source asset reference
→ planner can send as-is or edit → becomes PLANNER_RESPONSE_ADDED event
```

Not a free-form chatbot. Scoped to the proposal's venues + the planner's tenant catalog only.
No data crosses tenant boundaries.

```typescript
export interface AIAssistRequest {
  proposalId: string;
  proposalVenueId: string;
  question: string;
}

export interface AIAssistResponse {
  suggestedText: string;
  sourceCitations: Array<{
    assetId: string;
    assetName: string;
    excerpt: string;
  }>;
  confidence: number;
}
```

---

## 9. Themes / Skins

Same CSS token pattern as `ui-venue-management.md` §7. The client board uses
`data-skin="client-board"` set by the public share layout.

```css
/* Client board — clean, minimal, agency-branded */
[data-skin="client-board"] {
  --vmi-form-tab-active: var(--brand-primary, var(--mantine-color-blue-6));
  --vmi-section-bg: #ffffff;
  --vmi-badge-high: transparent; /* no confidence shown to client */
  --vmi-badge-medium: transparent;
  --vmi-badge-low: transparent;
  --vmi-badge-manual: transparent;
  --vmi-badge-catalog: transparent;
}

/* Planner board — information-dense, full provenance visible */
[data-skin="planner-board"] {
  --vmi-form-tab-active: var(--mantine-color-blue-6);
  --vmi-section-bg: var(--mantine-color-gray-0);
}
```

White-label (Enterprise plan): `--brand-primary` is set from tenant branding config.
Logo and custom domain replace VenueMi defaults. `"Powered by VenueMi"` footer is suppressed.

---

## 10. API Integration

```typescript
// src/shared/api/proposal.ts  (tenant app — planner side)

// Proposals
GET    /api/v1/venues/proposals                      → ProposalListResponse
POST   /api/v1/venues/proposals                      → Proposal
GET    /api/v1/venues/proposals/{id}                 → Proposal + ProposalVenue[]
PATCH  /api/v1/venues/proposals/{id}                 → update title/clientName/eventDate
POST   /api/v1/venues/proposals/{id}/share           → generate/refresh shareToken → shareUrl
POST   /api/v1/venues/proposals/{id}/archive

// Proposal venues
POST   /api/v1/venues/proposals/{id}/venues          → add venue (venueId, order)
DELETE /api/v1/venues/proposals/{id}/venues/{pvId}
PATCH  /api/v1/venues/proposals/{id}/venues/{pvId}   → update order, exposedFields, plannerNote
POST   /api/v1/venues/proposals/{id}/venues/{pvId}/labels/{labelId}
DELETE /api/v1/venues/proposals/{id}/venues/{pvId}/labels/{labelId}

// Labels
GET    /api/v1/venues/proposals/{id}/labels
POST   /api/v1/venues/proposals/{id}/labels
PATCH  /api/v1/venues/proposals/{id}/labels/{labelId}
DELETE /api/v1/venues/proposals/{id}/labels/{labelId}

// History
GET    /api/v1/venues/proposals/{id}/events          → ProposalEvent[]

// Snapshot
GET    /api/v1/venues/proposals/{id}/snapshot        → ProposalSnapshot (only if APPROVED)

// AI assist
POST   /api/v1/venues/proposals/{id}/ai-assist       → AIAssistResponse

// src/shared/api/client-board.ts  (public — no auth)

GET    /api/v1/venues/share/{token}                              → ClientBoardResponse (venues with exposedFields only)
PATCH  /api/v1/venues/share/{token}/venues/{pvId}/preference    → set ClientPreference
POST   /api/v1/venues/share/{token}/venues/{pvId}/notes         → ClientNote
POST   /api/v1/venues/share/{token}/approve                     → triggers PROPOSAL_APPROVED + snapshot
```

---

## 11. Routes

| Path                     | App    | Auth     | Component             |
| ------------------------ | ------ | -------- | --------------------- |
| `/venues/proposals`      | tenant | MEMBER+  | `ProposalListPage`    |
| `/venues/proposals/new`  | tenant | MEMBER+  | `CreateProposalModal` |
| `/venues/proposals/{id}` | tenant | MEMBER+  | `ProposalBoardPage`   |
| `/venues/share/{token}`  | tenant | **none** | `ClientBoardPage`     |

`/venues/share/{token}` is a public route — no `AuthGuard`, no `X-Tenant-ID` header. The server resolves
the tenant from the token.

---

## 12. Prototype Strategy

> **Prototype scope note.**
> The prototype is a **fully interactive implementation** — not a mockup or a clickable wireframe.
> Every button, form, field, transition, and screen state must work. Navigation between pages is
> real (TanStack Router). Proposal creation flows through the full modal. Venues are added,
> reordered, and removed. Labels are created and assigned. `exposedFields` toggling is live.
> The client board opens at `/share/:token` with real routing — no auth, full interaction.
> Client preference selectors work. The approve flow goes through the confirmation step and
> renders the locked snapshot state. The history timeline appends events as actions happen.
> The only thing emulated is the API layer — all network calls are intercepted by MSW handlers
> returning fixture data.
>
> This means every screen state must be represented in fixtures: DRAFT board, SHARED board with
> no client activity yet, IN_REVIEW board with client comments, APPROVED board with locked
> snapshot, client board before preference set, client board after shortlisting, the approve
> confirmation, and the post-approve read-only state. Missing states are UX bugs — catch them
> here, not after backend integration.

Shared code (`@venuemi/ui-types`) is referenced via `file:` protocol pointing directly
to TypeScript source — no build step, no publish. See
[ui-shared-packages.md](ui-shared-packages.md) §7 for the exact setup.

Same approach as `ui-venue-management.md` §11. Prototype in `foundation-ui-blank` first.

```
foundation-ui-blank/src/
├── addons/
│   └── deal-workspace/          ← the addon — develop here first, copy verbatim later
│       ├── index.ts             ← PUBLIC BARREL — only import path for the rest of the app
│       ├── types.ts             ← re-exports Proposal, ProposalVenue, ProposalEvent,
│       │                           ProposalSnapshot, ProposalLabel, ProposalVenueLabel,
│       │                           ClientPreference, ProposalStatus, ProposalEventType,
│       │                           DataConfidence, DataConfidenceLevel,
│       │                           AIAssistRequest, AIAssistResponse
│       │                           from @venuemi/ui-types (no redefinition)
│       ├── constants.ts         ← re-exports PROPOSAL_STATUSES, PROPOSAL_EVENT_TYPES,
│       │                           CLIENT_PREFERENCES, CONFIDENCE_THRESHOLDS,
│       │                           isProposalStatus, isClientPreference
│       │                           from @venuemi/ui-types (no redefinition)
│       └── confidence.ts        ← re-exports computeDataConfidence from @venuemi/ui-types
│                                   (no redefinition; kept as a named re-export for
│                                   discoverability within the addon)
├── entities/
│   └── proposal/
│       ├── index.ts
│       └── types.ts             ← ProposalListResponse; re-exports from @/addons/deal-workspace
├── features/
│   ├── create-proposal/
│   ├── edit-proposal-venues/
│   └── share-proposal/
├── widgets/
│   ├── proposal-board/
│   └── proposal-collaboration/
├── pages/
│   ├── venues/
│   │   ├── proposals/           ← /venues/proposals  (planner list)
│   │   └── proposals_.$id/      ← /venues/proposals/:id  (planner board)
│   └── venues/
│       └── share_.$token/       ← /venues/share/:token  (client board — public, no auth guard)
│           └── index.tsx        ← ClientBoardPage
│                                   NOTE: this page CONSUMES the addon but is NOT part of it.
│                                   ClientBoardPage lives in pages/, not in addons/deal-workspace/.
└── shared/
    ├── api/
    │   ├── proposal.ts          ← proposalApi (planner-side, requires auth header)
    │   └── client-board.ts      ← clientBoardApi (public, no auth header, no X-Tenant-ID)
    └── mocks/
        └── handlers/
            └── proposal.ts
```

### MSW fixture scope for prototype

```
GET  /api/v1/venues/proposals             → 5–8 proposals across all statuses
GET  /api/v1/venues/proposals/{id}         → full proposal with 3–5 venues, labels, events
POST /api/v1/venues/proposals             → create (echo back with id)
PATCH /api/v1/venues/proposals/{id}/venues/{pvId}  → update preference/note
GET  /api/v1/venues/share/{token}          → client board response (2 fixture tokens)
PATCH /api/v1/venues/share/{token}/venues/{pvId}/preference
POST /api/v1/venues/share/{token}/approve  → returns snapshot
GET  /api/v1/venues/proposals/{id}/events  → 8–12 events across actorTypes
```

Fixture data must include:

- At least one `APPROVED` proposal with a snapshot
- At least one `IN_REVIEW` proposal with client comments
- One proposal with `brandingEnabled: true` for white-label skin testing
- `CLIENT_OPENED` + `CLIENT_PREFERENCE_SET` events to exercise the activity feed

---

## 13. File Contracts

The addon file layout is **identical** across prototype, planner tenant app, and any future consumer. No rewrites on promotion — only wiring (auth guards, feature gates, route registration).

```
src/addons/deal-workspace/
  index.ts         PUBLIC BARREL — every import from outside the addon goes here.
                   Never import from addons/deal-workspace/types or any internal path.

  types.ts         Re-exports from @venuemi/ui-types:
                   Proposal, ProposalSummary, ProposalListResponse,
                   ProposalVenue, ProposalVenueSnapshot,
                   ProposalEvent, ProposalSnapshot,
                   ProposalLabel, ProposalVenueLabel,
                   ClientPreference, ProposalStatus, ProposalEventType,
                   DataConfidence, DataConfidenceLevel,
                   AIAssistRequest, AIAssistResponse.
                   No redefinitions — these types live in @venuemi/ui-types.

  constants.ts     Re-exports from @venuemi/ui-types:
                   PROPOSAL_STATUSES, PROPOSAL_EVENT_TYPES,
                   CLIENT_PREFERENCES, CONFIDENCE_THRESHOLDS,
                   isProposalStatus, isClientPreference, isProposalEventType.
                   No redefinitions.

  confidence.ts    Re-exports computeDataConfidence from @venuemi/ui-types.
                   Kept as a named re-export for addon discoverability.

src/entities/proposal/
  types.ts         ProposalListResponse re-export; no other additions.
                   Re-exports domain types from @/addons/deal-workspace.
  index.ts         Re-exports types.ts.

src/shared/api/
  proposal.ts      proposalApi — planner-side calls (auth header required).
  client-board.ts  clientBoardApi — public share calls (no auth header, no X-Tenant-ID).

src/pages/
  venues/proposals/        ← planner proposal list (auth guard)
  venues/proposals_.$id/   ← planner board (auth guard)
  venues/share_.$token/    ← client board (NO auth guard — public route)
                             ClientBoardPage is a page that consumes the addon.
                             It is NOT part of addons/deal-workspace/.
```

Architecture tests assert:
- No file outside `src/addons/deal-workspace/` imports from `@venuemi/ui-types` directly for deal types.
- No file redefines `ProposalStatus`, `PROPOSAL_STATUSES`, or `CONFIDENCE_THRESHOLDS`.
- `ClientBoardPage` lives in `src/pages/`, not in `src/addons/`.

---

**Docs:** [Architecture Overview](architecture-overview.md) · [UI: Venue Management](ui-venue-management.md) · [Data Model](data-model.md) · [API](api.md) · [Events](events.md) · [Product](../business/digital-sales-room-for-events/product.md)
