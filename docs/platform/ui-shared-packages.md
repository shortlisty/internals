# Shortlisty — UI Shared npm Packages

> **Audience:** Engineers, architects.
> **Purpose:** Defines shared UI code published to npmjs as private scoped packages,
> consumed by `foundation-ui-blank`, `foundation-ui-app`, and `foundation-ui-platform-admin`
> as independent microservice frontends.

---

## Related Documents

- [ui-venue-management.md](ui-venue-management.md) — venue CRUD, form spec, AnyVenue
- [ui-deal-workspace.md](ui-deal-workspace.md) — proposal, client board, approval snapshot
- [architecture-overview.md](architecture-overview.md) — FSD layers, addon pattern

---

## 1. Context

Three apps — separate repos, separate deploys, no monorepo. Shared UI code is published to
npmjs under the `@shortlisty` scope and consumed as regular versioned dependencies.

The split is **types first, components second**. The shared package contains zero React
dependencies — it is pure TypeScript: interfaces, enums, constants, type guards, and pure
utility functions. Each app builds its own addon on top of the shared types. Components,
renderers, and form logic live in the app-level addons and are never shared directly.

```
@shortlisty/ui-types          ← pure TS: interfaces, constants, type guards, pure utils
                                    no React, no Mantine, no build-time deps
        │
        ├── foundation-ui-blank/src/addons/venue-management/    ← components, registry
        ├── foundation-ui-app/src/addons/venue-management/      ← components, registry
        ├── foundation-ui-platform-admin/src/addons/venue-management/
        │
        └── foundation-ui-app/src/addons/deal-workspace/        ← proposal components
```

---

## 2. Package: `@shortlisty/ui-types`

Single shared package. No React, no UI framework dependency. Can be imported anywhere —
frontend apps, potential SSR layer, test utilities.

```
packages/ui-types/
└── src/
    ├── venue/
    │   ├── venue.types.ts      AnyVenue, VenueContext, VenueMetadata, FieldProvenance,
    │   │                       ProfileStage, VenueStatus, VenueSource, CateringPolicy,
    │   │                       AnnotationType, VenueAnnotation
    │   ├── venue.constants.ts  PROFILE_STAGES, VENUE_STATUSES, VENUE_SOURCES,
    │   │                       VENUE_CONTEXTS, ANNOTATION_TYPES, CATERING_POLICIES
    │   ├── venue.guards.ts     isProfileStage, isVenueStatus, isVenueContext, …
    │   ├── form-spec.types.ts   FieldDefinition, TabDefinition, SectionDefinition,
    │   │                       VenueFormSpec, FieldType, FieldAction, EnumOption
    │   ├── asset.types.ts      VenueAsset, VenueAssetSummary, AssetTableData,
    │   │                       AssetType, PhotoCategory, ExtractionStatus
    │   └── asset.constants.ts  ASSET_TYPES, PHOTO_CATEGORIES, EXTRACTION_STATUSES,
    │                           ASSET_TYPE_LABELS, PHOTO_CATEGORY_LABELS,
    │                           PHOTO_ASSET_TYPES, TABULAR_ASSET_TYPES,
    │                           assetAcceptsPhotoCategory, assetProducesTableData
    ├── deal/
    │   ├── deal.types.ts       Proposal (with mandatory ownerId, ownerName, ownerEmail),
    │   │                       ProposalSummary, ProposalVenue,
    │   │                       ProposalVenueSnapshot, ProposalEvent, ProposalSnapshot,
    │   │                       ProposalLabel, ProposalVenueLabel,
    │   │                       ClientPreference, ProposalStatus, ProposalEventType,
    │   │                       DataConfidence, DataConfidenceLevel,
    │   │                       AIAssistRequest, AIAssistResponse
    │   ├── deal.constants.ts   PROPOSAL_STATUSES, PROPOSAL_EVENT_TYPES,
    │   │                       CLIENT_PREFERENCES, CONFIDENCE_THRESHOLDS
    │   ├── deal.guards.ts      isProposalStatus, isClientPreference, …
    │   └── confidence.ts       computeDataConfidence() — pure function, no React dep
    └── index.ts                public barrel — re-exports everything above
```

