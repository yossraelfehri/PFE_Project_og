# 📐 Deluge Workflows Design — Nexus Real Estate Management Platform

> Complete technical documentation of the Nexus platform Deluge scripts, illustrated by Mermaid diagrams covering architecture, flows, sequences, activities, state transitions, and class diagrams.

---

## 1. Global System Architecture

```mermaid
graph TB
    subgraph FRONTEND["🌐 Frontend — Node.js / HTML"]
        UI[User Interface]
        API[api-proxy.js]
    end

    subgraph CREATOR["☁️ Zoho Creator — Deluge Workflows"]
        direction TB
        WF_USER[Add_User\nDelete_User\nsend_reset_email\nsend_verification_email]
        WF_RES[status&calcul\nCheck_Property_Availability\ngenerer_contrat_location\nDuration_period]
        WF_PROP[Validate_Property_Fields\nApprove_Property\nReject_Property\nDelete_Property]
        WF_PUR[contrat_apres_acceptation\nnotify_seller_on_purchase]
        WF_CON[apres_signature_contrat\nextract_signed_pdf]
    end

    subgraph ZOHO_SERVICES["🔗 Integrated Zoho Services"]
        SIGN[Zoho Sign\nE-Signature]
        BOOKS[Zoho Books\nInvoicing]
        CRM[Zoho CRM\nLead Management]
        FLOW[Zoho Flow\nWebhooks]
        ANALYTICS[Zoho Analytics\nDashboards]
    end

    UI -->|HTTP Requests| API
    API -->|Zoho Creator API| CREATOR
    WF_CON -->|createUsingTemplate| SIGN
    WF_CON -->|createRecord| BOOKS
    WF_RES -->|sync_reservation_to_crm| CRM
    WF_PUR -->|sync_purchase_to_crm| CRM
    SIGN -->|Webhook completed| FLOW
    FLOW -->|Update Contract Status| CREATOR
    ANALYTICS -.->|Embedded Dashboard| UI
```

---

## 2. Class Diagram — Creator Data Model

```mermaid
classDiagram
    class User {
        +ID : String
        +ID1 : Integer
        +full_name : Object
        +Email : String
        +Phone_Number : String
        +Password : String
        +Role : Enum[User,Agent,Admin]
        +Reset_Token : String
        +Reset_Token_Expiry : String
        +Verification_Token : String
        +CRM_Contact_ID : String
    }

    class Property {
        +ID : String
        +ID1 : Integer
        +title : String
        +description : String
        +location : String
        +Price1 : Decimal
        +status : Enum
        +type_field : Enum
        +Surface1 : Integer
        +Rooms1 : Integer
        +Bathrooms1 : Integer
        +Floor : Integer
        +Year_Built : Date
        +prix_nuit : Decimal
        +loyer_mensuel : Decimal
        +caution_courte : Decimal
        +caution_longue : Decimal
        +Validation_Status : Enum
        +User : Lookup~User~
    }

    class Reservation {
        +ID : String
        +ID1 : Integer
        +Start_Date : Date
        +End_Date : Date
        +Status : Enum
        +Duration_days : Integer
        +Duration_Text : String
        +type_location : Enum
        +Montant : Decimal
        +Caution : Decimal
        +Commission_rate : Decimal
        +Commission_montant : Decimal
        +Net_Proprietaire : Decimal
        +Advance_Amount : Decimal
        +Advance_Payment_Status : Enum
        +Payment_Deadline : DateTime
        +CRM_Deal_ID : String
        +User : Lookup~User~
        +Property1 : Lookup~Property~
    }

    class Purchase {
        +ID : String
        +ID1 : Integer
        +Status : Enum
        +Request_Date : Date
        +Message : String
        +Preference_de_contact : String
        +Advance_Amount : Decimal
        +Advance_Payment_Status : Enum
        +Payment_Deadline : DateTime
        +CRM_Deal_ID : String
        +Buyer : Lookup~User~
        +Seller : Lookup~User~
        +Property : Lookup~Property~
    }

    class Contract {
        +ID : String
        +ID1 : Integer
        +type_field : Enum
        +Status : Enum
        +prix_total : Decimal
        +Creation_date : Date
        +Date_signature : Date
        +Zoho_Sign_ID : String
        +Contrat_PDF_URL : String
        +Signe_Acheteur : String
        +Signe_Vendeur : String
        +Buyer : Lookup~User~
        +Seller : Lookup~User~
        +Property : Lookup~Property~
        +Reservation : Lookup~Reservation~
        +Purchase : Lookup~Purchase~
    }

    class Payment {
        +ID : String
        +Amount : Decimal
        +Payment_Date : Date
        +Status : Enum
        +type_field : Enum
        +methode : Enum
        +Ref_transaction : String
        +Date_limite : Date
        +Commission_rate : Decimal
        +Commission_montant : Decimal
        +Net_Proprietaire : Decimal
        +Zoho_Books_Invoice_ID : String
        +Contract : Lookup~Contract~
    }

    User "1" --> "0..*" Property : owns
    User "1" --> "0..*" Reservation : makes
    User "1" --> "0..*" Purchase : submits
    Property "1" --> "0..*" Reservation : receives
    Property "1" --> "0..*" Purchase : receives
    Reservation "1" --> "0..1" Contract : generates
    Purchase "1" --> "0..1" Contract : generates
    Contract "1" --> "0..1" Payment : triggers
```

