# VenueMi — UI: Venue Management

> **Audience:** Frontend engineers, designers.
> **Purpose:** Component structure, field registry pattern, shared entity abstraction, form layout
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

- **DRY / KISS first.** One field registry, one set of form components, one entity abstraction.
  Both `Venue` (tenant) and `MasterVenue` (platform) share the same canonical metadata schema and
  therefore the same form. Differences are handled by a single `context` discriminator — not by
  duplicating components.
- **MasterVenue is the more mature entity.** It has richer data (multi-source scraped, admin-curated,
  high-confidence). Tenant `Venue` is catching up as the product grows. The UI layer reflects this:
  features present on `MasterVenue` first, promoted to tenant `Venue` as backend adds support.
- **Schema-driven form.** The form is built from a static field registry (TS constant). Adding a
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
  /** Present for 'tenant'. Absent (undefined) for 'master_catalog'. */
  metadataSources?: Record<string, FieldProvenance>;
  createdAt: string;
  updatedAt: string;

  // ── Tenant-only fields (undefined on master_catalog) ──────────────
  displayName?: string | null;
  profileStage?: ProfileStage;
  source?: VenueSource;
  masterVenueId?: string | null;
  lastUsedInSalesRoomAt?: string | null;

  // ── Master Catalog-only fields (undefined on tenant) ──────────────
  /** Overall confidence score for master catalog record (0.0–1.0). */
  confidence?: number;
  /** Data source channel: 'platform_seed' | 'admin_import' | 'web_scrape'. */
  catalogSource?: string;
}

export type VenueStatus = "DRAFT" | "ACTIVE" | "ARCHIVED";
export type ProfileStage = "SEEDED" | "ENRICHED" | "CURATED" | "READY";
export type VenueSource = "MANUAL" | "FILE_IMPORT" | "MASTER_CATALOG" | "BULK_CSV";
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

export type {
  VenueContext,
  AnyVenue,
  VenueStatus,
  ProfileStage,
  VenueSource,
} from "@/addons/venue-management/types";

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
```

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
│       └── registry/
│           ├── field-registry.types.ts
│           ├── venue-field-registry.ts
│           └── registry.utils.ts
├── entities/
│   └── venue/
│       ├── index.ts
│       └── types.ts               ← VenueSummary, VenueDetail, VenueAnnotation
├── features/
│   ├── create-venue/
│   ├── edit-venue-metadata/       ← tabbed form via field registry
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

## 5. Field Registry

### Types

```typescript
// src/addons/venue-management/registry/field-registry.types.ts

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
  sourceBadge?: boolean;
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

export interface VenueFormRegistry {
  tabs: TabDefinition[];
  sections: SectionDefinition[];
  fields: FieldDefinition[];
}
```

### Registry data

One registry, used by both apps unchanged.

```typescript
// src/addons/venue-management/registry/venue-field-registry.ts

export const VENUE_FIELD_REGISTRY: VenueFormRegistry = {
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
      sourceBadge: true,
    },
    {
      key: "capacity.configurations.banquet",
      label: "Banquet",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 2,
      unit: "guests",
      sourceBadge: true,
    },
    {
      key: "capacity.configurations.theater",
      label: "Theater",
      type: "integer",
      tab: "basics",
      section: "capacity",
      order: 3,
      unit: "guests",
      sourceBadge: true,
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
      sourceBadge: true,
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
      sourceBadge: true,
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
      sourceBadge: true,
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
      sourceBadge: true,
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
      sourceBadge: true,
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
      sourceBadge: true,
    },
    {
      key: "pricing.minimum_spend",
      label: "Minimum Spend",
      type: "integer",
      tab: "details",
      section: "pricing",
      order: 2,
      sourceBadge: true,
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
      sourceBadge: true,
    },
    {
      key: "ratings.google_review_count",
      label: "Reviews",
      type: "integer",
      tab: "details",
      section: "ratings",
      order: 2,
      sourceBadge: true,
    },
  ],
};
```

### Registry utils

```typescript
// src/addons/venue-management/registry/registry.utils.ts

import type { VenueFormRegistry, FieldDefinition } from "./field-registry.types";
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
  registry: VenueFormRegistry,
  metadata: VenueMetadata,
  stage: ProfileStage,
): FieldDefinition[] {
  return registry.fields.filter((f) => {
    if (f.requiredForStage !== stage) return false;
    const v = resolveFieldValue(metadata, f.key);
    return v == null || v === "";
  });
}

