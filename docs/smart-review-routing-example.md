# Smart Review Routing Example

This document demonstrates how the intelligent review track selection works in practice.

## Overview

Smart Review Routing (Optimization 2) automatically analyzes each task and assigns one of three review tracks:
- **Fast Track**: Code Quality only (skips spec review)
- **Standard Track**: Spec Compliance → Code Quality
- **Heavy Track**: Spec Compliance → Code Quality → Integration

## Example Feature: E-commerce Product Catalog

Let's walk through a feature implementation with 6 tasks to see how tracks are assigned.

---

## Task Analysis

### Task 1: Add price formatting utility

**Task Description:**
```markdown
Create a utility function to format prices with currency symbols.
- Input: number (price in cents), currency code (string)
- Output: formatted string (e.g., "$12.99", "€10.50")
- Support USD, EUR, GBP
- Add unit tests
```

**Track Assignment:** **Fast Track** ✅

**Analysis:**
- Files: 1 (`utils/formatting.ts`)
- Lines: ~35 lines
- API: No new public API
- Architecture: No changes
- Domain: Single (utility)
- Security: Not sensitive

**Review Process:**
```
Implementer → Code Quality Review (1 round max) → Complete
```

**Rationale:** Pure utility function, single file, no integration concerns. Spec is crystal clear.

---

### Task 2: Product entity and repository

**Task Description:**
```markdown
Create Product entity and repository layer.
- Entity: Product (id, name, description, price, image_url, category_id)
- Repository: CRUD operations (create, findById, findAll, update, delete)
- Database migrations
- Unit tests for repository
```

**Track Assignment:** **Standard Track** ⚙️

**Analysis:**
- Files: 3 (`entities/Product.ts`, `repositories/ProductRepository.ts`, `migrations/...`)
- Lines: ~180 lines
- API: New data layer API
- Architecture: Standard repository pattern
- Domain: Single (backend)
- Security: Not sensitive

**Review Process:**
```
Implementer → Spec Compliance Review → Code Quality Review (2 rounds max) → Complete
```

**Rationale:** Multi-file, new API layer. Need to verify all CRUD operations are implemented and tests cover edge cases.

---

### Task 3: Product listing REST API

**Task Description:**
```markdown
Create REST API for product listing.
- GET /api/products (list all with pagination)
- GET /api/products/:id (get single product)
- POST /api/products (admin only - create product)
- PUT /api/products/:id (admin only - update)
- DELETE /api/products/:id (admin only - delete)
- Request/response validation
- Error handling (404, 400, 401, 403)
- Integration tests
```

**Track Assignment:** **Standard Track** ⚙️

**Analysis:**
- Files: 4 (`routes/products.ts`, `controllers/ProductController.ts`, `validators/product.ts`, tests)
- Lines: ~250 lines
- API: New REST API
- Architecture: Standard MVC pattern
- Domain: Single (backend)
- Security: Auth required but standard pattern

**Review Process:**
```
Implementer → Spec Compliance Review → Code Quality Review (2 rounds max) → Complete
```

**Rationale:** New public API with multiple endpoints. Need to verify all endpoints, status codes, and error handling match spec.

---

### Task 4: Frontend product catalog component

**Task Description:**
```markdown
Create product catalog React component.
- Display product grid (responsive: 1/2/3 columns)
- Product card: image, name, price, "Add to Cart" button
- Pagination controls
- Loading states
- Error states
- Call backend API /api/products
- Component tests with React Testing Library
```

**Track Assignment:** **Standard Track** ⚙️

**Analysis:**
- Files: 3 (`components/ProductCatalog.tsx`, `hooks/useProducts.ts`, tests)
- Lines: ~200 lines
- API: Calls existing backend API
- Architecture: Standard React component
- Domain: Single (frontend)
- Security: Not sensitive

**Review Process:**
```
Implementer → Spec Compliance Review → Code Quality Review (2 rounds max) → Complete
```

**Rationale:** Frontend component with API integration. Standard complexity, need to verify all UI states and API calls.

---

### Task 5: Product search with Elasticsearch

**Task Description:**
```markdown
Implement full-text product search.
- Elasticsearch index setup
- Indexing pipeline (sync products to ES)
- Search endpoint: POST /api/products/search
- Query builder (multi-field search: name, description, category)
- Relevance scoring
- Pagination
- Fallback to database if ES unavailable
- Performance tests (< 100ms p95)
```

**Track Assignment:** **Standard Track** ⚙️

**Analysis:**
- Files: 5 (search service, indexing, endpoint, config, tests)
- Lines: ~300 lines
- API: New search API
- Architecture: Adds external dependency (Elasticsearch)
- Domain: Single (backend) but external integration
- Security: Not sensitive

**Review Process:**
```
Implementer → Spec Compliance Review → Code Quality Review (2 rounds max) → Complete
```

**Rationale:** Complex but single-domain. External dependency but no cross-domain coordination. Standard track with careful review of error handling and fallback logic.

---

### Task 6: Checkout payment integration

**Task Description:**
```markdown
Integrate Stripe payment for checkout.

Backend:
- POST /api/checkout/session (create Stripe session)
- POST /api/checkout/webhook (handle Stripe events)
- Store payment records in database
- Handle success/failure states
- Idempotent payment processing
- Stripe API key management (environment vars)

Frontend:
- Checkout page component
- Stripe Elements integration
- Payment form with card input
- Success/failure redirect handling
- Loading states during payment

Integration:
- Frontend calls backend /api/checkout/session
- Backend creates Stripe session, returns session ID
- Frontend loads Stripe checkout with session ID
- Stripe webhook notifies backend of payment completion
- Backend updates order status
```

