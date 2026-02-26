# Cabal MCP

Cabal's MCP server lets you search people and companies, discover warm introduction paths through your team's network, and manage connectors.

- **Endpoint**: `POST /mcp`
- **Protocol**: MCP `2025-03-26` (JSON-RPC 2.0)
- **Auth**: OAuth 2.0 Bearer token
- **Scopes**: `read` (all tools), `write` (required for `add_connectors`)
- **Server**: `cabal-mcp` v0.1.0

---

## Core Concept: The UUID Chain

Most workflows follow a **search-then-act** pattern. Search tools return UUIDs; connector and intro tools consume them. Never fabricate UUIDs -- always obtain them from a prior tool call.

| ID Type | Produced By | Consumed By |
|---------|------------|-------------|
| Company UUID | `search_companies` | `get_company_connectors`, `get_company_connections`, `get_connector_intros_at_company` |
| Person UUID | `search_people` | `get_person_connectors`, `add_connectors` |
| Connector UUID | `get_company_connectors`, `get_person_connectors` | `get_connector_intros_at_company` |
| Location name | `lookup_locations` | `search_companies`, `search_people` |
| Industry name | `search_companies_industries` | `search_companies` |
| Investor ID (integer) | `search_companies_investors` | `search_companies` |
| Company UUID (for people filters) | `search_people_companies` | `search_people` |
| People list UUID | `search_people_person_lists` | `search_people` |
| Company list UUID | `search_people_company_lists` | `search_people` |

Always use helper tools (`lookup_locations`, `search_companies_industries`, etc.) to resolve valid filter values rather than guessing.

---

## Tool Reference

### Identity

#### `whoami`

Returns the authenticated user's profile information.

- **Params**: none
- **Returns**: JSON with first name, last name, name, email, title, team name, and team role

---

### Search

#### `search_companies`

Search the global company database by name, domain, industry, location, size, funding, and investors.

- **Params**:
  - `query` (string) -- company name or domain
  - `industries` (string[]) -- use `search_companies_industries` to find valid values
  - `location_names` (string[]) -- use `lookup_locations` to find valid values
  - `sizes` (string[]) -- employee count ranges (e.g. "11-50", "51-200")
  - `latest_funding_stages` (string[]) -- e.g. "Series A", "Series B"
  - `total_funding_raised` (string[]) -- funding amount ranges
  - `founded` (string[]) -- year founded (e.g. "2020")
  - `investors` (integer[]) -- investor IDs from `search_companies_investors`
  - `page` (integer, default 1), `per_page` (integer, default 10, max 25)
- **Returns**: companies with UUID, domain, headline, industry, location, size, founded, funding
- **Chains to**: `get_company_connectors`, `get_company_connections`, `get_connector_intros_at_company`

#### `search_people`

Search the global people database by name, company, headline, LinkedIn URL, job function, seniority, and location.

- **Params**:
  - `query` (string) -- person name, company name, headline keywords, or LinkedIn URL
  - `functions` (string[]) -- e.g. "Engineering", "Sales", "Marketing", "Finance", "Human Resources", "Operations", "Product", "Design", "Legal"
  - `levels` (string[]) -- e.g. "C-Suite", "VP", "Director", "Manager", "Senior", "Entry"
  - `location_names` (string[]) -- use `lookup_locations` to find valid values
  - `current_company_uuids` (string[]) -- use `search_people_companies` to find valid UUIDs
  - `former_company_uuids` (string[]) -- use `search_people_companies` to find valid UUIDs
  - `profile_list_uuids` (string[]) -- use `search_people_person_lists` to find valid UUIDs
  - `company_list_uuids` (string[]) -- use `search_people_company_lists` to find valid UUIDs
  - `page` (integer, default 1), `per_page` (integer, default 10, max 25)
- **Returns**: people with UUID, profile URL, headline, company, location, LinkedIn URL
- **Chains to**: `get_person_connectors`, `add_connectors`

---

### Filter Helpers

