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

`Extract EC2 ID from incident, ask for approval, then call remediation API and log results.`

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



