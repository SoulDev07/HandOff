# Product Requirements Document
# Support Ticket Assignment

**Status:** Draft
**Version:** 1.3
**Last Updated:** 12 August 2026

---

## 1. Overview

Support agents work across different hours, days, and timezones. When team leads manually route tickets, the process slows down as the team grows.

This product automatically assigns incoming tickets to available agents. It prioritizes quick responses by choosing agents who can start working the ticket soonest, while keeping workloads balanced across the team based on capacity.

---

## 2. Problem Statement

Manual assignment causes several operational problems:
* Tickets sit unassigned when team leads are offline or busy.
* Team leads spend hours triaging instead of managing the team.
* Workload is uneven, leaving some agents overwhelmed while others have light queues.
* Off-hours assignments send tickets to agents outside their working shifts.
* Round-robin assignment ignores differences in weekly capacity and past workload.
* Teams lack a record explaining why specific agents received tickets.

---

## 3. Goals & Success Criteria

### Primary Goals
1. Automatically route tickets to available working agents.
2. Route off-hours tickets to the earliest scheduled agent with available capacity (up to 7 days ahead).
3. Enforce weekly capacity limits so agents are not overassigned.
4. Balance work using short-term capacity percentages and long-term history.
5. Provide clear text explanations for every assignment decision.

### Success Criteria
* Automatically assign every ticket for which an eligible agent exists within the next 7 days; otherwise leave it unassigned with a clear reason.
* Off-hours tickets queue to the agent starting work earliest.
* The dashboard shows current coverage gaps and unassigned tickets.

---

## 4. Target Users

### Team Lead
Manages team setup and monitors routing. Responsibilities include:
* Setting up agent schedules, timezones, and weekly capacity limits.
* Configuring company workdays, company timezone, and default ticket effort estimates.
* Monitoring team coverage and capacity bottlenecks.
* Reviewing unassigned tickets and assignment reasons.

### Support Agent
Receives and resolves tickets.
* Has a set timezone, work schedule, and weekly capacity.
* Does not configure routing rules directly.

---

## 5. Scope

### In Scope
* **Company Settings:** Workday schedule, company timezone (for dashboard display and company workday interpretation), and effort estimates by priority (for example: P1 = 8h, P4 = 1h).
* **Agent Management:** Timezones, daily shift schedules (one block per day), and weekly capacity limits (in hours).
* **Assignment Engine:** Rules that evaluate availability and capacity to pick the best agent.
* **Coverage Dashboard:** Real-time view of active agents, priority capacity, and unassigned tickets.

### Out of Scope
* Authentication and user roles.
* Split shifts (multiple availability blocks per day).
* Holidays and PTO calendars.
* Skill-based routing and AI classification.
* Support for multiple teams per company.

---

## 6. Assumptions

* **Single Team:** Each company has one support team pool.
* **Equal Capabilities:** All agents can take any ticket.
* **Contiguous Shifts:** Agents work one block of time per day.
* **Fixed Effort:** Ticket effort depends only on its priority setting.
* **Strict Limits:** High priority tickets do not bypass capacity limits or shift hours.
* **Pre-Categorized Tickets:** Incoming tickets arrive with priority already set.

---

## 7. Assignment Model

### 7.1 Availability & Timezones
Agents set schedules in their local IANA timezone. The engine expands recurring local schedules into concrete UTC shift intervals `(shift_start_utc, shift_end_utc)` to handle daylight saving changes, cross-midnight shifts, and look-ahead evaluations on a unified UTC timeline. If a shift crosses midnight (for example, Monday 22:00 to Tuesday 06:00 local time), it is expanded as a single continuous shift interval starting at `shift_start_utc`. Each agent's weekly capacity is evaluated according to Monday - Sunday in their local timezone. The company timezone defines company-wide workday boundaries and formats coverage views on the dashboard.

### 7.2 Ticket Priority & Effort
Every ticket priority maps to a configurable default effort in hours:

