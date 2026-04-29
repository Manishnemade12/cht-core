Contact Management REST APIs for CHT-Core
Name, Contact Information & Skills
Name: [Your Name]
Email: [
your.email@example.com
]
GitHub: [github.com/your-username]
Portfolio: [your-portfolio-link]
Skills: JavaScript (ES6+), Node.js, TypeScript, CouchDB/PouchDB, REST API Design, Docker, Mocha/Chai/Sinon Testing, Express.js, Git, Linux
Title
Contact Management REST APIs for CHT-Core: Implementing Delete, Move & Merge Contact Operations

Summary
This proposal presents a modular, incremental approach to implementing three new REST API endpoints in CHT-Core for contact management operations — delete, move, and merge — currently only available via the cht-conf CLI tool. The implementation adapts the proven logic from cht-conf's HierarchyOperations module into cht-core's API layer, following the established controller patterns (
person.js
, 
place.js
, 
contact.js
) and authentication model (auth.assertPermissions). Each endpoint is built as a self-contained module within api/src/controllers/ and api/src/services/, backed by a shared hierarchy-operations service library in shared-libs/. A key architectural decision addresses a TODO in the problem statement — for hierarchy operations involving large subtrees, a synchronous-first approach with configurable timeout fallback is proposed, where small operations complete synchronously while large operations can optionally leverage a polling-based status pattern. The work is scoped across 10 weeks with clear milestones, comprehensive test coverage (unit + integration), and full API documentation.

Project Details
1. Architectural Understanding & Analysis
1.1 Current Architecture
The CHT-Core system follows a monorepo architecture with the API server (api/), background processor (sentinel/), and webapp (webapp/) as primary services. The API layer uses Express.js with controllers in api/src/controllers/ and services in api/src/services/, backed by CouchDB.

Existing Controller Patterns (Reference):

api/src/controllers/person.js
 — Uses @medic/cht-datasource bindings via ctx.bind(), auth via auth.assertPermissions({ isOnline: true, hasAny: [...] }), and serverUtils.doOrError for error wrapping
api/src/controllers/place.js
 — Same pattern with Place namespace
api/src/controllers/bulk-docs.js
 — Handles batched delete operations with batchSize: 50
Route Registration Pattern (from 
api/src/routing.js
):

javascript
app.postJson('/api/v1/person', person.v1.create);
app.putJson('/api/v1/person/:uuid', person.v1.update);
app.get('/api/v1/contact/:uuid', contact.v1.get);
Authentication Pattern (from 
api/src/auth.js
):

javascript
// hasAny pattern — at least one permission needed
await auth.assertPermissions(req, { isOnline: true, hasAny: ['can_delete_contacts', 'can_edit'] });
// hasAll pattern — all permissions needed
await auth.assertPermissions(req, { isOnline: true, hasAll: ['can_edit'] });
1.2 cht-conf Reference Implementation Analysis
All three operations in cht-conf delegate to a unified HierarchyOperations module:

cht-conf/src/lib/hierarchy-operations/
├── index.js              # Orchestrator — exports move(), merge(), delete()
├── delete-hierarchy.js   # Recursive delete with report cleanup
├── hierarchy-data-source.js  # CouchDB query layer (views, batching)
├── lineage-manipulation.js   # Lineage tree operations (create, replace, minify)
├── lineage-constraints.js    # Hierarchy validation rules
├── replace-lineage.js        # Parent/contact lineage replacement
└── jsdocFolder.js            # File-system staging (not needed for API)
Key Architectural Insights from cht-conf:

Operation	Core Function	Key Logic
Delete	deleteHierarchy()	Fetch descendants via contacts_by_depth view → mark each contact as _deleted: true → delete all reports via reports_by_subject view → optionally set cht_disable_linked_users for places
Move	moveHierarchy(merge=false)	Validate hierarchy constraints (parent type, circular check) → fetch descendants → replace parent lineage in subtree → update ancestor primary contacts → rebind report creator lineage via reports_by_freetext view
Merge	moveHierarchy(merge=true)	Same as move, plus: validate same contact type → mark source as _deleted: true → reassign report subjects (patient_id, patient_uuid, place_id, place_uuid) to destination → optionally merge primary contacts
Critical CouchDB Views Used:

