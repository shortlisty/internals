# VenueMi — UI: Venue Management

> **Audience:** Frontend engineers, designers.
> **Purpose:** Component structure, form spec pattern, shared entity abstraction, form layout
> contract, and file placement for venue CRUD in `foundation-ui-app` (tenant) and
> `foundation-ui-platform-admin` (admin).

---

## Related Documents

- [data-model.md](data-model.md) — `venues`, `venue_annotations`, `master_venue`, canonical field set, `profile_stage`
- [api.md](api.md) — REST endpoints consumed by these features
- [architecture-overview.md](architecture-overview.md) — UI integration section, FSD layer rules
- [ui-shared-packages.md](ui-shared-packages.md) — shared workspace packages, drift prevention rules

---

## 1. Guiding Principles

- **DRY / KISS first.** One form spec, one set of form components, one entity abstraction.
  Both `Venue` (tenant) and `MasterVenue` (platform) share the same canonical metadata schema and
  therefore the same form. Differences are handled by a single `context` discriminator — not by
  duplicating components.
- **MasterVenue is the more mature entity.** It has richer data (multi-source scraped, admin-curated,
  high-confidence). Tenant `Venue` is catching up as the product grows. The UI layer reflects this:
  features present on `MasterVenue` first, promoted to tenant `Venue` as backend adds support.
- **Schema-driven form.** The form is built from a static form spec (TS constant). Adding a
  canonical field requires one registry entry — no new component.
- **Fast list rendering.** List views consume flat summary DTOs resolved server-side. No JSONB
  traversal on the client for list display.
- **Progressive disclosure.** `SEEDED` → show 4–6 fields. Full form is always reachable but never
  forced.

---

## 2. Shared Entity Abstraction

`Venue` and `MasterVenue` share the same canonical `metadata` JSONB schema. The UI treats them as
variants of one abstract entity `AnyVenue`, differentiated by a `context` discriminator.

```typescript
// src/addons/venue-management/types.ts

import type { VenueMetadata, FieldProvenance } from "@/entities/venue";

/**
 * Discriminator — controls which UI affordances are rendered.
 *
 * 'tenant'         — personal record owned by the tenant. Has profile_stage, annotations,
 *                    assets, extraction provenance. Written by tenant members.
 *
 * 'master_catalog' — platform reference record. Has confidence score, multi-source aliases,
 *                    external provider links. Written only by PLATFORM_ADMIN.
 *                    More data-complete than a typical tenant Venue at any given moment.
 */
export type VenueContext = "tenant" | "master_catalog";

/** Common fields shared by both entities — used in shared components. */
export interface AnyVenue {
  id: string;
  context: VenueContext;
  name: string;
  city: string | null;
  countryCode: string | null;
  address: string | null;
  location: { lat: number; lng: number } | null;
  websiteUrl: string | null;
  description: string | null;
  primaryPhotoUrl: string | null;
  metadata: VenueMetadata;
  createdAt: string;
  updatedAt: string;

  // ── Tenant-only fields (undefined on master_catalog) ──────────────
  displayName?: string | null;
  profileStage?: ProfileStage;
  status?: VenueStatus;
  source?: VenueSource;
  masterVenueId?: string | null;
  lastUsedInSalesRoomAt?: string | null;
  /** Per-field provenance map — keys are VenueMetadata dot-paths. */
  metadataSources?: Record<string, FieldProvenance>;
  metadataAggregatedAt?: string | null;

  // ── Master Catalog-only fields (undefined on tenant) ──────────────
  /** Overall confidence score (0.0–1.0). Admin-facing — not shown to tenants. */
  confidence?: number;
  /** Ingestion channel: 'platform_seed' | 'admin_import' | 'web_scrape'. */
  catalogSource?: string;
  /** Known alternate names for this venue. */
  aliases?: string[];
  lastSyncedAt?: string | null;
}

export type VenueStatus = "DRAFT" | "ACTIVE" | "ARCHIVED";
export type ProfileStage = "SEEDED" | "ENRICHED" | "CURATED" | "READY";
export type VenueSource = "MANUAL" | "FILE_IMPORT" | "MASTER_CATALOG" | "BULK_CSV";

// ── Annotation ───────────────────────────────────────────────────────────────
// Annotation types belong in the addon — they are part of the venue domain,
// not app-specific projections.

export type AnnotationType = "NOTE" | "TAG" | "RATING" | "BOOKMARK" | "INTERNAL_FLAG";

export interface VenueAnnotation {
  id: string;
  venueId: string;
  annotationType: AnnotationType;
  textValue: string | null;
  colorHex: string | null;
  numericValue: number | null;
  isPrivate: boolean;
  createdBy: string;
  createdAt: string;
  updatedAt: string;
}

// ── Re-exports from @venuemi/ui-types ────────────────────────────────────────
// Types and constants that are pure TS (no React) live in the shared package.
// The addon imports them so consumers only need to import from the addon — never
// from @venuemi/ui-types directly in app code.

export type {
  AnyVenue as _AnyVenueBase, // imported above; re-exported via index.ts as AnyVenue
  VenueMetadata,
  FieldProvenance,
  CateringPolicy,
  PROFILE_STAGES,
  VENUE_STATUSES,
  VENUE_SOURCES,
  VENUE_CONTEXTS,
  ANNOTATION_TYPES,
  CATERING_POLICIES,
  isProfileStage,
  isVenueStatus,
  isVenueContext,
} from "@venuemi/ui-types";
```

