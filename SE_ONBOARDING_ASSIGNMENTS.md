# Software Engineer Onboarding Assignments

This document contains progressive assignments designed to help new Software Engineers gain a deep understanding of both **Care Frontend (care_fe)** and **Care Backend (care)** systems.

---

## Overview

**Care** is a Digital Public Good for TeleICU & Healthcare Capacity Management. The system consists of:
- **care_fe**: React 19 + TypeScript frontend with TanStack Query
- **care**: Django REST Framework backend with PostgreSQL

**Duration**: 4-6 weeks (depending on experience level)

---

## Pre-requisites

Before starting, ensure you can:
- [ ] Run `npm run dev` and access the frontend at `localhost:4000`
- [ ] Run the backend and access API at `localhost:9000`
- [ ] Login with test credentials
- [ ] Run `npm run playwright:test` successfully

**Read These First**:
1. `CLAUDE.md` - Project guidelines
2. `src/Utils/request/README.md` - API patterns documentation
3. `flowdiagram.md` - System flows and architecture

---

## Assignment Structure

Each assignment includes:
- **Objective**: What you'll build/learn
- **Learning Goals**: Skills you'll develop
- **Files to Study**: Reference material
- **Tasks**: Step-by-step implementation
- **Bonus**: Optional enhancements
- **Deliverables**: What to submit for review

---

## Week 1: Foundations

### Assignment 1: API Pattern Explorer (Day 1-2)

**Objective**: Understand how the frontend communicates with the backend.

**Learning Goals**:
- Understand the type-safe API route pattern
- Learn TanStack Query usage (query, mutation)
- Understand error handling flow

**Files to Study**:
```
src/Utils/request/
├── query.ts          # Query factory with debounce + pagination
├── mutate.ts         # Mutation factory
├── types.ts          # Type definitions
├── errorHandler.ts   # Global error handling
└── README.md         # Documentation

src/types/user/userApi.ts    # Example API definition
src/types/auth/authApi.ts    # Auth API patterns
```

**Tasks**:

1. **Trace a Query Flow**
   - Open browser DevTools Network tab
   - Login to the application
   - Find the API call that fetches the current user
   - Document the request/response structure
   - Find the corresponding code in `src/types/auth/authApi.ts`

2. **Trace a Mutation Flow**
   - Create a new user through the UI
   - Document the API call made
   - Find the mutation code that triggers it
   - Understand how success/error is handled

3. **Create a Documentation**
   Write a brief document explaining:
   - How `query()` function works
   - How `mutate()` function works
   - How errors are handled globally
   - What `debounced` and `paginated` variants do

**Deliverables**:
- [ ] API flow documentation (1-2 pages)
- [ ] Screenshots of network requests traced

---

### Assignment 2: Component Anatomy (Day 3-4)

**Objective**: Understand component structure from simple to complex.

**Learning Goals**:
- React component patterns used in the codebase
- Form handling with react-hook-form + Zod
- Sheet/Modal patterns with Radix UI

**Files to Study**:
```
# Simple (start here)
src/components/Common/BackButton.tsx         # 33 lines

# Medium complexity
src/pages/Organization/components/AddUserSheet.tsx  # 69 lines

# Complex (reference)
src/components/Users/UserForm.tsx            # 866 lines
```

**Tasks**:

1. **Analyze BackButton.tsx**
   Answer these questions:
   - What hook is used for navigation?
   - How are props typed and extended?
   - What pattern is used for default values?

2. **Analyze AddUserSheet.tsx**
   Answer these questions:
   - How is sheet open/close state managed?
   - How does it compose with UserForm?
   - What callback pattern is used for success?

3. **Create a Simple Component**
   Create `src/components/Common/ConfirmDialog.tsx`:
   ```typescript
   interface ConfirmDialogProps {
     open: boolean;
     onOpenChange: (open: boolean) => void;
     title: string;
     description: string;
     onConfirm: () => void;
     onCancel?: () => void;
     confirmText?: string;
     cancelText?: string;
     variant?: "default" | "destructive";
   }
   ```
   - Use `AlertDialog` from `@/components/ui/alert-dialog`
   - Follow existing component patterns
   - Add proper TypeScript types

**Deliverables**:
- [ ] Answers to analysis questions
- [ ] Working ConfirmDialog component
- [ ] Component used in at least one place

