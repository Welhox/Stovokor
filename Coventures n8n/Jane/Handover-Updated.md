# Jane - User Access Management System Handover

## Project Overview

**Purpose**: Automated user provisioning/deprovisioning across Google Workspace, HubSpot, Slack, and Agileday for Coventures.

**Primary Interface**: Slack bot ("Jane") in a private channel, responding to `@Jane` mentions.

**n8n Instance**: Cloud version 2.x (upgraded February 2026)

**Architecture Pattern**: Federated query (fetch from platforms on-demand, no local database sync)

---

## n8n 2.0 Important Changes

⚠️ **Publish/Save Paradigm**: 
- **Save** = preserves edits without affecting production
- **Publish** = explicitly pushes changes live
- Always remember to **Publish** after making workflow changes!

**Human-in-the-loop in Sub-workflows**: Now works correctly! Slack confirmations in sub-workflows return results properly to parent workflows.

**Task Runners**: Code nodes run in isolated environments by default.

---

## Architecture

### Core Workflows (Active)

| Workflow | ID | Purpose |
|----------|-----|---------|
| Agent - Jane v2 | `tbgqVGrkuJ5o7ztO` | Main AI agent (Slack trigger → authorization → LangChain agent) |
| Jane - Add User | `1TSrbZXz1HQgkr63` | Provisions users across platforms |
| Jane - Remove User | `BScqiDMW8JEaT81h` | Offboards users, includes Slack confirmation step |

### Platform Sub-workflows

| Workflow | ID | Notes |
|----------|-----|-------|
| Tool - Googleworkspace | `VwTo4rWU4Z1yJe7x` | User CRUD via Admin API (add/modify/delete=suspend) |
| Jane - Hubspot | `lr6erTORXWMOOCWT` | User creation (minimal permissions - no roles API without Enterprise) |
| Jane - Agileday | `6NNP0C50VCewy4oP` | Employee provisioning (segments: employee/subcontractor) |

### Tool Workflows (Agent Tools) - NEW ARCHITECTURE

| Workflow | ID | Purpose | Status |
|----------|-----|---------|--------|
| **Tool - Check Access** | `Ukl1o4WdH8GAxcG1` | Query single user across all platforms by email | ✅ New |
| **Tool - List Users** | `2uDCT1p955INp2tE` | List all users with platform access (federated query) | ✅ New |
| Tool - Update Agileday User | `nTr2xjch7XpsHcl2` | Update Agileday fields (team, title, etc.) — params: `email`, `firstName`, `lastName`, `title`, `primaryTeam`, `nationality`, `startDate` | ✅ Active |
| Tool - Send Agileday Onboarding | `u8vIm8gI9STFFRrB` | Send onboarding email via Agileday — params: `email`, `personal_email`, `first_name`, `last_name` | ✅ Active |

### Legacy/Deprecated Workflows

| Workflow | ID | Status | Notes |
|----------|-----|--------|-------|
| Jane - Tool: Get User | `hyKauylmzmwayEU6` | ⚠️ Deprecated | Replace with Tool - Check Access |
| Jane - Tool: List Users | `L3mvZ32zgLMVyeXa` | ⚠️ Deprecated | Replace with Tool - List Users |
| Jane - Tool: Update Database | `UPNAsxAmUa3kwdnN` | ⚠️ Deprecated | No longer using local DB sync |
| Agent - Jane | `LTTghFVxhYrA22u5` | 🗄️ Archived | Original non-AI version |
| Jane - Slack Interactions | `NSjA8SHyI7FWZguE` | 🗄️ Archived | Original modal-based interface |
| Fetch Agileday Users | `NdPsniUVQQMe6P53` | ⚠️ Deprecated | Merged into Tool - List Users |
| Fetch Agileday User Details | `Vo8CtITQcQgisJqV` | ⚠️ Deprecated | Merged into Tool - Check Access |
| Fetch HubSpot Users | `cH6oBvllD5PVLW2f` | ⚠️ Deprecated | Merged into Tool - List Users |

---

## Federated Query Architecture

### Why Federated (vs Database Sync)?
- Small user base (<100 users)
- Infrequent queries
- Always fresh data from source platforms
- No sync workflow to maintain
- Simpler architecture