> **Rule:** All `as const` arrays (`PROFILE_STAGES`, `VENUE_STATUSES`, etc.), type guards
> (`isProfileStage`, …), and `VenueMetadata`/`FieldProvenance` live in `@venuemi/ui-types`.
> The addon imports them and re-exports through `index.ts` so consuming app code never imports
> directly from `@venuemi/ui-types` — it always imports from `@/addons/venue-management`.
```

**Rule:** shared components (`VenueProfileHeader`, `VenueProfileTabs`, `VenueFormField`,
`VenueListItem`) accept `AnyVenue` and branch on `context` only where rendering genuinely differs.
Tenant-only panels (`VenueAnnotationsPanel`, `VenueQuickFillBar`) are not rendered when
`context === 'master_catalog'`.

---

## 3. Entity Types

### Tenant `Venue`

```typescript
// src/entities/venue/types.ts
// App-specific projection shapes only. Domain types re-exported from the addon.

export type {
  AnyVenue,
  AnnotationType,
  VenueAnnotation,
  VenueContext,
  VenueSource,
  VenueStatus,
  ProfileStage,
} from "@/addons/venue-management";

export type {
  AssetType,
  PhotoCategory,
  ExtractionStatus,
  AssetTableData,
  VenueAsset,
  VenueAssetSummary,
} from "@venuemi/ui-types";