---

### Assignment 3: Testing Fundamentals (Day 5)

**Objective**: Write E2E tests using Playwright.

**Learning Goals**:
- Playwright test structure
- Page object patterns
- API response interception
- Test data generation with Faker

**Files to Study**:
```
playwright.config.ts
tests/setup/auth.setup.ts
tests/auth/login.spec.ts                        # Simple test
tests/organization/user/userCreation.spec.ts   # Complex test
```

**Tasks**:

1. **Run and Analyze Existing Tests**
   ```bash
   npm run playwright:test:ui
   ```
   - Run the login test
   - Observe the browser automation
   - Read the test code alongside

2. **Write a New Test**
   Create `tests/organization/user/userSearch.spec.ts`:
   ```typescript
   // Test should:
   // 1. Navigate to organization users page
   // 2. Search for a user by name
   // 3. Verify search results appear
   // 4. Clear search and verify all users return
   ```

**Deliverables**:
- [ ] Working test file
- [ ] Test passing in CI

---

## Week 2: Core Features

### Assignment 4: Patient Search Enhancement (Day 1-3)

**Objective**: Add advanced filtering to patient search.

**Learning Goals**:
- Search and filter patterns
- Query parameter management
- Debounced API calls
- Filter state management

**Files to Study**:
```
src/components/Patient/PatientIndex.tsx
src/components/Patient/PatientIdentifierFilter.tsx
src/hooks/useFilters.tsx
src/types/emr/patient/patientApi.ts
```

**Tasks**:

1. **Understand Current Implementation**
   - How does patient search currently work?
   - What filters are available?
   - How are filters persisted in URL?

2. **Add Date Range Filter**
   Add ability to filter patients by:
   - Registration date range (from/to)
   - Last encounter date range

   Requirements:
   - Use existing `DateRangePicker` component
   - Persist filter in URL query params
   - Clear filters should reset dates
   - API should receive date params

3. **Backend Investigation**
   - Find the patient list API endpoint in care backend
   - Verify it supports date filtering
   - If not, document what backend changes are needed

**Deliverables**:
- [ ] Working date range filter
- [ ] Updated API call with date params
- [ ] Documentation of backend requirements

---

### Assignment 5: User Activity Log (Day 4-5)

**Objective**: Display user's recent activity in their profile.

**Learning Goals**:
- New feature implementation end-to-end
- API integration patterns
- Timeline/list component creation
- Pagination handling

**Frontend Tasks**:

1. **Create Activity Types**
   Create `src/types/user/activity.ts`:
   ```typescript
   export interface UserActivity {
     id: string;
     action: "login" | "patient_view" | "encounter_create" | "prescription_add";
     resource_type: string;
     resource_id: string;
     resource_name: string;
     timestamp: string;
     ip_address?: string;
     facility_name?: string;
   }
   ```

2. **Create Activity API**
   Create `src/types/user/activityApi.ts`:
   ```typescript
   // GET /api/v1/users/{user_id}/activity/
   // Query params: page, limit, action_type, from_date, to_date
   ```

3. **Create Activity List Component**
   Create `src/components/Users/UserActivityLog.tsx`:
   - Paginated list of activities
   - Filter by action type
   - Relative timestamps ("2 hours ago")
   - Link to resource when applicable

4. **Integrate in User Profile**
   - Add "Activity" tab to user profile page
   - Show last 10 activities in user card

**Backend Tasks** (care repository):

5. **Create Activity Model**
   - Create new Django model for activity tracking
   - Add migration

6. **Create Activity API**
   - Create DRF serializer
   - Create viewset with filtering
   - Add to URL router

7. **Add Activity Logging**
   - Create middleware or signal to log activities
   - Log at least: login, patient view, encounter create

**Deliverables**:
- [ ] Frontend components and types
- [ ] Backend model, API, and logging
- [ ] E2E test for activity display

---

## Week 3: Healthcare Domain

### Assignment 6: Medication Interaction Warning (Day 1-3)

**Objective**: Show warnings when prescribing medications that may interact.

**Learning Goals**:
- Healthcare domain understanding
- Real-time validation patterns
- Alert/warning UI patterns
- Cross-referencing data

