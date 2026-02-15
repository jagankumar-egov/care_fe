# Care Frontend Plugin Customization Guide

This document details all customization capabilities available to external plugins in the Care frontend system.

---

## Table of Contents

1. [Overview](#overview)
2. [Model/Data Extensions](#modeldata-extensions)
3. [Page Overrides](#page-overrides)
4. [Component Injection Points](#component-injection-points)
5. [Form Customization](#form-customization)
6. [Navigation Customization](#navigation-customization)
7. [Route Registration](#route-registration)
8. [Device Extensions](#device-extensions)
9. [Plugin Manifest Reference](#plugin-manifest-reference)

---

## Overview

Care Frontend uses a **Module Federation** based plugin system that allows external plugins to:

- Add custom data fields to entities
- Inject UI components at specific points
- Add custom pages/routes
- Extend navigation menus
- Add custom encounter tabs
- Define custom device types

### Plugin Loading Flow

```
Backend API (/api/v1/plug_config/)
         ↓
    PluginEngine.tsx
         ↓
    Module Federation (remoteEntry.js)
         ↓
    Plugin Manifest (./manifest)
         ↓
    Components, Routes, Navigation
```

---

## Model/Data Extensions

### Supported Entities for Custom Fields

| Entity | Extension Support | API Endpoint |
|--------|------------------|--------------|
| `patient` | Yes | `/api/v1/extensions/` |
| `encounter` | Yes | `/api/v1/extensions/` |
| `account` | Yes | `/api/v1/extensions/` |
| `payment_reconciliation` | Yes | `/api/v1/extensions/` |
| `supply_delivery` | Yes | `/api/v1/extensions/` |
| `supply_delivery_order` | Yes | `/api/v1/extensions/` |
| `product` | Yes | `/api/v1/extensions/` |

### Extension Schema Definition

Extensions use JSON Schema to define custom fields:

```typescript
interface ExtensionConfig {
  name: string;              // Extension identifier (e.g., "insurance_details")
  owner: string;             // Owner/plugin name
  version: string;           // Schema version
  write_schema: JSONSchema;  // Schema for create/update forms
  read_schema: JSONSchema;   // Schema for display
  retrieve_schema: JSONSchema; // Schema for API retrieval
}
```

### JSON Schema Example

```json
{
  "type": "object",
  "properties": {
    "insurance_provider": {
      "type": "string",
      "title": "Insurance Provider",
      "x-ui": {
        "control": "autocomplete",
        "metadata": {
          "searchApi": "/api/v1/insurance/providers/"
        }
      }
    },
    "policy_number": {
      "type": "string",
      "title": "Policy Number",
      "minLength": 5,
      "maxLength": 20
    },
    "coverage_type": {
      "type": "string",
      "enum": ["basic", "premium", "comprehensive"],
      "title": "Coverage Type",
      "x-ui": {
        "control": "radio"
      }
    }
  },
  "required": ["insurance_provider", "policy_number"]
}
```

### Supported UI Controls

| Control | JSON Schema | Description |
|---------|-------------|-------------|
| Text Input | `type: "string"` | Standard text field |
| Number | `type: "number"` or `type: "integer"` | Numeric input with min/max |
| Textarea | `x-ui: { control: "textarea" }` | Multi-line text |
| Select | `enum: [...]` | Dropdown selection |
| Radio | `x-ui: { control: "radio" }` | Radio button group |
| Checkbox | `type: "boolean"` | Boolean checkbox |
| Date | `format: "date"` | Date picker |
| DateTime | `format: "date-time"` | DateTime picker |
| Time | `format: "time"` | Time picker |
| Email | `format: "email"` | Email input |
| URL | `format: "uri"` | URL input |
| Autocomplete | `x-ui: { control: "autocomplete" }` | Searchable dropdown |
| Array/Table | `type: "array"` | Dynamic list of items |
| Section | `x-ui: { control: "section" }` | Grouped fields |
| Grid | `x-ui: { control: "grid" }` | Grid layout |

### Conditional Fields

Extensions support conditional visibility using JSON Schema `if/then/else`:

```json
{
  "if": {
    "properties": {
      "has_insurance": { "const": true }
    }
  },
  "then": {
    "required": ["policy_number"],
    "properties": {
      "policy_number": { "x-ui": { "visible": true } }
    }
  },
  "else": {
    "properties": {
      "policy_number": { "x-ui": { "visible": false } }
    }
  }
}
```

### Where Extensions Are Used

| File | Entity | Usage |
|------|--------|-------|
| `src/pages/Facility/billing/account/AccountSheet.tsx` | Account | Account creation/edit form |
| `src/pages/Facility/billing/PaymentReconciliationSheet.tsx` | Payment | Payment reconciliation |
| `src/pages/Facility/services/inventory/externalSupply/deliveryOrder/DeliveryOrderForm.tsx` | Supply Delivery | Delivery order forms |

---

## Page Overrides

### Complete Page Replacement

Plugins can add entirely new pages via the `routes` manifest property:

```typescript
// In plugin manifest.tsx
const manifest = {
  plugin: "my_plugin",
  routes: {
    "/my-plugin/dashboard": () => <MyDashboard />,
    "/my-plugin/settings": () => <MySettings />,
    "/facility/:facilityId/my-feature": ({ facilityId }) => (
      <MyFeaturePage facilityId={facilityId} />
    ),
  },
};
```

### Route Parameter Access

Routes receive URL parameters as function arguments:

```typescript
routes: {
  "/facility/:facilityId/patient/:patientId/custom": ({ facilityId, patientId }) => (
    <CustomPage facilityId={facilityId} patientId={patientId} />
  ),
}
```

### Route Priority

Plugin routes are merged with core routes. Core routes take precedence for conflicts:

```typescript
// In AppRouter.tsx
routes = {
  ...pluginRoutes,      // Plugin routes (lower priority)
  ...organizationRoutes,
  ...Routes,            // Core routes (higher priority)
  ...AdminRouter,
};
```

---

## Component Injection Points

### Available Injection Points

| Component Name | Location | Props Received | Use Case |
|----------------|----------|----------------|----------|
| `PatientHomeActions` | Patient home page | `patient`, `facilityId`, `className` | Add action buttons |
| `PatientRegistrationForm` | Patient registration | `form`, `facilityId`, `patientId` | Add custom form fields |
| `PatientDetailsTabDemographyGeneralInfo` | Patient demographics tab | `facilityId`, `patientId`, `patientData` | Add demographic sections |
| `PatientInfoCardActions` | Patient info card | `facilityId`, `patient`, `className` | Card-level actions |
| `PatientInfoCardQuickActions` | Encounter patient card | `encounter`, `className` | Quick action buttons |
| `PatientInfoCardMarkAsComplete` | Mark complete dialog | `encounter` | Custom completion logic |
| `PatientSearchActions` | Patient search | `facilityId`, `className` | Search result actions |
| `EncounterActions` | Encounter command dialog | `encounter`, `className` | Encounter actions |
| `FacilityHomeActions` | Facility home page | `facility`, `className` | Facility dashboard actions |
| `DoctorConnectButtons` | Doctor connect UI | `user` | Communication buttons |
| `Scribe` | Questionnaire form | `formState`, `setFormState` | AI scribe integration |
| `InvoiceRecordPaymentOptions` | Invoice payment | `facilityId`, `invoice` | Custom payment methods |
| `ServiceRequestAction` | Service request | `serviceRequestId` | Service request actions |

### Usage in Plugin

```typescript
// Plugin manifest.tsx
import { lazy } from "react";

const manifest = {
  plugin: "my_plugin",
  components: {
    PatientHomeActions: lazy(() => import("./components/PatientActions")),
    FacilityHomeActions: lazy(() => import("./components/FacilityActions")),
    Scribe: lazy(() => import("./components/MyScribe")),
  },
};
```

### Component Implementation

```typescript
// components/PatientActions.tsx
import { PatientRead } from "care-fe-types";

interface Props {
  patient: PatientRead;
  facilityId?: string;
  className?: string;
  __meta?: {
    url?: string;
    name?: string;
    config?: Record<string, unknown>;
  };
}

export default function PatientActions({ patient, facilityId, __meta }: Props) {
  const handleCustomAction = () => {
    // Access plugin config
    const apiEndpoint = __meta?.config?.apiEndpoint;
    // Perform action
  };

  return (
    <button onClick={handleCustomAction}>
      Custom Action for {patient.name}
    </button>
  );
}
```

### Injection Point Locations in Code

| Injection Point | File Location |
|-----------------|---------------|
| `PatientHomeActions` | `src/components/Patient/PatientHome.tsx` |
| `PatientRegistrationForm` | `src/components/Patient/PatientRegistration.tsx` |
| `PatientDetailsTabDemographyGeneralInfo` | `src/components/Patient/PatientDetailsTab/Demography.tsx` |
| `EncounterActions` | `src/components/Encounter/EncounterCommandDialog.tsx` |
| `FacilityHomeActions` | `src/components/Facility/FacilityHome.tsx` |
| `Scribe` | `src/components/Questionnaire/QuestionnaireForm.tsx` |
| `InvoiceRecordPaymentOptions` | `src/pages/Facility/billing/invoice/InvoiceShow.tsx` |
| `PatientInfoCardMarkAsComplete` | `src/pages/Encounters/MarkEncounterAsCompletedDialog.tsx` |

---

## Form Customization

### Scribe Component (Questionnaire Override)

The `Scribe` component receives full access to questionnaire form state:

```typescript
interface ScribeProps {
  formState: QuestionnaireFormState[];
  setFormState: React.Dispatch<React.SetStateAction<QuestionnaireFormState[]>>;
}

// In your plugin
export default function MyScribe({ formState, setFormState }: ScribeProps) {
  const handleAIFill = async () => {
    // AI-generated responses
    const aiResponses = await generateResponses(formState);

    // Update form state
    setFormState(prevState =>
      prevState.map(item => ({
        ...item,
        values: aiResponses[item.id] || item.values,
      }))
    );
  };

  return (
    <button onClick={handleAIFill}>
      Fill with AI
    </button>
  );
}
```

### Patient Registration Form Extension

```typescript
interface PatientRegistrationFormProps {
  form: UseFormReturn<PatientFormValues>;
  facilityId?: string;
  patientId?: string;
}

export default function MyPatientFields({ form }: PatientRegistrationFormProps) {
  return (
    <div className="space-y-4">
      <FormField
        control={form.control}
        name="customField1"
        render={({ field }) => (
          <FormItem>
            <FormLabel>Custom Field</FormLabel>
            <FormControl>
              <Input {...field} />
            </FormControl>
          </FormItem>
        )}
      />
    </div>
  );
}
```

---

## Navigation Customization

### Navigation Item Types

```typescript
interface NavigationLink {
  url: string;
  name: string;
  icon?: React.ReactNode;
  children?: NavigationLink[];
}
```

### Available Navigation Slots

| Slot | Location | Manifest Property |
|------|----------|-------------------|
| Main Sidebar | Facility navigation | `navItems` |
| Billing Section | Billing module | `billingNavItems` |
| User Menu | User dropdown | `userNavItems` |
| Admin Section | Admin sidebar | `adminNavItems` |

### Navigation Example

```typescript
const manifest = {
  plugin: "my_plugin",

  // Main navigation
  navItems: [
    {
      name: "My Feature",
      url: "/my-plugin/feature",
      icon: <MyIcon />,
    },
  ],

  // Admin navigation with children
  adminNavItems: [
    {
      name: "My Plugin Admin",
      url: "/admin/my-plugin",
      icon: <AdminIcon />,
      children: [
        { name: "Settings", url: "/admin/my-plugin/settings" },
        { name: "Reports", url: "/admin/my-plugin/reports" },
      ],
    },
  ],

  // User menu items (appended to user profile dropdown)
  userNavItems: [
    {
      name: "my-history",  // Translation key
      url: "my-history",   // Relative to user profile URL
      icon: <HistoryIcon />,
    },
  ],
};
```

### How Navigation Is Rendered

**Admin Navigation** (`src/components/ui/sidebar/admin-nav.tsx`):
```typescript
const careApps = useCareApps();
const pluginNavItems = careApps.flatMap((c) =>
  !c.isLoading && c.adminNavItems ? c.adminNavItems : [],
);
// Merged with core admin links
```

**User Navigation** (`src/components/ui/sidebar/nav-user.tsx`):
```typescript
const pluginNavItems = careApps.flatMap((c) =>
  !c.isLoading && c.userNavItems ? c.userNavItems : [],
);
// Rendered in user dropdown menu
```

---

## Route Registration

### Standard Routes

```typescript
const manifest = {
  routes: {
    // Static route
    "/my-plugin": () => <HomePage />,

    // Route with parameters
    "/my-plugin/:id": ({ id }) => <DetailPage id={id} />,

    // Nested facility route
    "/facility/:facilityId/my-plugin": ({ facilityId }) => (
      <FacilityFeature facilityId={facilityId} />
    ),

    // Multi-parameter route
    "/facility/:facilityId/patient/:patientId/my-feature": ({ facilityId, patientId }) => (
      <PatientFeature facilityId={facilityId} patientId={patientId} />
    ),
  },
};
```

### Organization Tabs

Add tabs to organization pages:

```typescript
const manifest = {
  organizationTabs: [
    {
      name: "Analytics",
      slug: "analytics",
      icon: <ChartIcon />,
      component: lazy(() => import("./pages/OrgAnalytics")),
    },
  ],
};
```

This creates routes:
- `/organization/:id/analytics`
- `/organization/:navOrganizationId/children/:id/analytics`

### Encounter Tabs

Add custom tabs to encounter detail pages:

```typescript
const manifest = {
  encounterTabs: {
    "vitals-monitor": lazy(() => import("./tabs/VitalsMonitor")),
    "ai-summary": lazy(() => import("./tabs/AISummary")),
  },
};
```

Encounter tab components receive:
```typescript
interface PluginEncounterTabProps {
  encounter: EncounterRead;
  patient: PatientRead;
}
```

---

## Device Extensions

### Device Manifest Structure

```typescript
interface PluginDeviceManifest {
  type: string;  // Matches device.care_type

  // Custom icon for device list
  icon?: React.FC<React.HTMLAttributes<HTMLElement>>;

  // Configuration form when adding/editing device
  configureForm?: React.FC<{
    facilityId: string;
    metadata: Record<string, unknown>;
    onChange: (metadata: Record<string, unknown>) => void;
  }>;

  // Card displayed on device detail page
  showPageCard?: React.FC<{
    device: DeviceDetail;
    facilityId: string;
  }>;

  // Widget in encounter overview
  encounterOverview?: React.FC<{
    encounter: EncounterRead;
  }>;
}
```

### Device Plugin Example

```typescript
const manifest = {
  devices: [
    {
      type: "ventilator",
      icon: VentilatorIcon,
      configureForm: lazy(() => import("./devices/VentilatorConfig")),
      showPageCard: lazy(() => import("./devices/VentilatorCard")),
      encounterOverview: lazy(() => import("./devices/VentilatorOverview")),
    },
    {
      type: "ecg_monitor",
      icon: ECGIcon,
      configureForm: lazy(() => import("./devices/ECGConfig")),
      showPageCard: lazy(() => import("./devices/ECGCard")),
    },
  ],
};
```

---

## Plugin Manifest Reference

### Complete Manifest Structure

```typescript
interface PluginManifest {
  // Required: Plugin identifier
  plugin: string;

  // Custom routes
  routes?: Record<string, (params: any) => React.ReactNode>;

  // Extension capabilities
  extends?: ("DoctorConnectButtons" | "PatientExternalRegistration")[];

  // Component injections
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

  // Navigation
  navItems?: NavigationLink[];
  billingNavItems?: NavigationLink[];
  userNavItems?: NavigationLink[];
  adminNavItems?: NavigationLink[];

  // Tabs
  organizationTabs?: PluginOrganizationTab[];
  encounterTabs?: Record<string, LazyComponent>;

  // Devices
  devices?: PluginDeviceManifest[];
}
```

### Minimal Plugin Example

```typescript
// src/manifest.tsx
import { lazy } from "react";

const manifest = {
  plugin: "my_plugin",

  routes: {
    "/my-plugin": () => <div>My Plugin Home</div>,
  },

  adminNavItems: [
    {
      name: "My Plugin",
      url: "/my-plugin",
      icon: null,
    },
  ],
};

export default manifest;
```

### Full-Featured Plugin Example

```typescript
// src/manifest.tsx
import { lazy, Suspense } from "react";

const Dashboard = lazy(() => import("./pages/Dashboard"));
const Settings = lazy(() => import("./pages/Settings"));
const PatientWidget = lazy(() => import("./components/PatientWidget"));
const ScribeIntegration = lazy(() => import("./components/Scribe"));
const VitalsTab = lazy(() => import("./tabs/VitalsTab"));

function PageWrapper({ children }) {
  return (
    <Suspense fallback={<div>Loading...</div>}>
      {children}
    </Suspense>
  );
}

const manifest = {
  plugin: "comprehensive_plugin",

  routes: {
    "/my-plugin": () => <PageWrapper><Dashboard /></PageWrapper>,
    "/my-plugin/settings": () => <PageWrapper><Settings /></PageWrapper>,
    "/facility/:facilityId/my-plugin": ({ facilityId }) => (
      <PageWrapper><Dashboard facilityId={facilityId} /></PageWrapper>
    ),
  },

  extends: ["DoctorConnectButtons"],

  components: {
    PatientHomeActions: PatientWidget,
    Scribe: ScribeIntegration,
  },

  navItems: [
    {
      name: "My Plugin",
      url: "/my-plugin",
      icon: null,
    },
  ],

  adminNavItems: [
    {
      name: "Plugin Admin",
      url: "/my-plugin/settings",
      icon: null,
      children: [
        { name: "Settings", url: "/my-plugin/settings" },
      ],
    },
  ],

  encounterTabs: {
    "vitals-monitor": VitalsTab,
  },

  devices: [
    {
      type: "custom_device",
      icon: null,
      configureForm: lazy(() => import("./devices/Config")),
      showPageCard: lazy(() => import("./devices/Card")),
    },
  ],
};

export default manifest;
```

---

## Plugin Runtime Access

### Accessing Plugin Metadata

```typescript
// Available globally
window.__CARE_PLUGIN_RUNTIME__ = {
  meta: {
    [pluginSlug]: {
      url?: string;
      name?: string;
      config?: {
        apiEndpoint?: string;
        featureFlags?: Record<string, boolean>;
        // Custom configuration
      };
    }
  }
};

// In component via props
function MyComponent({ __meta }) {
  const { config } = __meta;
  // Use plugin-specific configuration
}
```

### Using Care Apps Hook

```typescript
import { useCareApps } from "@/hooks/useCareApps";

function MyComponent() {
  const careApps = useCareApps();

  // Access all loaded plugins
  careApps.forEach(plugin => {
    if (!plugin.isLoading) {
      console.log(plugin.plugin, plugin.meta);
    }
  });
}
```

---

## Summary: What Can Be Customized

| Category | Capability | Method |
|----------|-----------|--------|
| **Data Models** | Add custom fields to 7 entities | Extension Schema API |
| **Pages** | Add new pages/routes | `routes` in manifest |
| **Components** | Inject at 13 specific points | `components` in manifest |
| **Forms** | Add fields to patient registration | `PatientRegistrationForm` component |
| **Forms** | Override questionnaire behavior | `Scribe` component |
| **Navigation** | Add to 4 navigation areas | `*NavItems` in manifest |
| **Encounter** | Add custom tabs | `encounterTabs` in manifest |
| **Organization** | Add custom tabs | `organizationTabs` in manifest |
| **Devices** | Define custom device types | `devices` in manifest |
| **Payments** | Add payment methods | `InvoiceRecordPaymentOptions` component |

---

## Limitations

| What | Limitation |
|------|-----------|
| Core Models | Cannot modify core model structure (only extend via extensions) |
| Core Components | Cannot replace core components, only inject alongside |
| Authentication | Cannot override authentication flow |
| Permissions | Cannot modify permission system |
| API Endpoints | Cannot intercept/modify core API calls |
| Database Schema | Cannot modify database (backend responsibility) |

---

*Last Updated: February 2025*