---

## 3. General Platform Flow

```mermaid
flowchart TD
    A([👤 User registers]) --> B{Email verified?}
    B -->|No| C[send_verification_email\nworkflow]
    C --> D[Clicks verification link]
    D --> B
    B -->|Yes| E[Account created in Creator\nAdd_User workflow]
    E --> F[sync_user_to_crm\nCRM Contact created]

    F --> G{Action type}

    G -->|Rental| H[Search property\nShort/Long term rental]
    G -->|Purchase| I[Search property\nFor sale]

    H --> J[status&calcul workflow\nValidation + Amount calculation]
    J --> K[10% advance payment\nCard simulation]
    K --> L[Advance_Payment_Status = Paid\nStatus = Confirmed]
    L --> M[generer_contrat_location\nworkflow]

    I --> N[Purchase created\nauto_set_purchase_fields]
    N --> O[notify_seller_on_purchase]
    O --> P[5% advance payment\nCard simulation]
    P --> Q[contrat_apres_acceptation\nworkflow]

    M --> R[Zoho Sign\nContract sent]
    Q --> R

    R --> S[Both parties sign]
    S --> T[Zoho Flow webhook\nStatus = Signed]
    T --> U[extract_signed_pdf\nPDF saved]
    T --> V[apres_signature_contrat\nFinal payment created]
    V --> W[Zoho Books\nInvoice generated]
```

---

## 4. Sequence Diagram — `status&calcul` Workflow (Reservation)

```mermaid
sequenceDiagram
    actor U as User
    participant FE as Frontend
    participant API as api-proxy.js
    participant C as Zoho Creator
    participant WF as status&calcul Workflow
    participant WF2 as Check_Property_Availability

    U->>FE: Selects dates + property
    FE->>API: POST /api/reservations/create
    API->>C: POST /form/Reservation

    C->>WF2: Trigger: Submit validation
    WF2->>C: Reservation[Property1 == X && Status == "Confirmed"]
    alt Dates overlap
        WF2-->>C: alert + cancel submit
        C-->>API: Workflow error
        API-->>FE: 400 - Property already booked
        FE-->>U: Error message
    else Available
        WF2-->>C: Validation OK
    end

    C->>WF: Trigger: Submit validation
    WF->>WF: Check property type
    alt For Sale
        WF-->>C: alert + cancel submit
    else Short-term rental less than 30 days
        WF->>WF: amount = prix_nuit x nbDays
        WF->>WF: Advance_Amount = amount x 10%
        WF->>WF: Commission_rate = 15%
    else Long-term rental 30+ days
        WF->>WF: amount = monthly_rent
        WF->>WF: Advance_Amount = amount x 10%
        WF->>WF: Commission_rate = 50%
    end

    WF->>C: input.Status = "Pending"
    WF->>C: input.Payment_Deadline = now + 1h
    C-->>API: Reservation created {ID}
    API-->>FE: { success, reservationId, advanceAmount }
    FE-->>U: Redirect to advance-payment.html
```