These tools resolve human-readable names into the exact values and IDs that search tools accept. Call them before applying filters.

#### `lookup_locations`

Find valid location values for the `location_names` filter on `search_companies` and `search_people`.

- **Params**: `query` (string, **required**) -- e.g. "San Francisco", "New York", "London"
- **Returns**: matching location strings ranked by relevance

#### `search_companies_industries`

Find valid industry values for the `industries` filter on `search_companies`.

- **Params**: `query` (string, **required**) -- e.g. "AI", "fintech", "healthcare"
- **Returns**: matching industry strings ranked by relevance

#### `search_companies_investors`

Find investor company IDs for the `investors` filter on `search_companies`.

- **Params**: `query` (string, **required**) -- e.g. "Sequoia", "Andreessen"
- **Returns**: `{label, value}` pairs where `value` is an **integer** ID (not a UUID)

#### `search_people_companies`

Find company UUIDs for the `current_company_uuids` and `former_company_uuids` filters on `search_people`.

- **Params**: `query` (string, **required**) -- e.g. "Google", "Stripe"
- **Returns**: `{label, value}` pairs where `value` is a company UUID

#### `search_people_person_lists`

List accessible people lists to get UUIDs for the `profile_list_uuids` filter on `search_people`.

- **Params**: `query` (string, optional) -- filter lists by name
- **Returns**: `{label, value}` pairs where `value` is a list UUID

#### `search_people_company_lists`

List accessible company lists to get UUIDs for the `company_list_uuids` filter on `search_people`.

- **Params**: `query` (string, optional) -- filter lists by name
- **Returns**: `{label, value}` pairs where `value` is a list UUID

---

### Connectors & Intros

#### `get_company_connectors`

Find team members who can make warm introductions to people at a specific company. Results are sorted by connection strength.

- **Params**:
  - `company_uuid` (string, **required**) -- from `search_companies`
  - `query` (string) -- filter connectors by name
  - `page` (integer, default 1), `per_page` (integer, default 10, max 25)
- **Returns**: connectors with UUID, headline, company, LinkedIn, connection type (Direct/Inferred), and reasons
- **Chains to**: `get_connector_intros_at_company`

#### `get_company_connections`

Find all reachable people at a company -- people your team can reach through warm intros, grouped with their connectors.

- **Params**:
  - `company_uuid` (string, **required**) -- from `search_companies`
  - `query` (string) -- filter connections by name
  - `page` (integer, default 1), `per_page` (integer, default 10, max 25)
- **Returns**: people at the company with headline, LinkedIn, each with a list of connectors (name, UUID, Direct/Inferred)

#### `get_connector_intros_at_company`

Find which specific people at a company a given connector can introduce you to.

- **Params**:
  - `company_uuid` (string, **required**) -- from `search_companies`
  - `connector_uuid` (string, **required**) -- from `get_company_connectors`
  - `page` (integer, default 1), `per_page` (integer, default 10, max 25)
- **Returns**: people with UUID, headline, LinkedIn, connection type, shared company count, and overlap years

#### `get_person_connectors`

Find team members who can make a warm introduction to a specific person. Results are sorted by connection strength.

- **Params**:
  - `person_uuid` (string, **required**) -- from `search_people`
  - `query` (string) -- filter connectors by name
  - `page` (integer, default 1), `per_page` (integer, default 10, max 25)
- **Returns**: connectors with UUID, headline, company, LinkedIn, connection type (Direct/Inferred), and reasons

#### `list_team_connectors`

List existing connectors on the team. Use this to see who is already a connector before adding new ones.

- **Params**:
  - `query` (string) -- filter connectors by name
  - `page` (integer, default 1), `per_page` (integer, default 10, max 25)
- **Returns**: connectors with UUID, headline, global person headline, company, email, LinkedIn, origin, global person UUID, and relationship tags

---

### Write

#### `add_connectors`

Add one or more connectors to the team. Requires the `write` OAuth scope.

