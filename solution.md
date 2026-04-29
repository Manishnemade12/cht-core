# Contact Management REST APIs for CHT-Core

## Name, Contact Information & Skills

- **Name:** [Your Name]
- **Email:** [your.email@example.com]
- **GitHub:** [github.com/your-username]
- **Portfolio:** [your-portfolio-link]
- **Skills:** JavaScript (ES6+), Node.js, TypeScript, CouchDB/PouchDB, REST API Design, Docker, Mocha/Chai/Sinon Testing, Express.js, Git, Linux

---

## Title

**Contact Management REST APIs for CHT-Core: Implementing Delete, Move & Merge Contact Operations**

---

## Summary

Right now, if someone needs to delete, move, or merge contacts in CHT, they have to use the `cht-conf` CLI tool - which means only a system admin can do it. This is a real pain point for teams on the ground who need to fix duplicate contacts or reorganize facility hierarchies but don't have CLI access. It also forces downstream services like `cht-user-management` to hack around the limitation.

My plan is to bring these three operations into cht-core as proper REST API endpoints. I've spent time reading through both the cht-core API layer and the cht-conf source code to understand exactly how these operations work under the hood. The approach is straightforward: take the battle-tested logic from cht-conf's `HierarchyOperations` module, adapt it for a server-side context (direct CouchDB writes instead of staging files to disk), and expose it through controllers that follow the same patterns already used by the `person`, `place`, and `contact` endpoints. The whole thing is broken into a 12-week plan with delete and move done by the mid-point milestone.

---

## Project Details

### 1. Project Overview

#### a) Understanding of the Project

CHT-Core is a monorepo with an Express.js API server (`api/`), a background processor (`sentinel/`), and an Angular webapp (`webapp/`), all backed by CouchDB. The API layer follows a clean pattern - controllers in `api/src/controllers/`, services in `api/src/services/`, routes registered in `routing.js`, and auth handled through `auth.assertPermissions()`.

I went through the existing controllers (`person.js`, `place.js`, `contact.js`, `bulk-docs.js`) to understand the conventions. Controllers use `@medic/cht-datasource` bindings through `ctx.bind()`, wrap handlers in `serverUtils.doOrError` for error handling, and routes are registered via `app.postJson()` / `app.putJson()` helpers. Auth uses two patterns - `hasAll` (every permission required) and `hasAny` (at least one). For contact management, `hasAny` fits best since you'd want either a specific permission like `can_delete_contacts` or the general `can_edit`. There's also precedent for large write operations - `bulk-docs.js` already handles batched deletes with a configurable batch size.

**How cht-conf handles these operations:**

All three operations in cht-conf funnel through a single `HierarchyOperations` module. I read through every file in that module:

```
cht-conf/src/lib/hierarchy-operations/
├── index.js              # Main orchestrator - exports move(), merge(), delete()
├── delete-hierarchy.js   # Recursive delete + report cleanup
├── hierarchy-data-source.js  # CouchDB queries (views, batching)
├── lineage-manipulation.js   # Lineage tree operations
├── lineage-constraints.js    # Validation rules for hierarchy changes
├── replace-lineage.js        # Swaps out lineage references in docs
└── jsdocFolder.js            # Writes docs to disk (we won't need this)
```

Move and merge share the same core function (`moveHierarchy`). Merge is basically a move with `merge: true`, which also deletes the source contact and reassigns report subjects.

| Operation | What Happens |
|-----------|-------------|
| **Delete** | Fetches all descendants via `contacts_by_depth` view, marks each as `_deleted: true`. Deletes all linked reports via `reports_by_subject`. Optionally sets `cht_disable_linked_users` on places to disable linked users. |
| **Move** | Validates hierarchy constraints (no circular refs, parent type must be allowed). Rewrites `parent` lineage across entire subtree. Updates ancestors whose primary contact was moved. Rebinds `contact` lineage in all reports created by moved contacts. |
| **Merge** | Same as move, plus: enforces same contact type between source and destination. Marks source as `_deleted: true`. Reassigns report subjects (`patient_id`, `patient_uuid`, `place_id`, `place_uuid`) to destination. Optionally merges primary contacts. |

**CouchDB views used:**
- `medic/contacts_by_depth` - fetches all descendants of a contact
- `medic-client/reports_by_subject` - fetches reports by subject ID
- `medic-client/reports_by_freetext` - fetches reports by creator contact