| Priority | Label | Default Effort |
| :--- | :--- | :---: |
| **P1** | Critical | 8h |
| **P2** | High | 4h |
| **P3** | Normal | 2h |
| **P4** | Low | 1h |

* **Invalid or Missing Priority Handling:** Tickets must specify a valid priority (`P1`, `P2`, `P3`, or `P4`). If a ticket arrives with a missing or invalid priority, the system rejects assignment with a validation error (`400 Bad Request`) and leaves the ticket unassigned with the reason *"Invalid or missing ticket priority: effort cannot be determined."* Defaulting to a lower priority (such as P4) is explicitly prohibited to prevent under-budgeting critical tickets and masking upstream data integration bugs.
* **Target Shift Week Allocation:** Ticket effort is deducted 100% from the planning week containing the target shift's start date. It is never split across weeks and is not deducted from the arrival week.
* **Look-Ahead Capacity Budgeting:** Look-ahead eligibility checks evaluate capacity against the target shift's planning week. For example, if a ticket arrives on Friday (Week 1) and is assigned to a Monday shift (Week 2), the entire effort is deducted from Week 2's capacity budget, leaving Week 1 untouched.
* **Multi-Shift Work:** Ticket effort represents a budget allocation for the target week, not a requirement to finish the ticket within a single shift.

### 7.3 Capacity & Workload
* **Weekly Capacity:** Maximum hours an agent can spend on tickets per week.
* **Weekly Workload:** Sum of estimated effort (in hours) of all tickets assigned to the agent for the target planning week (regardless of status).
* **Unresolved Tickets Across Week Boundaries (Non-Rollover Rule):** Ticket effort represents a fixed initial budget allocation charged 100% to the target planning week (`target_shift_start`). If a ticket remains unresolved (Open or In Progress) into a subsequent week, its effort remains budgeted in its original target week and does not roll over into the new week's capacity budget. The new week starts with a fresh capacity budget. Unresolved tickets continue to be tracked under `Active Workload` for operational visibility on the dashboard, but do not reduce future weekly capacity budgets. Effort is only removed if a ticket is explicitly cancelled or unassigned by a team lead.
* **Remaining Weekly Budget:** Hours left in the agent's weekly ticket capacity:
  $$\text{Remaining Weekly Budget} = \text{Weekly Ticket Capacity} - \text{Weekly Workload}$$
* **Capacity Rate:** The proportion of scheduled shift hours dedicated to ticket work, capped at 1.0:
  $$\text{Capacity Rate} = \min\left(1.0, \frac{\text{Weekly Ticket Capacity}}{\text{Total Scheduled Availability Hours}}\right)$$
  Capping the Capacity Rate at 1.0 ensures that Capacity Rate represents a valid proportion of scheduled shift time ($\le 100\%$). An agent cannot dedicate more time to ticket work during a shift than the shift's actual physical duration. If total scheduled hours or ticket capacity is 0, Capacity Rate is 0.
* **Time-Supported Remaining Capacity:** Ticket work that fits into the agent's remaining shift hours:
  $$\text{Time-Supported Remaining Capacity} = \text{Remaining Scheduled Shift Hours} \times \text{Capacity Rate}$$
* **Effective Remaining Capacity:** The lower value between remaining weekly capacity and time-supported capacity:
  $$\text{Effective Remaining Capacity} = \min(\text{Remaining Weekly Budget}, \text{Time-Supported Remaining Capacity})$$
  For Tier 1 immediate assignment, an agent's effective remaining capacity must be greater than or equal to the ticket effort; otherwise, the ticket shifts to look-ahead routing.
* **Partial and Zero Hours Handling:** Calculations preserve exact fractional hours (for example, 2.5 remaining shift hours at a 0.8 capacity rate yields 2.0 hours of time-supported capacity). If an agent has 0 weekly capacity or 0 scheduled hours, effective capacity is 0, rendering the agent ineligible for assignment.