- **Params**:
  - `connectors` (array, **required**, 1-25 items) -- each item must include exactly one of:
    - `global_person_uuid` (string) -- UUID from `search_people`
    - `linkedin_url` (string) -- LinkedIn profile URL
    - `email` (string) -- email address (pair with optional `first_name` and `last_name`)
  - `send_invite` (boolean, default false) -- send email invitation (email-based connectors only)
- **Returns**: results grouped into Added, Already Existed, and Failed

---

## Common Workflows

### 1. Find who can intro to a company

```
search_companies(query: "Acme Corp")        --> company_uuid
get_company_connectors(company_uuid: "...")  --> list of connectors with UUIDs
get_connector_intros_at_company(            --> specific people the connector knows
  company_uuid: "...",
  connector_uuid: "..."
)
```

### 2. Find who can intro to a person

```
search_people(query: "Jane Smith")          --> person_uuid
get_person_connectors(person_uuid: "...")    --> connectors who know Jane
```

### 3. Search with filters (helper-first pattern)

```
lookup_locations(query: "San Francisco")    --> "San Francisco, California, US"
search_companies_industries(query: "AI")    --> "Artificial Intelligence"
search_companies(
  location_names: ["San Francisco, California, US"],
  industries: ["Artificial Intelligence"]
)
```

### 4. Find people at specific companies

```
search_people_companies(query: "Stripe")    --> company UUID
search_people(
  current_company_uuids: ["..."],
  levels: ["VP", "Director"]
)
```

### 5. See all reachable people at a company

```
search_companies(query: "Acme Corp")        --> company_uuid
get_company_connections(company_uuid: "...")  --> people + their connectors
```

### 6. Add a new connector

```
list_team_connectors(query: "Jane")         --> check if already exists
search_people(query: "Jane Smith")          --> get global_person_uuid
add_connectors(connectors: [
  { global_person_uuid: "..." }
])
```

---

## Best Practices

- **Use helper tools before filtering.** Don't guess location names, industry values, or investor IDs. Call `lookup_locations`, `search_companies_industries`, or `search_companies_investors` first.
- **Chain UUIDs between tools.** Every UUID you pass to a tool should come from a previous tool call. Never fabricate UUIDs.
- **Handle pagination.** Results are capped at 25 per page. Check the pagination header (total count, current page, total pages) and request additional pages when needed.
- **Start broad, narrow incrementally.** Begin with a simple query, then add filters to refine results.
- **Check before writing.** Call `list_team_connectors` before `add_connectors` to avoid duplicates.
- **Prefer `global_person_uuid` over email** when adding connectors. UUID-based adds link to rich profile data; email-based adds create minimal records.
- **Investor IDs are integers, not UUIDs.** The `investors` filter on `search_companies` takes integer IDs from `search_companies_investors`.
- **Responses are markdown-formatted text.** All tool responses are returned as markdown in the MCP `text` content block.

---

## Error Handling

**Tool errors** (`isError: true` in the MCP response) indicate a problem with your input or the operation:
- `Missing required parameter: <param>` -- a required argument was not provided
- `Invalid UUID format for <param>` -- the value is not a valid UUID
- `Company or connector not found.` -- the UUID doesn't match any record
- `You do not have permission to add connectors to this team.` -- the user lacks the required team role
- `Too many connectors (maximum 25).` -- the `connectors` array exceeds the batch limit

**Protocol errors** (JSON-RPC `error` object) indicate a problem with the request itself:
- `-32700` Parse error -- invalid JSON
- `-32600` Invalid request -- missing `jsonrpc: "2.0"`, bad request structure, or insufficient OAuth scope (e.g. `Insufficient scope: the '<tool>' tool requires the 'write' scope`)
- `-32601` Method not found -- unrecognized JSON-RPC method
- `-32602` Unknown tool -- the tool name doesn't exist
- `-32603` Internal error -- unexpected server error

**Empty results are not errors.** A search that returns zero results will return a success response with a "Found 0 results" message or a "No connectors found" message.