Reports are processed in batches of 10,000 to keep memory bounded.

#### b) Problems (if any)

1. **Long-running operations:** These operations can touch hundreds of descendants and thousands of reports. The problem statement flags this as an open question - block the response or use polling? Existing cht-core endpoints all block, but these are much heavier.

2. **Adapting cht-conf's two-step approach:** cht-conf stages modified documents to disk, then uploads separately. An API needs to write directly to CouchDB, which means handling partial failures carefully.

3. **Lineage consistency:** Moving a contact requires updating `parent` lineage in every descendant and `contact` lineage in every linked report. If writes fail midway, the hierarchy ends up inconsistent.

4. **Constraint validation complexity:** Move operations must check parent type rules, detect circular hierarchies, and verify primary contacts won't be orphaned. These rules interact in non-obvious ways.

#### c) Solutions

**Solution for long-running operations - Synchronous with batched writes:**

I'll use a blocking (synchronous) approach, which matches all existing endpoints in cht-core. Reports are processed in memory-safe batches of 10,000 (same as cht-conf) and CouchDB writes happen in smaller batches (~500 docs). If the client disconnects due to a timeout, the operation still completes server-side since the internal `.bulkDocs` transactions are atomic. Because the service layer is purely orchestrating logic, it can be easily wrapped in a job runner (polling pattern) later if mentors determine the blocking approach isn't sufficient for production hierarchies.

**Solution for cht-conf adaptation - Direct CouchDB writes via `db.medic.bulkDocs()`:**

Instead of the file-staging approach, we go straight to the database. The risk of partial failure is handled by batching - if one batch fails, we know exactly where it stopped.

**Solution for consistency - Modular service architecture:**

I'm splitting the logic into focused, testable modules:

```
api/src/
├── controllers/
│   └── contact-management.js    [NEW] - handles HTTP requests
└── services/
    └── contact-management/      [NEW] - the actual logic
        ├── index.js             - public interface (deleteContact, moveContact, mergeContact)
        ├── hierarchy-data.js    - CouchDB queries, adapted from cht-conf
        ├── lineage-ops.js       - lineage manipulation (replace, minify, create)
        ├── constraints.js       - hierarchy rule validation
        └── report-ops.js        - report deletion and reassignment

api/tests/mocha/
├── controllers/
│   └── contact-management.spec.js   [NEW]
└── services/
    └── contact-management/
        ├── index.spec.js            [NEW]
        ├── hierarchy-data.spec.js   [NEW]
        ├── lineage-ops.spec.js      [NEW]
        ├── constraints.spec.js      [NEW]
        └── report-ops.spec.js       [NEW]
```

Each module has a clear job and can be tested independently. The service layer does the heavy lifting; the controller is thin - just auth + input validation + calling the service.

**Solution for constraints - Ported validation logic:**

All validation rules will be ported directly from cht-conf's `lineage-constraints.js`. When a move or merge operation is requested, the service layer must strictly verify:
- **Existence checks:** Both the source and target contacts actually exist in the database.
- **Hierarchy rules:** The operation must not create a circular hierarchy (`source` cannot be an ancestor of `destination`), and the `destination` must be an allowed parent type for the `source` contact type, as dictated by app settings.
- **Merge specific:** A contact cannot be merged with itself, and source/destination must be the exact same contact type.
- **Lineage safety:** The operation won't improperly orphan a primary contact from its place, nor process two contacts from the same lineage subtree in a single move.

**Three new API endpoints:**

| Endpoint | Method | Auth | Purpose |
|----------|--------|------|---------|
| `/api/v1/contact/:uuid/delete` | POST | `can_delete_contacts` or `can_edit` | Recursively delete contact + descendants + reports |
| `/api/v1/contact/:uuid/move` | POST | `can_move_contacts` or `can_edit` | Move contact subtree to new parent |
| `/api/v1/contact/:uuid/merge` | POST | `can_merge_contacts` or `can_edit` | Merge duplicate contact into surviving contact |

I went with POST instead of DELETE because the requests carry JSON bodies with options, and HTTP DELETE isn't really meant for that. This is consistent with how `bulk-delete` works in cht-core already.

**Controller design follows existing patterns:**