### Tool - Check Access Flow
```
Trigger(email) → Parallel Fetch:
  ├── Agileday API (by email)
  ├── HubSpot API (filter users)
  ├── Google Workspace Admin API (projection: full)
  └── Slack API (users.list, filter by email)
→ Merge (4 inputs) → Format Response
```

**Input parameter:** `email` (required) — the user's email address.

**Output Format:**
```json
{
  "success": true,
  "email": "john@coventures.io",
  "name": "John Doe",
  "agileday": { "active": true, "team": "EiR Business", "title": "EiR" },
  "hubspot": { "active": true, "role": "user" },
  "google": {
    "active": true,
    "suspended": false,
    "isAdmin": false,
    "isDelegatedAdmin": false,
    "is2faEnabled": false,
    "changePasswordAtNextLogin": false,
    "orgUnit": "/",
    "lastLoginTime": "2026-01-15T10:00:00Z",
    "creationTime": "2024-03-01T09:00:00Z"
  },
  "slack": { "active": true, "display_name": "John", "title": "EiR" }
}
```

**Note:** Google field `is2faEnabled` corresponds to Google's `isEnrolledIn2Sv`. Requires `projection: full` on the gSuiteAdmin node — `projection: basic` (default) omits this field.

### Tool - List Users Flow
```
Trigger(platform?, status?, include_offboarded?) → Parallel Fetch:
  ├── Agileday API (all employees)
  ├── HubSpot API (all users)
  ├── Google Workspace Admin API (all users)
  └── Slack API (all workspace members)
→ Merge (4 inputs) → Consolidate by Email → Filter → Return
```

**Input parameters:**
- `platform` (optional): filter by platform — `agileday`, `hubspot`, `google`, `slack`
- `status` (optional): filter by status — `active`, `inactive`
- `include_offboarded` (optional, boolean, default `false`): when `true`, includes users who are disabled in Agileday and have no other active access (i.e. fully offboarded employees)

**Output Format:**
```json
{
  "success": true,
  "total": 45,
  "users": [
    {
      "email": "john@coventures.io",
      "name": "John Doe",
      "agileday": true,
      "hubspot": true,
      "google": true,
      "slack": true,
      "agileday_team": "EiR Business",
      "agileday_title": "EiR",
      "agileday_segment": "employee",
      "agileday_start_date": "2024-03-01",
      "agileday_end_date": null,
      "agileday_is_offboarding": false,
      "google_2fa_enabled": false,
      "google_is_delegated_admin": false,
      "slack_is_admin": false,
      "slack_is_restricted": false
    }
  ]
}
```

**Agileday status fields:**
- `agileday: true` = active/enabled in Agileday; `false` = disabled (offboarded) or not in Agileday
- `agileday_is_offboarding: true` = active employee with a future `agileday_end_date` (leaving soon)
- Users with `agileday: false` and no other system access are excluded by default (`include_offboarded: false`)

**Slack notes:** Bots and deleted workspace members are excluded. Requires `users:read` scope (and `users:read.email` for email matching — add this to the Slack app if not already present).

---

## Credentials Reference

| Platform | Credential Name | ID | Notes |
|----------|----------------|-----|-------|
| Slack | Slack Jane | `uCSWRk66y2n0sljA` | OAuth |
| Google Workspace | Google Workspace Jane | `39CdaiSskL9WToZb` | Service Account with Domain-Wide Delegation |
| HubSpot | HubSpot OAuth | `fwLzLaPlT3e0zi5q` | OAuth2 |
| Agileday | AgileDay Header Auth | `gZ02orJPHhrRUjnr` | API Key in header |
| Supabase | (check instance) | - | For chat memory only |

---

## User Roles → Platform Mapping

| Jane Role | Agileday Segment | Agileday Team | Use Case |
|-----------|------------------|---------------|----------|
| active_eir | employee | EiR Business | Active Entrepreneur in Residence |
| nbb_eir | employee | NBB Business | NBB Entrepreneur in Residence |
| network_eir | subcontractor | Network EiR | Network-level EiR |
| special_cases | subcontractor | Other | Custom arrangements |

