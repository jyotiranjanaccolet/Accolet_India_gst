# Accolet IndiaGST: Tripartite Relational Architecture

This document maps the data flow, state management, and operational workflows between the four core portals of the Accolet ecosystem:
1. **Client Dashboard** (`kpi_dashboard.html`)
2. **L4 Operations Portal** (`ca_dashboard.html`)
3. **L3 Review Center** (`l3_dashboard.html`)
4. **L2 Partner Command Center** (`partner_dashboard.html`)

Developers should use this document to design the backend database relations, API routing, and role-based access control (RBAC).

---

## 1. Core Compliance Workflow (GSTR-1 & GSTR-3B)

This is the primary pipeline for monthly return filing. Data originates from the L4 CA workspace, moves up to the L3 Data Reviewer for quality control, and is finally pushed to the Client for final sign-off.

```mermaid
sequenceDiagram
    autonumber
    actor L4 as L4 Data Preparer (Operations)
    actor L3 as L3 Data Reviewer (Quality Control)
    actor Client as CFO, Epson Ltd (Client Dashboard)
    
    Note over L4: State: DRAFTING
    L4->>L4: Uploads Excel & Validates Data
    L4->>L4: Completes Reconciliations (2B Match)
    L4->>L3: Clicks "Send to L3 for Review"
    
    Note over L3: State: L3_REVIEW
    L3->>L3: Receives Alert in 'Quality Approvals' Tab
    L3->>L3: Reviews GSTR-1/3B Summaries
    
    alt L3 Rejects
        L3-->>L4: Clicks "Reject to L4" (State resets to DRAFTING)
    else L3 Approves
        L3->>Client: Clicks "Approve & Send to Client"
    end
    
    Note over Client: State: PENDING_CLIENT_APPROVAL
    Client->>Client: Sees Return in 'Pending Approvals' Tab
    
    alt Client Queries Data
        Client-->>L3: Adds comments, clicks "Send Back to L3"
    else Client Approves
        Client->>Client: Clicks "Approve" 
        Note over Client: State: APPROVED_FOR_FILING
    end
```

---

## 2. Client Advice Hub (Ticketing System)

The "Seeking Advice" feature is a built-in support module. While the L4/L3 team handles routine work, complex tax queries are routed directly to the L2 Partner for expert legal opinions. The L2 Partner retains overarching visibility into the health of all clients.

```mermaid
sequenceDiagram
    autonumber
    actor Client as CFO, Epson Ltd
    actor L4 as L4 Data Preparer
    actor L2 as L2 Partner
    
    Note over Client: Tab 8: Seeking Advice
    Client->>Client: Fills out "Request New Advice" form
    Client->>L2: Submits Query (e.g., Rule 37 dispute)
    
    Note over L4: Tab 7: Client Queries
    L4-->>L4: Can view ticket (Read-Only context)
    
    Note over L2: Tab 2: Client Advice Hub
    L2->>L2: Sees Ticket in 'Awaiting Reply' queue
    L2->>L2: Drafts Legal/Tax opinion
    L2->>Client: Clicks "Send Reply to Client"
    
    Note over Client: Status changes to 'Solved'
    Client->>Client: Reads L2's expert response
```

---

## 3. Litigation & Notice Management

Notices issued by the GST Department follow a top-down assignment flow, ensuring the Partner maintains firm-wide oversight over legal risks.

```mermaid
sequenceDiagram
    autonumber
    participant GSTN as Govt GST Portal
    actor L2 as L2 Partner
    actor L4 as L4 Data Preparer
    actor Client as CFO, Epson Ltd
    
    GSTN->>L2: Issues Notice (e.g., ASMT-10)
    
    Note over L2: Tab 3: Notice Assignment
    L2->>L4: Assigns Notice to L4 Data Preparer with strict deadline
    
    Note over L4: Tab 6: Inbox & Tasks
    L4->>L4: Logs into workspace, sees Task "Working"
    L4->>L4: Prepares DRC-06 reply draft
    L4->>L2: Submits draft for review
    
    L2->>L2: Approves Draft
    L2->>GSTN: Files Reply on Portal
    
    Note over Client: Tab 6: Orders & Notices
    L2->>Client: Updates Status to "Filed"
    Client->>Client: Client can click "View Reply"
```

---

## 4. Database State Machine Relations

For the backend developer, entities like `GST_Returns` and `Advice_Tickets` must follow strict state transitions governed by the RBAC matrix.

### `GST_Returns` (GSTR-1, 3B, 6, 9)
*   `DRAFT` (Owned by L4 CA)
*   `L3_REVIEW` (Owned by L3 Data Reviewer; L4 is locked out)
*   `CLIENT_APPROVAL` (Owned by Client; Visible in Tab 5 of Client Dashboard)
*   `APPROVED_FOR_FILING` (Passed back to L4/L3 for API transmission to GSTN)
*   `FILED` (Terminal state; Triggers ARN generation in Client Tab 7)

### `Advice_Tickets`
*   `OPEN` (Created by Client)
*   `PARTNER_DRAFTING` (Partner is writing response)
*   `CLOSED_SOLVED` (Response sent to Client)

### `Notice_Lifecycle`
*   `UNASSIGNED` (Visible only to Partner)
*   `WORKING` (Assigned to CA; Visible to Client as "Working")
*   `PENDING_PARTNER` (CA submitted draft to Partner)
*   `FILED` (Reply submitted to Govt)
*   `UNDER_APPEAL` (Elevated legal status)
