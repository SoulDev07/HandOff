# Product Requirements Document
# Support Ticket Assignment

**Status:** Draft  
**Version:** 1.1
**Last Updated:** 12 August 2026

---

## 1. Overview

Support teams typically consist of agents working varied hours, days, and timezones. Currently, team leads manually monitor the ticket queue and assign incoming tickets. While this works for small teams, it becomes unmanageable as the team scales.

This product aims to fully automate the assignment of new support tickets to the appropriate available agents. It is designed to balance two competing priorities:
1. **Availability (Rapid Response):** Ensuring tickets are assigned to someone who can start working on them as soon as possible.
2. **Fairness (Workload Balance):** Ensuring that no agent is overwhelmed and that work is distributed equitably based on individual capacity.

The system will give team leads full visibility into team coverage, capacity constraints, and assignment rationale.

---

## 2. Problem Statement

The manual assignment process results in several critical issues:
* Tickets sit unassigned when the team lead is unavailable.
* Team leads spend excessive time triaging tickets instead of focusing on higher-level management.
* Workload distribution is often uneven; some agents are overloaded while others sit idle.
* Agents in different timezones may be assigned work outside their active hours.
* Simple round-robin systems fail to account for agents with different weekly capacities or those who have historically carried a heavier load.
* There is no historical record explaining *why* a ticket was assigned to a specific agent.

## 3. Goals & Success Criteria

### Primary Goals
1. **Automate Assignment:** Automatically route new tickets to an eligible working agent.
2. **Intelligent Look-Ahead:** If no one is currently working, intelligently look ahead (up to 7 days) and assign the ticket to the earliest scheduled agent with sufficient capacity.
3. **Respect Capacity:** Never assign a ticket if it exceeds the agent's remaining capacity in the target planning week.
4. **Ensure Fairness:** Distribute work based on proportional capacity (short-term) and historical workload (long-term).
5. **Transparency:** Provide a plain-language explanation for every assignment decision.

### Success Criteria
* 100% of eligible tickets are automatically assigned; tickets with no eligible agent remain unassigned with a clear reason.
* Off-hours tickets are correctly queued to the agent starting work the earliest.
* Agents report a more balanced workload, and no agent exceeds their defined weekly capacity.
* The team lead dashboard accurately surfaces coverage gaps and unassigned tickets in real-time.

---

## 4. Target Users

### Team Lead
The primary user of the management UI. Their responsibilities include:
* Managing agent profiles, timezones, and weekly availability schedules.
* Configuring global company settings (workdays, default ticket effort estimates).
* Monitoring the real-time coverage dashboard to identify capacity bottlenecks.
* Reviewing unassigned tickets and understanding the system's assignment logic.

### Support Agent
The end-user who receives and resolves tickets. 
* They have defined timezones, availability blocks, and maximum weekly ticket capacities.
* They interact with the tickets but do not configure the assignment engine directly.

---

## 5. Scope

### In Scope
* **Company Settings:** Definition of company workdays, timezones, and default ticket efforts (e.g., P1 = 8 hours, P4 = 1 hour).
* **Agent Management:** Configuration of individual agent timezones, daily availability windows (one block per day), and total weekly ticket capacity (in hours).
* **Assignment Engine:** The core algorithm that evaluates agent eligibility and selects the optimal candidate.
* **Coverage Dashboard:** Real-time visibility into who is working, available capacity by priority, and unassigned ticket queues.

### Out of Scope
* Authentication and role-based access control.
* Multiple availability blocks per day (split shifts).
* Holiday calendars and PTO tracking.
* Agent skill-based routing or AI ticket classification.
* Multiple support teams within a single company.

---

## 6. Assumptions

To keep the initial version focused, the following product decisions and assumptions have been made:
* **One Team per Company:** A company has only one global support team pool.
* **Universal Capability:** Every agent is assumed to be capable of handling every ticket (no domain or skill-based routing).
* **Single Availability Block:** Agents work a single contiguous block of time per day.
* **Fixed Ticket Effort:** Ticket effort is strictly derived from its global priority setting. Individual tickets cannot have custom effort values.
* **No Rules Overrides:** High-priority tickets (e.g., P1) cannot override an agent's maximum capacity limits or availability schedules.
* **External Ticket Creation:** The system assumes tickets are created upstream and fed into this assignment engine with their priority already defined.

