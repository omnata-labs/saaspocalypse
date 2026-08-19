# UI Design Decisions Template

When designing the UI for an app, document decisions in `apps/{name}/ui/DECISIONS.md`.

## Example DECISIONS.md

```markdown
# UI Decisions: Support App

## Navigation
- Permanent sidebar drawer (240px)
- Items: Dashboard (home), Tickets, Investigations
- Messages are viewed within ticket context, not standalone

## Pages

### Dashboard
- Summary cards: open ticket count, unassigned count, avg resolution time
- Recent activity feed
- Quick-create button

### Ticket List
- Table: ID, Title, Status, Priority, Assignee, Created
- Filters: status (multi-select), priority, assignee
- Search by title/short_id
- Sort by created_at desc
- Click row to navigate to detail

### Ticket Detail
- Header: short_id, title (editable), status chip, priority chip
- Two-column: timeline (left 70%) + metadata panel (right 30%)
- Timeline: messages chronological with author info
- Compose area at bottom
- Reference data: organization name fetched from /api/v1/ref/organizations

### Create Ticket
- Modal dialog
- Fields: title, organization (autocomplete from ref data), reporter email, priority
- Validation errors displayed inline at fields

## Data Access Patterns
- Internal data (tickets, messages): via /api/v1/{entity} which proxies CALL procs
- Reference data (organizations, accounts): via /api/v1/ref/{entity} (read-only, cached)
- Client-side join: ticket shows org name by fetching from ref endpoint

## Patterns
- Status changes via dropdown on detail page
- Validation errors render inline at field from proc response
- Loading states use MUI Skeleton
- Empty states show helpful text + create button
- Reference data cached for 5 minutes (staleTime in react-query)
```
