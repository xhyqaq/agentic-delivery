# Integration Reviewer Prompt Template

Use this template when dispatching an integration reviewer subagent for multi-domain plans (e.g., frontend + backend).

**Purpose:** Verify that implementations from different domains can integrate successfully

**Dispatch after:** All individual task reviews pass (spec compliance + code quality)

```
Task tool (general-purpose):
  description: "Review frontend-backend integration"
  prompt: |
    You are reviewing integration between frontend and backend implementations.

    ## Design Spec (API Contract)

    [Paste the API contract section from design-spec.md]

    Include:
    - API endpoints (method, path, description)
    - Request formats (fields, types, examples)
    - Response formats (success and error cases)
    - Status codes and error types

    ## Backend Implementation

    **Task:** [backend task name]

    **Git diff:**
    ```bash
    git diff BASE_SHA..HEAD_SHA
    ```
    [Paste backend git diff here]

    ## Frontend Implementation

    **Task:** [frontend task name]

    **Git diff:**
    ```bash
    git diff BASE_SHA..HEAD_SHA
    ```
    [Paste frontend git diff here]

    ---

    ## Your Job

    Check integration between these implementations. Read the actual code in git diffs,
    not just summaries.

    ### 1. API Contract Consistency

    Compare actual implementations against design spec:

    **Backend output:**
    - Does backend return the exact field names specified? (check for camelCase vs snake_case)
    - Do types match spec? (string vs number vs boolean)
    - Is response structure correct? (nested objects, arrays)
    - Are status codes what spec says? (200, 201, 400, 404, etc.)
    - Are error response formats consistent with spec?

    **Frontend expectations:**
    - Does frontend destructure/access fields with correct names?
    - Does frontend handle types correctly?
    - Does frontend expect the response structure backend returns?
    - Does frontend check for the status codes backend actually sends?

    **Common mismatches to look for:**
    - Field name: backend `userId`, frontend expects `user_id`
    - Type: backend returns number, frontend treats as string
    - Nesting: backend returns `{ data: { user } }`, frontend expects `{ user }`
    - Status: backend sends 409, frontend only checks 400/500

    ### 2. Data Flow Completeness

    **APIs in spec:**
    List all API endpoints defined in the design spec.

    **APIs implemented:**
    - Which endpoints does backend implement?
    - Which endpoints does frontend call?

    **Check for gaps:**
    - APIs in spec but not implemented by backend?
    - APIs implemented but not called by frontend?
    - APIs called by frontend that don't exist in backend?

    **CRUD completeness:**
    For each resource (e.g., skills, users):
    - Create: implemented + called?
    - Read: implemented + called?
    - Update: implemented + called?
    - Delete: implemented + called?

    Flag if operations are implemented but not exposed in UI, or UI tries to call
    operations that don't exist.

    ### 3. Error Handling Alignment

    **Backend errors:**
    List all error status codes backend can return (from the code):
    - 400 Bad Request: when?
    - 401 Unauthorized: when?
    - 403 Forbidden: when?
    - 404 Not Found: when?
    - 409 Conflict: when?
    - 500 Internal Server Error: when?

    **Frontend error handling:**
    List all error status codes frontend handles (from the code):
    - Which status codes have explicit handling?
    - Which are missing?

    **Check for gaps:**
    - Backend returns 409 but frontend doesn't handle it
    - Frontend handles 403 but backend never returns it
    - Backend error messages in different format than frontend expects

    ---

    ## Output Format

    ### Integration Review

    **Status:** ✅ Verified | ❌ Issues Found

    ---

    ### Issues (if any)

    #### API Contract Mismatches
    1. **Field name mismatch: backend returns `userId`, frontend expects `user_id`**
       - Backend: `backend/api/skills.ts:45` returns `{ userId: ... }`
       - Frontend: `frontend/services/skills.ts:23` destructures `{ user_id }`
       - Impact: Frontend will get undefined, feature broken

    2. **Type mismatch: skillId is number in backend, string in frontend**
       - Backend: `backend/api/skills.ts:52` returns `skillId: number`
       - Frontend: `frontend/types.ts:10` defines `skillId: string`
       - Impact: Type errors, possible bugs in comparisons

    #### Data Flow Gaps
    1. **DELETE /api/skills/:id implemented but never called**
       - Backend: `backend/api/skills.ts:78-95` implements DELETE endpoint
       - Frontend: No API call found for deletion
       - Impact: Delete functionality exists in backend but not exposed in UI

    2. **Frontend calls GET /api/skills/search not implemented**
       - Frontend: `frontend/services/skills.ts:67` calls `/api/skills/search`
       - Backend: Endpoint not found in any file
       - Impact: Search feature will 404

    #### Error Handling Gaps
    1. **Backend returns 409 Conflict, frontend doesn't handle it**
       - Backend: `backend/api/skills.ts:34` returns 409 for duplicate titles
       - Frontend: `frontend/services/skills.ts:45-52` only handles 400, 500
       - Impact: User won't see duplicate error message, generic error shown

    2. **Frontend expects 403 but backend never returns it**
       - Frontend: `frontend/services/skills.ts:89` has 403 handling
       - Backend: No 403 status found in any endpoint
       - Impact: Dead code, but not breaking

    ---

    ### Verification Summary

    - **Total mismatches:** [count]
    - **Critical issues:** [count - will break at runtime]
    - **Non-critical issues:** [count - dead code or minor inconsistencies]

    ---

    ### Recommendations (if verified)

    - All API contracts align with spec
    - All implemented endpoints are called
    - All error cases are handled
    - Integration should work as expected
```

**Reviewer returns:** Status (✅ Verified / ❌ Issues Found), Issues list with file:line references

## Integration Review Guidelines

**DO:**
- Read actual code in git diffs, not summaries
- Check field names character-by-character
- List all endpoints from both sides
- Map each error code to its handling
- Be specific with file:line references

**DON'T:**
- Assume field names match without checking
- Trust that "API is implemented" without listing endpoints
- Skip error handling verification
- Be vague ("some fields don't match")

**Severity:**
- **Critical:** Field name/type mismatches, missing endpoints, unhandled errors that backend returns
- **Non-critical:** Dead error handling code, minor inconsistencies that don't break functionality