---

## 7. Assignment Model

### 7.1 Availability & Timezones
Agents define schedules in their local IANA timezone. The system converts schedules to UTC for consistent availability and look-ahead calculations, including DST and overnight shifts. The weekly planning budget resets at Monday 00:00 in each agent's local timezone to avoid mid-shift capacity resets across global teams.

### 7.2 Ticket Priority & Effort
Every ticket priority is mapped to a configurable effort estimate (in hours). For example, a P1 ticket might cost 8 hours of capacity, while a P3 ticket costs 2 hours. This effort is directly deducted from the agent's available capacity when assigned.

### 7.3 Capacity & Workload
* **Weekly Capacity:** The maximum number of hours an agent is expected to spend on ticket work per week.
* **Effective Remaining Capacity:** The system calculates the lesser of an agent's remaining weekly capacity and the actual time-supported hours left in their scheduled shifts. For Tier 1 immediate assignment, an agent's effective remaining capacity must be greater than or equal to the ticket's effort; otherwise, the ticket falls back to look-ahead tiers.

### 7.4 Fairness Metrics
The system does not just count tickets; it calculates utilization percentages to account for agents with different capacities.
* **Projected Utilization (Short-Term):** Measures how loaded an agent will be if they take the incoming ticket.
* **Rolling Utilization (Long-Term):** Measures workload over the past 30 available working days to prevent repeatedly assigning disproportionate workload to the same agents.

---

## 8. Assignment Logic

The system prioritizes **Availability** over **Fairness**. It wants to get the ticket to someone as fast as possible, but will distribute it fairly among those who are available. 

The assignment engine operates in a tiered structure:

1. **Tier 1 (Immediate Active Assignment):** 
   * The system checks all agents currently working.
   * Filters out anyone without enough *Effective Remaining Capacity*.
   * If multiple agents are eligible, it selects the agent with the lowest **Projected Utilization**.
   * If projected utilization differs by no more than 10 percentage points, candidates are considered tied, it falls back to the agent with the lowest **Rolling Utilization**.

2. **Tier 2 (Current Week Look-Ahead):** 
   * If no eligible agents are currently working, the system looks ahead at the remaining shifts in the current week.
   * It assigns the ticket to the eligible agent whose shift starts the earliest. 

3. **Tier 3 (7-Day Look-Ahead):** 
   * If no one can take it this week, the search expands up to 7 days into the future, picking the earliest eligible shift.

4. **Tier 4 (Unassigned):** 
   * If the team is completely at capacity for the next 7 days, the system halts. The ticket is marked as **Unassigned** and flagged for the Team Lead.

*Note: Once a ticket is assigned, the assignment is sticky. The system will not automatically re-assign tickets if an agent's schedule changes.*

---

## 9. User Interface & Views

### Coverage Dashboard
A real-time overview for the Team Lead that displays:
* How many agents are currently scheduled vs. actively available.
* A breakdown of available capacity (e.g., "3 agents can take a P2 ticket, but no agents have capacity for a P1 ticket").
* Clear warnings if the team is approaching maximum utilization.

### Unassigned Tickets Panel
A dedicated view for tickets that the system could not route. Crucially, each ticket displays the plain-language reason generated by the system (e.g., *"Unassigned: All active and upcoming shifts for the next 7 days lack sufficient capacity to absorb 8h of effort."*)

---

## 10. Non-Functional Requirements

* **Determinism:** The assignment engine must be fully deterministic. Given the same inputs, it must yield the same assignee. Ties are broken alphabetically by Agent ID as a last resort.
* **Concurrency:** The system must utilize database locks or transactions to prevent race conditions where a single ticket is assigned to multiple agents.
* **Explainability:** The algorithm must never be a "black box." Every decision must yield a clear, human-readable justification.

---

## 11. Future Improvements
* Allow Team Leads to initiate an "Emergency P1 Override" to bypass capacity rules.
* Implement PTO integration and Holiday calendars.
* Support split shifts and multiple availability blocks per day.