```javascript
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
      const result = await contactMgmt.deleteContact(uuid, {
        disableUsers: !!disable_users
      });
      return res.json(result);
    }),  
    // move and merge follow the identical structure...
  }
};
```

**Routes added to `routing.js`:**
```javascript
app.postJson('/api/v1/contact/:uuid/delete', contactManagement.v1.delete);
app.postJson('/api/v1/contact/:uuid/move', contactManagement.v1.move);
app.postJson('/api/v1/contact/:uuid/merge', contactManagement.v1.merge);
```

**Service layer - Direct write approach:**

The biggest difference between cht-conf and this API is that we skip the file-staging step entirely and write directly to CouchDB using `db.medic.bulkDocs()`. Here is how the orchestration looks for a delete operation:

```javascript
// api/src/services/contact-management/index.js
const db = require('../../db');
const hierarchyData = require('./hierarchy-data');
const constraints = require('./constraints');
const reportOps = require('./report-ops');

const deleteContact = async (contactId, options = {}) => {
  const contactDoc = await hierarchyData.getContact(db.medic, contactId);

  // 1. Get every contact under this one using contacts_by_depth view
  const descendants = await hierarchyData.getDescendants(db.medic, contactId);
  const checker = await constraints.load(db.medic);

  // 2. Mark them all as deleted
  const deletedContacts = descendants.map(doc => ({
    _id: doc._id,
    _rev: doc._rev,
    _deleted: true,
    cht_disable_linked_users: options.disableUsers && checker.isPlace(doc),
  }));

  // 3. Clean up reports in memory-safe batches
  let deletedReports = 0;
  for (const descendant of descendants) {
    deletedReports += await reportOps.deleteReportsForContact(
      db.medic, descendant._id
    );
  }

  // 4. Execute a single atomic bulk write for the contacts
  await db.medic.bulkDocs(deletedContacts);

  return { ok: true, deleted_contacts: deletedContacts.length, deleted_reports: deletedReports };
};
```

**Data layer - CouchDB query abstraction:**
The `hierarchy-data.js` module wraps CouchDB calls to keep the core service logic clean. It exposes functions like `getContact()`, `getDescendants()` (using the `medic/contacts_by_depth` view), and `getAncestors()` using standard `db.query` and `db.allDocs` methods.

---

### 2. Implementation Details with Timelines

#### a) Milestone 1 - Foundation + Delete Contact API (Weeks 1–4)

**Week 1: Environment Setup & Codebase Deep-Dive**
- Set up local dev environment (Docker, CouchDB, Node.js 22) and run the existing test suite.
- Walk through `cht-conf/src/lib/hierarchy-operations/` with mentors, document edge cases and confirm expected behaviors for all three operations.

**Week 2: Shared Data Layer**
- Build `hierarchy-data.js` - the data access module used by all three endpoints. Covers descendant lookup via `contacts_by_depth`, ancestor fetching, and batched report queries via `reports_by_subject` and `reports_by_freetext`.
- Write thorough unit tests for the data layer (Mocha + Chai + Sinon following project conventions).

**Week 3: Delete Service & Controller**
- Implement `deleteContact()` in the service layer: recursive descendant marking (`_deleted: true`), batched report deletion, and `cht_disable_linked_users` support for place contacts.
- Wire up `POST /api/v1/contact/:uuid/delete` controller with `auth.assertPermissions` and register routes in `routing.js`.

**Week 4: Delete Hardening & Review**
- Integration tests against a live CouchDB instance covering edge cases (contact not found, no children, deeply nested hierarchies, very large subtrees).
- Code review with mentors, incorporate feedback.
- **Deliverable:** Fully tested Delete Contact API endpoint.

#### b) Milestone 2 - Move Contact API (Weeks 5–8)

**Week 5: Constraints & Lineage Modules**
- Port `lineage-constraints.js` from cht-conf: circular hierarchy detection, parent type validation from app settings, primary contact safety checks.
- Implement `lineage-ops.js` for lineage replacement across subtrees, lineage minification, and lineage construction from documents.

**Week 6: Move Service & Controller**
- Build `moveContact()` orchestrator: rewrite `parent` lineage in all descendants, update ancestor primary contacts, rebind `contact` lineage in linked reports using `reports_by_freetext`.
- Wire up `POST /api/v1/contact/:uuid/move` controller, register routes, write unit tests.
- **Mid-point milestone reached:** Both Delete and Move endpoints are functional with tests.