medic/contacts_by_depth — Fetches all descendants of a contact (key: [contactId])
medic-client/reports_by_subject — Fetches reports by subject IDs (patient_id, place_id)
medic-client/reports_by_freetext — Fetches reports by creator contact ID (key: contact:<id>)
Batching Strategy: Uses BATCH_SIZE = 10000 for reports, processes in paginated batches to prevent memory exhaustion.

2. Proposed API Endpoints
2.1 Delete Contact
POST /api/v1/contact/:uuid/delete
Field	Details
Method	POST
Path	/api/v1/contact/:uuid/delete
Auth	hasAny: ['can_delete_contacts', 'can_edit']
Body	{ "disable_users": boolean } (optional)
Response	{ "ok": true, "deleted_contacts": number, "deleted_reports": number }
Error Codes	400 (invalid input), 401 (not logged in), 403 (insufficient permissions), 404 (contact not found)
Why POST instead of DELETE? The operation has a JSON request body with options (disable_users), and the HTTP DELETE method conventionally does not carry a body. This follows the existing bulk-delete pattern in cht-core.

2.2 Move Contact
POST /api/v1/contact/:uuid/move
Field	Details
Method	POST
Path	/api/v1/contact/:uuid/move
Auth	hasAny: ['can_move_contacts', 'can_edit']
Body	{ "parent": "<destination_uuid>" | "root" }
Response	{ "ok": true, "updated_contacts": number, "updated_reports": number }
Error Codes	400 (hierarchy constraint violation, circular hierarchy, invalid parent type), 401, 403, 404
2.3 Merge Contact
POST /api/v1/contact/:uuid/merge
Field	Details
Method	POST
Path	/api/v1/contact/:uuid/merge
Auth	hasAny: ['can_merge_contacts', 'can_edit']
Body	{ "destination": "<surviving_contact_uuid>", "disable_users": boolean, "merge_primary_contacts": boolean }
Response	{ "ok": true, "moved_contacts": number, "updated_reports": number, "deleted_contacts": number }
Error Codes	400 (type mismatch, same contact), 401, 403, 404
3. Technical Architecture & Module Design
3.1 New File Structure
api/src/
├── controllers/
│   └── contact-management.js    [NEW] — Route handlers for delete/move/merge
└── services/
    └── contact-management/      [NEW] — Core business logic
        ├── index.js             — Public API (deleteContact, moveContact, mergeContact)
        ├── hierarchy-data.js    — CouchDB queries (adapted from hierarchy-data-source.js)
        ├── lineage-ops.js       — Lineage manipulation (adapted from lineage-manipulation.js)
        ├── constraints.js       — Hierarchy constraint validation
        └── report-ops.js        — Report reassignment and cleanup
shared-libs/
└── contact-types-utils/         [EXISTING] — Leveraged for contact type validation
api/tests/mocha/
├── controllers/
│   └── contact-management.spec.js   [NEW] — Controller unit tests
└── services/
    └── contact-management/
        ├── index.spec.js            [NEW] — Service integration tests
        ├── hierarchy-data.spec.js   [NEW]
        ├── lineage-ops.spec.js      [NEW]
        ├── constraints.spec.js      [NEW]
        └── report-ops.spec.js       [NEW]
