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
| Tool - Update Agileday User | `nTr2xjch7XpsHcl2` | Update Agileday fields (team, title, etc.) | ✅ Active |
| Tool - Send Agileday Onboarding | `u8vIm8gI9STFFRrB` | Send onboarding email via Agileday | ✅ Active |

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
  └── Google Workspace Admin API
→ Merge (3 inputs) → Format Response
```

**Output Format:**
```json
{
  "success": true,
  "email": "john@coventures.io",
  "name": "John Doe",
  "agileday": { "active": true, "team": "EiR Business", "title": "EiR" },
  "hubspot": { "active": true, "role": "user" },
  "google": { "active": true, "suspended": false }
}
```

### Tool - List Users Flow
```
Trigger(platform?, status?) → Parallel Fetch:
  ├── Agileday API (all employees)
  ├── HubSpot API (all users)
  └── Google Workspace Admin API (all users)
→ Merge (3 inputs) → Consolidate by Email → Filter → Return
```

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
      "agileday_team": "EiR Business"
    }
  ]
}
```

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

### Agileday
- Required fields: first_name, last_name, email, start_date, primary_team, business_unit
- Offboarding: set end_date + disable (profile preserved for historical data)
- Segments: "employee" or "subcontractor"
- API base: `https://coventures.agileday.io/api/v1/`

### Google Workspace
- Full automation via Admin API
- Domain: `coventures.io`
- Delete = Suspend (preserves files, 20-day recovery window)
- Invitation flow (ID assigned when user accepts)

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
3. **Parallel platform execution** - Merge node with 3 inputs for simultaneous fetches
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
- [ ] Connect new Tool - Check Access to Jane agent
- [ ] Connect new Tool - List Users to Jane agent
- [ ] Remove deprecated DB query tools from agent
- [ ] Test federated query workflows
- [ ] Implement audit logging to `jane_bot.audit_log` table
- [ ] Update add_user/remove_user to use check_access instead of DB lookups

---

## Debugging Tips

1. **Workflow not updating in production**: Did you click **Publish**? (n8n 2.0 change)

2. **Workflow invisible in GUI**: Usually Switch node version mismatch (downgrade from v3.4 to v2)

3. **Agent schema errors**: LangChain `$fromAI` parameters may send null; simplify tool inputs

4. **Connection failures**: Use `cleanStaleConnections` operation in partial updates

5. **Merge node issues**: Use `mode: "chooseBranch"` with `numberInputs: 3` for 3-input merges

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

---

## Changelog

### February 2026
- Upgraded n8n from 1.121.3 to 2.x
- Created Tool - Check Access (federated query)
- Created Tool - List Users (federated query)
- Migrated from database sync to federated query architecture
- Fixed Merge node to use 3 inputs with proper `targetIndex` connections

### Previous
- See transcript archives for historical changes