---

## Platform Limitations & Workarounds

### HubSpot
- **No roles API** without Enterprise subscription
- Users created with minimal permissions (`contacts-base`)
- **Workaround**: Slack notification with direct link for manual permission setup
- Portal ID: `7128688`

### Slack
- Manual invitation required (API costs avoided during development)
- Notification sent to admins instead of automatic user creation
- **User querying** now supported via `users.list` API (Pro plan compatible)
- Required scopes: `users:read` + `users:read.email` (add the latter to enable email-based matching)

### Agileday
- Required fields: first_name, last_name, email, start_date, primary_team, business_unit
- Offboarding: set end_date + disable (profile preserved for historical data)
- Segments: "employee" or "subcontractor"
- API base: `https://coventures.agileday.io/api/v1/`
- API returns ALL employees including disabled (`disabled: true/false`). `agileday` field = `!emp.disabled`
- `agileday_is_offboarding` = active (`!disabled`) AND has a future `end_date`

### Google Workspace
- Full automation via Admin API
- Domain: `coventures.io`
- Delete = Suspend (preserves files, 20-day recovery window)
- Invitation flow (ID assigned when user accepts)
- **`projection: full` required** on gSuiteAdmin node to get `isEnrolledIn2Sv` (2FA status). Default `projection: basic` omits it. Also set `output: raw` to prevent n8n from stripping custom fields.
- Field `is2faEnabled` in tool output = Google's `isEnrolledIn2Sv`

---

## Security Measures

### Authorization
- Slack user ID allowlist in "Check Authorization" IF node
- Currently authorized: `U08PW72HEN7` (Casimir)
- Private channel restriction

### Confirmation Flow
- Add/Modify: Requires "yes" confirmation
- Remove: Requires Slack button confirmation (✅ Confirm Removal / ❌ Cancel)

### Safety Rules (Agent System Prompt)
1. Always check user access before any action
2. Always confirm before write operations
3. One user per request
4. Never claim success without tool confirmation
5. Report per-platform status (including partial failures)

---

## Key Technical Decisions

1. **Federated queries > Database sync** - Fresh data, simpler architecture
2. **Parameterized queries** - Use `$1, $2` with `queryReplacement` arrays (SQL injection prevention)
3. **Parallel platform execution** - Merge node with 4 inputs for simultaneous fetches (Agileday, HubSpot, Google, Slack)
4. **Error handling**: `onError: "continueRegularOutput"` on fetch nodes
5. **Soft delete** - Offboard (status change) rather than delete users

---

## Common Operations

### Check User Access
```
User: "What access does john@coventures.io have?"
Jane: Calls Tool - Check Access → Returns platform status
```

### Add User
```
User message → Jane parses intent → Confirms details → 
Calls Jane - Add User → Parallel platform provisioning → 
Audit log → Report results
```

### Remove User
```
User message → Jane parses intent → Slack confirmation button → 
User confirms → Calls Jane - Remove User → 
Check user exists → Parallel platform removal → 
Audit log → Report results
```

---

## Known Issues / TODOs

### Current Issues
1. HubSpot permissions require manual admin step post-creation
2. Slack user creation is manual notification only
3. Google Workspace returns invitation (no ID until accepted)

### Pending Tasks
- [x] Connect new Tool - Check Access to Jane agent
- [x] Connect new Tool - List Users to Jane agent
- [x] Remove deprecated DB query tools from agent
- [x] Test federated query workflows
- [ ] Implement audit logging to `jane_bot.audit_log` table
- [ ] Update add_user/remove_user to use check_access instead of DB lookups

---

## Debugging Tips

1. **Workflow not updating in production**: Did you click **Publish**? (n8n 2.0 change)

2. **Workflow invisible in GUI**: Usually Switch node version mismatch (downgrade from v3.4 to v2)

3. **Agent schema errors**: LangChain `$fromAI` parameters may send null; simplify tool inputs. Also: the LLM infers parameter names from the tool schema — **parameter names in `workflowInputs.schema`, in the system prompt docs, and in the trigger `workflowInputs.values` must all match exactly**. gpt-4.1 is strict about this; mismatches that gpt-4.1-mini tolerated will cause `Received tool input did not match expected schema` errors.