**Files to Study**:
```
src/components/Questionnaire/QuestionTypes/MedicationRequestQuestion.tsx
src/components/Medication/
src/types/emr/medicationRequest/
```

**Tasks**:

1. **Understand Medication Flow**
   - How are medications prescribed in encounters?
   - What data is captured for each medication?
   - How does the medication autocomplete work?

2. **Design Interaction Check**
   Design (document only, don't implement backend):
   ```
   // When user selects a medication:
   // 1. Check against patient's current medications
   // 2. Check against other medications in current prescription
   // 3. Return interaction warnings

   // Interaction levels:
   // - severe: Block prescription, require override
   // - moderate: Show warning, allow continue
   // - mild: Show info tooltip
   ```

3. **Create Warning Component**
   Create `src/components/Medication/InteractionWarning.tsx`:
   - Alert banner for severe interactions
   - Warning tooltip for moderate
   - Info icon for mild
   - List of interacting medications

4. **Integrate with Prescription Form**
   - Call interaction check when medication selected
   - Display appropriate warnings
   - Track if user acknowledged warnings

**Deliverables**:
- [ ] Design document for interaction checking
- [ ] Warning component with all severity levels
- [ ] Integration with medication selection
- [ ] Mock data for testing (no backend required)

---

### Assignment 7: Encounter Summary Export (Day 4-5)

**Objective**: Export encounter summary as PDF.

**Learning Goals**:
- PDF generation in browser
- Data aggregation from multiple sources
- Print-friendly styling
- File download patterns

**Files to Study**:
```
src/components/Patient/TreatmentSummary.tsx
src/pages/Encounters/PrintPrescription.tsx
src/Utils/request/uploadFile.ts
```

**Tasks**:

1. **Analyze Print Prescription**
   - How does the print functionality work?
   - What styling is applied for printing?
   - How is the PDF generated?

2. **Create Encounter Summary PDF**
   Create `src/pages/Encounters/ExportEncounterSummary.tsx`:

   Include in export:
   - Patient demographics
   - Encounter details (date, type, location)
   - Vital signs recorded
   - Diagnoses
   - Prescriptions
   - Notes summary
   - Care team members

3. **Add Export Button**
   - Add "Export PDF" button to encounter view
   - Show loading state during generation
   - Auto-download when complete

4. **Print Styling**
   - Create print-specific CSS
   - Ensure proper page breaks
   - Add hospital letterhead placeholder

**Deliverables**:
- [ ] PDF export component
- [ ] Print-friendly styling
- [ ] Working export button in encounter view

---

## Week 4: Advanced Patterns

### Assignment 8: Real-time Bed Availability (Day 1-3)

**Objective**: Create a real-time dashboard showing bed availability across facilities.

**Learning Goals**:
- WebSocket/polling patterns
- Dashboard component design
- Data visualization
- Cross-facility data aggregation

**Tasks**:

1. **Design Dashboard**
   Create wireframe for bed dashboard showing:
   - Total beds vs. occupied vs. available
   - Breakdown by ward/location
   - ICU vs. general bed status
   - Trend over last 24 hours

2. **Create Dashboard Component**
   Create `src/pages/Facility/BedAvailabilityDashboard.tsx`:
   - Grid of facility cards
   - Each card shows bed stats
   - Color coding (green/yellow/red)
   - Auto-refresh every 30 seconds

3. **Add Polling**
   - Use TanStack Query's `refetchInterval`
   - Show "Last updated" timestamp
   - Handle stale data gracefully

4. **Create Visualization**
   - Use existing chart library or add simple one
   - Show occupancy percentage bar
   - Show trend line if historical data available

**Deliverables**:
- [ ] Dashboard wireframe
- [ ] Working dashboard component
- [ ] Auto-refresh functionality
- [ ] Basic visualization

---

### Assignment 9: Plugin Development (Day 4-5)

**Objective**: Create a simple plugin that extends the Care system.

**Learning Goals**:
- Plugin architecture understanding
- Module Federation basics
- Extension point patterns
- Plugin manifest structure

**Files to Study**:
```
src/PluginEngine.tsx
src/pluginTypes.ts
src/hooks/useCareApps.tsx
src/types/plugConfig/plugConfigApi.ts
```

**Tasks**:

1. **Understand Plugin System**
   Document:
   - How are plugins loaded?
   - What can plugins extend?
   - How does Module Federation work?

2. **Design a Plugin**
   Design a "Quick Notes" plugin:
   ```
   // Adds a floating button to patient pages
   // Click to add a quick note
   // Notes appear in a sidebar
   // Uses PatientHomeActions extension point
   ```

3. **Create Plugin Manifest**
   Document the manifest structure needed:
   ```typescript
   // What components would the plugin export?
   // What routes would it add?
   // What navigation items?
   ```

4. **Create Mock Plugin Component**
   Create components AS IF they were in a plugin:
   - `QuickNoteButton.tsx` - Floating action button
   - `QuickNotesSidebar.tsx` - Notes list
   - `QuickNoteForm.tsx` - Note input form

   Note: These won't be an actual separate plugin, but structured as if they were.

**Deliverables**:
- [ ] Plugin system documentation
- [ ] Plugin design document
- [ ] Mock plugin components

---

## Week 5-6: Capstone Project

### Assignment 10: Care Team Communication Hub

**Objective**: Build a comprehensive feature that touches both frontend and backend, demonstrating mastery of the codebase.

**Learning Goals**:
- End-to-end feature development
- All patterns learned combined
- Code review and iteration
- Documentation

**Feature Requirements**:

Create a "Care Team Hub" for each encounter that enables:

1. **Team Chat**
   - Real-time messaging between care team members
   - Message history persistence
   - Unread message indicators
   - @mentions with notifications

2. **Task Assignment**
   - Create tasks for team members
   - Task status tracking (pending/in-progress/done)
   - Due dates and priorities
   - Task completion notifications

3. **Shift Handoff**
   - Structured handoff notes template
   - Previous shift summary view
   - Acknowledgment tracking
   - Critical items highlighting

**Technical Requirements**:

**Frontend**:
- [ ] New encounter tab: "Team Hub"
- [ ] Chat component with message list
- [ ] Task board component
- [ ] Handoff form and view
- [ ] Real-time updates (polling or WebSocket)
- [ ] Mobile-responsive design

**Backend**:
- [ ] Message model and API
- [ ] Task model and API
- [ ] Handoff model and API
- [ ] Notification integration

**Testing**:
- [ ] E2E tests for main flows
- [ ] Error handling tests

**Documentation**:
- [ ] API documentation
- [ ] Component documentation
- [ ] User guide

**Deliverables**:
- [ ] Working feature in both repos
- [ ] Comprehensive tests
- [ ] Documentation
- [ ] Demo video/presentation

---

## Evaluation Criteria

### Code Quality
- Follows existing patterns and conventions
- Proper TypeScript types (no `any`)
- Error handling implemented
- Code is readable and maintainable

### Testing
- Tests cover happy path
- Tests cover error cases
- Tests are maintainable

### Documentation
- Code is self-documenting
- Complex logic has comments
- README updated if needed

### Healthcare Domain
- Understands clinical workflows
- Considers patient safety
- Follows healthcare data practices

---

## Resources

### Internal
- `CLAUDE.md` - Project guidelines
- `flowdiagram.md` - System architecture
- `src/Utils/request/README.md` - API documentation

### External
- [TanStack Query Docs](https://tanstack.com/query/latest)
- [React Hook Form Docs](https://react-hook-form.com/)
- [Zod Docs](https://zod.dev/)
- [Radix UI Docs](https://www.radix-ui.com/)
- [Playwright Docs](https://playwright.dev/)

---

## Mentorship & Support

- Daily standup to discuss progress
- Code review on each assignment
- Pair programming sessions available
- Questions channel: #care-dev

---

## Completion Checklist

| Week | Assignment | Status |
|------|------------|--------|
| 1 | API Pattern Explorer | [ ] |
| 1 | Component Anatomy | [ ] |
| 1 | Testing Fundamentals | [ ] |
| 2 | Patient Search Enhancement | [ ] |
| 2 | User Activity Log | [ ] |
| 3 | Medication Interaction Warning | [ ] |
| 3 | Encounter Summary Export | [ ] |
| 4 | Real-time Bed Availability | [ ] |
| 4 | Plugin Development | [ ] |
| 5-6 | Capstone: Care Team Hub | [ ] |

---

*Last Updated: February 2025*
