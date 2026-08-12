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
Agents set schedules in their local IANA timezone. The engine converts schedules to UTC to handle overnight shifts and daylight saving changes. Each agent's weekly capacity is evaluated according to Monday - Sunday in their local timezone. The company timezone is used to define company-wide workday boundaries and present real-time coverage on the dashboard.

### 7.2 Ticket Priority & Effort
Each priority maps to an estimated effort in hours (for example: P1 = 8h, P3 = 2h). When assigned, this effort deducts from the agent's weekly budget for the target planning week. Ticket effort represents a budget allocation and does not require an agent to complete the entire ticket within a single shift.

### 7.3 Capacity & Workload
* **Weekly Capacity:** Maximum hours an agent can spend on tickets per week.
* **Capacity Rate:** The proportion of scheduled shift hours dedicated to ticket work:
  $$\text{Capacity Rate} = \frac{\text{Weekly Ticket Capacity}}{\text{Total Scheduled Availability Hours}}$$
  If total scheduled hours or ticket capacity is 0, Capacity Rate is 0.
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
* **Rolling Utilization (Long-Term):** Historical workload percentage over the agent's recent working history:
  $$\text{Rolling Utilization} = \frac{\text{Effort Assigned During Rolling Window}}{\text{Capacity Rate} \times \text{Scheduled Hours During Rolling Window}}$$
  * **Window Definition:** Evaluates the agent's previous 30 available working days (calendar dates on which the agent is scheduled to work on a company workday).
  * **History Rules:**
    * **Fewer than 5 available working days:** Use the team's current rolling utilization average. If the entire team lacks sufficient history, default to 0%.
    * **5 to 29 available working days:** Calculate using all available working days of history rather than waiting for 30 days.
    * **30 or more available working days:** Cap the window at the most recent 30 available working days.

---

## 8. Assignment Logic

Availability determines the assignment tier. Within the same availability tier, fairness determines the assignee.

Routing proceeds through three tiers:

1. **Tier 1 (Immediate Active Assignment):** 
   * Check agents currently on shift.
   * Filter out agents without enough Effective Remaining Capacity.
   * Select the eligible agent with the lowest Projected Utilization.
   * If the difference between the lowest projected utilization and another candidate is ≤ 10 percentage points, those candidates are considered tied. For tied candidates, compare their Rolling Utilization and select the candidate with the lower value.

2. **Tier 2 (7-Day Look-Ahead):** 
   * If no active agent is eligible, evaluate upcoming shifts over the next 7 days.
   * Assign to the eligible agent whose shift starts earliest.
   * If multiple eligible agents share the same earliest shift start, use Projected Utilization and Rolling Utilization to select between them.

3. **Tier 3 (Unassigned):** 
   * If no agent has an eligible working shift and sufficient capacity within the next 7 days, leave the ticket unassigned with a recorded reason.

Changes to schedules, timezones, or weekly capacity affect future assignments only. Existing assignments remain unchanged.

---

## 9. User Interface & Views

### Coverage Dashboard
Shows current coverage state:
* Active vs. scheduled agents.
* Available capacity by ticket priority.
* Warnings when capacity is low or exhausted.

### Unassigned Tickets Panel
Lists unassigned tickets alongside the specific reason recorded by the engine.

---

## 10. Non-Functional Requirements

* **Determinism:** Given identical inputs, the engine returns the same result. Ties break alphabetically by Agent ID as a last resort.
* **Concurrency:** Database locks prevent assigning the same ticket twice.
* **Explainability:** Every assignment decision records a plain-text reason.

---

## 11. Future Improvements
* Emergency manual overrides for critical tickets.
* PTO and holiday calendar integration.
* Split-shift schedule support.
