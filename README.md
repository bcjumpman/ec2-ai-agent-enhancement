# EC2 AI Agent Enhancement – ServiceNow Implementation

This README describes the AI Agent enhancement built on top of the existing **EC2 Monitoring and Remediation** system.  
The enhancement adds a conversational interface (EC2 Remediation Assistant) that identifies EC2 instance IDs from incident text, requests human approval, and runs the same remediation API used by the manual UI action.

---

## System Overview

The AI Agent sits between ServiceNow workflows and the existing AWS Integration Server. It can:

- Read incident text and extract EC2 instance IDs.
- Ask the engineer for approval before taking action.
- Call the same remediation API used by `EC2RemediationHelper`.
- Log every attempt in the `Remediation Log` table.

Both manual (UI Action) and conversational paths use the same API and logging. This keeps audit records consistent.

---

## Architecture Diagram

See `Diagram.png` for the enhanced architecture:
- Existing path: AWS EC2 → AWS Integration Server → ServiceNow EC2 Instance table → Flow Designer.
- New path: Flow Designer → AI Agent (conversational tool) → Script Tool (calls RemediationHelper) → AWS Integration Server API.
- Both paths update the Remediation Log and can create incidents.

---

## Agent Instructions (drop into AI Agent `Instructions` field)


(Short version for the Agent UI:  
`Extract EC2 ID from incident, ask for approval, then call remediation API and log results.`)

---

## Implementation Steps

### 1. Confirm prerequisites
- WL2 manual system is working:
  - `EC2 Instance` and `Remediation Log` tables exist and contain test data.
  - `EC2RemediationHelper` Script Include works via UI Action.
  - AWS Integration Server connection and credentials are configured.

### 2. Create AI Agent
- Studio → **AI Agent Studio** → New Agent:
  - **Name**: `EC2 Remediation Assistant`
  - **Description**: conversational remediation specialist for DevOps
  - **Role/Instructions**: use the Agent Instructions above

### 3. Create Script Tool
- Add a **Script Tool** in AI Agent Studio:
  - Base logic on `EC2RemediationHelper`.
  - Input schema: `{ "instance_id": { "type": "string", "description": "EC2 id e.g. i-0abc1234" } }`
  - Execution mode: **Supervised** (requires human approval)
  - Tool behavior:
    - Map `instance_id` → lookup `EC2 Instance` record (sys_id).
    - Call the same REST message that `EC2RemediationHelper` uses.
    - Insert a `Remediation Log` entry identical in shape to the manual path.
    - Return `{ success, message, log_id, http_status }`.

> Note: adding a tool creates records in `sn_aia_agent_tool_m2m`. Those records must be captured in the update set.

### 4. Attach Tool to Agent
- Open the Agent → **Tools** → Add the Script Tool.
- Configure tool parameters to return human-readable responses.
- Test the tool in Studio with sample `instance_id` values.

### 5. Flow Designer integration
- Update the existing flow or create an execution plan that:
  - Accepts incident context (incident.sys_id).
  - Optionally calls AI Search as before.
  - Invokes the AI Agent (conversation) when interactive remediation is desired.
  - If agent executes remediation, capture results in the flow and link to incident.

### 6. UI / Approval
- Keep the manual **Trigger EC2 Remediation** button.
- For agent flow, require the agent to explicitly request and receive human approval before calling the tool.

---

## Testing & Validation

### Conversational scenarios
- **Direct**: `Restart instance i-09ae69f1cb71f622e`
- **Incident-based**: `Help me solve incident INC0001234` (agent reads short_description)
- **Invalid**: malformed IDs (agent asks for correction)
- **Approval**: agent must pause and await explicit "approve" before executing

### Integration checks
- Confirm API requests match manual path (endpoint, method, headers, body).
- Confirm Remediation Log entries are created with identical fields:
  - `ec2_instance` (reference).
  - `attempted_status`, `timestamp`, `request_payload`, `response_payload`, `http_status_code`, `response_time_ms`, `error_message`.
- Confirm `sn_aia_agent_tool_m2m` relationships exist.
- Confirm conversation logs appear in AI Agent Studio.