/** Flat projection for list view — server resolves photo URL and key metadata fields. */
export interface VenueSummary {
  id: string;
  context: "tenant";
  name: string;
  displayName: string | null;
  city: string | null;
  countryCode: string | null;
  status: VenueStatus;
  profileStage: ProfileStage;
  source: VenueSource;
  primaryPhotoUrl: string | null;
  capacityMaxTotal: number | null; // resolved server-side — no client JSONB parse
  cateringPolicy: string | null; // resolved server-side
  masterVenueId: string | null;
  lastUsedInSalesRoomAt: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface VenueSummaryListResponse {
  items: VenueSummary[];
  totalElements: number;
}

/** Full detail — loaded on venue profile page. */
export interface VenueDetail extends VenueSummary, AnyVenue {
  context: "tenant";
  metadataSources: Record<string, FieldProvenance>;
  metadataAggregatedAt: string | null;
}
```

> **Rule:** `src/entities/venue/types.ts` contains only app-specific projection shapes
> (`VenueSummary`, `VenueDetail`). All domain types — `AnyVenue`, `VenueMetadata`,
> `FieldProvenance`, `VenueAnnotation`, constants, guards — live in
> `src/addons/venue-management/` and are re-exported through its `index.ts`.
> Entity files must never define types that exist in the addon or in `@venuemi/ui-types`.

### `MasterVenue`

```typescript
// src/entities/master-venue/types.ts  (foundation-ui-platform-admin)

export type { AnyVenue } from "@/addons/venue-management/types";

/** Flat projection for master venue list. */
export interface MasterVenueSummary {
  id: string;
  context: "master_catalog";
  name: string;
  city: string | null;
  countryCode: string | null;
  confidence: number;
  catalogSource: string;
  primaryPhotoUrl: string | null;
  capacityMaxTotal: number | null;
  cateringPolicy: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface MasterVenueSummaryListResponse {
  items: MasterVenueSummary[];
  totalElements: number;
}

/** Full detail — used in admin edit form. */
export interface MasterVenueDetail extends MasterVenueSummary, AnyVenue {
  context: "master_catalog";
  aliases: string[];
  externalProviders: Array<{
    provider: string;
    providerExternalId: string;
    crawledAt: string;
  }>;
}
```

### `VenueMetadata` — shared by both

```typescript
// src/addons/venue-management/types.ts  (continued)

/** Canonical metadata shape. Shared by Venue and MasterVenue. All fields nullable. */
export interface VenueMetadata {
  _schema_version: number;
  capacity?: {
    max_total?: number | null;
    configurations?: {
      banquet?: number | null;
      theater?: number | null;
      classroom?: number | null;
      cocktail?: number | null;
      conference?: number | null;
    };
  };
  venue_type?: string[];
  location_notes?: string | null;
  catering?: {
    policy?: "in_house_exclusive" | "in_house_preferred" | "outside_allowed" | "no_catering" | null;
    kosher_available?: boolean | null;
    halal_available?: boolean | null;
    vegan_available?: boolean | null;
  };
  av_tech?: {
    built_in_av?: boolean | null;
    projector_lumens?: number | null;
    screens?: number | null;
    rigging_points?: boolean | null;
    internet_bandwidth_mbps?: number | null;
  };
  accessibility?: {
    ada_compliant?: boolean | null;
    elevator_access?: boolean | null;
    accessible_restrooms?: boolean | null;
    wheelchair_stage_access?: boolean | null;
  };
  logistics?: {
    load_in_access?: string | null;
    parking_spaces?: number | null;
    valet_available?: boolean | null;
    curfew_time?: string | null;
    setup_hours_before?: number | null;
    teardown_hours_after?: number | null;
  };
  outdoor_space?: {
    available?: boolean | null;
    covered?: boolean | null;
    sqm?: number | null;
  };
  exclusivity?: {
    exclusive_use?: boolean | null;
    shared_space?: boolean | null;
  };
  restrictions?: string[];
  amenities?: string[];
  contacts?: Array<{ name?: string; role?: string; email?: string; phone?: string }>;
  pricing?: {
    minimum_spend?: number | null;
    currency?: string | null;
    rental_fee_indicative?: number | null;
  };
  social?: {
    instagram_handle?: string | null;
    google_place_id?: string | null;
  };
  ratings?: {
    google_rating?: number | null;
    google_review_count?: number | null;
  };
}

/** Per-field provenance — from metadata_sources[fieldKey] on tenant Venue. */
export interface FieldProvenance {
  value: unknown;
  confidence: number;
  source_type: string;
  source_id: string | null;
  updated_at: string;
  alternatives?: Array<{ value: unknown; confidence: number; source_type: string }>;
}
```

---

## 4. App Placement

### `foundation-ui-app` (tenant)

```
src/
├── addons/
│   └── venue-management/          ← shared addon: types, registry, utils, CSS tokens
│       ├── index.ts
│       ├── types.ts               ← AnyVenue, VenueContext, VenueMetadata, FieldProvenance
│       └── spec/
│           ├── form-spec.types.ts
│           ├── venue-form-spec.ts
│           └── form-spec.utils.ts
├── entities/
│   └── venue/
│       ├── index.ts
│       └── types.ts               ← VenueSummary, VenueDetail, VenueAnnotation
├── features/
│   ├── create-venue/
│   ├── edit-venue-metadata/       ← tabbed form via form spec
│   ├── upload-venue-asset/
│   └── venue-quick-fill/          ← SEEDED→ENRICHED inline nudge
├── widgets/
│   ├── venue-list/
│   └── venue-profile/
└── pages/
    ├── venues/                    ← /venues
    └── venues_.$id/               ← /venues/:id
```

### `foundation-ui-platform-admin` (admin)

```
src/
├── addons/
│   └── venue-management/          ← same addon package (shared or copied)
│       └── ...                    ← identical to tenant app — no duplication
├── entities/
│   └── master-venue/
│       ├── index.ts
│       └── types.ts               ← MasterVenueSummary, MasterVenueDetail
├── features/
│   ├── create-master-venue/
│   └── edit-master-venue/         ← same VenueMetadataForm, context='master_catalog'
├── widgets/
│   └── master-venue-list/         ← same VenueListItem, context='master_catalog'
└── pages/
    └── master-venues/             ← /master-venues (PLATFORM_ADMIN only)
```

The `venue-management` addon is **shared between both apps** — either as a workspace package or
copied. Components, registry, and utils are identical. The admin app passes
`context='master_catalog'` to suppress tenant-only panels.

---

## 5. form spec

### Types

```typescript
// src/addons/venue-management/spec/form-spec.types.ts

import type { ProfileStage } from "../types";

export type FieldType =
  | "string"
  | "integer"
  | "decimal"
  | "boolean"
  | "enum"
  | "enum_multi"
  | "string_array"
  | "time"
  | "url";

export type FieldAction = "verify_place_id" | "open_asset_picker" | "enrich_from_web";

export interface EnumOption {
  value: string;
  label: string;
  color?: string;
}

export interface FieldDefinition {
  key: string; // dot-path into VenueMetadata
  label: string;
  type: FieldType;
  tab: string;
  section: string;
  order: number;
  unit?: string;
  placeholder?: string;
  hint?: string;
  options?: EnumOption[];
  requiredForStage?: ProfileStage; // tenant only — drives completion nudge
  quickFill?: boolean;
  showSourceBadge?: boolean;       // show provenance badge next to this field
  readOnly?: boolean;              // field cannot be edited (e.g. master catalog context)
  action?: FieldAction;
  min?: number;
  max?: number;
}

export interface SectionDefinition {
  key: string;
  label: string;
  tab: string;
  order: number;
  collapsible?: boolean;
  collapsedByDefault?: boolean;
}

export interface TabDefinition {
  key: string;
  label: string;
  order: number;
  icon?: string;
}

export interface VenueFormSpec {
  tabs: TabDefinition[];
  sections: SectionDefinition[];
  fields: FieldDefinition[];
}
```

### Registry data

One registry, used by both apps unchanged.

```typescript
// src/addons/venue-management/spec/venue-form-spec.ts

export const VENUE_FORM_SPEC: VenueFormSpec = {
  tabs: [
    { key: "basics", label: "Basics", order: 1, icon: "Building2" },
    { key: "technical", label: "Technical", order: 2, icon: "Cpu" },
    { key: "logistics", label: "Logistics", order: 3, icon: "Truck" },
    { key: "details", label: "Details", order: 4, icon: "Info" },
  ],
  sections: [
    { key: "capacity", label: "Capacity", tab: "basics", order: 1 },
    { key: "catering", label: "Catering", tab: "basics", order: 2 },
    {
      key: "outdoor",
      label: "Outdoor Space",
      tab: "basics",
      order: 3,
      collapsible: true,
      collapsedByDefault: true,
    },
    {
      key: "exclusivity",
      label: "Exclusivity",
      tab: "basics",
      order: 4,
      collapsible: true,
      collapsedByDefault: true,
    },
    { key: "av", label: "AV & Tech", tab: "technical", order: 1 },
    { key: "accessibility", label: "Accessibility", tab: "technical", order: 2, collapsible: true },
    { key: "setup", label: "Setup & Teardown", tab: "logistics", order: 1 },
    { key: "parking", label: "Parking", tab: "logistics", order: 2, collapsible: true },
    { key: "pricing", label: "Pricing", tab: "details", order: 1 },
    { key: "contacts", label: "Contacts", tab: "details", order: 2 },
    { key: "online", label: "Online Presence", tab: "details", order: 3, collapsible: true },
    {
      key: "ratings",
      label: "Ratings",
      tab: "details",
      order: 4,
      collapsible: true,
      collapsedByDefault: true,
    },
  ],
  fields: [
    // ── Basics / Capacity ─────────────────────────────────────────
    {
      key: "capacity.max_total",
      label: "Max Capacity",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 1,
      unit: "guests",
      requiredForStage: "ENRICHED",
      quickFill: true,
      showSourceBadge: true,
    },
    {
      key: "capacity.configurations.banquet",
      label: "Banquet",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 2,
      unit: "guests",
      showSourceBadge: true,
    },
    {
      key: "capacity.configurations.theater",
      label: "Theater",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 3,
      unit: "guests",
      showSourceBadge: true,
    },
    {
      key: "capacity.configurations.classroom",
      label: "Classroom",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 4,
      unit: "guests",
    },
    {
      key: "capacity.configurations.cocktail",
      label: "Cocktail",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 5,
      unit: "guests",
    },
    {
      key: "capacity.configurations.conference",
      label: "Conference",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 6,
      unit: "guests",
    },
    // ── Basics / Catering ─────────────────────────────────────────
    {
      key: "catering.policy",
      label: "Catering Policy",
      type: "enum",
      tab: "basics",
      section: "catering",
      order: 1,
      options: [
        { value: "in_house_exclusive", label: "In-house only", color: "#EF4444" },
        { value: "in_house_preferred", label: "In-house preferred", color: "#F59E0B" },
        { value: "outside_allowed", label: "Outside allowed", color: "#22C55E" },
        { value: "no_catering", label: "No catering", color: "#6B7280" },
      ],
      requiredForStage: "ENRICHED",
      quickFill: true,
      showSourceBadge: true,
    },
    {
      key: "catering.kosher_available",
      label: "Kosher",
      type: "boolean",
      tab: "basics",
      section: "catering",
      order: 2,
    },
    {
      key: "catering.halal_available",
      label: "Halal",
      type: "boolean",
      tab: "basics",
      section: "catering",
      order: 3,
    },
    {
      key: "catering.vegan_available",
      label: "Vegan",
      type: "boolean",
      tab: "basics",
      section: "catering",
      order: 4,
    },
    // ── Basics / Outdoor ──────────────────────────────────────────
    {
      key: "outdoor_space.available",
      label: "Outdoor Space",
      type: "boolean",
      tab: "basics",
      section: "outdoor",
      order: 1,
    },
    {
      key: "outdoor_space.covered",
      label: "Covered",
      type: "boolean",
      tab: "basics",
      section: "outdoor",
      order: 2,
    },
    {
      key: "outdoor_space.sqm",
      label: "Area",
      type: "integer",
      tab: "basics",
      section: "outdoor",
      order: 3,
      unit: "sqm",
    },
    // ── Basics / Exclusivity ──────────────────────────────────────
    {
      key: "exclusivity.exclusive_use",
      label: "Exclusive Use",
      type: "boolean",
      tab: "basics",
      section: "exclusivity",
      order: 1,
    },
    {
      key: "exclusivity.shared_space",
      label: "Shared Space",
      type: "boolean",
      tab: "basics",
      section: "exclusivity",
      order: 2,
    },
    // ── Technical / AV ────────────────────────────────────────────
    {
      key: "av_tech.built_in_av",
      label: "Built-in AV",
      type: "boolean",
      tab: "technical",
      section: "av",
      order: 1,
    },
    {
      key: "av_tech.projector_lumens",
      label: "Projector",
      type: "integer",
      tab: "technical",
      section: "av",
      order: 2,
      unit: "lm",
      showSourceBadge: true,
    },
    {
      key: "av_tech.screens",
      label: "Screens",
      type: "integer",
      tab: "technical",
      section: "av",
      order: 3,
    },
    {
      key: "av_tech.rigging_points",
      label: "Rigging",
      type: "boolean",
      tab: "technical",
      section: "av",
      order: 4,
    },
    {
      key: "av_tech.internet_bandwidth_mbps",
      label: "Internet",
      type: "integer",
      tab: "technical",
      section: "av",
      order: 5,
      unit: "Mbps",
      showSourceBadge: true,
    },
    // ── Technical / Accessibility ─────────────────────────────────
    {
      key: "accessibility.ada_compliant",
      label: "ADA Compliant",
      type: "boolean",
      tab: "technical",
      section: "accessibility",
      order: 1,
    },
    {
      key: "accessibility.elevator_access",
      label: "Elevator Access",
      type: "boolean",
      tab: "technical",
      section: "accessibility",
      order: 2,
    },
    {
      key: "accessibility.accessible_restrooms",
      label: "Accessible Restrooms",
      type: "boolean",
      tab: "technical",
      section: "accessibility",
      order: 3,
    },
    {
      key: "accessibility.wheelchair_stage_access",
      label: "Wheelchair Stage",
      type: "boolean",
      tab: "technical",
      section: "accessibility",
      order: 4,
    },
    // ── Logistics / Setup ─────────────────────────────────────────
    {
      key: "logistics.setup_hours_before",
      label: "Setup Before",
      type: "integer",
      tab: "logistics",
      section: "setup",
      order: 1,
      unit: "h",
    },
    {
      key: "logistics.teardown_hours_after",
      label: "Teardown After",
      type: "integer",
      tab: "logistics",
      section: "setup",
      order: 2,
      unit: "h",
    },
    {
      key: "logistics.curfew_time",
      label: "Curfew",
      type: "time",
      tab: "logistics",
      section: "setup",
      order: 3,
      showSourceBadge: true,
    },
    {
      key: "logistics.load_in_access",
      label: "Load-in Access",
      type: "string",
      tab: "logistics",
      section: "setup",
      order: 4,
    },
    {
      key: "restrictions",
      label: "Restrictions",
      type: "string_array",
      tab: "logistics",
      section: "setup",
      order: 5,
      showSourceBadge: true,
    },
    // ── Logistics / Parking ───────────────────────────────────────
    {
      key: "logistics.parking_spaces",
      label: "Parking Spaces",
      type: "integer",
      tab: "logistics",
      section: "parking",
      order: 1,
    },
    {
      key: "logistics.valet_available",
      label: "Valet",
      type: "boolean",
      tab: "logistics",
      section: "parking",
      order: 2,
    },
    // ── Details / Pricing ─────────────────────────────────────────
    {
      key: "pricing.rental_fee_indicative",
      label: "Rental Fee",
      type: "integer",
      tab: "details",
      section: "pricing",
      order: 1,
      showSourceBadge: true,
    },
    {
      key: "pricing.minimum_spend",
      label: "Minimum Spend",
      type: "integer",
      tab: "details",
      section: "pricing",
      order: 2,
      showSourceBadge: true,
    },
    {
      key: "pricing.currency",
      label: "Currency",
      type: "string",
      tab: "details",
      section: "pricing",
      order: 3,
    },
    // ── Details / Online ──────────────────────────────────────────
    {
      key: "social.google_place_id",
      label: "Google Place ID",
      type: "string",
      tab: "details",
      section: "online",
      order: 1,
      hint: "Enables automatic enrichment of address, photos and ratings",
      action: "verify_place_id",
    },
    {
      key: "social.instagram_handle",
      label: "Instagram Handle",
      type: "string",
      tab: "details",
      section: "online",
      order: 2,
      placeholder: "@venue_name",
    },
    // ── Details / Ratings ─────────────────────────────────────────
    {
      key: "ratings.google_rating",
      label: "Google Rating",
      type: "decimal",
      tab: "details",
      section: "ratings",
      order: 1,
      min: 0,
      max: 5,
      showSourceBadge: true,
    },
    {
      key: "ratings.google_review_count",
      label: "Reviews",
      type: "integer",
      tab: "details",
      section: "ratings",
      order: 2,
      showSourceBadge: true,
    },
  ],
};
```

### Registry utils

```typescript
// src/addons/venue-management/spec/form-spec.utils.ts

import type { VenueFormSpec, FieldDefinition } from "./form-spec.types";
import type { ProfileStage, VenueMetadata } from "../types";

export function resolveFieldValue(metadata: VenueMetadata, key: string): unknown {
  return key.split(".").reduce<unknown>((obj, k) => {
    if (obj != null && typeof obj === "object") return (obj as Record<string, unknown>)[k];
    return undefined;
  }, metadata as unknown);
}

export function setFieldValue(metadata: VenueMetadata, key: string, value: unknown): VenueMetadata {
  const parts = key.split(".");
  const result = structuredClone(metadata) as Record<string, unknown>;
  let cursor = result;
  for (let i = 0; i < parts.length - 1; i++) {
    if (cursor[parts[i]] == null || typeof cursor[parts[i]] !== "object") cursor[parts[i]] = {};
    cursor = cursor[parts[i]] as Record<string, unknown>;
  }
  cursor[parts[parts.length - 1]] = value;
  return result as VenueMetadata;
}

export function missingFieldsForStage(
  registry: VenueFormSpec,
  metadata: VenueMetadata,
  stage: ProfileStage,
): FieldDefinition[] {
  return registry.fields.filter((f) => {
    if (f.requiredForStage !== stage) return false;
    const v = resolveFieldValue(metadata, f.key);
    return v == null || v === "";
  });
}

export function groupByTabAndSection(registry: VenueFormSpec) {
  return registry.tabs
    .slice()
    .sort((a, b) => a.order - b.order)
    .map((tab) => ({
      tab,
      sections: registry.sections
        .filter((s) => s.tab === tab.key)
        .sort((a, b) => a.order - b.order)
        .map((section) => ({
          section,
          fields: registry.fields
            .filter((f) => f.tab === tab.key && f.section === section.key)
            .sort((a, b) => a.order - b.order),
        })),
    }));
}
```

---

## 6. Component Structure

All components accept `AnyVenue`. Tenant-only panels are gated on `context`.

```
VenueProfilePage  (/venues/:id  or  /master-venues/:id)
│
├── VenueProfileHeader          ← name, city, status/confidence badge
│     context='tenant'      → ProfileStagePill + displayName (editable inline)
│     context='master_catalog' → ConfidenceBadge + catalogSource chip
│
├── VenueProfileTabs            ← tab bar; completion dots (tenant only)
│   └── [tab panel × 4]
│       └── VenueFormSection    ← collapsible
│           └── VenueFormField  ← dispatches on FieldDefinition.type
│               ├── IntegerField / DecimalField
│               ├── StringField / UrlField
│               ├── EnumField          colored option chips
│               ├── BooleanField       toggle
│               ├── TimeField
│               ├── StringArrayField   tag-input
│               └── SourceBadge        confidence chip + alternatives popover
│                     context='tenant'         → full provenance (confidence, source, alternatives)
│                     context='master_catalog' → catalog confidence at row level
│
├── VenueAnnotationsPanel       ← rendered only when context='tenant'
└── VenueAssetGallery           ← rendered only when context='tenant'
      │
      ├── AssetPhotoGallery          ← tabs per PhotoCategory; ordered by display_order
      │     thumbnail grid → lightbox on click
      │     empty category tabs hidden; 'Other' always last
      │
      ├── AssetDocumentList          ← groups: Floor Plans, Venue Decks, Spec Sheets,
      │     Menus, Price Lists, Misc  Menus, CAD Files, Misc
      │     each row: icon(AssetType), label|fileName, sizeBytes, cdnUrl download,
      │     extractionStatus chip (PENDING/IN_PROGRESS animate; FAILED shows warning)
      │
      ├── AssetDataTableViewer       ← rendered when asset has tableData
      │     inline table: headers + rows, max 10 rows shown, "Show all" expands
      │     source sheet name shown as subtitle
      │
      └── AssetUploadZone            ← drag-and-drop or file picker
            accepts: image/*, .pdf, .docx, .csv, .xlsx, .dwg, .dxf, .mp4, .mov
            on drop: infers AssetType from MIME + extension
            if AssetType === 'PHOTO': prompts PhotoCategory selector before confirm
            upload progress per file → fires POST /api/v1/venues/{id}/assets/initiate → PUT S3 → POST confirm

VenueListPage  (/venues  or  /master-venues)
├── VenueListFilters            ← city, status/context-appropriate filters
├── VenueList
│   └── VenueListItem           ← same component, context controls badge column
│         context='tenant'         → ProfileStagePill + quick-fill nudge
│         context='master_catalog' → ConfidenceBadge + source chip
└── VenueQuickFillBar           ← rendered only when context='tenant'
```

### `VenueFormField` dispatch

```typescript
// No switch-case — renderer map is the full dispatch table.
const FIELD_RENDERERS: Record<FieldType, React.ComponentType<VenueFormFieldProps>> = {
  string: StringField,
  integer: IntegerField,
  decimal: DecimalField,
  boolean: BooleanField,
  enum: EnumField,
  enum_multi: EnumMultiField,
  string_array: StringArrayField,
  time: TimeField,
  url: UrlField,
};

// Props accepted by all shared components
export interface VenueFormFieldProps {
  definition: FieldDefinition;
  value: unknown;
  provenance?: FieldProvenance; // undefined for master_catalog (row-level confidence only)
  onChange: (key: string, value: unknown) => void;
  readOnly?: boolean;
}
```

### `SourceBadge`

```typescript
// Confidence thresholds
const CONFIDENCE_LEVELS = [
  { min: 0.9, label: "High", color: "green" },
  { min: 0.7, label: "Medium", color: "yellow" },
  { min: 0.0, label: "Low", color: "gray" },
] as const;

// Source type overrides badge label and color
const SOURCE_OVERRIDES: Record<string, { label: string; color: string }> = {
  USER_INPUT: { label: "Manual", color: "blue" },
  MC_INHERIT: { label: "Catalog", color: "violet" },
};
// Click → popover with alternatives[] for inline conflict resolution
```

---

## 7. Themes / Skins

The addon owns its CSS tokens. Skins override tokens — no component changes, no prop drilling.
The `data-skin` attribute is set by the parent layout (Sales Room, admin app, tenant app).

```css
/* ── Default: tenant app ─────────────────────────────── */
:root {
  --vmi-form-tab-active: var(--mantine-color-blue-6);
  --vmi-section-bg: var(--mantine-color-gray-0);
  --vmi-badge-high: var(--mantine-color-green-6);
  --vmi-badge-medium: var(--mantine-color-yellow-6);
  --vmi-badge-low: var(--mantine-color-gray-5);
  --vmi-badge-manual: var(--mantine-color-blue-6);
  --vmi-badge-catalog: var(--mantine-color-violet-6);
}

/* ── Admin app: master catalog ───────────────────────── */
[data-skin="admin"] {
  --vmi-form-tab-active: var(--mantine-color-red-6);
  --vmi-section-bg: var(--mantine-color-gray-1);
}

/* ── Sales Room: client-facing microsite ─────────────── */
[data-skin="sales-room"] {
  --vmi-form-tab-active: var(--brand-primary);
  --vmi-section-bg: #ffffff;
  --vmi-badge-high: transparent; /* hide confidence from client */
  --vmi-badge-medium: transparent;
  --vmi-badge-low: transparent;
}
```

---

## 8. API Integration

```typescript
// src/shared/api/venue.ts  (tenant app)

// List — flat summary, no client JSONB
GET  /api/v1/venues?city=&status=&profileStage=&page=&size=
// Full detail
GET  /api/v1/venues/{id}
// Create
POST /api/v1/venues
// Patch one metadata field → triggers aggregation + stage recalc
PATCH /api/v1/venues/{id}/metadata   { fieldKey: string; value: unknown }
// Annotations
GET|POST        /api/v1/venues/{id}/annotations
PATCH|DELETE    /api/v1/venues/{id}/annotations/{annotationId}
// Assets — summary list (no tableData, fast for gallery render)
GET    /api/v1/venues/{id}/assets?type=&category=   → VenueAssetSummary[]
// Full asset (includes tableData — loaded on demand)
GET    /api/v1/venues/{id}/assets/{assetId}          → VenueAsset
// Two-phase upload
POST   /api/v1/venues/{id}/assets/initiate   { assetType, photoCategory?, fileName, contentType, sizeBytes }
PUT    <presignedS3Url>                     direct from browser
POST   /api/v1/venues/{id}/assets/{assetId}/confirm
// Update label, photoCategory, displayOrder
PATCH  /api/v1/venues/{id}/assets/{assetId}
DELETE /api/v1/venues/{id}/assets/{assetId}

// src/shared/api/master-venue.ts  (admin app)

GET  /api/v1/venues/admin/master-venues?city=&page=&size=
GET  /api/v1/venues/admin/master-venues/{id}
POST /api/v1/venues/admin/master-venues
PUT  /api/v1/venues/admin/master-venues/{id}

// master-venue MEMBER read (tenant app — search + browse master catalog)
GET  /api/v1/venues/master-venues?search=&city=&country_code=&capacity=&page=&size=
GET  /api/v1/venues/master-venues/{id}
```

---

## 9. Routes

| Path                 | App            | Auth            | Component          |
| -------------------- | -------------- | --------------- | ------------------ |
| `/venues`            | tenant         | MEMBER+         | `VenueListPage`    |
| `/venues/new`        | tenant         | MEMBER+         | `CreateVenueModal` |
| `/venues/:id`        | tenant         | MEMBER+         | `VenueProfilePage` |
| `/master-venues`     | platform admin | PLATFORM\_ADMIN | `VenueListPage`    |
| `/master-venues/:id` | platform admin | PLATFORM\_ADMIN | `VenueProfilePage` |

Note: `/master-venues` routes use the **same page components** as `/venues`, passing
`context='master_catalog'` — not separate page implementations.

---

## 10. Evolution Path

`MasterVenue` is currently the more complete entity (richer data, multi-source, higher confidence).
Tenant `Venue` gains parity over time. The shared `AnyVenue` abstraction means UI features added
for one context become available to the other by updating the `context` gate — no component
rewrite needed.

| Feature                 | MasterVenue  | Tenant Venue      |
| ----------------------- | ------------ | ----------------- |
| Canonical metadata      | ✅ MVP       | ✅ MVP            |
| Per-field provenance    | ✅ row-level | ✅ full per-field |
| Photo gallery           | Phase 2      | ✅ MVP            |
| Annotations / tags      | —            | ✅ MVP            |
| Profile stage           | —            | ✅ MVP            |
| Aliases                 | ✅ MVP       | Phase 2           |
| External provider links | ✅ MVP       | —                 |
| Sales Room export       | —            | Phase 2           |

---

## 11. Prototype Strategy

> **Prototype scope note.**
> The prototype is a **fully interactive implementation** — not a mockup or a clickable wireframe.
> Every button, form, field, transition, and screen state must work. Navigation between pages is
> real (TanStack Router). Forms submit and show validation errors. List items are clickable and
> open detail pages. Panels open, close, and collapse. `profile_stage` badges update after a
> quick-fill save. Confidence badges open popovers with alternatives. Annotations are added,
> edited, and deleted. The only thing emulated is the API layer — all network calls are
> intercepted by MSW handlers returning fixture data.
>
> Shared code (`@venuemi/ui-types`) is referenced via `file:` protocol pointing directly
> to TypeScript source — no build step, no publish. See
> [ui-shared-packages.md](ui-shared-packages.md) §7 for the exact setup.

Venue management UI is prototyped in **`foundation-ui-blank`** before being promoted to the
production apps. This keeps auth, billing, and routing concerns out of UX exploration.

### Why `foundation-ui-blank`

`foundation-ui-blank` has the same stack (Mantine, TanStack Router, React Query, Zustand,
ts-pattern, MSW) and the same FSD + addon lifecycle (`addonRegistry`, `loadAddons`,
`addonRegistry.initializeAll`). The addon developed here moves to `foundation-ui-app` unchanged —
no rewrite, only wiring.

### Prototype structure

```
foundation-ui-blank/src/
├── addons/
│   └── venue-management/        ← develop the addon here first
│       ├── types.ts
│       ├── index.ts
│       └── spec/
│           ├── form-spec.types.ts
│           ├── venue-form-spec.ts
│           └── form-spec.utils.ts
├── entities/
│   └── venue/                   ← VenueSummary, VenueDetail, VenueAnnotation
├── features/
│   ├── create-venue/
│   ├── edit-venue-metadata/
│   └── venue-quick-fill/
├── widgets/
│   ├── venue-list/
│   └── venue-profile/
├── pages/
│   ├── venues/                  ← /venues (no auth guard)
│   └── venues_.$id/             ← /venues/:id (no auth guard)
└── shared/
    ├── api/
    │   └── venue.ts             ← same venueApi shape as production
    └── mocks/
        └── handlers/
            └── venue.ts         ← MSW handlers (fixture data, no real backend needed)
```

### MSW fixture scope

Handlers to implement for a working prototype:

```
GET    /api/v1/venues              → VenueSummaryListResponse (10–20 fixtures)
GET    /api/v1/venues/{id}          → VenueDetail (full metadata + metadata_sources)
POST   /api/v1/venues              → create, return new VenueDetail
PATCH  /api/v1/venues/{id}/metadata → update one field, recalculate profileStage
GET    /api/v1/venues/{id}/annotations
POST   /api/v1/venues/{id}/annotations
DELETE /api/v1/venues/{id}/annotations/{annotationId}
```

Fixture data should cover all four `profileStage` values and both `context` variants
(`tenant`, `master_catalog`) to exercise full rendering logic.

### Promotion checklist (blank → foundation-ui-app)

When the prototype UX is approved:

- [ ] Copy `src/addons/venue-management/` verbatim — no changes
- [ ] Copy `src/entities/venue/` verbatim
- [ ] Copy `src/shared/api/venue.ts` verbatim — already points to real endpoints
- [ ] Copy `src/features/` and `src/widgets/` venue directories verbatim
- [ ] Add auth guards to page routes (`hasAnyAuthority('MEMBER', ...)`)
- [ ] Wire `FeatureGate` / `useEntitlements` where plan limits apply
- [ ] Replace MSW handlers with real backend (remove from `handlers/index.ts`)
- [ ] Add routes to TanStack Router tree (see §9)
- [ ] Repeat for `foundation-ui-platform-admin` with `context='master_catalog'`

MSW handlers from `foundation-ui-blank` are kept as-is for local dev and Playwright fixtures.

---

## 12. File Contracts (implementation reference)

Files to create. Contents are defined in §§ 2–8 above — this section is the index to prevent drift.

### `foundation-ui-blank` (prototype) → `foundation-ui-app` (production)

The addon file layout is **identical** across prototype, tenant app, and admin app. No rewrites,
no restructuring — only wiring changes (auth guards, feature gates, route registration).

```
src/addons/venue-management/
  index.ts                 PUBLIC BARREL — the only import path used by the rest of the app.
                           Re-exports everything below. Nothing outside this directory imports
                           addon internals directly.

