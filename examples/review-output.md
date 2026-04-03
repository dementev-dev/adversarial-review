# Example: Adversarial Code Review Output

This is a synthetic example showing what a typical adversarial review looks like.

---

## Adversarial Review — Round 1 (mode: code, model: gpt-5.4)

## Summary

This change adds a user deletion endpoint to the API. The endpoint accepts
a user ID, deletes the user record, and returns 204. The implementation
handles the happy path correctly but has two significant issues around
data integrity and authorization.

## Findings

### [severity: critical] User deletion does not cascade to dependent records

- **File:** src/api/users.py lines 45-52
- **What can go wrong:** Deleting a user leaves orphaned records in `orders`,
  `sessions`, and `audit_log` tables. Foreign key constraints are set to
  `NO ACTION`, so the delete succeeds but dependent data becomes unreferenceable.
  Queries joining on `user_id` will silently drop rows.
- **Why vulnerable:** The `delete_user()` function calls
  `db.users.delete(user_id)` directly without handling related tables.
  No cascade is configured at the database level.
- **Impact:** Data integrity violation. Orphaned orders cannot be attributed
  to any user. Financial reconciliation reports will undercount.
- **Recommendation:** Either add `ON DELETE CASCADE` to foreign keys in a
  migration, or delete dependent records explicitly in a transaction before
  deleting the user.

### [severity: high] Missing authorization check on delete endpoint

- **File:** src/api/users.py lines 38-40
- **What can go wrong:** Any authenticated user can delete any other user
  by sending `DELETE /api/users/{id}`. The endpoint checks authentication
  (`@require_auth`) but not authorization — there is no admin role check
  or ownership validation.
- **Why vulnerable:** The route decorator only has `@require_auth`, not
  `@require_role('admin')`. The `user_id` parameter comes from the URL
  path and is never compared to the authenticated user's ID.
- **Impact:** Privilege escalation. Any logged-in user can delete
  arbitrary accounts including admins.
- **Recommendation:** Add `@require_role('admin')` decorator, or if
  self-deletion is intended, verify `request.user.id == user_id`.

## Verdict

VERDICT: REVISE

---

### Fixes (Round 1)

- Added `ON DELETE CASCADE` migration for `orders.user_id` and `sessions.user_id`
- Added `@require_role('admin')` to the delete endpoint
- Added test for unauthorized deletion attempt (returns 403)

---

## Adversarial Review — Round 2 (mode: code, model: gpt-5.4)

## Summary

Both critical issues from round 1 are resolved. The cascade migration is correct
and the authorization check is in place. No new issues found.

## Findings

No actionable findings.

## Verdict

VERDICT: APPROVED
