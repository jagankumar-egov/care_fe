# Care Architecture Perspective

This document provides a deep analysis of the Care system's model hierarchy, plugin architecture, and extension mechanisms across both frontend (care_fe) and backend (care) systems.

---

## Table of Contents

1. [Model Hierarchy Overview](#model-hierarchy-overview)
2. [Base Models and Inheritance](#base-models-and-inheritance)
3. [Patient Model Extensions](#patient-model-extensions)
4. [Plugin Architecture](#plugin-architecture)
5. [Extension Mechanisms](#extension-mechanisms)
6. [Plugin Deployment](#plugin-deployment)
7. [Modifiable Behaviors](#modifiable-behaviors)

---

## Model Hierarchy Overview

### Frontend Type System (care_fe)

The frontend uses TypeScript interfaces that mirror the backend's FHIR-based models. Types follow a consistent pattern:

```
{Entity}Base          → Core fields (used for both create/read)
    ↓
{Entity}Create        → Fields for POST/PUT requests
{Entity}Update        → Fields for PATCH requests (often Partial<Create>)
    ↓
{Entity}ListRead      → List endpoint response (minimal relations)
    ↓
{Entity}Read          → Detail endpoint response (full relations + permissions)
```

### Backend Model System (care)

The backend uses Django models with FHIR compliance:

```
BaseModel                 → Audit fields (created_by, updated_by, timestamps)
    ↓
EMRBaseModel             → EMR-specific base (soft delete, facility context)
    ↓
Domain Models            → Patient, Encounter, Observation, etc.
```

---

## Base Models and Inheritance

### Frontend Base Types

#### 1. Code (Foundation for All Coded Concepts)

**File**: `src/types/base/code/code.ts`

```typescript
// Used by: Diagnosis, Observation, Medication, ServiceRequest, etc.
interface Code {
  system: string;    // Coding system URI (ICD-10, SNOMED, etc.)
  code: string;      // The code value
  display: string;   // Human-readable display
}

interface CodeConceptMinimal {
  code: string;
  display: string;
  system: string;
  designation: Designation[];
}
```

**Types inheriting Code pattern**:
| Type | Usage |
|------|-------|
| `Diagnosis.code` | ICD-10/SNOMED diagnosis codes |
| `Observation.main_code` | LOINC observation codes |
| `MedicationRequest.medication` | Drug codes |
| `ServiceRequest.code` | Procedure/service codes |
| `AllergyIntolerance.code` | Allergy codes |
| `FacilityRead.instance_discount_codes` | Discount type codes |
| `FacilityRead.instance_tax_codes` | Tax type codes |

#### 2. Period (Time Range)

```typescript
interface Period {
  start: string;  // ISO datetime
  end?: string;   // ISO datetime (optional for ongoing)
}
```

**Used by**: Encounter, Account, Schedule, Availability

#### 3. Permissions Mixin

```typescript
interface Permissions {
  permissions: string[];  // Array of permission strings
}

// Extended version for facilities
interface FacilityPermissions extends Permissions {
  root_org_permissions: string[];
  child_org_permissions: string[];
}
```

**Applied to**: `PatientRead`, `EncounterRead`, `FacilityRead`, `CurrentUserRead`

### Backend Base Models (Django - care repository)

#### BaseModel

```python
class BaseModel(models.Model):
    """Base model with audit fields"""
    id = models.UUIDField(primary_key=True, default=uuid4)
    external_id = models.UUIDField(unique=True, default=uuid4)
    created_date = models.DateTimeField(auto_now_add=True)
    modified_date = models.DateTimeField(auto_now=True)
    deleted = models.BooleanField(default=False)

    class Meta:
        abstract = True
```

#### EMRBaseModel

```python
class EMRBaseModel(BaseModel):
    """EMR base with facility context and soft delete"""
    created_by = models.ForeignKey(User, on_delete=models.PROTECT)
    updated_by = models.ForeignKey(User, on_delete=models.PROTECT)
    facility = models.ForeignKey(Facility, on_delete=models.CASCADE)

    class Meta:
        abstract = True
```

### Model Inheritance Map

```
┌─────────────────────────────────────────────────────────────────┐
│                    BACKEND (Django Models)                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   BaseModel (Abstract)                                           │
│   ├── id, external_id, created_date, modified_date, deleted     │
│   │                                                              │
│   ├── User                                                       │
│   ├── Facility                                                   │
│   ├── Organization                                               │
│   ├── PlugConfig                                                 │
│   │                                                              │
│   └── EMRBaseModel (Abstract)                                    │
│       ├── created_by, updated_by, facility                       │
│       │                                                          │
│       ├── Patient                                                │
│       │   └── PatientIdentifier                                  │
│       │                                                          │
│       ├── Encounter                                              │
│       │   ├── EncounterLocation                                  │
│       │   └── CareTeam                                           │
│       │                                                          │
│       ├── Observation                                            │
│       ├── Diagnosis (Condition)                                  │
│       ├── MedicationRequest                                      │
│       ├── MedicationStatement                                    │
│       ├── ServiceRequest                                         │
│       ├── AllergyIntolerance                                     │
│       ├── Prescription                                           │
│       ├── Consent                                                │
│       │                                                          │
│       └── Account (Billing)                                      │
│           ├── Invoice                                            │
│           ├── ChargeItem                                         │
│           └── PaymentReconciliation                              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                   FRONTEND (TypeScript Types)                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Base Types                                                     │
│   ├── Code (system, code, display)                              │
│   ├── Period (start, end)                                       │
│   ├── Permissions (permissions[])                               │
│   └── TagConfig (id, name, color)                               │
│                                                                  │
│   Entity Types (follow Base → ListRead → Read pattern)          │
│   ├── Patient (PatientBase → PatientListRead → PatientRead)    │
│   ├── Encounter (EncounterBase → EncounterListRead → Read)     │
│   ├── User (UserBase → UserReadMinimal → UserRead)             │
│   ├── Facility (FacilityBareMinimum → FacilityBase → Read)     │
│   ├── Device (DeviceBase → DeviceList → DeviceDetail)          │
│   └── Account (AccountBase → AccountList → AccountRead)        │
│                                                                  │
│   EMR Types (all reference Code base type)                      │
│   ├── Diagnosis                                                 │
│   ├── Observation                                               │
│   ├── MedicationRequest                                         │
│   ├── ServiceRequest                                            │
│   └── AllergyIntolerance                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Count of Types by Category

| Category | Count | Examples |
|----------|-------|----------|
| **Base Types** | 6 | Code, Period, Permissions, Duration, MonetaryComponent, QualifiedRange |
| **Patient-related** | 8 | PatientBase, PatientRead, PatientListRead, PatientCreate, PatientIdentifier, etc. |
| **Encounter-related** | 12 | EncounterBase, EncounterRead, CareTeam, LocationAssociation, etc. |
| **EMR Clinical** | 20+ | Diagnosis, Observation, MedicationRequest, Prescription, etc. |
| **Billing** | 15+ | Account, Invoice, ChargeItem, Payment, etc. |
| **Organization** | 8 | Organization, FacilityOrganization, Role, Permission |
| **User** | 6 | UserBase, UserRead, CurrentUserRead, etc. |
| **Device** | 5 | DeviceBase, DeviceDetail, DeviceAssociation |

---

## Patient Model Extensions

### Current Patient Type Structure

```typescript
// Base patient data
interface PatientBase {
  name: string;
  gender: GenderChoices;
  phone_number: string;
  emergency_phone_number?: string;
  address?: string;
  permanent_address?: string;
  pincode?: number;
  deceased_datetime?: string | null;
  blood_group?: BloodGroupChoices;
  date_of_birth?: string | null;
}

// List view
interface PatientListRead extends PatientBase {
  id: string;
  year_of_birth: number | null;
  created_date: string;
  modified_date: string;
  instance_tags: TagConfig[];      // Can be extended via tags
  facility_tags: TagConfig[];      // Facility-specific tags
}

// Full read with relations
interface PatientRead extends PatientListRead, Permissions {
  geo_organization?: Organization;
  created_by?: UserReadMinimal;
  updated_by?: UserReadMinimal;
  instance_identifiers: PatientIdentifier[];   // Extensible identifiers
  facility_identifiers: PatientIdentifier[];
}
```

### Ways to Extend Patient Model

#### 1. Patient Identifiers (Built-in Extension)

**Configuration**: Facility-level patient identifier types

```typescript
interface PatientIdentifierConfig {
  id: string;
  identifier_type: string;        // Custom identifier type
  identifier_type_display: string;
  min_length?: number;
  max_length?: number;
  validation_regex?: string;
}

interface PatientIdentifier {
  id: string;
  patient: string;
  identifier_type: PatientIdentifierConfig;
  value: string;
}
```

**Usage**: Add custom ID types like MRN, Insurance ID, National ID, etc.

#### 2. Tags System (Dynamic Extension)

```typescript
interface TagConfig {
  id: string;
  name: string;
  slug: string;
  color?: string;
}
```

**Usage**:
- `instance_tags`: Global patient tags
- `facility_tags`: Facility-specific patient categorization

#### 3. Plugin UI Extensions

Plugins can extend Patient UI through these injection points:

| Extension Point | What It Can Do |
|----------------|----------------|
| `PatientRegistrationForm` | Add custom fields to registration form |
| `PatientHomeActions` | Add action buttons on patient page |
| `PatientInfoCardActions` | Add actions to patient info card |
| `PatientInfoCardQuickActions` | Add quick action buttons |
| `PatientDetailsTabDemographyGeneralInfo` | Add custom sections to demographics |
| `PatientSearchActions` | Add custom search actions |

#### 4. Backend Extensions API (For Custom Fields)

**Extension System** (for dynamic fields):

```typescript
// Define extension schema
interface ExtensionConfig {
  name: string;
  owner: string;          // Plugin/module owner
  version: string;
  write_schema: JSONSchema;
  read_schema: JSONSchema;
}

// Supported entity types for extensions
enum ExtensionEntityType {
  account = "account",
  encounter = "encounter",
  patient = "patient",           // Patient extension support
  payment_reconciliation = "payment_reconciliation",
  supply_delivery = "supply_delivery",
  product = "product",
}
```

**Usage Flow**:
1. Register extension schema via backend API
2. Store custom data in extension fields
3. Frontend reads extension data alongside entity

---

## Plugin Architecture

### Overview

Care uses **Webpack Module Federation** (via Vite) for runtime plugin loading:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Plugin Architecture                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  Plugin A    │     │  Plugin B    │     │  Plugin C    │    │
│  │  (Remote)    │     │  (Remote)    │     │  (Remote)    │    │
│  └──────┬───────┘     └──────┬───────┘     └──────┬───────┘    │
│         │                    │                    │             │
│         └────────────────────┼────────────────────┘             │
│                              │                                   │
│                    Module Federation                             │
│                    (Runtime Loading)                             │
│                              │                                   │
│         ┌────────────────────┼────────────────────┐             │
│         │                    │                    │             │
│         ▼                    ▼                    ▼             │
│  ┌──────────────┐     ┌──────────────┐     ┌──────────────┐    │
│  │  Manifest    │     │  Components  │     │   Routes     │    │
│  │  Loading     │     │  Injection   │     │  Generation  │    │
│  └──────────────┘     └──────────────┘     └──────────────┘    │
│                              │                                   │
│                              ▼                                   │
│                    ┌──────────────────┐                         │
│                    │   Care Core App  │                         │
│                    │   (Host Module)  │                         │
│                    └──────────────────┘                         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Plugin Configuration

**Backend Storage**: `/api/v1/plug_config/`

```typescript
interface PlugConfig {
  slug: string;              // Unique plugin identifier
  meta: {
    url: string;             // Remote module URL
    name: string;            // Display name
    config?: {               // Plugin-specific configuration
      [key: string]: unknown;
    };
  };
}
```

### Plugin Manifest Structure

Each plugin must export a manifest from `./manifest`:

```typescript
interface PluginManifest {
  plugin: string;                    // Plugin identifier

  // Component Extensions (13 injection points)
  components?: {
    DoctorConnectButtons?: LazyComponent;
    Scribe?: LazyComponent;
    PatientHomeActions?: LazyComponent;
    PatientInfoCardQuickActions?: LazyComponent;
    EncounterActions?: LazyComponent;
    PatientInfoCardMarkAsComplete?: LazyComponent;
    FacilityHomeActions?: LazyComponent;
    PatientRegistrationForm?: LazyComponent;
    PatientDetailsTabDemographyGeneralInfo?: LazyComponent;
    InvoiceRecordPaymentOptions?: LazyComponent;
    PatientSearchActions?: LazyComponent;
    PatientInfoCardActions?: LazyComponent;
    ServiceRequestAction?: LazyComponent;
  };

  // Route Extensions
  routes?: Record<string, LazyComponent>;

  // Navigation Extensions
  navItems?: NavigationLink[];
  billingNavItems?: NavigationLink[];
  userNavItems?: NavigationLink[];
  adminNavItems?: NavigationLink[];

  // Tab Extensions
  organizationTabs?: PluginOrganizationTab[];
  encounterTabs?: Record<string, LazyComponent>;

  // Device Extensions
  devices?: PluginDeviceManifest[];

  // Behavior Extensions
  extends?: ("DoctorConnectButtons" | "PatientExternalRegistration")[];
}
```

### Plugin Loading Flow

```typescript
// 1. Fetch enabled plugins from backend
const { data: plugConfigs } = useQuery({
  queryKey: ["plugConfigs"],
  queryFn: query(plugConfigApi.list),
});

// 2. For each plugin, load manifest via Module Federation
const loadPlugin = async (config: PlugConfig) => {
  // Register remote module
  setFederationRemote(config.slug, {
    url: () => Promise.resolve(config.meta.url),
    format: "esm",
    from: "vite",
    externalType: "promise",
  });

  // Load manifest
  const module = await getFederationRemote(config.slug, "./manifest");
  return unwrapModule(module) as PluginManifest;
};

// 3. Provide to app via context
<CareAppsContext.Provider value={loadedPlugins}>
  <App />
</CareAppsContext.Provider>
```

### Component Injection Pattern

**PLUGIN_Component Usage**:

```tsx
// In PatientHome.tsx
<PLUGIN_Component
  __name="PatientHomeActions"
  patient={patientData}
  facilityId={facilityId}
/>

// PLUGIN_Component implementation
function PLUGIN_Component(props) {
  const careApps = useCareApps();

  return careApps.map(plugin => {
    const Component = plugin.components?.[props.__name];
    if (!Component) return null;

    return (
      <PluginErrorBoundary key={plugin.plugin}>
        <Suspense fallback={<Spinner />}>
          <Component {...props} __meta={plugin.meta} />
        </Suspense>
      </PluginErrorBoundary>
    );
  });
}
```

---

## Extension Mechanisms

### 1. Component Extensions (UI)

| Extension Point | Props Received | Use Case |
|----------------|----------------|----------|
| `PatientRegistrationForm` | `{ form: UseFormReturn }` | Add custom form fields |
| `PatientHomeActions` | `{ patient, facilityId }` | Add action buttons |
| `PatientInfoCardActions` | `{ facilityId, patient }` | Card-level actions |
| `PatientInfoCardQuickActions` | `{ encounter }` | Quick encounter actions |
| `PatientDetailsTabDemographyGeneralInfo` | `{ facilityId, patientId, patientData }` | Custom demographics |
| `EncounterActions` | `{ encounter }` | Encounter page actions |
| `FacilityHomeActions` | `{ facility }` | Facility dashboard actions |
| `DoctorConnectButtons` | `{ user }` | Communication buttons |
| `Scribe` | `{ formState }` | AI note-taking integration |
| `InvoiceRecordPaymentOptions` | `{ facilityId, invoice }` | Payment method options |
| `PatientSearchActions` | `{ facilityId }` | Search result actions |
| `ServiceRequestAction` | `{ serviceRequestId }` | Service request actions |

### 2. Navigation Extensions

```typescript
interface NavigationLink {
  title: string;
  url: string;
  icon?: string;
  requiredPermissions?: string[];
}

// Plugin can add to:
navItems: NavigationLink[];        // Main sidebar
billingNavItems: NavigationLink[]; // Billing section
userNavItems: NavigationLink[];    // User menu
adminNavItems: NavigationLink[];   // Admin section
```

### 3. Route Extensions

```typescript
// Plugin manifest
routes: {
  "/plugin/custom-page": lazy(() => import("./CustomPage")),
  "/plugin/settings": lazy(() => import("./PluginSettings")),
}

// Automatically merged into app router
```

### 4. Tab Extensions

**Encounter Tabs**:
```typescript
encounterTabs: {
  "vitals-monitor": lazy(() => import("./VitalsMonitor")),
  "ai-summary": lazy(() => import("./AISummary")),
}
```

**Organization Tabs**:
```typescript
organizationTabs: [
  {
    name: "Analytics",
    slug: "analytics",
    icon: <ChartIcon />,
    component: AnalyticsDashboard,
  }
]
```

### 5. Device Extensions

```typescript
interface PluginDeviceManifest {
  type: string;  // Matches device.care_type

  // Custom icon for device type
  icon?: React.FC;

  // Configuration form for device setup
  configureForm?: React.FC<{
    facilityId: string;
    metadata: Record<string, unknown>;
    onChange: (metadata: Record<string, unknown>) => void;
  }>;

  // Device display card
  showPageCard?: React.FC<{
    device: DeviceDetail;
    facilityId: string;
  }>;

  // Encounter integration
  encounterOverview?: React.FC<{
    encounter: EncounterRead;
  }>;
}
```

### 6. Data Extensions (Backend)

**Extension API**: `/api/v1/extensions/`

```typescript
// Register custom fields for entities
POST /api/v1/extensions/
{
  entity_type: "patient",
  name: "insurance_details",
  owner: "insurance-plugin",
  write_schema: { /* JSON Schema */ },
  read_schema: { /* JSON Schema */ }
}

// Store extension data
PATCH /api/v1/patient/{id}/
{
  extensions: {
    "insurance-plugin:insurance_details": {
      provider: "ABC Insurance",
      policy_number: "INS-12345"
    }
  }
}
```

---

## Plugin Deployment

### Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Production Deployment                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    CDN / Static Host                     │   │
│   │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     │   │
│   │  │  Care Core  │  │  Plugin A   │  │  Plugin B   │     │   │
│   │  │   Bundle    │  │   Bundle    │  │   Bundle    │     │   │
│   │  │  (main.js)  │  │ (remoteA.js)│  │(remoteB.js) │     │   │
│   │  └─────────────┘  └─────────────┘  └─────────────┘     │   │
│   └─────────────────────────────────────────────────────────┘   │
│                              │                                   │
│                              ▼                                   │
│   ┌─────────────────────────────────────────────────────────┐   │
│   │                    Care Backend API                      │   │
│   │                                                          │   │
│   │  /api/v1/plug_config/                                   │   │
│   │  ┌─────────────────────────────────────────────────┐   │   │
│   │  │ [                                                │   │   │
│   │  │   {                                              │   │   │
│   │  │     "slug": "plugin-a",                          │   │   │
│   │  │     "meta": {                                    │   │   │
│   │  │       "url": "https://cdn.example.com/plugin-a/",│   │   │
│   │  │       "name": "Plugin A"                         │   │   │
│   │  │     }                                            │   │   │
│   │  │   }                                              │   │   │
│   │  │ ]                                                │   │   │
│   │  └─────────────────────────────────────────────────┘   │   │
│   └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Plugin Build Requirements

**1. Vite Configuration for Plugin**:

```typescript
// vite.config.ts (plugin)
import { federation } from "@originjs/vite-plugin-federation";

export default defineConfig({
  plugins: [
    react(),
    federation({
      name: "plugin-name",
      filename: "remoteEntry.js",
      exposes: {
        "./manifest": "./src/manifest.ts",
      },
      shared: [
        "react",
        "react-dom",
        "react-i18next",
        "@tanstack/react-query",
        "raviger",
        "sonner",
        "decimal.js",
      ],
    }),
  ],
});
```

**2. Plugin Manifest Export**:

```typescript
// src/manifest.ts
import { lazy } from "react";
import type { PluginManifest } from "care-fe-types";

export const manifest: PluginManifest = {
  plugin: "my-plugin",

  components: {
    PatientHomeActions: lazy(() => import("./components/PatientActions")),
  },

  encounterTabs: {
    "my-custom-tab": lazy(() => import("./tabs/CustomTab")),
  },

  navItems: [
    {
      title: "My Feature",
      url: "/my-plugin/feature",
      icon: "star",
    },
  ],

  routes: {
    "/my-plugin/feature": lazy(() => import("./pages/FeaturePage")),
  },
};
```

### Deployment Steps

**1. Build Plugin**:
```bash
cd plugin-directory
npm run build
# Outputs: dist/remoteEntry.js + chunks
```

**2. Deploy to CDN**:
```bash
# Upload to any static hosting
aws s3 sync dist/ s3://plugins-bucket/my-plugin/
# Or: Deploy to Vercel, Netlify, CloudFlare Pages
```

**3. Register Plugin in Backend**:
```bash
# Via API
curl -X POST https://care-api/api/v1/plug_config/ \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "slug": "my-plugin",
    "meta": {
      "url": "https://cdn.example.com/my-plugin/remoteEntry.js",
      "name": "My Plugin",
      "config": {}
    }
  }'

# Or via Django admin
```

**4. Enable for Facilities** (if facility-specific):
```python
# Backend model can support facility-level plugin enabling
PlugConfig.objects.create(
    slug="my-plugin",
    meta={"url": "...", "name": "..."},
    facilities=[facility1, facility2]  # Optional scoping
)
```

### Shared Dependencies

Plugins share these dependencies with the host (loaded once):

| Package | Version | Purpose |
|---------|---------|---------|
| `react` | 19.x | UI Framework |
| `react-dom` | 19.x | DOM Rendering |
| `react-i18next` | Latest | Internationalization |
| `@tanstack/react-query` | 5.x | Data Fetching |
| `raviger` | Latest | Routing |
| `sonner` | Latest | Toast Notifications |
| `decimal.js` | Latest | Decimal Math |

### Environment Configuration

**Plugin receives config via props**:

```typescript
// Plugin component receives __meta
interface PluginProps<T> = T & {
  __meta: {
    url: string;
    name: string;
    config?: {
      apiEndpoint?: string;
      featureFlags?: Record<string, boolean>;
      // Custom plugin configuration
    };
  };
};

// Usage in plugin component
function MyPluginComponent({ patient, __meta }: PluginProps) {
  const apiUrl = __meta.config?.apiEndpoint;
  // Use plugin-specific configuration
}
```

---

## Modifiable Behaviors

### Summary of What Plugins Can Modify

| Category | Behavior | How |
|----------|----------|-----|
| **Patient UI** | Registration form fields | `PatientRegistrationForm` component |
| **Patient UI** | Home page actions | `PatientHomeActions` component |
| **Patient UI** | Demographics display | `PatientDetailsTabDemographyGeneralInfo` |
| **Patient UI** | Search result actions | `PatientSearchActions` component |
| **Patient UI** | Info card actions | `PatientInfoCardActions`, `PatientInfoCardQuickActions` |
| **Encounter UI** | Custom tabs | `encounterTabs` manifest entry |
| **Encounter UI** | Action buttons | `EncounterActions` component |
| **Encounter UI** | Completion logic | `PatientInfoCardMarkAsComplete` |
| **Facility UI** | Dashboard actions | `FacilityHomeActions` component |
| **Billing UI** | Payment options | `InvoiceRecordPaymentOptions` component |
| **Navigation** | Main menu items | `navItems` manifest entry |
| **Navigation** | Billing menu items | `billingNavItems` manifest entry |
| **Navigation** | User menu items | `userNavItems` manifest entry |
| **Navigation** | Admin menu items | `adminNavItems` manifest entry |
| **Routing** | Custom pages | `routes` manifest entry |
| **Organization** | Custom tabs | `organizationTabs` manifest entry |
| **Devices** | Custom device types | `devices` manifest entry |
| **Communication** | Doctor connect buttons | `DoctorConnectButtons` component |
| **Documentation** | AI scribe integration | `Scribe` component |
| **Service Requests** | Custom actions | `ServiceRequestAction` component |
| **Data** | Custom entity fields | Backend Extensions API |
| **External Auth** | Patient registration | `PatientExternalRegistration` extension |

### What Plugins CANNOT Modify

| Aspect | Reason |
|--------|--------|
| Core data models | Requires backend schema changes |
| Permission system | Security-critical, backend-controlled |
| Authentication flow | Security-critical, core functionality |
| API endpoints | Backend responsibility |
| Database schema | Requires migrations |
| Core business logic | Handled by backend |

### Extension Strategy by Use Case

| Use Case | Recommended Approach |
|----------|---------------------|
| Add patient fields | 1. Patient Identifiers, 2. Tags, 3. Extensions API |
| Custom patient workflow | Component injection + custom routes |
| Custom device type | Device manifest with configureForm |
| External integration | Custom routes + navItems |
| Analytics dashboard | Organization tabs + custom routes |
| Payment gateway | `InvoiceRecordPaymentOptions` |
| AI/ML features | `Scribe` component + custom tabs |
| Communication tools | `DoctorConnectButtons` |
| Custom reports | Custom routes + encounter tabs |

---

## Quick Reference

### Files to Study

| Topic | File Path |
|-------|-----------|
| Plugin Engine | `src/PluginEngine.tsx` |
| Plugin Types | `src/pluginTypes.ts` |
| Plugin Hooks | `src/hooks/useCareApps.tsx` |
| Plugin Config API | `src/types/plugConfig/plugConfigApi.ts` |
| Extensions Types | `src/types/extensions/extensions.ts` |
| Patient Types | `src/types/emr/patient/patient.ts` |
| Encounter Types | `src/types/emr/encounter/encounter.ts` |
| Base Types | `src/types/base/` |

### API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /api/v1/plug_config/` | List enabled plugins |
| `POST /api/v1/plug_config/` | Register new plugin |
| `GET /api/v1/extensions/` | List extension schemas |
| `POST /api/v1/extensions/` | Register extension schema |

---

*Last Updated: February 2025*