### 7.4 Fairness Metrics
The engine compares agents using percentage utilization rather than raw ticket counts:
* **Projected Utilization (Short-Term):** Expected workload percentage after taking the ticket:
  $$\text{Projected Utilization} = \frac{\text{Weekly Workload} + \text{Ticket Effort}}{\text{Weekly Ticket Capacity}}$$
  * **Numerator:** `Weekly Workload + Ticket Effort` (where `Weekly Workload` is the total effort of all tickets assigned to the agent for the target shift's planning week).
  * **Denominator:** `Weekly Ticket Capacity` (the agent's weekly ticket capacity in hours).
  * **Target Week Scoping:** Scoped to the planning week containing the target shift. Tier 1 evaluates the current planning week. Look-ahead tiers evaluate the specific planning week to which the target shift belongs. Tickets assigned to future shifts count in their target shift's planning week budget, not the arrival week.
* **Rolling Utilization (Long-Term):** Historical workload percentage over the agent's recent working history:
  $$\text{Rolling Utilization} = \frac{\text{Effort Assigned During Rolling Window}}{\text{Capacity Rate} \times \text{Scheduled Hours During Rolling Window}}$$
  * **Window Definition:** Evaluates the agent's previous 30 available working days (calendar dates on which the agent is scheduled to work on a company workday).
  * **History Rules:**
    * **Fewer than 5 available working days:** Use the team's current rolling utilization average. If the entire team lacks sufficient history, default to 0%.
    * **5 to 29 available working days:** Calculate using all available working days of history rather than waiting for 30 days.
    * **30 or more available working days:** Cap the window at the most recent 30 available working days.

---

## 8. Assignment logic

Availability determines the assignment tier. Within the same availability tier, fairness determines the assignee.

### Availability vs. Fairness Trade-Off
The system intentionally prioritizes **Availability (Earliest Response)** over absolute **Fairness**:
* **Why Availability takes priority:** Evaluating absolute fairness across the entire 7-day window would route tickets to the least-loaded agent even if their next shift starts 3 days later, causing customer tickets to sit unserviced.
* **How Tiers enforce priority:** Earliest shift start time takes precedence in look-ahead routing. Fairness metrics (Projected and Rolling Utilization) are used to pick among active agents in Tier 1, or as tie-breakers in Tier 2 when multiple eligible agents share the exact same earliest shift start time.

Routing proceeds through three tiers:

1. **Tier 1 (Immediate active assignment):** 
   * Check agents currently on shift.
   * Filter out agents without enough Effective Remaining Capacity.
   * Select the eligible agent with the lowest Projected Utilization.
   * If the difference between the lowest projected utilization and another candidate is ≤ 10 percentage points, those candidates are considered tied. For tied candidates, compare their Rolling Utilization and select the candidate with the lower value.

2. **Tier 2 (7-day look-ahead):** 
   * If no active agent is eligible, evaluate upcoming shifts starting within the 168-hour UTC look-ahead window (`ticket_created_at_utc` to `ticket_created_at_utc + 168h`).
   * Assign to the eligible agent whose shift starts earliest.
   * If multiple eligible agents share the same earliest shift start, use Projected Utilization and Rolling Utilization to select between them.

3. **Tier 3 (Unassigned):** 
   * If no agent has an eligible working shift and sufficient capacity within the 168-hour look-ahead window, leave the ticket unassigned with a recorded reason.

* **Sticky Assignment Scope:** Once a ticket is assigned, the assignment is sticky. Changes to schedules, timezones, or weekly capacity apply strictly to new tickets arriving after the update ("future assignments"). Existing assignments (including tickets assigned via look-ahead to future shifts) remain unchanged and are never automatically re-routed by the engine. Re-assignment requires an explicit manual action by a team lead.

---

## 9. User Interface and views

The management interface acts as both a configuration portal and an operational control center for team leads.

### 9.1 Configuration & Management Workflows

#### 9.1.1 Agent Roster & Capacity Management
* **Add Agent Flow:** Modal/form requiring Agent Name, local IANA Timezone (searchable dropdown, for example `Asia/Kolkata`), and Weekly Ticket Capacity in hours (the system automatically generates a unique `agent_id`).
* **Edit Agent Flow:** Form to update an agent's timezone, weekly ticket capacity, or personal details.
* **Agent Deactivation (Soft-Delete):** Setting an agent's status to `Inactive`. Inactive agents are immediately excluded from Tier 1 and Tier 2 routing algorithms. Historical assignment records, audit logs, and rolling utilization history remain fully preserved.
* **Independent Capacity Setting:** Weekly Ticket Capacity and Availability Schedule are configured independently without imposing artificial constraints (for example, senior agents can be assigned higher weekly ticket capacity than scheduled shift hours).

#### 9.1.2 Agent Schedule Builder
* **Weekly Schedule Grid:** Per-day availability builder (Monday through Sunday) configured in the agent's local IANA timezone.
* **Daily Shift Window:** Each day toggles `On` or `Off`. When active, sets a single contiguous shift window (`Start Time` and `End Time`).

#### 9.1.3 Company Settings View
* **Company Workdays:** Checkboxes to select active operating days for the company pool (for example, Monday through Friday).
* **Company Timezone:** Searchable dropdown setting the primary company timezone used for dashboard formatting and workday boundaries.
* **Priority Effort Mapping:** Configurable number inputs setting default estimated effort in hours for each priority level (P1 = 8h, P2 = 4h, P3 = 2h, P4 = 1h).

#### 9.1.4 Manual Ticket Reassignment Flow
* **Reassignment Action:** The team lead can select any assigned ticket (active or future look-ahead) and click **"Reassign Ticket"**.
* **Reassignment Modal:** Displays a list of available/scheduled agents evaluated against the ticket's target shift planning week. The lead selects a new assignee (with optional manual capacity override if the lead chooses to over-assign an agent).
* **Capacity Rebalancing & State Update:** Upon manual reassignment:
  1. The system **releases** (deducts) the ticket effort from the previous agent's `Weekly Workload` for their target planning week.
  2. The system **allocates** (adds) the ticket effort to the new agent's `Weekly Workload` for their target planning week, and updates `target_shift_start` to match the new shift.
  3. A new structured audit record is persisted (`assignment_tier: "manual_reassignment"`, `previous_agent_id`, `new_agent_id`, `reassigned_by_lead: true`, and `reason`).

---

### 9.2 Operations Control Center (Coverage Dashboard)

#### 9.2.1 Top-level routing health bar
Surfaces current system state at a glance:
* **Active vs. Scheduled Agents:** Differentiates between agents currently on shift (`shift_start_utc <= now_utc <= shift_end_utc`) and agents scheduled to work later on the current workday.
* **Overall System Health Indicator:**
  * **Healthy (Green):** Active agents working and all ticket priorities (P1 to P4) currently covered.
  * **Limited Capacity (Yellow):** Active agents working, but capacity for critical priorities (for example, P1) is exhausted.
  * **Future Covered (Orange):** No active agent available now, but an eligible look-ahead shift exists within 7 days.
  * **Unassignable / No Coverage (Red):** No active or upcoming eligible shift available within 7 days.

#### 9.2.2 Priority capacity matrix
Answers whether the team can handle new incoming tickets right now:
* Displays counts of active agents with Effective Remaining Capacity $\ge$ Ticket Effort for each priority (P1 = 8h, P2 = 4h, P3 = 2h, P4 = 1h).
* Displays the next eligible shift timestamp if current active capacity for a priority level is 0.

#### 9.2.3 Coverage timeline
Provides a visual timeline of agent shifts across a **168-hour planning window** (`now_utc` to `now_utc + 168h`), matching the assignment engine's 7-day look-ahead window. It flags two distinct timeline issues:
* **Scheduled Coverage Gap:** Time windows within the 168-hour window where 0 agents are scheduled to work on a company workday.
* **Capacity Gap:** Time windows where agents are working, but their combined effective capacity is insufficient for higher-priority tickets.

#### 9.2.4 Capacity-first agent table
Exposes exact agent-level routing metrics:
* **Agent Status and Schedule:** Shift window, local timezone, and current active status (Working, Away, Unavailable).
* **Capacity Metrics:** Weekly Capacity, Remaining Weekly Budget, and Effective Remaining Capacity.
* **Fairness Metrics:** Projected Utilization percentage and 30-Day Rolling Utilization percentage.
* **Priority Eligibility:** Per-priority eligibility indicators (`P1`, `P2`, `P3`, `P4`) with reasons if restricted (for example: *"Cannot accept P1: 4h effective capacity remaining"*).
* **Capacity State Categories:** **Healthy** (can accept P1), **Limited** (can accept P3/P4 but not P1), **Exhausted** (effective capacity = 0h), or **Unavailable** (outside shift).

#### 9.2.5 Attention required panel
Surfaces tickets requiring manual intervention:
* **Unassigned Tickets Queue:** Lists unassigned tickets alongside evaluation details (why Tier 1, Tier 2, and Tier 3 failed), plain-language reason strings, and a **"Retry Assignment" / "Re-run Routing" action button**.
* **Retry Action Flow:** Clicking "Retry Assignment" triggers `POST /companies/{company_id}/tickets/{ticket_id}/assignment`. If an agent has since logged in or gained capacity, the ticket transitions to `assigned`. If still unassigned, the UI updates the reason string with the latest evaluation result.
* **Future Reservation Visibility:** Displays look-ahead capacity reservations for upcoming shifts (`target_shift_start`), showing team leads how future shifts are pre-allocated.

#### 9.2.6 State and threshold definitions
To guarantee consistent dashboard behavior across implementations, states follow these explicit thresholds:
* **Dashboard Planning Window:** Evaluates the 168-hour UTC window starting from the current timestamp (`now_utc` to `now_utc + 168h`).
* **Active Agent:** Current UTC timestamp falls within expanded shift interval (`shift_start_utc <= now_utc <= shift_end_utc`).
* **Scheduled Agent:** Agent has a scheduled availability block on the current workday, regardless of whether shift has started.
* **No Active Coverage State:** Current-time condition where `active_agents == 0` at `now_utc`. Indicates no agent is currently working right now (does not necessarily mean a schedule gap exists, as shifts may start later today).
* **Scheduled Coverage Gap:** A time interval within the 168-hour planning window during which 0 agents are scheduled to work on a company workday.
* **Capacity Gap:** A time interval where agents are scheduled or active, but 0 eligible agents have Effective Remaining Capacity $\ge$ Ticket Effort for a given priority.
* **Exhausted Capacity:** Agent capacity state where Effective Remaining Capacity is 0h; or priority status where 0 active agents have Effective Remaining Capacity $\ge$ Ticket Effort for that priority.
* **Low / Limited Capacity:** Agent capacity state where an active agent cannot accept P1 (8h) but can accept lower priorities; or dashboard warning where 0 active agents can accept P1 or team projected utilization exceeds 80%.

---

## 10. API Contract Specification

### 10.1 Endpoint Definition
```http
POST /companies/{company_id}/tickets/{ticket_id}/assignment
```

### 10.2 Request & Priority Resolution
* **Path Parameters:** `company_id` (string/UUID), `ticket_id` (string/UUID).
* **Priority Source:** The engine loads the pre-created ticket record from the database using `ticket_id` to retrieve its `company_id`, `priority`, and `created_at_utc`. If a request payload is provided, its `priority` must match the ticket record.
* **Priority Validation:** Ticket priority must be valid (`P1`, `P2`, `P3`, or `P4`).

### 10.3 Response Schemas

#### A. Assigned Ticket Response (`200 OK`)
```json
{
  "ticket_id": "ticket_123",
  "company_id": "company_abc",
  "status": "assigned",
  "assigned_agent_id": "agent_42",
  "assigned_agent_name": "Ananya",
  "assigned_at": "2026-08-13T10:30:00Z",
  "target_shift_start": "2026-08-13T10:30:00Z",
  "assignment_tier": "tier_1_active",
  "metrics_at_assignment": {
    "weekly_capacity": 32.0,
    "weekly_workload": 12.0,
    "effective_remaining_capacity": 9.6,
    "projected_utilization": 0.625,
    "rolling_utilization": 0.58,
    "eligible_candidates_count": 3
  },
  "reason": "Assigned to Ananya: lowest projected utilization (62.5%) among 3 eligible active candidates."
}
```

#### B. Unassigned Ticket Response (`200 OK`)
```json
{
  "ticket_id": "ticket_125",
  "company_id": "company_abc",
  "status": "unassigned",
  "assigned_agent_id": null,
  "assigned_agent_name": null,
  "assigned_at": "2026-08-13T10:30:00Z",
  "target_shift_start": null,
  "assignment_tier": "tier_3_unassigned",
  "metrics_at_assignment": {
    "weekly_capacity": null,
    "weekly_workload": null,
    "effective_remaining_capacity": null,
    "projected_utilization": null,
    "rolling_utilization": null,
    "eligible_candidates_count": 0
  },
  "reason": "Unassigned: 0 eligible candidates found within the 168-hour look-ahead window."
}
```

#### C. Error Responses
* **`404 Not Found`:** Returned when `ticket_id` or `company_id` does not exist in the database.
```json
{
  "error": "NOT_FOUND",
  "message": "Ticket ticket_999 or Company company_abc not found."
}
```
* **`400 Bad Request`:** Returned when ticket priority is missing, invalid (not P1 to P4), or `company_id` mismatches.
```json
{
  "error": "VALIDATION_ERROR",
  "message": "Invalid or missing ticket priority: effort cannot be determined."
}
```

### 10.4 Idempotency & Retry Contract
* **Retrying an `assigned` Ticket:** Calling the endpoint for a ticket that has already been successfully assigned returns the existing assignment payload (`200 OK`) as an **idempotent no-op**. It does not re-run routing algorithms or alter capacity budgets.
* **Retrying an `unassigned` Ticket:** Calling the endpoint for a ticket whose previous evaluation returned `unassigned` **re-evaluates routing against current system state**. This allows retry jobs or UI "Retry Assignment" clicks to successfully assign tickets once agents log in or gain capacity.

---

## 11. Non-Functional Requirements

* **Determinism:** Given identical inputs, the engine returns the same result. Ties break alphabetically by Agent ID as a last resort.
* **Idempotency:** The `ticket_id` serves as the idempotent request key. Re-delivering or calling the assignment API for an already-assigned `ticket_id` is a no-op that returns the existing assignment details as-is without re-running the assignment algorithm or altering capacity budgets.
* **Concurrency & Locking:** Assignment operations execute within an isolated database transaction using row-level locking (e.g., `SELECT ... FOR UPDATE` on ticket and assignment records) alongside a database `UNIQUE(ticket_id)` constraint. If multiple workers receive the same `ticket_id` simultaneously, exactly one worker completes the assignment while concurrent requests wait and gracefully return the created assignment via the idempotent no-op path.
* **Explainability and Auditability:** Every assignment decision (whether assigned or unassigned) persists a structured audit record containing both machine-readable metrics and a human-readable text explanation.

---

## 12. Future Improvements
* Emergency manual overrides for critical tickets.
* PTO and holiday calendar integration.
* Split-shift schedule support.
