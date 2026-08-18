# VenueMi — UI Shared Code Strategy

> **Audience:** Frontend engineers.
> **Purpose:** Defines how shared UI code is maintained across `foundation-ui-blank`,
> `foundation-ui-app`, and `foundation-ui-platform-admin` — three independent apps,
> each with its own deploy, no monorepo.

---

## Related Documents

- [ui-venue-management.md](ui-venue-management.md) — venue CRUD, field registry, AnyVenue
- [ui-deal-workspace.md](ui-deal-workspace.md) — proposal, client board, approval snapshot
- [architecture-overview.md](architecture-overview.md) — FSD layers, addon pattern

---

## 1. Context

Each app is an independent microservice frontend — separate repo, separate deploy, no shared
build pipeline. There is no monorepo or workspace package infrastructure.

Shared code is maintained by **convention and discipline**, not by a build system. The drift
prevention strategy is documentation + code review rules + architecture tests within each app.

---

## 2. What is shared (by copy)

Two addons are copied across apps. The copy is intentional and explicit — not accidental
duplication.

| Addon                          | Contents                                               | Copied to                                            |
| ------------------------------ | ------------------------------------------------------ | ---------------------------------------------------- |
| `src/addons/venue-management/` | `types.ts`, `constants.ts`, `registry/`, `components/` | `foundation-ui-app` → `foundation-ui-platform-admin` |
| `src/addons/deal-workspace/`   | `types.ts`, `constants.ts`, `confidence.ts`            | stays in `foundation-ui-app` only                    |

`foundation-ui-blank` is the **origin** — addons are authored there during prototyping, then
promoted to production apps. After promotion, `foundation-ui-app` becomes the canonical source.

---

## 3. Canonical source of truth

```
foundation-ui-blank      ← origin during prototype phase
      │
      │  promote (one-time, when UX approved)
      ▼
foundation-ui-app        ← canonical source after promotion
      │
      │  sync (on meaningful change)
      ▼
foundation-ui-platform-admin
```

**Rule:** changes to shared addon code are always made first in `foundation-ui-app`, then
manually applied to `foundation-ui-platform-admin`. Never the reverse.

`foundation-ui-blank` is kept in sync with `foundation-ui-app` during active prototype work
to avoid divergence before promotion.

---

## 4. What is shared

### `venue-management` addon — shared between `foundation-ui-app` and `foundation-ui-platform-admin`

```
src/addons/venue-management/
  types.ts        AnyVenue, VenueContext, VenueMetadata, FieldProvenance,
                  ProfileStage, VenueStatus, VenueSource, CateringPolicy,
                  AnnotationType, VenueAnnotation

  constants.ts    PROFILE_STAGES, VENUE_STATUSES, VENUE_SOURCES,
                  VENUE_CONTEXTS, ANNOTATION_TYPES, CATERING_POLICIES
                  + type guards (isProfileStage, isVenueContext, …)

  registry/
    field-registry.types.ts    FieldDefinition, TabDefinition, SectionDefinition,
                               VenueFormRegistry, FieldType, FieldAction
    venue-field-registry.ts    VENUE_FIELD_REGISTRY  ← single source of truth
    registry.utils.ts          resolveFieldValue, setFieldValue,
                               missingFieldsForStage, groupByTabAndSection

  components/
    VenueFormField.tsx          dispatcher + all field type renderers
    VenueFormSection.tsx        collapsible section wrapper
    VenueFormTabs.tsx           tab bar with completion dots
    SourceBadge.tsx             confidence chip + provenance popover
    ProfileStagePill.tsx        SEEDED/ENRICHED/CURATED/READY badge

  index.ts        public barrel
```

### `deal-workspace` addon — `foundation-ui-app` only