---

## 5. Sequence Diagram — Complete E-Signature Flow

```mermaid
sequenceDiagram
    actor B as Buyer/Tenant
    actor S as Seller/Owner
    participant API as api-proxy.js
    participant C as Creator
    participant WF as Contract Generation
    participant ZS as Zoho Sign
    participant ZF as Zoho Flow
    participant WF2 as extract_signed_pdf
    participant WF3 as apres_signature_contrat
    participant ZB as Zoho Books

    API->>C: PATCH Advance_Payment_Status = "Paid"
    API->>C: PATCH Status = "Confirmed"
    C->>WF: Trigger: Status updated
    WF->>C: insert into Contract [Draft]
    WF->>WF: Prepare fieldTextData + fieldDateData
    WF->>ZS: createUsingTemplate(templateId, params)
    ZS-->>WF: { request_id, status: "success" }
    WF->>C: update Contract [Zoho_Sign_ID, Status="Sent for signature"]
    WF->>B: Email "Contract ready to sign"
    WF->>S: Email "Contract sent"

    ZS->>B: Zoho Sign Email
    ZS->>S: Zoho Sign Email
    B->>ZS: Signs the document
    S->>ZS: Signs the document

    ZS->>ZF: Webhook "Document completed"
    ZF->>C: Update Contract Status = "Signed"

    C->>WF2: Trigger: Status = "Signed"
    WF2->>ZS: GET /requests/{Zoho_Sign_ID}
    ZS-->>WF2: { document_id }
    WF2->>C: update Contract [Contrat_PDF_URL, Date_signature]

    C->>WF3: Trigger: Status = "Signed" (Contract)
    WF3->>C: insert into Payment [Pending]
    WF3->>ZB: createRecord("contacts")
    ZB-->>WF3: { contact_id }
    WF3->>ZB: createRecord("invoices")
    ZB-->>WF3: { invoice_id }
    WF3->>C: update Payment [Zoho_Books_Invoice_ID]
    WF3->>B: Email "Payment required"
```

---

## 6. Sequence Diagram — Password Reset Flow

```mermaid
sequenceDiagram
    actor U as User
    participant FE as Frontend
    participant API as api-proxy.js
    participant C as Zoho Creator
    participant WF as send_reset_email

    U->>FE: Clicks "Forgot password"
    FE->>API: POST /api/auth/forgot-password {email}
    API->>C: GET All_Users?criteria=Email=="X"
    C-->>API: { user }
    API->>API: crypto.randomBytes(32) → token
    API->>API: expiry = Date.now() + 30min
    API->>C: PATCH All_Users/{userId}\n{Reset_Token, Reset_Token_Expiry}

    C->>WF: Trigger: Reset_Token field updated
    WF->>WF: resetLink = localhost:3000/reset.html?token=X&email=Y
    WF->>U: Reset email sent

    API-->>FE: { success: true }
    FE-->>U: "Email sent"

    U->>FE: Clicks link → reset.html
    FE->>API: POST /api/auth/verify-reset-token {email, token}
    API->>C: GET All_Users?criteria=Email=="X"
    API->>C: GET All_Users/{userId}?field_config=all
    C-->>API: { Reset_Token, Reset_Token_Expiry }
    API->>API: Verify token + expiry
    API-->>FE: { valid: true }

    U->>FE: Enters new password
    FE->>API: POST /api/auth/reset-password\n{email, token, newPassword}
    API->>C: PATCH All_Users/{userId}\n{Password, Reset_Token:"", Reset_Token_Expiry:""}
    API-->>FE: { success: true }
    FE-->>U: Redirect to login after 3s
```

---

## 7. Activity Diagram — `Add_User` Workflow