4. **Connection failures**: Use `cleanStaleConnections` operation in partial updates

5. **Merge node issues**: Use `mode: "chooseBranch"` with `numberInputs: 4` for 4-input merges (Tool - Check Access and Tool - List Users)

6. **Google Workspace validation warnings**: `userId` mode "userEmail" shows as invalid in validator but works at runtime

---

## Quick Reference: n8n Node Types

```
Slack trigger:     @n8n/n8n-nodes-langchain.chatTrigger
AI Agent:          @n8n/n8n-nodes-langchain.agent
PostgreSQL:        n8n-nodes-base.postgres
HTTP Request:      n8n-nodes-base.httpRequest
Execute Workflow:  n8n-nodes-base.executeWorkflow
IF:                n8n-nodes-base.if
Merge:             n8n-nodes-base.merge
Code:              n8n-nodes-base.code
Google Admin:      n8n-nodes-base.gSuiteAdmin
```

---

## API Endpoints Reference

| Platform | Endpoint | Auth |
|----------|----------|------|
| Agileday - Get by email | `GET /api/v1/employee/email/{email}` | Header: X-API-KEY |
| Agileday - Get all | `GET /api/v1/employee` | Header: X-API-KEY |
| HubSpot - Get users | `GET /settings/v3/users` | OAuth Bearer |
| Google - Get user | `GET /admin/directory/v1/users/{email}` | Service Account |
| Google - Get all | `GET /admin/directory/v1/users?domain=coventures.io` | Service Account |
| Slack - Get all users | `GET /api/users.list` | OAuth (via n8n Slack node) |
| Slack - Lookup by email | `GET /api/users.lookupByEmail?email={email}` | OAuth (requires `users:read.email`) |

---

## Changelog

### February 2026 (late)
- **Model upgrade**: Agent - Jane v2 (`TEST model` node) upgraded from `gpt-4.1-mini` → `gpt-4.1` for stronger temporal reasoning and agentic tasks
- **Live date injection**: Jane's system prompt converted to n8n expression using `$now.toFormat('yyyy-MM-dd')` — Jane now knows the exact current date at runtime
- **Date interpretation rules**: Added `## DATE INTERPRETATION` section to system prompt (previous month = calendar month, last 30 days = rolling window)
- **Agileday offboarding**:
  - Added `agileday_is_offboarding` computed field (true = active employee with future end date)
  - Added `include_offboarded` parameter to Tool - List Users (default false — excludes fully disabled/offboarded users from normal queries)
  - Updated Jane's system prompt with Agileday status interpretation guide
- **Google 2FA field**:
  - Renamed `google_is_enrolled_2sv` → `google_2fa_enabled` in Tool - List Users output
  - Renamed `isEnrolledIn2Sv` → `is2faEnabled` in Tool - Check Access output
  - Set `projection: full` + `output: raw` on gSuiteAdmin nodes — `projection: basic` (default) omits 2FA field
- **Tool parameter schema fixes** (gpt-4.1 stricter than mini):
  - Tool - List Users: corrected system prompt to use `platform`/`status` (not `filter_system`/`filter_status`); added `include_offboarded` to `workflowInputs.schema`
  - Tool - Check Access: added explicit `check_access(email)` documentation to system prompt; prevents model from inferring `filter_email`
  - Tool - Send Agileday Onboarding: trimmed 9 phantom parameters from `workflowInputs.schema` (Slack context fields leaked in from trigger passthrough — unused by workflow, caused LLM confusion)

### February 2026 (early)
- Upgraded n8n from 1.121.3 to 2.x
- Created Tool - Check Access (federated query)
- Created Tool - List Users (federated query)
- Migrated from database sync to federated query architecture
- Fixed Merge node to use 4 inputs with proper `targetIndex` connections
- Added Slack user querying to Tool - Check Access and Tool - List Users (4-platform federated query)
  - Slack Pro plan compatible via `users.list` API
  - Requires `users:read.email` scope on the Slack app for email-based matching

### Previous
- See transcript archives for historical changes