  types.ts                 AnyVenue, VenueContext, VenueMetadata, FieldProvenance,
                           ProfileStage, VenueStatus, VenueSource, CateringPolicy,
                           AnnotationType, VenueAnnotation.
                           Imports base types/constants from @venuemi/ui-types;
                           re-exports through index.ts so app code never imports
                           @venuemi/ui-types directly.

  spec/
    form-spec.types.ts     FieldType, FieldAction, EnumOption, FieldDefinition,
                           SectionDefinition, TabDefinition, VenueFormSpec.
                           Imports ProfileStage from ./types (not from @venuemi/ui-types directly).

    venue-form-spec.ts     VENUE_FORM_SPEC — the single source of truth for
                           tabs, sections, and fields.

    form-spec.utils.ts     resolveFieldValue, setFieldValue,
                           missingFieldsForStage, groupByTabAndSection.

  components/              React + Mantine components. Never exported outside the addon
                           as individual files — only through index.ts.
    VenueFormField.tsx     Imports FieldDefinition, FieldType from ./spec/form-spec.types
    VenueFormSection.tsx
    VenueFormTabs.tsx
    SourceBadge.tsx        Imports FieldProvenance from ./types
    ProfileStagePill.tsx   Imports ProfileStage, PROFILE_STAGES from ./types
```

> **Addon rule (enforced from day one, including in the prototype):**
> Every import into a page, widget, or feature that touches venue domain logic goes through
> `@/addons/venue-management` — never through `@/addons/venue-management/types` or any
> internal path. Architecture tests assert this.

```
src/entities/venue/
  types.ts                 VenueSummary, VenueSummaryListResponse, VenueDetail.
                           App-specific projection shapes only.
                           Re-exports domain types from @/addons/venue-management.
  index.ts                 Re-exports types.ts.

src/shared/api/
  venue.ts                 venueApi object: list, getById, create, patchMetadata,
                           listAnnotations, createAnnotation, updateAnnotation,
                           deleteAnnotation, listAssets, getAsset,
                           initiateUpload, confirmUpload, updateAsset, deleteAsset,
                           listFromMasterCatalog, promoteFromMasterCatalog.
                           ListVenuesParams, PatchMetadataRequest interfaces.
                           Lives in shared/api/ by FSD convention (HTTP layer);
                           the API shape is dictated by the addon's entity types.
```

### `foundation-ui-platform-admin`

```
src/addons/venue-management/
  ...                      Identical copy of the tenant app addon — same index.ts,
                           types.ts, spec/, and components/. The admin app passes
                           context='master_catalog' to suppress tenant-only panels.
                           No separate admin addon. No duplication.

src/entities/master-venue/
  types.ts                 MasterVenueSummary, MasterVenueSummaryListResponse,
                           MasterVenueDetail, MasterVenueAlias, MasterVenueExternal.
                           Re-exports AnyVenue + VenueMetadata from @/addons/venue-management.
  index.ts                 Re-exports types.ts.

src/shared/api/
  master-venue.ts          masterVenueApi object:
                             list (MEMBER — search/filter/paginate),
                             getById (MEMBER),
                             adminList, adminGetById, adminCreate, adminUpdate,
                             adminDelete, dedupCheck, bulkImport, merge.
                           ListMasterVenuesParams, AdminListMasterVenuesParams interfaces.
```

### Constant anchors (prevent magic strings)

All `as const` arrays and derived types live in `@venuemi/ui-types`. The addon imports and
re-exports them — app code never imports from `@venuemi/ui-types` directly.

```typescript
// src/addons/venue-management/types.ts — re-export pattern

// From @venuemi/ui-types (single source of truth — do not redefine here)
export {
  PROFILE_STAGES, VENUE_STATUSES, VENUE_SOURCES, VENUE_SOURCE_LABELS, VENUE_CONTEXTS,
  ANNOTATION_TYPES, ANNOTATION_TYPE_LABELS, CATERING_POLICIES, CATERING_POLICY_LABELS,
  PROFILE_STAGE_LABELS, VENUE_STATUS_LABELS,
  isProfileStage, isVenueStatus, isVenueSource, isVenueContext, isCateringPolicy,
} from "@venuemi/ui-types";

export type {
  ProfileStage, VenueStatus, VenueSource, VenueContext,
  AnnotationType, CateringPolicy,
  AnyVenue, VenueMetadata, FieldProvenance,
} from "@venuemi/ui-types";
```

Architecture tests in each app assert:
- No file in `src/` outside `src/addons/venue-management/` imports directly from `@venuemi/ui-types`.
- No file in `src/` redefines `VenueMetadata`, `ProfileStage`, `PROFILE_STAGES`, or any other
  type or constant that already exists in `@venuemi/ui-types`.
- Every feature, widget, and page that uses venue types imports from `@/addons/venue-management`
  or `@/entities/venue` — never from addon internals or from `@venuemi/ui-types` directly.

---

**Docs:** [Architecture Overview](architecture-overview.md) · [Data Model](data-model.md) · [API](api.md) · [Search](search.md)
