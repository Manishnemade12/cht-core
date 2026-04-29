Ticket Contents
Currently, the CHT (Community Health Toolkit) app allows users to perform basic contact management tasks such as creating an editing contacts. Users with the proper permissions can also delete contacts, but only if there are no child contacts associated with the contact to delete.

More advanced contact management current requires a system administrator to use the cht-conf CLI tool. cht-conf can perform operations like moving, merging, and deleting contacts in the hierarchy (note that the delete-contacts action in cht-conf is recursive so that it also deletes all the child contacts). Having this functionality in cht-conf creates friction for users who cannot perform these actions themselves through the webapp, especially when dealing with duplicate contacts or reorganizing health facility hierarchies.

It also forces services like cht-user-management to either re-implement these operations or else somehow leverage cht-conf under the hood to perform them.

This project aims to add REST API endpoints in cht-core for these actions. The work can be done incrementally (one action at a time), giving it a limited initial scope with room to expand. The goal is simply to have api support for these operations. Updating the webapp to actually give the user access to these operations is out of scope.

Goals & Mid-Point Milestone

Understand the existing cht-conf move/merge/delete contact logic

Implement the delete-contacts API endpoint with support for recursive hierarchy deletion

Implement the move-contacts API endpoint for relocating contacts between parent hierarchies

Implement the merge-contacts API endpoint for combining duplicate contacts and reassigning all linked documents (reports, tasks) to the surviving contact

Write API documentation for all new endpoints in cht-docs

Goals achieved by mid-point milestone: Delete-contacts and move-contacts API endpoints are functional with unit and integration tests
Expected Outcome
New REST API endpoints available in cht-core:

DELETE /api/v1/contact/:uuid (or POST /api/v1/contact/:uuid/delete) for recursively deleting a contact and its children in one step
POST /api/v1/contact/:uuid/move for moving a contact (and its subtree) to a new parent
POST /api/v1/contact/:uuid/merge for merging two duplicate contacts, reassigning all linked documents to the surviving contact
These endpoints replace the need to use the cht-conf CLI for these operations, enabling them to be performed programmatically or through the webapp.

Implementation Details
The existing cht-conf implementations serve as the reference for the core logic:

cht-conf/src/fn/move-contacts.js
cht-conf/src/fn/merge-contacts.js
cht-conf/src/fn/delete-contacts.js
New API controllers should be added in api/src/controllers/ following the patterns established by the existing person, place, and report controllers.

Key technical considerations:

TODO these actions may involve long-running processes. We need to determine if we should block our REST response until the process actually completes (our typical approach for existing endpoints) or if we should implement some kind of "polling" pattern where we queue the operation and then immediately return a reference to it. Then the consumer can poll a /api/v1/operation endpoint to get the status (and know when it completes).
Permission model should follow the hasAny pattern (e.g. can_delete_contacts OR can_edit)
Delete must handle recursive cleanup of child contacts and all associated documents
Move must update parent lineage references across the entire subtree and linked documents
Merge must handle conflict resolution and reassignment of all linked reports and tasks to the surviving contact
Product Name
Community Health Toolkit: cht-core

Organization Name
Medic

Domain
Healthcare

Tech Skills Needed
Docker, JavaScript, Mocha, Node.js, TypeScript

Organizational Mentor
@sugat009, @jkuester

Complexity
High

Category
API, Backend