`package.json`:

```json
{
  "name": "@shortlisty/ui-types",
  "version": "0.1.0",
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "devDependencies": {
    "typescript": "^6"
  }
}
```

No `peerDependencies`. Zero runtime dependencies. TypeScript is dev-only.

### Asset types — `venue/asset.types.ts` and `venue/asset.constants.ts`

```typescript
// venue/asset.types.ts

export type AssetType =
  | "PHOTO"
  | "FLOOR_PLAN"
  | "PDF_DECK"
  | "SPEC_SHEET"
  | "MENU"
  | "PRICE_LIST"
  | "DATA_TABLE"
  | "VIDEO"
  | "CAD_FILE"
  | "MISC";

export type PhotoCategory =
  "EXTERIOR" | "INTERIOR" | "SETUP" | "DETAIL" | "CATERING" | "OUTDOOR" | "TEAM" | "OTHER";

export type ExtractionStatus = "PENDING" | "IN_PROGRESS" | "COMPLETED" | "FAILED";

/** Parsed tabular content — populated for DATA_TABLE and PRICE_LIST (csv/xlsx). */
export interface AssetTableData {
  headers: string[];
  rows: string[][];
  sourceSheet: string | null; // original worksheet name for xlsx
  rowCount: number;
  parsedAt: string;
}

/** Full asset record — used in venue profile / asset management view. */
export interface VenueAsset {
  id: string;
  venueId: string;
  assetType: AssetType;
  photoCategory: PhotoCategory | null; // set only when assetType === 'PHOTO'
  displayOrder: number;
  label: string | null;
  fileName: string;
  contentType: string;
  sizeBytes: number;
  cdnUrl: string | null;
  thumbnailCdnUrl: string | null; // set for PHOTO and VIDEO after processing
  tableData: AssetTableData | null; // set for DATA_TABLE and PRICE_LIST (csv/xlsx)
  extractionStatus: ExtractionStatus;
  uploadedBy: string;
  uploadedAt: string;
  updatedAt: string;
}

/**
 * Flat projection for gallery and list rendering.
 * Omits tableData and raw extraction fields — loaded on demand.
 */
export interface VenueAssetSummary {
  id: string;
  venueId: string;
  assetType: AssetType;
  photoCategory: PhotoCategory | null;
  displayOrder: number;
  label: string | null;
  fileName: string;
  sizeBytes: number;
  cdnUrl: string | null;
  thumbnailCdnUrl: string | null;
  extractionStatus: ExtractionStatus;
  uploadedAt: string;
}
```

```typescript
// venue/asset.constants.ts

import type { AssetType, PhotoCategory, ExtractionStatus } from "./asset.types";

export const ASSET_TYPES = [
  "PHOTO",
  "FLOOR_PLAN",
  "PDF_DECK",
  "SPEC_SHEET",
  "MENU",
  "PRICE_LIST",
  "DATA_TABLE",
  "VIDEO",
  "CAD_FILE",
  "MISC",
] as const;

export const PHOTO_CATEGORIES = [
  "EXTERIOR",
  "INTERIOR",
  "SETUP",
  "DETAIL",
  "CATERING",
  "OUTDOOR",
  "TEAM",
  "OTHER",
] as const;

export const EXTRACTION_STATUSES = ["PENDING", "IN_PROGRESS", "COMPLETED", "FAILED"] as const;

export const ASSET_TYPE_LABELS: Record<AssetType, string> = {
  PHOTO: "Photo",
  FLOOR_PLAN: "Floor Plan",
  PDF_DECK: "Venue Deck",
  SPEC_SHEET: "Spec Sheet",
  MENU: "Menu",
  PRICE_LIST: "Price List",
  DATA_TABLE: "Data Table",
  VIDEO: "Video",
  CAD_FILE: "CAD File",
  MISC: "Other",
};

export const PHOTO_CATEGORY_LABELS: Record<PhotoCategory, string> = {
  EXTERIOR: "Exterior",
  INTERIOR: "Interior",
  SETUP: "Setup",
  DETAIL: "Detail",
  CATERING: "Catering",
  OUTDOOR: "Outdoor",
  TEAM: "Team",
  OTHER: "Other",
};

/** Asset types that carry photo_category. */
export const PHOTO_ASSET_TYPES = ["PHOTO"] as const satisfies AssetType[];

/** Asset types that produce table_data on the backend. */
export const TABULAR_ASSET_TYPES = ["DATA_TABLE", "PRICE_LIST"] as const satisfies AssetType[];

// ── Type guards ────────────────────────────────────────────────────────────

export const isAssetType = (v: unknown): v is AssetType =>
  (ASSET_TYPES as readonly string[]).includes(v as string);
export const isPhotoCategory = (v: unknown): v is PhotoCategory =>
  (PHOTO_CATEGORIES as readonly string[]).includes(v as string);
export const isExtractionStatus = (v: unknown): v is ExtractionStatus =>
  (EXTRACTION_STATUSES as readonly string[]).includes(v as string);

// ── Pure helpers (no React dep) ────────────────────────────────────────────

export const assetAcceptsPhotoCategory = (type: AssetType): boolean =>
  (PHOTO_ASSET_TYPES as readonly string[]).includes(type);

export const assetProducesTableData = (type: AssetType): boolean =>
  (TABULAR_ASSET_TYPES as readonly string[]).includes(type);

export const assetHasThumbnail = (type: AssetType): boolean => type === "PHOTO" || type === "VIDEO";
```