```mermaid
flowchart TD
    START([▶ User form submitted]) --> A[Get input.Password]
    A --> B{Starts with\n$2b$ or $2a$?}
    B -->|Yes — already hashed| G
    B -->|No| C{length >= 8?}
    C -->|No| E1[alert: 8 characters minimum\ncancel submit]
    C -->|Yes| D{Contains uppercase?}
    D -->|No| E2[alert: uppercase required\ncancel submit]
    D -->|Yes| E{Contains digit?}
    E -->|No| E3[alert: digit required\ncancel submit]
    E -->|Yes| F{Contains special char\n!@#$%^&*()}
    F -->|No| E4[alert: special character required\ncancel submit]
    F -->|Yes| G[User[Email == input.Email]]
    G --> H{Email already exists?}
    H -->|Yes| E5[alert: email already in use\ncancel submit]
    H -->|No| I{Role empty?}
    I -->|Yes| J[input.Role = 'User']
    I -->|No| K
    J --> K([✅ Record creation allowed])
    E1 --> STOP([❌ End])
    E2 --> STOP
    E3 --> STOP
    E4 --> STOP
    E5 --> STOP
```

---

## 8. Activity Diagram — `generer_contrat_location` Workflow

```mermaid
flowchart TD
    START([▶ Advance_Payment_Status = Paid]) --> A{Contract already exists\nfor this reservation?}
    A -->|Yes| STOP([❌ return — duplicate])
    A -->|No| B[Fetch Reservation, Property, Seller]
    B --> C[insert into Contract\nStatus = Draft]
    C --> D[deadline = today + 7 days]
    D --> E{type_location?}

    E -->|Short-term rental| F1[templateId = 48394\nfieldTextData: ref_con, nom_loc,\nnom_prop, surface_bien,\nprix/nuit, nb_nuit...\nfieldDateData: Date_deb, Date_fin]
    E -->|Long-term rental| F2[templateId = 48344\nfieldTextData: ref_contrat, nom_loc,\nnom_b, surface, nb_piece, etage,\nLoyer_mensuel, equiv...\nfieldDateData: Date_priseEff,\nDate_ech, Date_limite]
    E -->|Other| STOP2([❌ No matching template])

    F1 --> G[buyerAction + sellerAction\nMaps with action_id and role]
    F2 --> G
    G --> H[zoho.sign.createUsingTemplate\ntemplate, parameters, zoho_sign]
    H --> I{status == success?}
    I -->|No| J[log Zoho Sign error]
    J --> STOP3([❌ End with error])
    I -->|Yes| K[request_id = resp.requests.request_id]
    K --> L[update Contract\nZoho_Sign_ID = request_id\nStatus = Sent for signature]
    L --> M[sendmail → Buyer]
    M --> N[sendmail → Seller]
    N --> END([✅ Contract sent for signature])
```

---

## 9. State Transition Diagram — Reservation

```mermaid
stateDiagram-v2
    [*] --> Pending : Creation\nstatus&calcul workflow
    Pending --> Confirmed : Advance payment\nAPI PATCH
    Pending --> Cancelled : User cancellation\nAPI cancel
    Pending --> [*] : Deadline expired\nScheduled workflow\nauto-delete
    Confirmed --> Cancelled : Cancellation\nadmin/user
    Confirmed --> ContractGenerated : generer_contrat_location\nworkflow

    state ContractGenerated {
        [*] --> SentForSignature : Zoho Sign\ncreateUsingTemplate
        SentForSignature --> Signed : Both parties sign\nZoho Flow webhook
    }

    ContractGenerated --> Completed : Final payment\nconfirmed
```

---

## 10. State Transition Diagram — Purchase Request

```mermaid
stateDiagram-v2
    [*] --> Pending : Purchase created\nauto_set_purchase_fields

    Pending --> Accepted : 5% advance payment\nAPI PATCH Status=Accepted
    Pending --> Cancelled : User cancellation
    Pending --> [*] : Deadline expired\ndelete_unpaid_purchases

    Accepted --> ContractSent : contrat_apres_acceptation\nZoho Sign

    state ContractSent {
        [*] --> SentForSignature : createUsingTemplate
        SentForSignature --> Signed : Zoho Flow Webhook
    }

    ContractSent --> FinalPayment : apres_signature_contrat\nPayment created
    FinalPayment --> Completed : Payment confirmed\nProperty = Sold
```

