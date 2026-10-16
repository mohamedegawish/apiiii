# Admin Reports And Offline Audit

## Reports

- Student, teacher, and circle reports are wired to the statistics API.
- Each report validates the selected entity server-side through its policy.
- Recalculation is restricted to administrative roles; it is not queued offline because it is a server-side aggregate operation.
- Statistics responses are now cached locally after a successful read and are shown when the same report is reopened without connectivity.
- Searchable report pickers cache their last successful first page for offline selection.

## Offline Writes

- Attendance, memorization sessions, revision sessions, assessments, homework, and learning-record evaluations use the persistent sync queue.
- The teacher session workspace learning records are now queued when offline.
- Offline create followed by edit is coalesced into one queued create, preserving the latest draft.
- Connectivity restoration drains the queue automatically in FIFO order.
- App restart preserves the queue.
- Network/server failures stay queued until delivery; logical failures remain visible as blocked operations with retry.
- The sync screen shows pending, blocked, retry count, and the last error.

## Scope Boundary

Circle creation and its dependent weekly schedules are queued offline; the sync layer remaps the server-generated circle ID before uploading schedules. Finance and user-provisioning writes remain online-only because they require server-side accounting/identity transactions. Existing cached reports and field operations remain usable offline.

## Verification

- `flutter analyze` admin app: passed.
- Admin sync tests: passed, including offline evaluation coalescing, reconnect upload, persistent queue, blocked-operation retry, and offline report cache.
- Full Laravel suite: passed, 352 tests and 1433 assertions.