---

## 3. App-level addons (not shared)

Each app's addon imports from `@shortlisty/ui-types` and builds React/Mantine components on top.
The addon is the app's private implementation — it is never published.

```
src/addons/venue-management/          ← per-app, not shared
  spec/
    venue-form-spec.ts           VENUE_FORM_SPEC — imports FieldDefinition
                                      from @shortlisty/ui-types
    form-spec.utils.ts                 resolveFieldValue, setFieldValue,
                                      missingFieldsForStage, groupByTabAndSection —
                                      imports VenueMetadata, FieldDefinition
                                      from @shortlisty/ui-types
  components/
    VenueFormField.tsx                imports FieldDefinition, FieldType
    VenueFormSection.tsx
    VenueFormTabs.tsx
    SourceBadge.tsx                   imports FieldProvenance
    ProfileStagePill.tsx              imports ProfileStage, PROFILE_STAGES
  index.ts
```

```
src/addons/deal-workspace/            ← foundation-ui-app only, not shared
  components/
    ProposalBoardCard.tsx             imports ProposalVenue, ProposalLabel
    ProposalHistoryTimeline.tsx       imports ProposalEvent, PROPOSAL_EVENT_TYPES
    ClientPreferenceSelector.tsx      imports ClientPreference, CLIENT_PREFERENCES
    DataConfidenceBadge.tsx           imports DataConfidence, CONFIDENCE_THRESHOLDS
  index.ts
```

**Rule:** addons import types and constants from `@shortlisty/ui-types`. They never redefine
interfaces or constants that exist in the shared package.

---

## 4. Versioning

`@shortlisty/ui-types` follows semver:

| Change                                                  | Bump  |
| ------------------------------------------------------- | ----- |
| New optional field in `VenueMetadata`                   | patch |
| New type or constant exported                           | minor |
| New field in `FieldDefinition`                          | minor |
| Breaking type change, field removed, enum value renamed | major |

Adding a canonical metadata field is a **patch** — new optional key, all existing
consumers unaffected. Renaming `SEEDED` → `INITIAL` is **major** — all addons must update.

---

## 5. What stays app-local

| Code                                                      | Why app-local                                           |
| --------------------------------------------------------- | ------------------------------------------------------- |
| `VENUE_FORM_SPEC` constant                                | Field labels, order, UI hints are app-specific concerns |
| `VenueFormField` and all React components                 | React + Mantine dep, per-app customisation              |
| `entities/venue/types.ts` — `VenueSummary`, `VenueDetail` | DTO projections differ per app                          |
| `entities/master-venue/types.ts`                          | Only in `foundation-ui-platform-admin`                  |
| `entities/proposal/types.ts`                              | Only in `foundation-ui-app`                             |
| `shared/api/*.ts`                                         | Different base URLs, auth headers per app               |
| MSW handlers and fixtures                                 | Only in `foundation-ui-blank`                           |
| Auth guards, `FeatureGate`, `useEntitlements`             | Only in production apps                                 |
| CSS theme token overrides (`--vmi-*`)                     | Per-app / per-skin                                      |