---

## 11. State Transition Diagram — Contract

```mermaid
stateDiagram-v2
    [*] --> Draft : Insert into Contract\nGeneration workflow

    Draft --> SentForSignature : Zoho Sign\ncreateUsingTemplate success
    Draft --> [*] : Zoho Sign error

    SentForSignature --> Signed : Zoho Flow webhook\nDocument completed

    Signed --> PDFExtracted : extract_signed_pdf\nworkflow
    PDFExtracted --> PaymentCreated : apres_signature_contrat\nworkflow

    note right of Signed
        Date_signature = today
        Contrat_PDF_URL = Zoho Sign URL
    end note

    note right of PaymentCreated
        Payment record created
        Zoho Books invoice generated
        Email sent to buyer
    end note
```

---

## 12. State Transition Diagram — Payment

```mermaid
stateDiagram-v2
    [*] --> Pending : apres_signature_contrat\nInsert into Payment

    Pending --> Successful : Admin confirms\nor Zoho Books validated
    Pending --> Failed : Timeout or\nbank rejection
    Pending --> Expired : Date_limite exceeded

    Successful --> [*] : Property updated\nSold or Rented
    Failed --> Pending : New attempt
    Expired --> [*] : Archived
```

---

## 13. State Transition Diagram — Property

```mermaid
stateDiagram-v2
    [*] --> Pending : Owner publishes\nValidation_Status = pending

    Pending --> Approved : Admin approves\nApprove_Property workflow
    Pending --> Rejected : Admin rejects\nReject_Property workflow

    Rejected --> [*] : Deleted\nauto Delete_Property

    Approved --> Available : Visible on the site
    Available --> UnderNegotiation : Purchase accepted
    Available --> Rented : Reservation confirmed\n+ contract signed
    UnderNegotiation --> Sold : Final payment successful
    Rented --> Available : End of rental period
    Sold --> [*] : Archived
```

---

## 14. Sequence Diagram — CRM Synchronization

```mermaid
sequenceDiagram
    participant C as Zoho Creator
    participant WF1 as sync_user_to_crm
    participant WF2 as sync_reservation_to_crm
    participant WF3 as sync_purchase_to_crm
    participant CRM as Zoho CRM

    Note over C,CRM: Workflow 1 — New user registered
    C->>WF1: Trigger: User created
    WF1->>CRM: createRecord("Contacts", contactMap)
    CRM-->>WF1: { contact_id }
    WF1->>C: update User [CRM_Contact_ID]

    Note over C,CRM: Workflow 2 — Reservation confirmed
    C->>WF2: Trigger: Status = "Confirmed"
    WF2->>C: Fetch buyer.CRM_Contact_ID
    WF2->>CRM: createRecord("Deals", dealMap)\nStage = Qualification
    CRM-->>WF2: { deal_id }
    WF2->>C: update Reservation [CRM_Deal_ID]

    Note over C,CRM: Workflow 3 — Purchase accepted
    C->>WF3: Trigger: Status = "Accepted"
    WF3->>CRM: createRecord("Deals", dealMap)\nStage = Qualification
    CRM-->>WF3: { deal_id }
    WF3->>C: update Purchase [CRM_Deal_ID]

    Note over C,CRM: Pipeline update — Contract signed
    C->>CRM: updateRecord("Deals")\nStage = Contract Sent

    Note over C,CRM: Pipeline update — Payment successful
    C->>CRM: updateRecord("Deals")\nStage = Closed Won
```

---

## 15. Flow Diagram — Automatic Deletion (Payment Deadline)