**Track Assignment:** **Heavy Track** 🔥

**Analysis:**
- Files: 8+ (backend routes/services/webhooks, frontend components/hooks, config)
- Lines: ~400+ lines
- API: New payment API (backend) + integration with Stripe
- Architecture: Cross-domain coordination (frontend ↔ backend ↔ Stripe)
- Domain: **Multiple (frontend + backend + external service)**
- Security: **HIGHLY SENSITIVE** (payment credentials, PCI compliance)

**Review Process:**
```
Implementer
  → Spec Compliance Review
  → Code Quality Review
  → Integration Review (verify contract between frontend/backend/Stripe)
  (3 rounds max per stage)
  → Complete
```

**Rationale:**
1. **Cross-domain:** Frontend and backend must coordinate precisely
2. **Security-sensitive:** Handles payment credentials, requires special scrutiny
3. **External API:** Stripe integration adds complexity
4. **Contract verification needed:**
   - Frontend expects session ID format from backend
   - Backend expects webhook payload format from Stripe
   - Success/failure flow must work end-to-end
5. **High risk:** Payment failures are costly, need extra verification

**Integration Review Checks:**
- Does backend session creation return format frontend expects?
- Does frontend handle all backend error codes (400, 500, etc.)?
- Does webhook handler cover all Stripe event types?
- Are success/failure redirects configured correctly on both sides?
- Is payment idempotency implemented correctly?

---

## Summary

### Track Distribution

| Track | Task Count | Review Calls per Task | Total Calls |
|-------|-----------|----------------------|-------------|
| Fast Track | 1 | 1 (Code Quality only) | 1 |
| Standard Track | 4 | 2 (Spec + Quality) | 8 |
| Heavy Track | 1 | 3 (Spec + Quality + Integration) | 3 |
| **Total** | **6** | - | **12** |

### Performance Comparison

**Without Smart Routing (all Standard Track):**
- 6 tasks × 2 reviews = 12 review calls
- Estimated time: ~12 minutes

**With Smart Routing:**
- 1 Fast × 1 + 4 Standard × 2 + 1 Heavy × 3 = 12 review calls
- Estimated time: ~11 minutes
- **But:** Fast Track task fails faster (1 round vs 2), Heavy Track gets extra scrutiny where needed

### Key Benefits

1. **Fast Track tasks** (Task 1):
   - Skip unnecessary spec review for obvious cases
   - Fail fast with 1 round max
   - Saves ~1 minute per simple task

2. **Standard Track tasks** (Tasks 2-5):
   - Balanced review for typical features
   - 2 rounds is sufficient for most cases

3. **Heavy Track tasks** (Task 6):
   - Extra integration verification prevents runtime failures
   - 3 rounds allow for complex fixes
   - Catches cross-domain contract mismatches

### Example Issues Caught by Integration Review

Task 6 Integration Review found:
- ❌ Backend returns `{ sessionId: "..." }`, Frontend expects `{ session_id: "..." }` (camelCase vs snake_case mismatch)
- ❌ Frontend only handles 400/500 errors, Backend also returns 409 Conflict for duplicate payments
- ❌ Stripe webhook expects endpoint at `/api/webhooks/stripe`, Backend registered `/api/checkout/webhook` (URL mismatch)

Without Heavy Track, these would fail at runtime (after deployment).

---

## Decision Tree Visualization

```
Task ready for implementation
       │
       ▼
  Analyze characteristics
       │
       ├─ Single file < 50 lines, no API, no arch change?
       │  YES → Fast Track (Code Quality only, 1 round)
       │
       ├─ Cross-domain (frontend+backend) OR security-sensitive OR arch change?
       │  YES → Heavy Track (Spec+Quality+Integration, 3 rounds)
       │
       └─ Otherwise → Standard Track (Spec+Quality, 2 rounds)
```

---

## Implementation Notes

### For Orchestrators

When assigning tracks at Stage 5A:

```markdown
**Review Track Assignments:**

Task 1: [name] → [Track]
  Reason: [brief explanation]
  Files: [count], Lines: [estimate]
  API: [Yes/No], Domain: [single/multi], Security: [Yes/No]

Task 2: ...
```

### For Reviewers

Track information is passed to reviewers:

```
Unified Reviewer receives:
- Task track: "Heavy"
- Max rounds: 3
- Integration check required: Yes

If track == "Heavy" and all reviews pass:
  → Dispatch Integration Reviewer
  → Verify cross-domain contracts
```

### For Users

Users see track assignments in logs:

```
Starting Stage 5: Implementation

Review Track Assignments:
✓ Task 1 (Fast Track) - utility function
✓ Task 2-5 (Standard Track) - typical features
✓ Task 6 (Heavy Track) - payment integration (security-sensitive, cross-domain)

Dispatching implementers...
```

---

## Future Enhancements

Potential improvements to Smart Routing:

1. **Learning from history:** Track success rates per track, adjust thresholds
2. **User overrides:** Allow user to force track if auto-detection is wrong
3. **Complexity scoring:** Numeric complexity score (0-100) instead of three discrete tracks
4. **Project-specific rules:** Custom track rules per project (e.g., all auth code → Heavy Track)

---

**Status:** Implemented as of 2026-03-25
**Related:** Optimization 2 in `docs/optimization-proposals.md`
