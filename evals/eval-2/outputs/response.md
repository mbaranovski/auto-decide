# Response

**Decision:** Going with Option C -- server-side filtering, sorting, and pagination with the dual-mode CSV export (immediate download for small datasets, async email for large ones). This is the only scalable option that delivers all three features the user asked for without deferring anything to a follow-up; Option B drops CSV export to a later PR, and Option A won't hold up past 10k rows in practice.

Now, let's start building. Here's the implementation plan:

## Implementation Plan

### 1. Data Table Foundation
- Set up the admin dashboard layout with a reusable `DataTable` component
- Server-side pagination with configurable page size (25, 50, 100 rows)
- Column definitions with type metadata (string, number, date, enum) to drive filter UI

### 2. Server-Side Filtering
- Filter API accepts a structured query: `{ column, operator, value }` per filter
- Operators vary by column type: `contains`, `equals`, `gt`, `lt`, `between` for numbers/dates; `contains`, `equals`, `in` for strings/enums
- Filters are composable (AND logic by default)
- Filter state is synced to URL query params so filtered views are shareable

### 3. Server-Side Sorting
- Sort API accepts `{ column, direction }` -- single-column sort to start, multi-column if needed later
- Default sort defined per table (e.g., `created_at DESC`)
- Sort state also synced to URL query params

### 4. CSV Export (Dual-Mode)
- **Small datasets (under threshold, e.g., 5,000 rows after filters applied):** Synchronous endpoint streams the CSV directly as a download. Immediate, no waiting.
- **Large datasets (over threshold):** Async endpoint kicks off a background job. The job generates the CSV, uploads it to file storage, and emails the user a download link. The UI shows a toast: "Your export is being prepared -- you'll receive an email when it's ready."
- The export always respects the current filters and sort order -- the user gets exactly what they're looking at.

### 5. API Endpoints
- `GET /api/admin/{resource}` -- paginated, filtered, sorted list
- `GET /api/admin/{resource}/export` -- CSV export (returns file or 202 with job ID)
- `GET /api/admin/{resource}/export/{jobId}/status` -- poll export job status (optional, if we want in-app progress)

Let me start with the data table component and the server-side list endpoint.
