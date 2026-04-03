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

### [severity: critical] User deletion without transaction leaves partial state on failure

- **File:** src/api/users.py lines 45-52
- **What can go wrong:** The `delete_user()` function deletes from `sessions`,
  `orders`, and `users` in three separate queries without a transaction.
  If the process crashes or the DB connection drops after deleting sessions
  but before deleting the user, the user record persists with missing session
  history. There is no retry or cleanup mechanism.
- **Why vulnerable:** The function calls `db.sessions.delete(user_id)`,
  `db.orders.delete(user_id)`, and `db.users.delete(user_id)` sequentially
  without wrapping them in `db.transaction()`.
- **Impact:** Data loss. Partial deletion leaves the database in an
  inconsistent state that requires manual intervention to fix.
- **Recommendation:** Wrap all three deletes in a single transaction:
  `with db.transaction(): ...`. If any step fails, the entire operation
  rolls back.

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

- Wrapped all three deletes (`sessions`, `orders`, `users`) in `db.transaction()`
- Added `@require_role('admin')` to the delete endpoint
- Added test for unauthorized deletion attempt (returns 403)

---

## Adversarial Review — Round 2 (mode: code, model: gpt-5.4)

## Summary

Both issues from round 1 are resolved. The transaction wraps all deletes
atomically and the authorization check is in place. No new issues found.

## Findings

No actionable findings.

## Verdict

VERDICT: APPROVED