---

## 6. Drift prevention rules

1. **Interfaces and constants live in `@shortlisty/ui-types` only.** No addon or entity file
   redefines `VenueMetadata`, `ProfileStage`, `ProposalStatus`, or any other type that exists
   in the shared package. Architecture tests assert this.

2. **String literal types derive from `as const` arrays** defined in `@shortlisty/ui-types`.
   Addons import the array and derive the type — never write a manual union.

3. **`computeDataConfidence()` lives in `@shortlisty/ui-types`** — pure function, no React,
   same logic in all apps, testable in isolation without a DOM.

4. **`Proposal.ownerId` is non-nullable** — architecture test asserts no code path sets
   `ownerId` to `null` or `undefined`. Owner can be reassigned but never unset.
   `ownerName` and `ownerEmail` are snapshot fields kept in sync on read from IAM.

5. **Architecture test in each app** asserts:
   - No file in `src/` declares `interface VenueMetadata`
   - No file in `src/` declares `const PROFILE_STAGES`
   - `@shortlisty/ui-types` is listed in `package.json` dependencies

---

## 7. Prototype phase — `file:` protocol

During prototyping `packages/ui-types/` lives alongside the apps. Apps reference it via
pnpm's `file:` protocol — resolves directly to TypeScript source, no build step needed.

### Directory layout

```
IQKV/ms/
├── packages/
│   └── ui-types/              ← pure TS source, no build config yet
│       ├── package.json
│       └── src/
│           ├── venue/
│           ├── deal/
│           └── index.ts
├── foundation-ui-blank/
├── foundation-ui-app/
└── foundation-ui-platform-admin/
```

### `package.json` in package (prototype)

```json
{
  "name": "@shortlisty/ui-types",
  "version": "0.0.0",
  "private": true,
  "exports": { ".": "./src/index.ts" }
}
```

`"exports"` points to `src/index.ts` — Vite resolves TypeScript source directly.

### Consumer `package.json`

```json
{
  "dependencies": {
    "@shortlisty/ui-types": "file:../../packages/ui-types"
  }
}
```

```bash
pnpm install   # symlinks packages/ui-types into node_modules/@shortlisty/ui-types
```

Changes to `packages/ui-types/src/` are immediately visible — pnpm symlinks the directory,
not a copy.

### `tsconfig.json` path alias (each consuming app)

```json
{
  "compilerOptions": {
    "paths": {
      "@shortlisty/ui-types": ["../../packages/ui-types/src/index.ts"]
    }
  }
}
```

---

## 8. Promotion to npmjs

Triggered when prototypes are stable and the shared API is no longer changing daily.

**Step 1 — build config** (no source changes):

```bash
pnpm add -D tsup   # in packages/ui-types/
```

```ts
// tsup.config.ts
import { defineConfig } from "tsup";
export default defineConfig({
  entry: ["src/index.ts"],
  format: ["esm"],
  dts: true,
  clean: true,
});
```

Update `package.json` exports:

```json
{
  "exports": { ".": { "import": "./dist/index.js", "types": "./dist/index.d.ts" } },
  "scripts": { "build": "tsup" }
}
```

**Step 2 — publish**:

```bash
pnpm build
npm publish --access public   # under @shortlisty org on npmjs
```

**Step 3 — update all consumers** (all three apps, same commit):

```json
"@shortlisty/ui-types": "^0.1.0"
```

Remove `file:` entry and `paths` override from `tsconfig.json`.

**Step 4 — architecture tests**: add assertions from §6.

> Source files in `packages/ui-types/src/` do not change during promotion —
> only build config and `package.json` exports are updated around them.

---

**Docs:** [UI: Venue Management](ui-venue-management.md) · [UI: Deal Workspace](ui-deal-workspace.md) · [Architecture Overview](architecture-overview.md)