3.2 Controller Design
javascript
// api/src/controllers/contact-management.js
const auth = require('../auth');
const serverUtils = require('../server-utils');
const contactMgmt = require('../services/contact-management');
module.exports = {
  v1: {
    delete: serverUtils.doOrError(async (req, res) => {
      await auth.assertPermissions(req, {
        isOnline: true,
        hasAny: ['can_delete_contacts', 'can_edit']
      });
      const { uuid } = req.params;
      const { disable_users } = req.body || {};
      const result = await contactMgmt.deleteContact(uuid, { disableUsers: !!disable_users });
      return res.json(result);
    }),
    move: serverUtils.doOrError(async (req, res) => {
      await auth.assertPermissions(req, {
        isOnline: true,
        hasAny: ['can_move_contacts', 'can_edit']
      });
      const { uuid } = req.params;
      const { parent } = req.body;
      if (!parent) {
        return serverUtils.error({ status: 400, message: 'Missing required field: parent' }, req, res);
      }
      const result = await contactMgmt.moveContact(uuid, parent);
      return res.json(result);
    }),
    merge: serverUtils.doOrError(async (req, res) => {
      await auth.assertPermissions(req, {
        isOnline: true,
        hasAny: ['can_merge_contacts', 'can_edit']
      });
      const { uuid } = req.params;
      const { destination, disable_users, merge_primary_contacts } = req.body || {};
      if (!destination) {
        return serverUtils.error({ status: 400, message: 'Missing required field: destination' }, req, res);
      }
      const result = await contactMgmt.mergeContact(uuid, destination, {
        disableUsers: !!disable_users,
        mergePrimaryContacts: !!merge_primary_contacts,
      });
      return res.json(result);
    }),
  },
};
3.3 Route Registration
javascript
// In api/src/routing.js — add alongside existing contact routes
const contactManagement = require('./controllers/contact-management');
app.postJson('/api/v1/contact/:uuid/delete', contactManagement.v1.delete);
app.postJson('/api/v1/contact/:uuid/move', contactManagement.v1.move);
app.postJson('/api/v1/contact/:uuid/merge', contactManagement.v1.merge);
3.4 Service Layer Architecture
The service layer adapts cht-conf's HierarchyOperations logic to work directly with the cht-core CouchDB database (via api/src/db.js) instead of PouchDB. The key difference is that instead of writing docs to disk (as cht-conf does for its two-step delete-contacts + upload-docs workflow), the service directly performs bulk writes via db.medic.bulkDocs().

javascript
// api/src/services/contact-management/index.js
const db = require('../../db');
const hierarchyData = require('./hierarchy-data');
const lineageOps = require('./lineage-ops');
const constraints = require('./constraints');
const reportOps = require('./report-ops');
const deleteContact = async (contactId, options = {}) => {
  // 1. Fetch the contact doc
  const contactDoc = await hierarchyData.getContact(db.medic, contactId);
  
  // 2. Fetch all descendants via contacts_by_depth view
  const descendants = await hierarchyData.getDescendants(db.medic, contactId);
  
  // 3. Load constraint checker
  const checker = await constraints.load(db.medic);
  
  // 4. Mark all contacts (including self) as deleted
  const deletedContacts = descendants.map(doc => ({
    _id: doc._id,
    _rev: doc._rev,
    _deleted: true,
    cht_disable_linked_users: options.disableUsers && checker.isPlace(doc),
  }));
  
  // 5. Delete associated reports in batches
  let deletedReports = 0;
  for (const descendant of descendants) {
    deletedReports += await reportOps.deleteReportsForContact(db.medic, descendant._id);
  }
  
  // 6. Bulk-write deleted contacts
  await db.medic.bulkDocs(deletedContacts);
  
  return {
    ok: true,
    deleted_contacts: deletedContacts.length,
    deleted_reports: deletedReports,
  };
};
3.5 Long-Running Operations: Design Decision
TODO from problem statement: These actions may involve long-running processes. We need to determine if we should block our REST response or implement a polling pattern.

Proposed Approach: Synchronous with Batched Writes

After analyzing the existing patterns in cht-core, I propose using a synchronous approach for the initial implementation, which aligns with how all existing endpoints work. The rationale:

Consistency: Every existing cht-core API endpoint blocks until completion — bulk-delete, create, update all follow this pattern.
Simplicity: A polling pattern introduces significant complexity (job queue, status storage, cleanup) that may be premature for an initial implementation.
Batching mitigates memory issues: By processing reports in batches of 10,000 (following cht-conf's BATCH_SIZE) and performing bulkDocs in smaller batches (e.g., 500 docs), memory usage stays bounded.
Client timeout handling: If a client disconnects due to timeout, the server-side operation continues to completion — CouchDB writes are atomic per bulkDocs call, so partial completion is safe.
Future Enhancement (Phase 2, if needed): If mentors determine that a polling pattern is required for very large hierarchies, the architecture supports easy extension:

POST /api/v1/contact/:uuid/delete → 202 { "operation_id": "..." }
GET /api/v1/operation/:id → { "status": "in_progress", "progress": { ... } }
The service layer already separates the operation logic from the controller, so adding an async wrapper would only require changes to the controller layer.

4. Hierarchy Data Layer Adaptation
The core data layer adapts cht-conf's hierarchy-data-source.js to use cht-core's db module:

javascript
// api/src/services/contact-management/hierarchy-data.js
const BATCH_SIZE = 10000;
const HIERARCHY_ROOT = 'root';
const getContact = async (db, contactId) => {
  if (contactId === HIERARCHY_ROOT) return undefined;
  try {
    return await db.get(contactId);
  } catch (err) {
    if (err.status === 404) {
      throw { status: 404, message: `Contact '${contactId}' not found` };
    }
    throw err;
  }
};
const getDescendants = async (db, contactId) => {
  const result = await db.query('medic/contacts_by_depth', {
    key: [contactId],
    include_docs: true,
  });
  return result.rows.map(row => row.doc).filter(Boolean);
};
const getAncestors = async (db, contactDoc) => {
  const ancestorIds = pluckIdsFromLineage(contactDoc.parent);
  const result = await db.allDocs({ keys: ancestorIds, include_docs: true });
  return result.rows.map(row => row.doc).filter(Boolean);
};
5. Lineage Constraint Validation
Constraints are ported from cht-conf's lineage-constraints.js:

Constraint	Move	Merge	Delete
Contact exists	✅	✅	✅
No circular hierarchy	✅	—	—
Parent type allowed	✅	—	—
Same contact type	—	✅ (source & destination)	—
No self-reference	✅	✅	—
Primary contact safety	✅ (descendant can't be ancestor's primary)	✅	—
No same-lineage batch ops	✅	✅	—
6. Work Breakdown Structure & Timeline
Phase 1: Foundation & Delete (Weeks 1–3) — Required
Week	Deliverable	Status
Week 1	Investigation & Setup	Required
Set up local dev environment (Docker, CouchDB, Node.js 22)	
Deep-dive into cht-conf HierarchyOperations with mentor guidance	
Create api/src/services/contact-management/ module skeleton	
Implement hierarchy-data.js (CouchDB query abstraction)	
Write unit tests for hierarchy-data.js	
Week 2	Delete Contact Implementation	Required
Implement deleteContact() in service layer	
Implement recursive descendant deletion via contacts_by_depth	
Implement batched report deletion via reports_by_subject	
Implement cht_disable_linked_users for place contacts	
Wire up controller and route (POST /api/v1/contact/:uuid/delete)	
Write comprehensive unit tests (controller + service)	
Week 3	Delete Testing & Refactoring	Required
Write integration tests for delete endpoint	
Edge case testing (contact not found, no children, large subtrees)	
Code review and mentor feedback incorporation	
Phase 2: Move Contact (Weeks 4–6) — Required
Week	Deliverable	Status
Week 4	Move Contact Core Logic	Required
Implement constraints.js (hierarchy validation rules)	
Implement lineage-ops.js (lineage replacement, minification)	
Unit tests for constraints and lineage operations	
Week 5	Move Contact Integration	Required
Implement moveContact() orchestrator in service layer	
Handle subtree lineage updates (all descendants)	
Handle ancestor primary contact updates	
Implement report creator lineage updates via reports_by_freetext	
Wire up controller and route (POST /api/v1/contact/:uuid/move)	
Week 6	Move Testing	Required
Unit tests for moveContact controller + service	
Integration tests for move endpoint	
Edge case testing (move to root, cross-type violations, circular hierarchy)	
🏁 Mid-Point Milestone (End of Week 6):

✅ Delete-contacts API endpoint functional with unit and integration tests
✅ Move-contacts API endpoint functional with unit and integration tests
Phase 3: Merge Contact (Weeks 7–9) — Required
Week	Deliverable	Status
Week 7	Merge Contact Core Logic	Required
Implement report-ops.js (report subject reassignment)	
Implement merge-specific constraints (same type, primary contact merge)	
Report reassignment for patient_id, patient_uuid, place_id, place_uuid	
Week 8	Merge Contact Integration	Required
Implement mergeContact() orchestrator in service layer	
Handle source contact deletion after merge	
Handle optional primary contact merging	
Wire up controller and route (POST /api/v1/contact/:uuid/merge)	
Unit tests for all merge components	
Week 9	Merge Testing & Integration	Required
Integration tests for merge endpoint	
Cross-endpoint testing (merge + move interactions)	
Performance testing with large hierarchies	
Phase 4: Documentation & Polish (Week 10) — Required
Week	Deliverable	Status
Week 10	Documentation & Final Review	Required
Write OpenAPI documentation for all 3 endpoints (following existing @openapi JSDoc patterns)	
Write API documentation for cht-docs site	
Final code review and mentor feedback	
Address any outstanding review comments	
Optional Deliverables (If Time Permits)
Deliverable	Description
Polling pattern for long-running operations	POST /api/v1/operation/:id status endpoint
Dry-run mode	?dry_run=true query parameter to preview affected documents without writing
Batch operations	Support comma-separated contact IDs for batch delete/move
7. Testing Strategy
7.1 Unit Tests (Mocha + Chai + Sinon)
Following cht-core conventions, unit tests mirror source paths:

api/tests/mocha/controllers/contact-management.spec.js
api/tests/mocha/services/contact-management/index.spec.js
api/tests/mocha/services/contact-management/hierarchy-data.spec.js
api/tests/mocha/services/contact-management/lineage-ops.spec.js
api/tests/mocha/services/contact-management/constraints.spec.js
api/tests/mocha/services/contact-management/report-ops.spec.js
Run with:

bash
npm run unit-api
# Or specifically:
npx mocha api/tests/mocha/controllers/contact-management.spec.js
npx mocha api/tests/mocha/services/contact-management/**/*.spec.js
Key test scenarios:

Auth: Verify 403 for missing permissions, 401 for unauthenticated
Delete: Single contact, contact with children, contact with reports, place with disable_users
Move: Valid move, circular hierarchy rejection, invalid parent type, move to root
Merge: Same-type merge, type mismatch rejection, report subject reassignment, primary contact merge
7.2 Integration Tests
tests/integration/api/controllers/contact-management.spec.js
Run with:

bash
npm run integration-api
Key integration scenarios:

Full delete workflow with CouchDB verification
Move with lineage hierarchy verification post-move
Merge with report reassignment verification
Availability
Hours per week: 35–40 hours/week (full-time commitment)
Other commitments: No other internships or substantial commitments during the coding period (July–August). I can dedicate focused time to this project.
Timezone: IST (UTC+5:30) — overlapping with mentor availability
Personal Information & Motivation
About Me
[Share your background — e.g., year of study, university, relevant coursework in databases, distributed systems, or API design]

Open Source Experience
[Share your open-source contributions — any PRs, issues solved, or projects contributed to. Mention any experience with:]

Node.js/Express API development
CouchDB or document databases
Testing frameworks (Mocha, Chai, Sinon)
Healthcare technology or community-focused projects
[If you've solved any cht-core issues, mention them here!]
Key Projects
[List 2–3 relevant projects that demonstrate:]

REST API design and implementation
Working with document databases or hierarchical data structures
Writing comprehensive test suites
Handling complex data operations (migration, transformation, batch processing)
Motivation
I am deeply motivated to contribute to the Community Health Toolkit because:

Real-World Impact: CHT serves community health workers across dozens of countries. Enabling them to manage contacts programmatically through APIs — instead of requiring a system administrator with CLI access — directly removes friction from their daily workflows and empowers the people closest to the communities they serve.

Technical Growth: This project sits at the intersection of several areas I'm passionate about — API design, hierarchical data management, database operations, and building reliable systems. The challenge of adapting cht-conf's file-based staging approach into a real-time API server context is a fascinating architectural problem.

Open-Source Ecosystem: This work has a multiplicative effect — services like cht-user-management can consume these APIs instead of re-implementing the logic or depending on cht-conf under the hood, making the entire CHT ecosystem more cohesive and maintainable.

C4GT Mission: Code for GovTech's mission to create digital public goods resonates strongly with me. Contributing to healthcare infrastructure that serves underserved communities through open-source software is exactly the kind of work I want to be doing.

References
CHT-Core GitHub Repository
cht-conf HierarchyOperations Source
CHT Documentation — Architecture Overview
CHT Documentation — Contributing Guide