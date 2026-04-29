# Architecture & Framework Diagrams

## 1. System Architecture - Where New APIs Fit

```mermaid
graph TB
    Client["Client / Webapp / cht-user-management"]
    NGINX["NGINX Proxy"]
    API["API Server (Express.js)"]
    Sentinel["Sentinel (Background Processor)"]
    CouchDB["CouchDB"]

    subgraph "New Contact Management APIs"
        Delete["POST /contact/:uuid/delete"]
        Move["POST /contact/:uuid/move"]
        Merge["POST /contact/:uuid/merge"]
    end

    Client -->|HTTP| NGINX
    NGINX -->|Proxy| API
    API --> Delete
    API --> Move
    API --> Merge
    Delete -->|bulkDocs| CouchDB
    Move -->|bulkDocs| CouchDB
    Merge -->|bulkDocs| CouchDB
    Sentinel -->|Watches changes| CouchDB
```

## 2. Service Layer Module Architecture

```mermaid
graph TD
    Controller["contact-management.js (Controller)"]
    Auth["auth.assertPermissions"]
    ServerUtils["serverUtils.doOrError"]

    subgraph "Service Layer (api/src/services/contact-management/)"
        Index["index.js — deleteContact / moveContact / mergeContact"]
        HierarchyData["hierarchy-data.js — CouchDB view queries"]
        Constraints["constraints.js — Hierarchy validation"]
        LineageOps["lineage-ops.js — Lineage manipulation"]
        ReportOps["report-ops.js — Report reassignment"]
    end

    DB["db.medic (CouchDB)"]

    Controller --> Auth
    Controller --> ServerUtils
    Controller --> Index
    Index --> HierarchyData
    Index --> Constraints
    Index --> LineageOps
    Index --> ReportOps
    HierarchyData --> DB
    ReportOps --> DB
    Index -->|bulkDocs| DB
```

## 3. Request Flow - Delete Contact Operation

```mermaid
sequenceDiagram
    participant C as Client
    participant R as routing.js
    participant Ctrl as Controller
    participant Auth as auth.js
    participant Svc as Service Layer
    participant HD as hierarchy-data.js
    participant RO as report-ops.js
    participant DB as CouchDB

    C->>R: POST /api/v1/contact/:uuid/delete
    R->>Ctrl: contactManagement.v1.delete
    Ctrl->>Auth: assertPermissions (can_delete_contacts)
    Auth-->>Ctrl: Authorized
    Ctrl->>Svc: deleteContact(uuid, options)
    Svc->>HD: getDescendants(uuid)
    HD->>DB: query contacts_by_depth view
    DB-->>HD: descendant docs
    HD-->>Svc: descendants[]
    loop For each descendant
        Svc->>RO: deleteReportsForContact(id)
        RO->>DB: query reports_by_subject
        RO->>DB: bulkDocs (delete reports)
    end
    Svc->>DB: bulkDocs (delete contacts)
    DB-->>Svc: write confirmed
    Svc-->>Ctrl: {ok, deleted_contacts, deleted_reports}
    Ctrl-->>C: 200 JSON response
```

## 4. Data Flow - Move vs Merge Operations

```mermaid
flowchart TD
    Start["Incoming Request"]

    Start --> Validate["Validate Constraints"]
    Validate --> CheckCircular{"Circular Hierarchy?"}
    CheckCircular -->|Yes| Reject["400 Error"]
    CheckCircular -->|No| CheckParent{"Valid Parent Type?"}
    CheckParent -->|No| Reject
    CheckParent -->|Yes| IsMerge{"Is Merge?"}

    IsMerge -->|No - Move| FetchDesc["Fetch Descendants via contacts_by_depth"]
    IsMerge -->|Yes - Merge| CheckType{"Same Contact Type?"}
    CheckType -->|No| Reject
    CheckType -->|Yes| FetchDesc

    FetchDesc --> RewriteLineage["Rewrite parent lineage in subtree"]
    RewriteLineage --> UpdateAncestors["Update ancestor primary contacts"]
    UpdateAncestors --> UpdateReports["Update report contact lineage via reports_by_freetext"]

    UpdateReports --> IsMerge2{"Is Merge?"}
    IsMerge2 -->|No| WriteDocs["bulkDocs write to CouchDB"]
    IsMerge2 -->|Yes| Reassign["Reassign report subjects (patient_id, place_id)"]
    Reassign --> DeleteSource["Mark source as _deleted: true"]
    DeleteSource --> WriteDocs
    WriteDocs --> Done["Return success response"]
```