**Week 7: Move Edge Cases & Integration Tests**
- Test move-specific edge cases: move to root (`"root"` as parent), cross-type violations, circular hierarchy attempts, moving a contact that is an ancestor's primary contact.
- Integration tests on live CouchDB to verify lineage integrity post-move.

**Week 8: Stabilization & Buffer**
- Address mentor review feedback from Weeks 5-7. Refactor shared modules if needed based on real-world testing results.
- Ensure the Delete + Move test suites pass cleanly in CI (`npm run unit-api` and `npm run integration-api`).
- **Deliverable:** Production-ready Move Contact API endpoint. Delete + Move fully stabilized.

#### c) Milestone 3 - Merge Contact API + Documentation (Weeks 9–12)

**Week 9: Report Reassignment & Merge Constraints**
- Build `report-ops.js` to reassign report subjects, swapping `patient_id`, `patient_uuid`, `place_id`, `place_uuid` from the source contact to the destination.
- Implement merge-specific constraints: source and destination must be the same contact type, cannot merge a contact with itself.

**Week 10: Merge Service & Controller**
- Wire up `mergeContact()` orchestrator that combines move logic with source deletion and report subject reassignment. Support `merge_primary_contacts` and `disable_users` options.
- Deliver `POST /api/v1/contact/:uuid/merge` controller and complete unit testing.

**Week 11: Cross-Endpoint Integration & Performance**
- Full integration test suite across all three endpoints running against a live CouchDB instance.
- Performance testing with large hierarchies (1000+ descendants, 10000+ reports) to validate batching strategy.
- Code review with mentors.

**Week 12: Documentation & Final Polish**
- Write OpenAPI specs for all three endpoints using the `@openapi` JSDoc convention in the codebase.
- Author API documentation for the `cht-docs` site covering usage examples, permissions, and error handling.
- Final review pass, address remaining comments, submit final PR.
- **Deliverable:** Merge Contact API endpoint + complete documentation for all three endpoints.

**Optional Stretch Goals (if time permits):**
- Dry-run mode (`?dry_run=true`) to preview affected documents without writing.
- Batch operations to accept multiple contact IDs in a single request.

---

### Testing Strategy

All tests follow cht-core conventions using Mocha + Chai + Sinon, with `sinon.restore()` in `afterEach`. Unit tests mirror the source path structure:

```
api/tests/mocha/controllers/contact-management.spec.js
api/tests/mocha/services/contact-management/index.spec.js
api/tests/mocha/services/contact-management/hierarchy-data.spec.js
api/tests/mocha/services/contact-management/lineage-ops.spec.js
api/tests/mocha/services/contact-management/constraints.spec.js
api/tests/mocha/services/contact-management/report-ops.spec.js
```

Integration tests run against a live CouchDB to verify actual document mutations, lineage correctness, and report reassignment:

```
tests/integration/api/controllers/contact-management.spec.js
```

---

## Availability

- **35–40 hours per week** - I'm treating this as full-time work
- **No competing commitments** during July–August
- **Timezone:** IST (UTC+5:30)

---

## Personal Information & Motivation

### About Me & Open Source Experience

I am a Full Stack Developer with deep expertise in JavaScript, Node.js, and document databases (CouchDB) — the core languages and tools required for this project. I am also an active open-source contributor, heavily involved in the Meshery and Layer5 communities, where I've gained strong experience architecting and maintaining complex, distributed systems.

### Why This Project

This project solves a major pain point for community health workers by moving critical contact-management capabilities from a restrictive CLI tool into secure, accessible REST APIs. Adapting a file-based CLI implementation to run safely against a live CouchDB database using batched operations is exactly the kind of challenging, high-impact backend architecture I enjoy building.

---

## References

- [CHT-Core Repository](https://github.com/medic/cht-core)
- [cht-conf HierarchyOperations](https://github.com/medic/cht-conf/tree/main/src/lib/hierarchy-operations)
- [CHT Architecture Docs](https://docs.communityhealthtoolkit.org/technical-overview/architecture/cht-core/)
- [CHT Contributing Guide](https://docs.communityhealthtoolkit.org/community/contributing/code/core/dev-environment/)
