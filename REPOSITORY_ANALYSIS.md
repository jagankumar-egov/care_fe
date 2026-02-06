# Care Frontend - Repository Analysis

> A comprehensive analysis of the Care frontend codebase covering architecture, dependencies, code practices, and more.

---

## Table of Contents

- [1. Architecture Overview](#1-architecture-overview)
- [2. External Package Dependencies](#2-external-package-dependencies)
- [3. Code Practices Assessment](#3-code-practices-assessment)
- [4. Authentication System](#4-authentication-system)
- [5. Reusable Components & Utilities](#5-reusable-components--utilities)
- [6. Documentation Quality](#6-documentation-quality)
- [7. Build Process & Tools](#7-build-process--tools)
- [8. Styling & Theming](#8-styling--theming)
- [9. Summary Scorecard](#9-summary-scorecard)

---

## 1. Architecture Overview

### Folder Structure

```
src/
├── App.tsx                    # Main app with provider hierarchy
├── index.tsx                  # Entry point (i18n, Sentry, service worker)
├── CAREUI/                    # Custom UI component library
├── Routers/                   # Route definitions (raviger-based)
│   ├── AppRouter.tsx          # Authenticated routes
│   ├── PublicRouter.tsx       # Login/public routes
│   ├── PatientRouter.tsx      # Patient OTP routes
│   └── routes/                # Feature-based route modules
├── Providers/                 # Context providers (Auth, History, Patient)
├── components/                # Feature-organized components
│   ├── ui/                    # 88 Radix UI-based primitives
│   ├── Common/                # 57 shared utilities
│   └── [Feature]/             # Domain components (Patient, Facility, etc.)
├── pages/                     # Page components by feature
├── hooks/                     # 33 custom hooks
├── atoms/                     # Jotai state atoms
├── types/                     # 32 domain-organized type folders
├── Utils/                     # Utilities including request layer
├── context/                   # React contexts (Permission, Shortcut)
├── Locale/                    # i18n configuration
└── config/                    # App configuration
```

### State Management (Multi-layered)

| Layer | Technology | Purpose |
|-------|------------|---------|
| Server State | TanStack Query v5 | API data, caching, pagination |
| Global State | Jotai | User, developer mode, nav state |
| Auth State | React Context | User session, tokens |
| Local State | React useState | Component-specific |

### Routing

- **Library**: Raviger (lightweight, ~3KB)
- **Pattern**: Feature-based route modules merged in AppRouter
- **Type-safe**: Route params extracted via TypeScript generics

### Provider Hierarchy

```
QueryClientProvider
└── Suspense
    └── PubSubProvider
        └── ShortcutProvider
            └── PluginEngine
                └── HistoryAPIProvider
                    └── AuthUserProvider
                        ├── PublicRouter (unauthorized)
                        ├── PatientRouter (OTP auth)
                        └── AppRouter (authenticated)
```

---

## 2. External Package Dependencies

### Core Framework

| Package | Version | Purpose |
|---------|---------|---------|
| `react` | 19.1.1 | UI framework (latest) |
| `typescript` | 5.6.3 | Type safety |
| `vite` | 6.0.0 | Build tooling |

### UI & Components

| Package | Purpose |
|---------|---------|
| `@radix-ui/*` (18 packages) | Accessible, unstyled primitives |
| `tailwindcss` 4.1.3 | Utility-first CSS |
| `lucide-react` | Icon library (548 icons) |
| `class-variance-authority` | Component variant management |
| `framer-motion` | Animations |
| `sonner` | Toast notifications |
| `vaul` | Drawer component |
| `cmdk` | Command palette |

### Data & State

| Package | Purpose |
|---------|---------|
| `@tanstack/react-query` 5.64 | Server state, caching |
| `jotai` | Atomic global state |
| `react-hook-form` 7.55 | Form management |
| `zod` 3.23 | Schema validation |

### Utilities

| Package | Purpose |
|---------|---------|
| `date-fns` / `dayjs` | Date manipulation |
| `decimal.js` | Financial calculations |
| `i18next` | Internationalization (6 languages) |
| `raviger` | Routing |
| `markdown-it` | Markdown parsing |

### Specialized

| Package | Purpose |
|---------|---------|
| `react-pdf` / `jspdf` | PDF viewing/generation |
| `react-easy-crop` | Image cropping |
| `pigeon-maps` | Map display |
| `recharts` | Charts/graphs |
| `@excalidraw/excalidraw` | Drawing/diagrams |
| `qrcode.react` | QR code generation |
| `react-webcam` | Camera capture |
| `@yudiel/react-qr-scanner` | QR code scanning |

---

## 3. Code Practices Assessment

### Strengths

| Practice | Implementation |
|----------|----------------|
| **TypeScript Strict Mode** | Enabled with no implicit any |
| **Path Aliases** | `@/*` for clean imports |
| **Import Order** | Enforced: 3rd-party → library → CAREUI → UI → components → hooks → utils |
| **Naming Conventions** | PascalCase components, camelCase functions |
| **Component Organization** | Feature-based with clear separation |
| **Error Handling** | Centralized with session expiry detection |
| **Type-safe API Layer** | Generic routes with TypeScript inference |

### Code Quality Tools

```
ESLint 9 (flat config) + TypeScript rules
Prettier (double quotes, 2-space, semicolons)
Husky + lint-staged (pre-commit hooks)
unimported (unused import detection)
```

### ESLint Configuration Highlights

- **TypeScript**: `@typescript-eslint/eslint-plugin` with strict rules
- **React**: React 19 + React Hooks rules
- **Custom Rules**:
  - No relative import paths (enforces `@/` alias)
  - i18next literal string detection
  - Undefined translation key checking

### Prettier Configuration

```json
{
  "singleQuote": false,
  "useTabs": false,
  "tabWidth": 2,
  "semi": true,
  "arrowParens": "always"
}
```

### Areas for Improvement

- Some deprecated utilities still present
- Variable documentation levels across modules
- No Storybook for component documentation

---

## 4. Authentication System

### Architecture

```
AuthUserProvider
├── Login: useMutation(authApi.login)
├── MFA: useMutation(authApi.mfa.login)
├── Token Refresh: useQuery with refetchInterval
├── Logout: Clear tokens + redirect
└── Multi-tab sync: Storage event listener
```

### Token Management

| Token | Storage Key | Purpose |
|-------|-------------|---------|
| Access Token | `care_access_token` | API authentication (Bearer) |
| Refresh Token | `care_refresh_token` | Token renewal |
| Patient Token | `care_patient_token` | Patient OTP auth |

### Security Features

- **MFA Support**: TOTP (6-digit) + backup codes (8-digit)
- **Session Expiry**: Auto-redirect to `/session-expired`
- **Multi-tab Logout**: Storage event synchronization
- **Permission System**: Role-based via `PermissionContext`
- **Token Refresh**: Automatic background refresh with configurable interval

### Protected Routes Pattern

```tsx
// App.tsx
<AuthUserProvider
  unauthorized={<PublicRouter />}    // Login pages
  otpAuthorized={<PatientRouter />}  // Patient portal
>
  <AppRouter />                       // Full app (requires auth)
</AuthUserProvider>
```

### API Authentication

```typescript
// Authorization header automatically injected
headers.set("Authorization", `Bearer ${accessToken}`);
```

### Default Login Credentials (Local Development)

| Role | Username | Password |
|------|----------|----------|
| Administrator | `administrator_2_0` | `Coronasafe@123` |
| Doctor | `doctor_2_0` | `Coronasafe@123` |
| Nurse | `nurse_2_0` | `Coronasafe@123` |
| Staff | `staff_2_0` | `Coronasafe@123` |
| Facility Admin | `facility_admin_2_0` | `Coronasafe@123` |
| Volunteer | `volunteer_2_0` | `Coronasafe@123` |

---

## 5. Reusable Components & Utilities

### Component Layers

| Layer | Count | Purpose |
|-------|-------|---------|
| `components/ui/` | 88 files | Radix UI primitives (Button, Dialog, Form, etc.) |
| `components/Common/` | 57 files | Shared utilities (Avatar, Pagination, Search) |
| `CAREUI/` | 15 files | CARE-specific (Icons, Callout, Calendar) |
| Feature components | 200+ | Domain-specific (Patient, Facility, Medicine) |

### Key UI Components (`components/ui/`)

#### Form Components
- `input`, `textarea`, `checkbox`, `radio-group`, `switch`
- `select`, `autocomplete`, `phone-input`, `date-picker`
- `form` (React Hook Form integration)

#### Layout Components
- `card`, `separator`, `tabs`, `accordion`
- `collapsible`, `resizable-panels`

#### Overlay Components
- `dialog`, `drawer`, `alert-dialog`
- `popover`, `hover-card`, `tooltip`
- `dropdown-menu`, `context-menu`

#### Data Display
- `table`, `pagination`, `badge`, `avatar`
- `progress`, `skeleton`, `calendar`

### CAREUI Components

```
CAREUI/
├── display/
│   ├── Callout.tsx         # 5 variants: primary, secondary, warning, alert, danger
│   ├── ColoredIndicator.tsx
│   └── FilterBadge.tsx
├── interactive/
│   ├── Calendar.tsx
│   ├── WeekdayCheckbox.tsx
│   └── Zoom.tsx
├── icons/
│   ├── CareIcon.tsx        # 2000+ Unicon icons
│   ├── CustomIcons.tsx     # Custom SVG definitions
│   └── UniconPaths.json    # Icon path data
└── misc/
    └── PrintPreview.tsx
```

### Custom Hooks (33 total)

#### Data Management
| Hook | Purpose |
|------|---------|
| `useFilters` | Filter/pagination with query string sync |
| `useFileManager` | File operations (view, download, archive) |
| `useFileUpload` | Multi-file upload with compression |
| `useReportManager` | Report file management |
| `useLocationManagement` | Facility location CRUD |

#### Forms & Validation
| Hook | Purpose |
|------|---------|
| `useExtensions` | Dynamic extension field handling |
| `useConditionalFields` | Conditional field visibility |
| `useExtensionSchemas` | Extension schema management |

#### Input & Interaction
| Hook | Purpose |
|------|---------|
| `useBarcodeScanner` | External barcode scanner detection |
| `useKeyboardShortcuts` | Advanced keyboard shortcut handling |
| `useDragAndDrop` | Drag and drop events |

#### UI & Display
| Hook | Purpose |
|------|---------|
| `useBreakpoints` | Responsive breakpoint detection |
| `useMobile` | Mobile device detection |
| `useSidebarState` | Sidebar open/closed state |
| `useTimer` | Timer/countdown management |

#### Authentication
| Hook | Purpose |
|------|---------|
| `useAuthUser` | Current authenticated user |
| `usePatientUser` | Patient user context |
| `usePatientSignOut` | Patient logout handler |

### API Utilities (`Utils/request/`)

#### Query Functions

```typescript
// Standard query
useQuery({
  queryKey: ["patient", id],
  queryFn: query(patientApi.get, { pathParams: { id } }),
});

// Debounced query (for search)
queryFn: query.debounced(patientApi.search, {
  debounceInterval: 500,
  queryParams: { search: term },
});

// Paginated query (fetches all pages)
queryFn: query.paginated(patientApi.list, {
  pageSize: 10,
  maxPages: null,
});
```

#### Mutation Functions

```typescript
const { mutate } = useMutation({
  mutationFn: mutate(patientApi.create),
  onSuccess: () => queryClient.invalidateQueries(["patients"]),
});
```

### General Utilities (`Utils/utils.ts`)

#### Date/Time
- `formatDateTime()` - Format dates for display
- `relativeDate()` - "2 hours ago" format
- `getReadableDuration()` - Human-readable duration
- `getMonthStartAndEnd()` - Date range utilities

#### Formatting
- `formatName()` - Name formatting
- `properCase()` - Title case conversion
- `formatPatientAge()` - Age calculation from DOB
- `formatTruncatedList()` - List truncation

#### Device Detection
- `isIOSDevice`, `isMacDevice`, `isAppleDevice`
- `isAndroidDevice`, `isTouchDevice`
- `isUserOnline()` - Network status

#### Validation (`Utils/validators.ts`)
- Phone number validation (Zod)
- Coordinate validation (lat/long)
- Pincode validation
- Age validation (1-120)

---

## 6. Documentation Quality

### What Exists

| Type | Quality | Location |
|------|---------|----------|
| API Request README | Excellent | `src/Utils/request/README.md` |
| Project Guidelines | Good | `CLAUDE.md`, `AGENTS.md` |
| Copilot Instructions | Good | `.github/instructions/` (14 files) |
| Test Documentation | Good | `tests/README.md` |

### Code Comments

#### JSDoc Examples

```typescript
// CareIcon.tsx
/**
 * ### CARE's Official Icon Library.
 * @param className icon class name
 * @returns icon component
 * @example ```<CareIcon icon="l-hospital"/> ```
 * @see [icon library](https://iconscout.com/unicons/)
 */

// Calendar.tsx
/**
 * A custom calendar component built on top of react-day-picker.
 * @param props The props for the calendar.
 * @default yearRange 12
 */
```

### What's Missing

- No Storybook for interactive component documentation
- No comprehensive component API documentation
- Limited inline comments in complex components
- No auto-generated API documentation

---

## 7. Build Process & Tools

### Development Commands

```bash
# Start development server (localhost:4000)
npm run dev

# Run linting
npm run lint
npm run lint-fix

# Format code
npm run format

# Run tests
npm run playwright:test        # Headless
npm run playwright:test:ui     # Interactive UI
npm run playwright:test:headed # Visible browser
```

### Build Commands

```bash
# Full production build
npm run build

# Individual build steps
npm run setup              # Generate plugin maps
npm run build:meta         # Generate build version
npm run supported-browsers # Generate browser regex
npm run build:react        # Vite production build
```

### Build Pipeline

```
npm run build
├── npm run setup              # Generate src/pluginMap.ts
├── npm run build:meta         # Generate public/build-meta.json (UUID)
├── npm run supported-browsers # Generate src/supportedBrowsers.ts
└── npm run build:react        # Vite production build
    ├── TypeScript compilation (ES2022)
    ├── Tree-shake unused icons
    ├── Generate PWA manifest
    ├── Module federation setup
    └── Output to build/
```

### Vite Configuration Highlights

```typescript
// vite.config.mts
{
  plugins: [
    tailwindcss(),           // Tailwind CSS v4
    react(),                 // React 19 support
    federation({...}),       // Module federation for plugins
    checker({ typescript: true, eslint: {...} }),
    VitePWA({...}),         // Progressive Web App
  ],
  build: {
    target: "es2022",
    outDir: "build",
    sourcemap: true,
  },
  server: {
    port: 4000,
  }
}
```

### CI/CD Workflows (GitHub Actions)

| Workflow | Trigger | Purpose |
|----------|---------|---------|
| `linter.yml` | PRs to develop | ESLint + unused imports |
| `deploy.yaml` | Push to develop/staging/tags | Docker build + AWS deploy |
| `playwright.yaml` | PRs | E2E tests (3 parallel shards) |

### Docker Build

```dockerfile
# Stage 1: Build
FROM node:22-bookworm-slim
RUN npm install --ignore-scripts
RUN npm run postinstall
RUN npm run setup
RUN npm run build

# Stage 2: Serve
FROM nginx:stable-alpine
COPY build/ /usr/share/nginx/html
EXPOSE 80
```

### TypeScript Configuration

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "strict": true,
    "jsx": "react-jsx",
    "paths": {
      "@/*": ["./src/*"],
      "@core/*": ["src/*"],
      "@careConfig": ["./care.config.ts"]
    }
  }
}
```

---

## 8. Styling & Theming

### Tailwind Configuration

```javascript
// tailwind.config.js
module.exports = {
  darkMode: ["class"],
  theme: {
    extend: {
      fontFamily: {
        sans: ["Figtree", ...defaultTheme.fontFamily.sans],
      },
      colors: {
        primary: {
          DEFAULT: "#0d9f6e",
          50: "#f0fdf5",
          // ... 100-900
        },
        secondary: {
          // Purple palette 50-900
        },
        warning: colors.amber,
        alert: colors.violet,
        danger: colors.red,
      },
    },
  },
  plugins: [
    require("@tailwindcss/forms"),
    require("@tailwindcss/typography"),
    require("@tailwindcss/container-queries"),
  ],
};
```

### Styling Stack

| Tool | Purpose |
|------|---------|
| Tailwind CSS 4.1.3 | Utility-first CSS |
| Class Variance Authority | Component variants |
| tailwind-merge + clsx | Class merging (`cn()` utility) |
| CSS Variables | Dynamic theming |

### Component Variant Pattern (CVA)

```typescript
// Example: Button variants
const buttonVariants = cva(
  "inline-flex items-center justify-center rounded-md font-medium transition-colors",
  {
    variants: {
      variant: {
        primary: "bg-primary-700 text-white hover:bg-primary-800",
        secondary: "bg-secondary-100 text-secondary-900",
        destructive: "bg-red-500 text-white hover:bg-red-600",
        outline: "border border-gray-300 bg-transparent",
        ghost: "hover:bg-gray-100",
        link: "text-primary-700 underline-offset-4 hover:underline",
        warning: "bg-amber-500 text-white",
        alert: "bg-violet-500 text-white",
      },
      size: {
        xs: "h-7 px-2 text-xs",
        sm: "h-8 px-3 text-sm",
        default: "h-9 px-4",
        md: "h-10 px-4",
        lg: "h-11 px-6",
        icon: "h-9 w-9",
      },
    },
    defaultVariants: {
      variant: "primary",
      size: "default",
    },
  }
);
```

### CSS Variables (Sidebar Theming)

```css
:root {
  --radius: 0.5rem;
  --sidebar-background: 220 14.3% 95.9%;
  --sidebar-foreground: 240 5.3% 26.1%;
  --sidebar-primary: 240 5.9% 10%;
  --sidebar-accent: 240 4.8% 95.9%;
  --sidebar-border: transparent;
}
```

### Theme Customization Options

| Aspect | Customizable | Method |
|--------|--------------|--------|
| Colors | Yes | `tailwind.config.js` theme extension |
| Fonts | Yes | Font imports + Tailwind config |
| Dark Mode | Partially | Class-based (infrastructure ready) |
| Logos | Yes | Environment variables |
| Spacing | Yes | Tailwind theme extension |
| Border Radius | Yes | CSS variable `--radius` |

### Dark Mode Status

- **Infrastructure**: Ready (class-based, `next-themes` installed)
- **Implementation**: Partial (only Toaster component uses it)
- **Components**: Have `dark:` class prefixes but not actively toggled

### Print Utilities

```css
/* Available print classes */
.print-page-break-before
.print-page-break-after
.print-avoid-break
.print-landscape
.print-portrait
.print-a4
.print-letter
.print-text-small / medium / large / auto
.print-scale-90 / 85 / 80 / 75 / fit
.print-logo-small / medium / large
```

---

## 9. Summary Scorecard

| Aspect | Rating | Notes |
|--------|--------|-------|
| **Architecture** | ⭐⭐⭐⭐⭐ | Clean separation, feature-based, scalable |
| **Type Safety** | ⭐⭐⭐⭐⭐ | Strict TypeScript, generic API layer |
| **Code Practices** | ⭐⭐⭐⭐ | Consistent patterns, good linting |
| **Authentication** | ⭐⭐⭐⭐⭐ | MFA, refresh tokens, multi-tab sync |
| **Reusability** | ⭐⭐⭐⭐⭐ | 88 UI primitives, 33 hooks, rich utils |
| **Documentation** | ⭐⭐⭐ | Good API docs, lacks component docs |
| **Build Tools** | ⭐⭐⭐⭐⭐ | Modern Vite, comprehensive CI/CD |
| **Theming** | ⭐⭐⭐⭐ | Well-structured, dark mode incomplete |

### Key Strengths

1. **Modern Stack**: React 19, Vite 6, Tailwind 4, TypeScript 5.6
2. **Enterprise Features**: Module federation, PWA, i18n (6 languages)
3. **Strict Quality**: ESLint flat config, Prettier, pre-commit hooks
4. **Performance**: Tree-shaking, code splitting, async chunks
5. **Testing**: Playwright with parallel shards and backend integration
6. **Extensibility**: Plugin system via module federation

### Areas for Enhancement

1. Add Storybook for interactive component documentation
2. Complete dark mode implementation
3. Expand JSDoc coverage for complex components
4. Add component usage guides

---

## Quick Start

```bash
# Install dependencies
npm install

# Create .env.local for local backend
echo "REACT_CARE_API_URL=http://127.0.0.1:9000" > .env.local

# Start development server
npm run dev

# Open browser
open http://localhost:4000
```

---

*Generated: January 2025*