### Acceptance criteria
- Agent extracts instance IDs in ≥ one tested incident.
- Agent waits for and records human approval before action.
- Agent calls the same API and creates a Remediation Log row identical in structure to the UI action path.
- Manual and conversational remediations produce comparable results in logs and instance status.

---

## Script comparison (manual vs agent tool)

| Item | Manual (EC2RemediationHelper) | Agent Script Tool |
|------|-------------------------------|-------------------|
| Input | `instance_id` or passed `sys_id` | `instance_id` (from conversation or incident) |
| Lookup | query by sys_id or instance_id | query by instance_id → get sys_id |
| Execution | RESTMessageV2 call to AWS Integration Server | same call via Script Tool code |
| Approval | human clicks UI button | agent requests and waits for approval |
| Logging | writes Remediation Log | writes Remediation Log (same fields) |
| Return | JSON-like object | structured response for agent + readable message |

---

## Update set instructions (what to capture)

Before testing the agent, **make the update set current** and capture:

**AI Agent components**
- `sn_aia_agent` record: *EC2 Remediation Assistant*
- `sn_aia_agent_tool_m2m` records linking agent → script tool
- `sn_aia_agent_tool` (the tool record)
- `sn_aia_execution_plan` and related execution records
- `sn_aia_execution_task` (or whatever execution task table your instance uses) — capture all tasks tied to the plan

**Existing system components (from WL2)**
- `x_snc_ec2_monito_0_ec2_instance` (sample records)
- `x_snc_ec2_monito_0_remediation_log` (sample records)
- `EC2RemediationHelper` Script Include (if kept or referenced)
- UI Action: `Trigger EC2 Remediation`

**Integration & security**
- Connection & Credential Alias
- HTTP Connection record
- Basic Auth Credentials (do not include secrets in GitHub — record references only)
- Table ACLs for:
  - `EC2 Instance`
  - `Remediation Log`
  - Script Include ACL
  - UI Action ACL

**Other**
- Flow Designer execution plan or flow (force save)
- Knowledge Article records used by AI Search (if referenced)
  
> Before exporting update set: remove Slack webhook secrets from any flow action. Replace with placeholder text.

---

## How to add records to your update set (quick steps)

1. Open `System Update Sets > Local Update Sets`. Create `EC2 AI Agent Enhancement`. Set to *In Progress* and Make Current.
2. Create agent and tools in AI Agent Studio.
3. For each created item (agent, tool, plan, tasks): open the record → Related Links → **Add to Update Set** (or use the table list view and right-click → Add to Update Set).
4. For execution plan tasks: open plan → open tasks list → select all tasks → Right-click → Add to Update Set.
5. For `sn_aia_agent_tool_m2m` and related records: open table list view, filter by your agent sys_id → add to update set.
6. When done, change update set state to **Complete**, then **Retrieved Update Sets** → Export to XML.

---

## Deliverables (what to include in GitHub)

- `ec2-ai-agent-enhancement.xml` (update set export)
- `README.md` (this file)
- `Diagram.png` (enhanced architecture)
- Screenshots:
  - Agent record (`sn_aia_agent`)
  - Script Tool record and schema
  - `sn_aia_agent_tool_m2m` entries (show relationship)
  - Execution plan and tasks
  - Remediation Log entries created by agent
  - Slack message (agent-driven) [remove webhook token before commit]
  - Flow/Incident created when agent executes

---

## Quick test checklist

- [ ] Agent extracts ID from incident short_description
- [ ] Agent requests explicit approval
- [ ] Agent calls remediation API after approval
- [ ] Remediation Log entry created and matches manual path
- [ ] Manual UI Action still works unchanged
- [ ] Conversation logs available in AI Agent Studio
- [ ] Update set contains `sn_aia_agent_tool_m2m` and execution tasks

---

## Notes & tips

- Use supervised mode for the Script Tool while testing. This forces the human-in-the-loop step.
- Use a sandbox Slack webhook for tests. Remove the real webhook before exporting the update set.
- Keep the manual UI action as fallback. It gives a quick safety path during early testing.

---

If you want, I can:
- produce the exact **Agent Tools JSON schema** you can paste into the Script Tool setup, and  
- generate the **update-set checklist table** formatted for insertion in your README.

Which one do you want next?


