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
   - [Medication Management Flow](#medication-management-flow)
   - [Care Team Management Flow](#care-team-management-flow)
   - [Facility Management Flow](#facility-management-flow)
   - [Inventory & Supply Chain Flow](#inventory--supply-chain-flow)
   - [Device Management Flow](#device-management-flow)
   - [Scheduling & Appointments Flow](#scheduling--appointments-flow)
   - [Public Appointment Booking Flow](#public-appointment-booking-flow)
   - [Appointment Queuing Flow](#appointment-queuing-flow)
   - [Billing & Accounts Flow](#billing--accounts-flow)
   - [Consent Management Flow](#consent-management-flow)
   - [File Management Flow](#file-management-flow)
   - [Questionnaire & Templates Flow](#questionnaire--templates-flow)
   - [User Preferences Flow](#user-preferences-flow)
4. [Plugin Extension Points](#plugin-extension-points)
5. [Permission System](#permission-system)
6. [State Management](#state-management)
7. [Recent Features](#recent-features)

---

## System Overview

```mermaid
flowchart TB
    subgraph Frontend["Care Frontend Application"]
        PR[Public Router] --> Auth[Authentication]
        PR --> PublicAppt[Public Appointments]
        AR[App Router] --> Features[Feature Modules]
        PatR[Patient Router] --> PatientFeatures[Patient Features]
    end

    subgraph Actors["System Actors"]
        Admin[Administrator]
        Doctor[Doctor]
        Nurse[Nurse]
        Staff[Staff]
        Volunteer[Volunteer]
        Patient[Patient/Public User]
    end

    subgraph Plugins["Plugin System"]
        PE[Plugin Engine]
        PM[Plugin Manifests]
        PC[Plugin Components]
        PD[Plugin Devices]
    end

    subgraph StateManagement["State Management"]
        Jotai[Jotai Atoms]
        Context[React Context]
        TanStack[TanStack Query]
    end

    Actors --> Frontend
    PE --> Features
    PE --> PatientFeatures
    StateManagement --> Frontend
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
| **Patient** | Public user | Self-registration, appointment booking, viewing records |

### Role Hierarchy

```mermaid
flowchart TD
    SA[Super Admin] --> OA[Organization Admin]
    OA --> GA[Geo Admin]
    GA --> FA[Facility Admin]
    FA --> D[Doctor]
    FA --> N[Nurse]
    FA --> S[Staff]
    FA --> V[Volunteer]

    subgraph Permissions
        OA -.-> OP[Organization Permissions]
        GA -.-> GP[Geographic Permissions]
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
        PLUGIN_REG --> Identifiers[Add Patient Identifiers]
        Identifiers --> SavePatient[Save Patient]
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
        Tabs --> Allergies[Allergies]

        Demo --> PLUGIN_DEMO[/"[PLUGIN: PatientDetailsTabDemographyGeneralInfo]<br/>Custom demographics"/]
    end

    subgraph Actions["Patient Actions"]
        PH --> PLUGIN_CARD[/"[PLUGIN: PatientInfoCardActions]<br/>Card actions"/]
        PH --> PLUGIN_QUICK[/"[PLUGIN: PatientInfoCardQuickActions]<br/>Quick actions"/]
        PH --> NewEncounter[Create Encounter]
        PH --> BookAppt[Book Appointment]
        PH --> UploadFile[Upload File]
        PH --> AddAllergy[Add Allergy/Intolerance]
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
        CreateEnc --> SelectType[Select Encounter Class]
        SelectType --> SetLocation[Set Location]
        SetLocation --> AssignTeam[Assign Care Team]
        AssignTeam --> EncCreated[Encounter Created]
    end

    subgraph ClinicalDocumentation["Clinical Documentation"]
        EncCreated --> EncHome[Encounter Home]
        EncHome --> PLUGIN_ENC[/"[PLUGIN: EncounterActions]<br/>Custom encounter actions"/]

        EncHome --> Tabs{Documentation Tabs}
        Tabs --> Overview[Overview]
        Tabs --> Quest[Questionnaires/Responses]
        Tabs --> Vitals[Observations/Vitals]
        Tabs --> Diagnosis[Diagnosis]
        Tabs --> Symptoms[Symptoms]
        Tabs --> Prescriptions[Prescriptions/Medicines]
        Tabs --> Notes[Clinical Notes]
        Tabs --> Consents[Consents]
        Tabs --> DevicesTab[Devices]
        Tabs --> DiagReports[Diagnostic Reports]
        Tabs --> ServiceReqs[Service Requests]
        Tabs --> FilesTab[Files]
        Tabs --> Plots[Plots]
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
        GenerateReport --> PrintPrescription[Print Prescription]
        PrintPrescription --> End([Encounter Closed])
    end

    style PLUGIN_ENC fill:#fff3e0,stroke:#ff9800
    style PLUGIN_TABS fill:#fff3e0,stroke:#ff9800
    style PLUGIN_SCRIBE fill:#fff3e0,stroke:#ff9800
    style PLUGIN_SR fill:#fff3e0,stroke:#ff9800
    style PLUGIN_COMPLETE fill:#fff3e0,stroke:#ff9800
```

---

### Medication Management Flow

```mermaid
flowchart TD
    subgraph MedicationRequest["Medication Request"]
        Start([Encounter Active]) --> CreateMed[Create Medication Request]
        CreateMed --> SelectMed[Select Medication]
        SelectMed --> SetDosage[Set Dosage & Frequency]
        SetDosage --> SetDuration[Set Duration]
        SetDuration --> AddNotes[Add Notes]
        AddNotes --> SaveMed[Save Medication]
    end

    subgraph MedicationAdministration["Medication Administration"]
        SaveMed --> MedList[Medication List]
        MedList --> AdminMed[Administer Medication]
        AdminMed --> RecordAdmin[Record Administration]
        RecordAdmin --> PrintAdmin[Print Administration Record]
    end

    subgraph PharmacyQueue["Pharmacy Queue"]
        MedList --> PharmQueue[Pharmacy Queue]
        PharmQueue --> DispenseMed[Dispense Medication]
        DispenseMed --> SubstitutionCheck{Substitution Needed?}
        SubstitutionCheck -->|Yes| SubSheet[Substitution Sheet]
        SubSheet --> SelectAlt[Select Alternative]
        SelectAlt --> CompletePrescription
        SubstitutionCheck -->|No| CompletePrescription[Complete Prescription]
    end

    subgraph Billing["Medication Billing"]
        CompletePrescription --> BillMed[Bill Medication]
        BillMed --> AddToInvoice[Add to Invoice]
    end

    subgraph History["Medication History"]
        MedList --> MedStatement[Medication Statement]
        MedStatement --> HistoricalMeds[Historical Medications]
    end
```

---

### Care Team Management Flow

```mermaid
flowchart TD
    subgraph TeamSetup["Care Team Setup"]
        Start([Encounter/Patient]) --> ViewTeam[View Care Team]
        ViewTeam --> AddMember{Add Member?}
        AddMember -->|Yes| SearchUser[Search User]
        SearchUser --> SelectRole[Select Role]
        SelectRole --> AssignMember[Assign to Team]
        AssignMember --> TeamUpdated[Team Updated]
    end

    subgraph TeamManagement["Team Management"]
        TeamUpdated --> TeamList[Care Team List]
        TeamList --> EditMember[Edit Member Role]
        TeamList --> RemoveMember[Remove Member]
        TeamList --> FilterByTeam[Filter Encounters by Team]
    end

    subgraph Communication["Team Communication"]
        TeamList --> PLUGIN_DOCTOR[/"[PLUGIN: DoctorConnectButtons]<br/>Custom communication"/]
        PLUGIN_DOCTOR --> VideoCall[Video Call]
        PLUGIN_DOCTOR --> Message[Send Message]
    end

    style PLUGIN_DOCTOR fill:#fff3e0,stroke:#ff9800
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

    subgraph FacilitySettings["Facility Settings"]
        FacHome --> Settings[Settings]
        Settings --> FacOrgs[Facility Organizations]
        Settings --> DeviceConfig[Device Configuration]
        Settings --> Templates[Encounter Templates]
    end

    style PLUGIN_FAC fill:#fff3e0,stroke:#ff9800
```

---

### Inventory & Supply Chain Flow

```mermaid
flowchart TD
    subgraph InternalTransfer["Internal Transfer"]
        Start([Facility]) --> Inventory[Inventory Management]
        Inventory --> InternalXfer[Internal Transfer]
        InternalXfer --> SelectSource[Select Source Location]
        SelectSource --> SelectDest[Select Destination]
        SelectDest --> SelectItems[Select Items]
        SelectItems --> SelectLots[Select Stock Lots]
        SelectLots --> ConfirmXfer[Confirm Transfer]
    end

    subgraph ExternalSupply["External Supply Chain"]
        Inventory --> ExternalSupply[External Supply]
        ExternalSupply --> PurchaseOrder[Purchase Order]
        ExternalSupply --> DeliveryOrder[Delivery Order]
        DeliveryOrder --> ToReceive[Items To Receive]
        PurchaseOrder --> ToDispatch[Items To Dispatch]
    end

    subgraph StockManagement["Stock Management"]
        Inventory --> StockLots[Stock Lot Management]
        StockLots --> ViewLots[View Stock Lots]
        StockLots --> ExpiryTracking[Expiry Tracking]
        StockLots --> LotSelection[Lot Selection for Dispense]
    end

    subgraph ProductKnowledge["Product Knowledge"]
        Inventory --> Products[Product Catalog]
        Products --> ProductDetails[Product Details]
        Products --> SupplierInfo[Supplier Information]
    end
```

---

### Device Management Flow

```mermaid
flowchart TD
    subgraph DeviceSetup["Device Setup"]
        Start([Facility Settings]) --> Devices[Device Management]
        Devices --> AddDevice[Add Device]
        AddDevice --> SelectType[Select Device Type]
        SelectType --> PLUGIN_DEV[/"[PLUGIN: devices]<br/>Custom device types"/]
        PLUGIN_DEV --> ConfigDevice[Configure Device]
        ConfigDevice --> SaveDevice[Save Device]
    end

    subgraph EncounterDevices["Encounter Device Association"]
        SaveDevice --> DeviceList[Device List]
        DeviceList --> AssocDevice[Associate Device Sheet]
        AssocDevice --> SelectEncounter[Select Encounter]
        SelectEncounter --> LinkDevice[Link Device to Encounter]
    end

    subgraph DeviceData["Device Data Display"]
        LinkDevice --> EncDevices[Encounter Devices Tab]
        EncDevices --> DeviceOverview[Device Overview Card]
        DeviceOverview --> PLUGIN_CARD[/"[PLUGIN: devices.showPageCard]<br/>Custom device display"/]
        EncDevices --> EncOverview[/"[PLUGIN: devices.encounterOverview]<br/>Device in encounter"/]
    end

    style PLUGIN_DEV fill:#fff3e0,stroke:#ff9800
    style PLUGIN_CARD fill:#fff3e0,stroke:#ff9800
    style EncOverview fill:#fff3e0,stroke:#ff9800
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
        ManageSchedule --> ServiceType[Set Service Type]
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
        CheckIn --> GenerateToken[Generate Token]
        GenerateToken --> StartEnc[Start Encounter]
    end

    style PLUGIN_SEARCH fill:#fff3e0,stroke:#ff9800
```

---

### Public Appointment Booking Flow

```mermaid
flowchart TD
    subgraph PublicAccess["Public Access (No Auth)"]
        Start([Public User]) --> FacilityPage[Facility Public Page]
        FacilityPage --> ViewDoctors[View Available Doctors]
        ViewDoctors --> SelectDoctor[Select Doctor]
        SelectDoctor --> ViewSchedule[View Schedule]
    end

    subgraph PatientAuth["Patient Authentication"]
        ViewSchedule --> SelectSlot[Select Time Slot]
        SelectSlot --> PatientLogin{Has Account?}
        PatientLogin -->|Yes| Login[Patient Login]
        PatientLogin -->|No| Register[Patient Registration]
        Register --> FillDetails[Fill Patient Details]
        FillDetails --> VerifyPhone[Verify Phone/OTP]
    end

    subgraph Booking["Complete Booking"]
        Login --> ConfirmBooking[Confirm Booking]
        VerifyPhone --> ConfirmBooking
        ConfirmBooking --> BookingSuccess[Booking Success]
        BookingSuccess --> ViewAppointment[View Appointment Details]
    end
```

---

### Appointment Queuing Flow

```mermaid
flowchart TD
    subgraph QueueSetup["Queue Setup"]
        Start([Practitioner]) --> ManageQueue[Manage Queue]
        ManageQueue --> TokenCategories[Token Categories]
        TokenCategories --> CreateCategory[Create Token Category]
    end

    subgraph TokenGeneration["Token Generation"]
        Patient([Patient Arrives]) --> CheckIn[Check-In]
        CheckIn --> SelectCategory[Select Token Category]
        SelectCategory --> GenerateToken[Generate Token]
        GenerateToken --> TokenNumber[Token Number Assigned]
    end

    subgraph QueueManagement["Queue Management"]
        TokenNumber --> QueueDisplay[Queue Display]
        QueueDisplay --> OngoingQueue[Ongoing Queue]
        QueueDisplay --> CompletedQueue[Completed Queue]
        OngoingQueue --> CallPatient[Call Patient]
        CallPatient --> StartEncounter[Start Encounter]
        StartEncounter --> TokenEncLink[Link Token to Encounter]
    end

    subgraph QueueRedirect["Token Redirect"]
        TokenEncLink --> TokenRedirect[Token Encounter Redirect]
        TokenRedirect --> EncounterPage[Go to Encounter]
    end
```

---

### Billing & Accounts Flow

```mermaid
flowchart TD
    subgraph AccountManagement["Account Management"]
        Start([Patient]) --> CreateAccount[Create Patient Account]
        CreateAccount --> AccountDetails[Account Details]
        AccountDetails --> ViewCharges[View Charges]
    end

    subgraph ChargeItems["Charge Items"]
        ViewCharges --> AddCharges[Add Charges]
        AddCharges --> BedCharges[Bed Charges]
        AddCharges --> ServiceCharges[Service Charges]
        AddCharges --> MedCharges[Medication Charges]
        AddCharges --> ConsumableCharges[Consumable Charges]
    end

    subgraph InvoiceManagement["Invoice Management"]
        AddCharges --> CreateInvoice[Create Invoice]
        CreateInvoice --> ApplyDiscount[Apply Discount]
        ApplyDiscount --> ReviewInvoice[Review Invoice]
        ReviewInvoice --> FinalizeInvoice{Finalize?}
        FinalizeInvoice -->|Yes| InvoiceFinalized[Invoice Finalized]
        FinalizeInvoice -->|No| EditInvoice[Edit Invoice]
        EditInvoice --> ReviewInvoice
    end

    subgraph PaymentProcessing["Payment Processing"]
        InvoiceFinalized --> RecordPayment[Record Payment]
        RecordPayment --> PLUGIN_PAY[/"[PLUGIN: InvoiceRecordPaymentOptions]<br/>Custom payment methods"/]
        PLUGIN_PAY --> PayMethod{Payment Method}
        PayMethod --> Cash[Cash]
        PayMethod --> Card[Card]
        PayMethod --> Insurance[Insurance]
        PayMethod --> CustomPay[Plugin Payment]
    end

    subgraph Reconciliation["Reconciliation"]
        Cash --> Reconcile[Payment Reconciliation]
        Card --> Reconcile
        Insurance --> Reconcile
        CustomPay --> Reconcile
        Reconcile --> LocationFilter[Filter by Location]
        LocationFilter --> GenerateReceipt[Generate Receipt]
        GenerateReceipt --> AutoPrint[Auto Print]
    end

    style PLUGIN_PAY fill:#fff3e0,stroke:#ff9800
```

---

### Consent Management Flow

```mermaid
flowchart TD
    subgraph ConsentCreation["Consent Creation"]
        Start([Encounter]) --> ConsentsTab[Consents Tab]
        ConsentsTab --> AddConsent[Add Consent]
        AddConsent --> ConsentForm[Consent Form Sheet]
        ConsentForm --> SelectType[Select Consent Type]
        SelectType --> FillDetails[Fill Consent Details]
        FillDetails --> PatientSign[Patient Signature]
        PatientSign --> SaveConsent[Save Consent]
    end

    subgraph ConsentManagement["Consent Management"]
        SaveConsent --> ConsentList[Consent List]
        ConsentList --> ViewConsent[View Consent Detail]
        ViewConsent --> ConsentDetail[Consent Detail Page]
        ConsentDetail --> UpdateConsent[Update Consent]
        ConsentDetail --> RevokeConsent[Revoke Consent]
    end
```

---

### File Management Flow

```mermaid
flowchart TD
    subgraph FileUpload["File Upload"]
        Start([Patient/Encounter]) --> FilesTab[Files Tab]
        FilesTab --> UploadFile[Upload File]
        UploadFile --> SelectFile[Select File]
        SelectFile --> Compress{Compress?}
        Compress -->|Yes| CompressFile[Compress File]
        Compress -->|No| ProcessFile[Process File]
        CompressFile --> ProcessFile
        ProcessFile --> SaveFile[Save File]
    end

    subgraph FileManagement["File Management"]
        SaveFile --> FileList[File List]
        FileList --> ViewFile[View File]
        FileList --> DownloadFile[Download File]
        FileList --> ArchiveFile[Archive File]
        FileList --> DeleteFile[Delete File]
    end

    subgraph FileAccess["File Access Control"]
        ViewFile --> CheckAccess{Has Access?}
        CheckAccess -->|Yes| DisplayFile[Display File]
        CheckAccess -->|No| AccessDenied[Access Denied]
    end
```

---

### Questionnaire & Templates Flow

```mermaid
flowchart TD
    subgraph QuestionnaireManagement["Questionnaire Management"]
        Start([Admin]) --> QuestList[Questionnaire List]
        QuestList --> CreateQuest[Create Questionnaire]
        CreateQuest --> QuestEditor[Questionnaire Editor]
        QuestEditor --> AddQuestions[Add Questions]
        AddQuestions --> SetConditions[Set Conditions]
        SetConditions --> EncClassEval[Encounter Class Evaluator]
        EncClassEval --> SaveQuest[Save Questionnaire]
    end

    subgraph ResponseTemplates["Response Templates"]
        SaveQuest --> ResponseTemplates[Response Templates]
        ResponseTemplates --> ManageTemplates[Manage Templates Sheet]
        ManageTemplates --> FacOrgSelector[Facility Organization Selector]
        FacOrgSelector --> CreateTemplate[Create Template]
    end

    subgraph EncounterTemplates["Encounter Templates"]
        Start2([Facility]) --> TemplateBuilder[Template Builder]
        TemplateBuilder --> SelectQuests[Select Questionnaires]
        SelectQuests --> ArrangeOrder[Arrange Order]
        ArrangeOrder --> SaveTemplate[Save Template]
        SaveTemplate --> ApplyToEnc[Apply to Encounters]
    end

    subgraph ValueSets["ValueSet Management"]
        QuestEditor --> ValueSets[ValueSets]
        ValueSets --> CreateValueSet[Create ValueSet]
        CreateValueSet --> AddOptions[Add Options]
        AddOptions --> UseInQuest[Use in Questionnaire]
    end
```

---

### User Preferences Flow

```mermaid
flowchart TD
    subgraph Preferences["User Preferences"]
        Start([User]) --> Settings[User Settings]
        Settings --> Preferences[Preferences]
        Preferences --> PinnedLinks[Pinned Links]
    end

    subgraph PinPages["Pin Pages"]
        AnyPage([Any Page]) --> PinDialog[Pin Page Dialog]
        PinDialog --> SetLabel[Set Label]
        SetLabel --> ConfirmPin[Confirm Pin]
        ConfirmPin --> AddToSidebar[Add to Sidebar]
    end

    subgraph ManagePins["Manage Pins"]
        PinnedLinks --> ViewPins[View Pinned Pages]
        ViewPins --> ReorderPins[Reorder Pins]
        ViewPins --> UnpinPage[Unpin Page]
        ViewPins --> EditLabel[Edit Label]
    end
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

### Device Plugin Manifest

Plugins can provide custom device types with:

```typescript
interface PluginDeviceManifest {
  type: string;              // Device care_type
  icon: IconComponent;       // Custom device icon
  configureForm: Component;  // Device configuration UI
  showPageCard: Component;   // Device display card
  encounterOverview: Component; // Device data in encounter
}
```

### Encounter Tab Extensions

Plugins can add custom tabs to encounters:

| Built-in Tabs | Description |
|--------------|-------------|
| Overview | Encounter summary |
| Responses | Questionnaire responses |
| Observations | Vitals and observations |
| Medicines | Medications/Prescriptions |
| Notes | Clinical notes |
| Consents | Patient consents |
| Devices | Associated devices |
| Diagnostic Reports | Lab/diagnostic reports |
| Service Requests | Service requests |
| Files | Uploaded files |
| Plots | Data visualization |

Custom tabs via `encounterTabs` plugin manifest.

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
        Accounts[Accounts]
        Reconciliation[Reconciliation]
        PLUGIN_BILL[/"[PLUGIN: billingNavItems]"/]
    end

    subgraph UserNav["User Menu"]
        Profile[Profile]
        Settings[Settings]
        Preferences[Preferences]
        PLUGIN_USER[/"[PLUGIN: userNavItems]"/]
    end

    subgraph AdminNav["Admin Navigation"]
        Users[Users]
        Roles[Roles]
        Questionnaires[Questionnaires]
        Plugins[Plugins]
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
        Extends[Extension Types]
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

### Plugin Extension Types

```typescript
type SupportedPluginExtensions =
  | "DoctorConnectButtons"      // Extend doctor communication
  | "PatientExternalRegistration"; // External patient registration
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

    subgraph SchedulePerms["Schedule Permissions"]
        S1[can_list_booking]
        S2[can_write_booking]
        S3[can_reschedule_booking]
        S4[can_write_schedule]
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

## State Management

### Context Providers

| Context | Purpose | File |
|---------|---------|------|
| `PermissionContext` | Permission checking | `src/context/PermissionContext.tsx` |
| `ShortcutContext` | Keyboard shortcuts | `src/context/ShortcutContext.tsx` |
| `CareAppsContext` | Plugin management | `src/hooks/useCareApps.tsx` |
| `EncounterProvider` | Encounter state | `src/pages/Encounters/utils/EncounterProvider.tsx` |
| `AuthUserProvider` | Authentication state | `src/Providers/AuthUserProvider.tsx` |
| `PatientUserProvider` | Patient portal context | `src/Providers/PatientUserProvider.tsx` |
| `HistoryAPIProvider` | Navigation history | `src/Providers/HistoryAPIProvider.tsx` |

### Jotai Atoms (Global State)

| Atom | Purpose | File |
|------|---------|------|
| `developerMode` | Developer mode toggle | `src/atoms/developerMode.ts` |
| `encounterFilterAtom` | Encounter filter state | `src/atoms/encounterFilterAtom.ts` |
| `navExpansionAtom` | Sidebar expansion state | `src/atoms/navExpansionAtom.ts` |
| `paymentReconcilationLocationAtom` | Payment location filter | `src/atoms/paymentReconcilationLocationAtom.ts` |
| `pharmacyAtom` | Pharmacy queue state | `src/atoms/pharmacy.ts` |
| `scheduleServiceTypeAtom` | Schedule service type cache | `src/atoms/scheduleServiceTypeAtom.ts` |
| `userAtom` | Current user state | `src/atoms/user-atom.ts` |

---

## Recent Features

### Latest Updates (2025)

| Feature | Description | Related Flow |
|---------|-------------|--------------|
| Care Team Support | Care team user types and filtering | Care Team Management |
| Facility Organization Selector | Organization-specific templates | Questionnaire Templates |
| User Preferences | Pinned links/bookmarks | User Preferences |
| Schedule Service Type Caching | Performance optimization | Scheduling |
| Encounter Class in Medication | Encounter-aware medications | Medication Management |
| Token Encounter Linking | Queue-encounter integration | Appointment Queuing |
| Condition Editor Enhancements | Encounter class in conditions | Questionnaire Templates |
| External Links Integration | Links in definitions | Diagnostic Reports |
| Auto Print on Payment | Automatic receipt printing | Billing |
| Medication Notes in Billing | Display med notes in bills | Medication/Billing |

---

## Key Files Reference

| Category | File Path |
|----------|-----------|
| **Plugin System** | |
| Plugin Engine | `src/PluginEngine.tsx` |
| Plugin Types | `src/pluginTypes.ts` |
| Plugin Hooks | `src/hooks/useCareApps.tsx` |
| Plugin API | `src/types/plugConfig/plugConfigApi.ts` |
| Plugin Devices Hook | `src/pages/Facility/settings/devices/hooks/usePluginDevices.ts` |
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
| **State Management** | |
| Jotai Atoms | `src/atoms/` |
| Context Providers | `src/context/`, `src/Providers/` |
| **Key Components** | |
| Encounter Provider | `src/pages/Encounters/utils/EncounterProvider.tsx` |
| Care Team | `src/components/CareTeam/` |
| Medication | `src/components/Medication/` |
| Inventory | `src/pages/Facility/services/inventory/` |
| Questionnaire Editor | `src/components/Questionnaire/` |
| Template Builder | `src/pages/Encounters/TemplateBuilder/` |

---

## Legend

| Symbol | Meaning |
|--------|---------|
| `[PLUGIN: name]` | Plugin extension point - behavior can be customized via plugins |
| Orange boxes | Plugin-customizable components |
| Green boxes | Successful completion states |
| Blue boxes | Starting points |
| Dashed arrows | Permission relationships |