```mermaid
flowchart TD
    SCHED([⏰ Scheduled Workflow\nevery hour]) --> A

    A[Fetch Reservations\nAdvance_Payment_Status == Pending\nPayment_Deadline less than now]
    A --> B{Records found?}
    B -->|No| END1([✅ Nothing to delete])
    B -->|Yes| C[For each expired reservation]
    C --> D[delete from Reservation\nID == reservation.ID]
    D --> E{Next?}
    E -->|Yes| C
    E -->|No| F

    F[Fetch Purchases\nAdvance_Payment_Status == Pending\nPayment_Deadline less than now]
    F --> G{Records found?}
    G -->|No| END2([✅ Nothing to delete])
    G -->|Yes| H[For each expired purchase]
    H --> I[delete from Purchase\nID == purchase.ID]
    I --> J{Next?}
    J -->|Yes| H
    J -->|No| END3([✅ Cleanup complete])
```

---

## 16. Activity Diagram — `apres_signature_contrat` Workflow

```mermaid
flowchart TD
    START([▶ Contract Status = Signed]) --> A{Payment already exists\nfor this contract?}
    A -->|Yes| STOP([❌ return — duplicate])
    A -->|No| B{type_field?}

    B -->|Purchase| C1[amount = prix_total x 10%\ncommission_rate = 3%\ntype = Deposit]
    B -->|Short-term rental| C2[amount = prix_total\ncommission_rate = 15%\ntype = Full payment]
    B -->|Long-term rental| C3[amount = prix_total\ncommission_rate = 50%\ntype = First month]
    B -->|Other| C4[amount = prix_total x 10%\ncommission_rate = 3%\ntype = Deposit]

    C1 --> D
    C2 --> D
    C3 --> D
    C4 --> D

    D[commission = amount x rate / 100\nnet_owner = amount - commission\ndeadline = today + 7 days\nref = PAY-ID-date]
    D --> E[insert into Payment\nStatus=Pending, method=Wire transfer]
    E --> F[Zoho Books\ncreateRecord contacts]
    F --> G[Zoho Books\ncreateRecord invoices]
    G --> H[update Payment\nZoho_Books_Invoice_ID]
    H --> I[sendmail to Buyer\nPayment details]
    I --> END([✅ Payment and invoice created])
```

---

## 17. Synthetic View — Workflow Interactions

```mermaid
graph LR
    subgraph USER_FORM["📋 User Form"]
        AU[Add_User]
        DU[Delete_User]
        SRE[send_reset_email]
        SVE[send_verification_email]
    end

    subgraph PROP_FORM["🏠 Property Form"]
        VPF[Validate_Property_Fields]
        APW[Approve_Property]
        RPW[Reject_Property]
        DPW[Delete_Property]
    end

    subgraph RES_FORM["📅 Reservation Form"]
        SC[status&calcul]
        CPA[Check_Property_Availability]
        GCL[generer_contrat_location]
        DP[Duration_period]
    end

    subgraph PUR_FORM["🛒 Purchase Form"]
        CAA[contrat_apres_acceptation]
        NSP[notify_seller_on_purchase]
    end

    subgraph CON_FORM["📄 Contract Form"]
        ASC[apres_signature_contrat]
        ESP[extract_signed_pdf]
    end

    subgraph SCHEDULED["⏰ Scheduled"]
        DUR[delete_unpaid_reservations]
        DUP[delete_unpaid_purchases]
    end

    AU -->|CRM Contact| CRM[(Zoho CRM)]
    GCL -->|E-Sign| ZS[(Zoho Sign)]
    CAA -->|E-Sign| ZS
    ASC -->|Invoice| ZB[(Zoho Books)]
    ZS -->|Webhook| ZF[(Zoho Flow)]
    ZF -->|Status Update| ESP
    ZF -->|Status Update| ASC
    SC -->|CRM Deal| CRM
    NSP -->|Email| MAIL[(Zoho Mail)]
    SRE -->|Email| MAIL
    SVE -->|Email| MAIL
```

---

*Documentation generated for the Final Year Project (PFE) — Nexus Real Estate Management Platform — Bachelor of Information Technology, Specialization: Information Systems Development*
