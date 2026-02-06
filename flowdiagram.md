# Care Frontend - Flow Diagrams

This document describes the main user flows, actors, and plugin extension points in the Care healthcare management system.

---

## Table of Contents

1. [System Overview](#system-overview)
2. [Actors & Roles](#actors--roles)
3. [User Flows](#user-flows)
   - [Authentication Flow](#authentication-flow)
   - [Patient Management Flow](#patient-management-flow)
   - [Clinical Encounter Flow](#clinical-encounter-flow)
   - [Facility Management Flow](#facility-management-flow)
   - [Scheduling & Appointments Flow](#scheduling--appointments-flow)
   - [Billing Flow](#billing-flow)
4. [Plugin Extension Points](#plugin-extension-points)
5. [Permission System](#permission-system)

---

## System Overview

```mermaid
flowchart TB
    subgraph Frontend["Care Frontend Application"]
        PR[Public Router] --> Auth[Authentication]
        AR[App Router] --> Features[Feature Modules]
        PatR[Patient Router] --> PatientFeatures[Patient Features]
    end

    subgraph Actors["System Actors"]
        Admin[Administrator]
        Doctor[Doctor]
        Nurse[Nurse]
        Staff[Staff]
        Volunteer[Volunteer]
    end

    subgraph Plugins["Plugin System"]
        PE[Plugin Engine]
        PM[Plugin Manifests]
        PC[Plugin Components]
    end

    Actors --> Frontend
    PE --> Features
    PE --> PatientFeatures
```

---

## Actors & Roles

### User Types

| Actor | Description | Primary Responsibilities |
|-------|-------------|-------------------------|
| **Administrator** | System administrator | User management, facility configuration, plugin management |
| **Doctor** | Healthcare practitioner | Patient consultations, prescriptions, clinical documentation |
| **Nurse** | Nursing staff | Patient care, vitals monitoring, medication administration |
| **Staff** | General facility staff | Administrative tasks, patient registration, scheduling |
| **Volunteer** | Volunteer workers | Support activities, patient assistance |

### Role Hierarchy

```mermaid
flowchart TD
    SA[Super Admin] --> OA[Organization Admin]
    OA --> FA[Facility Admin]
    FA --> D[Doctor]
    FA --> N[Nurse]
    FA --> S[Staff]
    FA --> V[Volunteer]

    subgraph Permissions
        OA -.-> OP[Organization Permissions]
        FA -.-> FP[Facility Permissions]
        D -.-> CP[Clinical Permissions]
        N -.-> NP[Nursing Permissions]
    end
```

---

## User Flows

### Authentication Flow

```mermaid
flowchart TD
    Start([User Visits App]) --> Landing[Landing Page]
    Landing --> Login[Login Page]
    Login --> Creds{Valid Credentials?}
    Creds -->|No| Login
    Creds -->|Yes| TwoFA{2FA Enabled?}
    TwoFA -->|Yes| OTP[Enter OTP]
    OTP --> OTPValid{OTP Valid?}
    OTPValid -->|No| OTP
    OTPValid -->|Yes| Dashboard
    TwoFA -->|No| Dashboard[User Dashboard]

    Dashboard --> FacilitySelect[Select Facility]
    FacilitySelect --> FacilityHome[Facility Home]

    style Start fill:#e1f5fe
    style Dashboard fill:#c8e6c9
    style FacilityHome fill:#c8e6c9
```

**Plugin Extension Point:** `[PLUGIN: PatientExternalRegistration]` - External authentication providers

---

### Patient Management Flow

```mermaid
flowchart TD
    subgraph Registration["Patient Registration"]
        Start([Start]) --> Search[Search Existing Patient]
        Search --> Found{Patient Found?}
        Found -->|Yes| SelectPatient[Select Patient]
        Found -->|No| CreateNew[Create New Patient]
        CreateNew --> RegForm[Registration Form]
        RegForm --> PLUGIN_REG[/"[PLUGIN: PatientRegistrationForm]<br/>Custom registration fields"/]
        PLUGIN_REG --> SavePatient[Save Patient]
    end

    subgraph PatientHome["Patient Home"]
        SelectPatient --> PH[Patient Home Page]
        SavePatient --> PH
        PH --> PLUGIN_PHA[/"[PLUGIN: PatientHomeActions]<br/>Custom home actions"/]
        PH --> Tabs{Select Tab}
        Tabs --> Demo[Demography]
        Tabs --> Clinical[Clinical Data]
        Tabs --> History[History]
        Tabs --> Encounters[Encounters]
        Tabs --> Files[Files]
        Tabs --> Appointments[Appointments]

        Demo --> PLUGIN_DEMO[/"[PLUGIN: PatientDetailsTabDemographyGeneralInfo]<br/>Custom demographics"/]
    end

    subgraph Actions["Patient Actions"]
        PH --> PLUGIN_CARD[/"[PLUGIN: PatientInfoCardActions]<br/>Card actions"/]
        PH --> PLUGIN_QUICK[/"[PLUGIN: PatientInfoCardQuickActions]<br/>Quick actions"/]
        PH --> NewEncounter[Create Encounter]
        PH --> BookAppt[Book Appointment]
        PH --> UploadFile[Upload File]
    end

    style PLUGIN_REG fill:#fff3e0,stroke:#ff9800
    style PLUGIN_PHA fill:#fff3e0,stroke:#ff9800
    style PLUGIN_DEMO fill:#fff3e0,stroke:#ff9800
    style PLUGIN_CARD fill:#fff3e0,stroke:#ff9800
    style PLUGIN_QUICK fill:#fff3e0,stroke:#ff9800
```

---

### Clinical Encounter Flow

```mermaid
flowchart TD
    subgraph EncounterCreation["Encounter Creation"]
        Start([Patient Selected]) --> CreateEnc[Create Encounter]
        CreateEnc --> SelectType[Select Encounter Type]
        SelectType --> SetLocation[Set Location]
        SetLocation --> AssignTeam[Assign Care Team]
        AssignTeam --> EncCreated[Encounter Created]
    end

    subgraph ClinicalDocumentation["Clinical Documentation"]
        EncCreated --> EncHome[Encounter Home]
        EncHome --> PLUGIN_ENC[/"[PLUGIN: EncounterActions]<br/>Custom encounter actions"/]

        EncHome --> Tabs{Documentation}
        Tabs --> Quest[Questionnaires]
        Tabs --> Vitals[Vital Signs]
        Tabs --> Diagnosis[Diagnosis]
        Tabs --> Symptoms[Symptoms]
        Tabs --> Prescriptions[Prescriptions]
        Tabs --> Notes[Clinical Notes]
        Tabs --> PLUGIN_TABS[/"[PLUGIN: encounterTabs]<br/>Custom encounter tabs"/]

        Quest --> PLUGIN_SCRIBE[/"[PLUGIN: Scribe]<br/>AI note-taking assistance"/]
    end

    subgraph TreatmentPlanning["Treatment Planning"]
        Diagnosis --> TreatmentPlan[Treatment Plan]
        Prescriptions --> MedOrders[Medication Orders]
        TreatmentPlan --> ServiceReq[Service Requests]
        ServiceReq --> PLUGIN_SR[/"[PLUGIN: ServiceRequestAction]<br/>Custom service request actions"/]
    end

    subgraph Completion["Encounter Completion"]
        EncHome --> MarkComplete[Mark as Complete]
        MarkComplete --> PLUGIN_COMPLETE[/"[PLUGIN: PatientInfoCardMarkAsComplete]<br/>Custom completion logic"/]
        PLUGIN_COMPLETE --> GenerateReport[Generate Report]
        GenerateReport --> End([Encounter Closed])
    end

    style PLUGIN_ENC fill:#fff3e0,stroke:#ff9800
    style PLUGIN_TABS fill:#fff3e0,stroke:#ff9800
    style PLUGIN_SCRIBE fill:#fff3e0,stroke:#ff9800
    style PLUGIN_SR fill:#fff3e0,stroke:#ff9800
    style PLUGIN_COMPLETE fill:#fff3e0,stroke:#ff9800
```

---

### Facility Management Flow

```mermaid
flowchart TD
    subgraph FacilityOverview["Facility Overview"]
        Start([Login]) --> SelectFac[Select Facility]
        SelectFac --> FacHome[Facility Home]
        FacHome --> PLUGIN_FAC[/"[PLUGIN: FacilityHomeActions]<br/>Custom facility actions"/]

        FacHome --> Dashboard[Dashboard]
        Dashboard --> BedStatus[Bed Availability]
        Dashboard --> ServiceStatus[Service Status]
        Dashboard --> StaffStatus[Staff On Duty]
    end

    subgraph LocationManagement["Location Management"]
        FacHome --> Locations[Manage Locations]
        Locations --> CreateLoc[Create Location]
        Locations --> EditLoc[Edit Location]
        Locations --> BedMgmt[Bed Management]
    end

    subgraph ServiceManagement["Service Management"]
        FacHome --> Services[Services]
        Services --> HealthServices[Healthcare Services]
        Services --> DiagServices[Diagnostic Services]
        Services --> ServiceReqs[Service Requests]
    end

    subgraph ResourceManagement["Resource Management"]
        FacHome --> Resources[Resources]
        Resources --> Devices[Medical Devices]
        Resources --> Consumables[Consumables]
        Resources --> Inventory[Inventory]

        Devices --> PLUGIN_DEV[/"[PLUGIN: devices]<br/>Custom device manifests"/]
    end

    style PLUGIN_FAC fill:#fff3e0,stroke:#ff9800
    style PLUGIN_DEV fill:#fff3e0,stroke:#ff9800
```

---

### Scheduling & Appointments Flow

```mermaid
flowchart TD
    subgraph ScheduleManagement["Schedule Management"]
        Start([Doctor/Staff]) --> ManageSchedule[Manage Schedule]
        ManageSchedule --> CreateSlots[Create Time Slots]
        ManageSchedule --> SetAvail[Set Availability]
        ManageSchedule --> Exceptions[Schedule Exceptions]
        CreateSlots --> Templates[Use Templates]
    end

    subgraph AppointmentBooking["Appointment Booking"]
        Patient([Patient/Staff]) --> SearchDoc[Search Doctor]
        SearchDoc --> PLUGIN_SEARCH[/"[PLUGIN: PatientSearchActions]<br/>Custom search actions"/]
        SearchDoc --> ViewAvail[View Availability]
        ViewAvail --> SelectSlot[Select Time Slot]
        SelectSlot --> ConfirmAppt[Confirm Appointment]
        ConfirmAppt --> Notification[Send Notification]
    end

    subgraph AppointmentManagement["Appointment Management"]
        ConfirmAppt --> ApptList[Appointment List]
        ApptList --> Reschedule[Reschedule]
        ApptList --> Cancel[Cancel]
        ApptList --> CheckIn[Check-In]
        CheckIn --> StartEnc[Start Encounter]
    end

    style PLUGIN_SEARCH fill:#fff3e0,stroke:#ff9800
```

---

### Billing Flow

```mermaid
flowchart TD
    subgraph ServiceCharging["Service Charging"]
        Start([Encounter/Service]) --> AddCharge[Add Charge]
        AddCharge --> SelectService[Select Service]
        SelectService --> SetQuantity[Set Quantity]
        SetQuantity --> ApplyDiscount[Apply Discount]
    end

    subgraph InvoiceManagement["Invoice Management"]
        ApplyDiscount --> CreateInvoice[Create Invoice]
        CreateInvoice --> ReviewInvoice[Review Invoice]
        ReviewInvoice --> SendInvoice[Send to Patient]
    end

    subgraph PaymentProcessing["Payment Processing"]
        SendInvoice --> RecordPayment[Record Payment]
        RecordPayment --> PLUGIN_PAY[/"[PLUGIN: InvoiceRecordPaymentOptions]<br/>Custom payment methods"/]
        PLUGIN_PAY --> PayMethod{Payment Method}
        PayMethod --> Cash[Cash]
        PayMethod --> Card[Card]
        PayMethod --> Insurance[Insurance]
        PayMethod --> CustomPay[Plugin Payment]
    end

    subgraph Reconciliation["Reconciliation"]
        Cash --> Reconcile[Reconcile Payment]
        Card --> Reconcile
        Insurance --> Reconcile
        CustomPay --> Reconcile
        Reconcile --> GenerateReceipt[Generate Receipt]
        GenerateReceipt --> End([Complete])
    end

    style PLUGIN_PAY fill:#fff3e0,stroke:#ff9800
```

---

## Plugin Extension Points

The system uses **Module Federation** for dynamic plugin loading. Plugins can extend the application at these points:

### Component Extension Points

| Extension Point | Location | Purpose |
|----------------|----------|---------|
| `PatientRegistrationForm` | Patient Registration | Add custom registration fields |
| `PatientHomeActions` | Patient Home Page | Add custom action buttons |
| `PatientInfoCardActions` | Patient Info Card | Add card-level actions |
| `PatientInfoCardQuickActions` | Patient Info Card | Add quick action buttons |
| `PatientInfoCardMarkAsComplete` | Patient Card | Custom completion logic |
| `PatientDetailsTabDemographyGeneralInfo` | Patient Demographics | Custom demographic fields |
| `PatientSearchActions` | Patient Search | Custom search features |
| `EncounterActions` | Encounter Page | Custom encounter actions |
| `FacilityHomeActions` | Facility Home | Custom facility actions |
| `DoctorConnectButtons` | Doctor Connect | Custom communication buttons |
| `Scribe` | Questionnaire Form | AI-assisted note taking |
| `ServiceRequestAction` | Service Requests | Custom service actions |
| `InvoiceRecordPaymentOptions` | Billing/Payment | Custom payment methods |

### Navigation Extension Points

```mermaid
flowchart LR
    subgraph MainNav["Main Navigation"]
        Home[Home]
        Patients[Patients]
        Facilities[Facilities]
        PLUGIN_NAV[/"[PLUGIN: navItems]"/]
    end

    subgraph BillingNav["Billing Navigation"]
        Invoices[Invoices]
        Payments[Payments]
        PLUGIN_BILL[/"[PLUGIN: billingNavItems]"/]
    end

    subgraph UserNav["User Menu"]
        Profile[Profile]
        Settings[Settings]
        PLUGIN_USER[/"[PLUGIN: userNavItems]"/]
    end

    subgraph AdminNav["Admin Navigation"]
        Users[Users]
        Roles[Roles]
        PLUGIN_ADMIN[/"[PLUGIN: adminNavItems]"/]
    end

    style PLUGIN_NAV fill:#fff3e0,stroke:#ff9800
    style PLUGIN_BILL fill:#fff3e0,stroke:#ff9800
    style PLUGIN_USER fill:#fff3e0,stroke:#ff9800
    style PLUGIN_ADMIN fill:#fff3e0,stroke:#ff9800
```

### Route Extension Points

Plugins can add custom routes:

```mermaid
flowchart TD
    Router[App Router] --> CoreRoutes[Core Routes]
    Router --> PLUGIN_ROUTES[/"[PLUGIN: routes]<br/>Custom URL routes"/]
    Router --> OrgRoutes[Organization Routes]
    OrgRoutes --> PLUGIN_ORG[/"[PLUGIN: organizationTabs]<br/>Custom org tabs"/]

    style PLUGIN_ROUTES fill:#fff3e0,stroke:#ff9800
    style PLUGIN_ORG fill:#fff3e0,stroke:#ff9800
```

### Plugin Architecture

```mermaid
flowchart TD
    subgraph Backend["Backend API"]
        PlugAPI["/api/v1/plug_config/"]
    end

    subgraph PluginEngine["Plugin Engine"]
        FetchConfig[Fetch Plugin Configs]
        ModFed[Module Federation]
        LoadManifest[Load Plugin Manifests]
        RenderPlugin[Render Plugin Components]
    end

    subgraph PluginManifest["Plugin Manifest"]
        Components[Component Exports]
        Routes[Route Definitions]
        NavItems[Navigation Items]
        Devices[Device Manifests]
        EncTabs[Encounter Tabs]
        OrgTabs[Organization Tabs]
    end

    subgraph ErrorHandling["Error Handling"]
        ErrorBoundary[Plugin Error Boundary]
        Suspense[Suspense Fallback]
    end

    PlugAPI --> FetchConfig
    FetchConfig --> ModFed
    ModFed --> LoadManifest
    LoadManifest --> PluginManifest
    PluginManifest --> RenderPlugin
    RenderPlugin --> ErrorBoundary
    RenderPlugin --> Suspense
```

---

## Permission System

### Permission Categories

```mermaid
flowchart TD
    subgraph PatientPerms["Patient Permissions"]
        P1[can_create_patient]
        P2[can_write_patient]
        P3[can_list_patients]
        P4[can_view_clinical_data]
    end

    subgraph EncounterPerms["Encounter Permissions"]
        E1[can_create_encounter]
        E2[can_write_encounter]
        E3[can_read_encounter]
        E4[can_read_encounter_clinical_data]
    end

    subgraph FacilityPerms["Facility Permissions"]
        F1[can_create_facility]
        F2[can_read_facility]
        F3[can_update_facility]
        F4[can_write_facility_locations]
    end

    subgraph OrgPerms["Organization Permissions"]
        O1[can_view_organization]
        O2[can_manage_organization]
        O3[can_manage_organization_users]
        O4[is_geo_admin]
    end

    subgraph AdminPerms["Admin Permissions"]
        A1[can_create_user]
        A2[can_list_user]
        A3[can_write_questionnaire]
        A4[can_manage_questionnaire]
    end
```

### Permission Flow

```mermaid
flowchart TD
    User([User]) --> Auth[Authenticate]
    Auth --> LoadPerms[Load Permissions]
    LoadPerms --> Sources{Permission Sources}
    Sources --> Direct[Direct User Permissions]
    Sources --> Facility[Facility Permissions]
    Sources --> Org[Organization Permissions]
    Sources --> Super[Super Admin Flag]

    Direct --> Merge[Merge Permissions]
    Facility --> Merge
    Org --> Merge
    Super --> Merge

    Merge --> Context[Permission Context]
    Context --> Check{Check Permission}
    Check -->|Has Permission| Allow[Allow Action]
    Check -->|No Permission| Deny[Deny Action]
```

---

## Key Files Reference

| Category | File Path |
|----------|-----------|
| **Plugin System** | |
| Plugin Engine | `src/PluginEngine.tsx` |
| Plugin Types | `src/pluginTypes.ts` |
| Plugin Hooks | `src/hooks/useCareApps.tsx` |
| Plugin API | `src/types/plugConfig/plugConfigApi.ts` |
| **Permission System** | |
| Permission Context | `src/context/PermissionContext.tsx` |
| Permission Constants | `src/common/Permissions.tsx` |
| Permission Types | `src/types/emr/permission/permission.ts` |
| Role Types | `src/types/emr/role/role.ts` |
| **Routing** | |
| App Router | `src/Routers/AppRouter.tsx` |
| Patient Router | `src/Routers/PatientRouter.tsx` |
| Public Router | `src/Routers/PublicRouter.tsx` |
| Route Definitions | `src/Routers/routes/` |

---

## Legend

| Symbol | Meaning |
|--------|---------|
| `[PLUGIN: name]` | Plugin extension point - behavior can be customized via plugins |
| Orange boxes | Plugin-customizable components |
| Green boxes | Successful completion states |
| Blue boxes | Starting points |