export function groupByTabAndSection(registry: VenueFormRegistry) {
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
GET  /api/v1/venues/:id
// Create
POST /api/v1/venues
// Patch one metadata field → triggers aggregation + stage recalc
PATCH /api/v1/venues/:id/metadata   { fieldKey: string; value: unknown }
// Annotations
GET|POST        /api/v1/venues/:id/annotations
PATCH|DELETE    /api/v1/venues/:id/annotations/:annotationId

// src/shared/api/master-venue.ts  (admin app)

GET  /api/v1/admin/master-venues?city=&page=&size=
GET  /api/v1/admin/master-venues/:id
POST /api/v1/admin/master-venues
PUT  /api/v1/admin/master-venues/:id
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
> This means the prototype exercises the full component tree, the field registry rendering, the
> `AnyVenue` context switching, the `SourceBadge` popover, `VenueAnnotationsPanel`, and
> `VenueQuickFillBar` — exactly as they will behave in production. UI bugs, awkward interactions,
> and missing states are caught here, not after backend integration.

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
│       └── registry/
│           ├── field-registry.types.ts
│           ├── venue-field-registry.ts
│           └── registry.utils.ts
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
GET    /api/v1/venues/:id          → VenueDetail (full metadata + metadata_sources)
POST   /api/v1/venues              → create, return new VenueDetail
PATCH  /api/v1/venues/:id/metadata → update one field, recalculate profileStage
GET    /api/v1/venues/:id/annotations
POST   /api/v1/venues/:id/annotations
DELETE /api/v1/venues/:id/annotations/:annotationId
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

```
src/addons/venue-management/
  types.ts                 AnyVenue, VenueContext, VenueMetadata, VenueMetadataContacts,
                           FieldProvenance, ProfileStage, VenueStatus, VenueSource
                           AnnotationType, VenueAnnotation
                           PROFILE_STAGES, VENUE_STATUSES, VENUE_SOURCES  ← string literal
                           constants, used in filters and type guards

  registry/
    field-registry.types.ts  FieldType, FieldAction, EnumOption,
                             FieldDefinition, SectionDefinition, TabDefinition,
                             VenueFormRegistry

    venue-field-registry.ts  VENUE_FIELD_REGISTRY  ← the single source of truth
                             for tabs, sections, and fields

    registry.utils.ts        resolveFieldValue, setFieldValue,
                             missingFieldsForStage, groupByTabAndSection

  index.ts                 public barrel — re-exports everything above;
                           nothing else in the project imports addon internals directly

src/entities/venue/
  types.ts                 VenueSummary, VenueSummaryListResponse, VenueDetail
                           (imports AnyVenue + VenueMetadata from addon)
  index.ts                 re-exports types.ts

src/shared/api/
  venue.ts                 venueApi object (same shape as iamApi):
                             list, getById, create, patchMetadata,
                             listAnnotations, createAnnotation,
                             updateAnnotation, deleteAnnotation
                           ListVenuesParams interface
                           PatchMetadataRequest interface
```

### `foundation-ui-platform-admin`

```
src/addons/venue-management/
  ...                      identical copy of tenant app addon
                           (or shared workspace package in future)

src/entities/master-venue/
  types.ts                 MasterVenueSummary, MasterVenueSummaryListResponse,
                           MasterVenueDetail, MasterVenueAlias, MasterVenueExternal
                           (imports AnyVenue + VenueMetadata from addon)
  index.ts                 re-exports types.ts

src/shared/api/
  master-venue.ts          masterVenueApi object:
                             list, getById, create, update, dedupCheck
                           ListMasterVenuesParams interface
```

### Constant anchors (prevent magic strings)

These constants must be defined in `addon/types.ts` and imported wherever needed — no inline string literals in components or API calls:

```typescript
export const PROFILE_STAGES = ["SEEDED", "ENRICHED", "CURATED", "READY"] as const;
export const VENUE_STATUSES = ["DRAFT", "ACTIVE", "ARCHIVED"] as const;
export const VENUE_SOURCES = ["MANUAL", "FILE_IMPORT", "MASTER_CATALOG", "BULK_CSV"] as const;
export const VENUE_CONTEXTS = ["tenant", "master_catalog"] as const;
export const ANNOTATION_TYPES = ["NOTE", "TAG", "RATING", "BOOKMARK", "INTERNAL_FLAG"] as const;

export const CATERING_POLICIES = [
  "in_house_exclusive",
  "in_house_preferred",
  "outside_allowed",
  "no_catering",
] as const;

// Derived types from constants — single source of truth, no enum drift
export type ProfileStage = (typeof PROFILE_STAGES)[number];
export type VenueStatus = (typeof VENUE_STATUSES)[number];
export type VenueSource = (typeof VENUE_SOURCES)[number];
export type VenueContext = (typeof VENUE_CONTEXTS)[number];
export type AnnotationType = (typeof ANNOTATION_TYPES)[number];
export type CateringPolicy = (typeof CATERING_POLICIES)[number];
```

Type guards generated from the same constants — no manual maintenance:

```typescript
export const isProfileStage = (v: unknown): v is ProfileStage =>
  PROFILE_STAGES.includes(v as ProfileStage);
export const isVenueStatus = (v: unknown): v is VenueStatus =>
  VENUE_STATUSES.includes(v as VenueStatus);
export const isVenueContext = (v: unknown): v is VenueContext =>
  VENUE_CONTEXTS.includes(v as VenueContext);
```

---

**Docs:** [Architecture Overview](architecture-overview.md) · [Data Model](data-model.md) · [API](api.md) · [Search](search.md)