```
src/addons/deal-workspace/
  types.ts        Proposal, ProposalSummary, ProposalVenue, ProposalVenueSnapshot,
                  ProposalEvent, ProposalSnapshot, ProposalLabel, ProposalVenueLabel,
                  ClientPreference, ProposalStatus, ProposalEventType,
                  DataConfidence, DataConfidenceLevel,
                  AIAssistRequest, AIAssistResponse

  constants.ts    PROPOSAL_STATUSES, PROPOSAL_EVENT_TYPES,
                  CLIENT_PREFERENCES, CONFIDENCE_THRESHOLDS + type guards

  confidence.ts   computeDataConfidence() — pure function, no React dep

  index.ts        public barrel
```

---

## 5. What stays app-local (not copied)

| Code                                                      | Why app-local                              |
| --------------------------------------------------------- | ------------------------------------------ |
| `entities/venue/types.ts` — `VenueSummary`, `VenueDetail` | List/detail DTO projections differ per app |
| `entities/master-venue/types.ts`                          | Only in `foundation-ui-platform-admin`     |
| `entities/proposal/types.ts`                              | Only in `foundation-ui-app`                |
| `shared/api/venue.ts`, `proposal.ts`, `master-venue.ts`   | Different base URLs, auth headers per app  |
| Page components, widget layouts                           | App-specific routing and layout shells     |
| MSW handlers and fixtures                                 | Only in `foundation-ui-blank`              |
| Auth guards, `FeatureGate`, `useEntitlements`             | Only in production apps                    |
| CSS theme tokens (`--vmi-*` overrides)                    | Per-app / per-skin                         |

---

## 6. Drift prevention rules

Without a build system enforcing consistency, these rules are enforced at code review:

1. **`VenueMetadata` is defined once** — in `src/addons/venue-management/types.ts` inside
   `foundation-ui-app`. The copy in `foundation-ui-platform-admin` must be byte-for-byte
   identical. Any change to `VenueMetadata` requires updating both apps in the same PR/commit.

2. **`VENUE_FIELD_REGISTRY` is defined once** — in `foundation-ui-app`. The copy in
   `foundation-ui-platform-admin` is synced manually. A field added to the registry in one app
   must be added in the other in the same change.

3. **String literal types derive from `as const` arrays** — no inline string union types anywhere.
   `ProfileStage` is `typeof PROFILE_STAGES[number]`, not a manually written union.

4. **`VenueFormField` and shared components are synced** — bug fixes applied to one app are
   applied to the other. The diff between the two files should always be empty.

5. **`computeDataConfidence()` is a pure function** — no React dependency, no app-specific
   imports. Testable in isolation. Same file, same logic in all apps that use it.

6. **Architecture test in each app** asserts:
   - `src/addons/venue-management/types.ts` exports `VenueMetadata` (smoke check that the file exists and exports correctly)
   - No other file in `src/` defines an interface named `VenueMetadata`
   - `src/addons/venue-management/registry/venue-field-registry.ts` exports `VENUE_FIELD_REGISTRY`

---

## 7. Sync checklist

When making a change to shared addon code in `foundation-ui-app`:

- [ ] Change made and tested in `foundation-ui-app`
- [ ] Same change applied to `foundation-ui-platform-admin` (if `venue-management` addon)
- [ ] Same change applied to `foundation-ui-blank` (if actively used for prototyping)
- [ ] Architecture tests pass in all affected apps
- [ ] Commit message references all affected apps explicitly

Template commit subject:

```
feat(venue-management): add outdoor_space fields to registry [app, admin, blank]
```

---

## 8. Future option — npm package

If the copy-and-sync discipline proves hard to maintain at scale (>3 apps, >2 engineers), the
natural upgrade is to extract the shared addons into a private npm package
(`@venuemi/ui-venue-core`, `@venuemi/ui-deal-core`) published to a private registry
(GitHub Packages, Verdaccio, or similar).

This is a non-breaking migration: the package contents are identical to the current addon files,
consumers just change the import path from `@/addons/venue-management` to
`@venuemi/ui-venue-core`. The upgrade is deferred until the copy-sync approach shows actual
friction — not preemptively.

---

**Docs:** [UI: Venue Management](ui-venue-management.md) · [UI: Deal Workspace](ui-deal-workspace.md) · [Architecture Overview](architecture-overview.md)
