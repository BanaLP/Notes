# Design: Timeframe From Jira Story Skill (v2)

## Status

Design approved through brainstorming checkpoints. Supersedes v1 design.

## Skill Name

`timeframe-from-story`

## Purpose

Calculate an engineering-only implementation timeframe from a Jira Story ticket, assign engineers based on ticket weight (heavy vs supporting), then optionally write the calculated schedule and assignee into each FE/BE Sub-task Engineer ticket using Jira fields:

- `Expected Start Date[Date]`
- `Expected End Date[Date]`
- `assignee`

The skill is intended for story-level planning where the user provides a Jira Story key or URL, engineer roster/capacity, desired start date, and any remaining SP commitments from other stories.

## Trigger Examples

- `Buat timeframe dari story CORB-71`
- `Generate timeline story https://lionparcel.atlassian.net/browse/CORB-71`
- `Hitung jadwal pengerjaan story <KEY>`

## Core Assumptions

- 1 SP = 1 hour of work.
- A working day has 8 hours, but engineer delivery capacity is configurable by SP/day.
- Default capacity when omitted:
    - Senior: 6 SP/day
    - Junior: 5 SP/day
- Ticket assignment is weight-aware:
    - Heavy tickets (complex analysis, high SP) → Senior first
    - Supporting tickets → Junior first
    - Senior overflow → Supporting tickets for full capacity utilization
- FE and BE are scheduled as parallel tracks by default.
- QA/UAT timeline is out of scope for v1.
- Weekends are non-working days.
- User can provide custom holidays/non-working dates.
- All required user input and confirmations must use OpenCode `question()`.

## Input Model

### Story Input

User can provide either:

- Jira URL: `https://lionparcel.atlassian.net/browse/CORB-71`
- Jira key: `CORB-71`

The skill extracts and normalizes the Story key.

### Engineer Roster Input

Roster uses simple list format:

```text
BE: Andi senior, Budi junior
FE: Cici senior, Dedi junior
```

Or with custom capacity:

```text
BE: Andi senior 8SP/day, Budi junior 6SP/day
```

If capacity is omitted, the skill applies defaults by level:

- Senior → 6 SP/day
- Junior → 5 SP/day

**All-junior detection:** If all engineers in a track are junior, the skill automatically switches to "distribute all evenly" mode — skips heavy/supporting classification for that track.

### Remaining SP Input

After start date and roster, the skill asks via `question()`:

```
Apakah ada engineer yang sudah punya sisa SP dari story lain?
Format: <Nama> <tanggal> <sisa SP>
Contoh: Andi 2026-06-16 3SP, Budi 2026-06-17 2SP
(Kosongkan jika tidak ada)
```

The skill parses this into a map:

```json
{
  "Andi": { "2026-06-16": 3 },
  "Budi": { "2026-06-17": 2 }
}
```

During scheduling, available SP for an engineer on a given date = capacity − remaining SP.

### Calendar Input

The skill asks via `question()` for:

- Start date in `YYYY-MM-DD` format.
- Custom holidays/non-working dates, comma-separated, optional.

If the start date falls on a weekend or custom holiday, the skill asks via `question()` whether to shift to the next working day or keep the date.

### Story Point Source

For each Sub-task Engineer, the skill attempts to read SP in this order:

1. `Story Point[Number]` / `Story Point`
2. `Story Point Dev[Number]` / `Story Point Dev`
3. If both are missing/unreadable, ask user via `question()` for that ticket's SP.

## Ticket Classification

### Auto-Detection Logic

A ticket is classified as **HEAVY** if:

- SP >= threshold (default 5, overridable by lead) **AND**
- Description contains at least one keyword from the heavy keyword list

A ticket is classified as **SUPPORTING** if neither condition is met.

If only one condition is met (high SP but no keyword, or keyword but low SP), the skill presents the ticket to the lead for manual classification via `question()`.

### Heavy Keyword List

```
analisa, analysis, investigate, complex, migration, refactor,
architecture, integration, debug, performance, optimization
```

### Classification Preview

After fetching child issues, the skill displays:

```
📋 Ticket Classification
[BE][Horde] Update client approval (12 SP) → 🔴 HEAVY (SP≥5 + keyword: 'analysis')
[BE][Horde] Add logging middleware (3 SP) → 🟢 SUPPORTING
[BE][Horde] Refactor auth module (8 SP) → 🔴 HEAVY (SP≥5 + keyword: 'refactor')

Konfirmasi klasifikasi atau override?
```

The lead can override classification per ticket via `question()`.

### All-Junior Track Behavior

If all engineers in a track are junior:

- Skip heavy/supporting classification for that track.
- All tickets are treated equally and distributed evenly across juniors by capacity.

## Scheduling Algorithm: Priority Queue with Capacity Fill

### Pre-Processing

1. Build day-grid per engineer:
    - For each working day starting from start date
    - `available_sp[engineer][date] = capacity[engineer] - remaining_sp[engineer][date]`

2. Classify tickets per track:
    - `heavy_queue[]` = tickets classified as HEAVY
    - `supporting_queue[]` = tickets classified as SUPPORTING
    - Sort each by Jira created order

### Phase 1: Heavy Tickets → Senior

For each heavy ticket in `heavy_queue`:

1. Find the senior in the same track with the earliest available slot.
2. Tie-breaker: senior with lowest total assigned SP.
3. Allocate SP across working days:
    - Day 1: `min(ticket_remaining_sp, available_sp[senior][day1])`
    - Day 2: `min(remaining, available_sp[senior][day2])`
    - Continue until ticket fully allocated.
4. Update `available_sp` grid.
5. Record: ticket → assignee, start_date, end_date.

### Phase 2: Supporting Tickets → Junior

For each supporting ticket in `supporting_queue`:

1. Find the junior in the same track with the earliest available slot.
2. Tie-breaker: junior with lowest total assigned SP.
3. Allocate SP across working days (same logic as Phase 1).
4. Update `available_sp` grid.
5. Record: ticket → assignee, start_date, end_date.

### Phase 3: Capacity Fill (Senior ← Supporting Overflow)

For each senior in track:

1. Calculate `total_assigned` = sum of all SP assigned to this senior.
2. Calculate `total_capacity` = sum of `available_sp` across all scheduled days.
3. If `total_assigned < total_capacity` AND `supporting_queue` has unassigned tickets:
    - Take unassigned supporting tickets from `supporting_queue`.
    - Assign to this senior until capacity is full.
    - Update `available_sp` grid.

### Edge Cases

| Scenario | Handling |
|---|---|
| All seniors full, heavy ticket remains | Heavy ticket overflows to junior (warning in preview) |
| All juniors full, supporting ticket remains | Supporting ticket spills to senior (Phase 3 handles this) |
| All-junior track | Skip Phase 1, all tickets go to Phase 2 directly |
| Engineer with remaining SP on start date | Available SP that day = capacity − remaining |
| Single engineer in track | All tickets to that engineer, spread across days |
| No seniors in track | All tickets treated as supporting, assigned to juniors |

## Preview Output

### Schedule Table

| Ticket | Track | SP | Type | Assignee | Level | Expected Start | Expected End | Notes |
|---|---|---:|---|---|---|---|---|---|
| BE-1 | BE | 12 | 🔴 Heavy | Andi | Senior | 2026-06-15 | 2026-06-16 | - |
| BE-2 | BE | 3 | 🟢 Support | Budi | Junior | 2026-06-15 | 2026-06-15 | - |
| BE-3 | BE | 8 | 🔴 Heavy | Andi | Senior | 2026-06-17 | 2026-06-17 | - |
| BE-4 | BE | 5 | 🟢 Support | Andi | Senior | 2026-06-18 | 2026-06-18 | ⚠️ Capacity fill |
| FE-1 | FE | 6 | 🟢 Support | Cici | Junior | 2026-06-15 | 2026-06-16 | - |

### Capacity Summary

```
📊 Capacity Utilization
BE Track:
  Andi (Senior, 6 SP/day): 25/30 SP used (83%) — Days: Jun 15-18
  Budi (Junior, 5 SP/day):  8/10 SP used (80%) — Days: Jun 15-16

FE Track:
  Cici (Junior, 5 SP/day):  10/10 SP used (100%) — Days: Jun 15-16
```

### Warnings

```
⚠️ Warnings:
- BE-4 assigned to Andi (Senior) as capacity fill — originally supporting ticket
- Budi under-capacity: 2 SP unused on Jun 16
- Andi remaining SP accounted: Jun 15 = 3SP (from Story CORB-70)
```

### Remaining SP Transparency

```
📌 Remaining SP from Other Stories:
  Andi: Jun 15 = 3SP remaining (available: 3/6)
  (No other remaining SP reported)
```

## Jira Update Behavior

### Update Scope

Only FE/BE Sub-task Engineer tickets are updated. The parent Story is not updated.

### Fields to Update

For each scheduled Sub-task Engineer:

| Field | Value | Source |
|---|---|---|
| `Expected Start Date[Date]` | Calculated start date | Scheduling algorithm |
| `Expected End Date[Date]` | Calculated end date | Scheduling algorithm |
| `assignee` | Engineer name/accountId | Phase 1-3 assignment |

### Assignee Resolution

The skill resolves engineer name to Jira accountId via:

1. Match name from roster input to Jira user search.
2. If ambiguous, present to lead via `question()` for confirmation.

### Field Resolution

Before updating, the skill resolves Jira field IDs for:

- `Expected Start Date[Date]`
- `Expected End Date[Date]`
- `Story Point[Number]` / `Story Point`
- `Story Point Dev[Number]` / `Story Point Dev`

If required date field IDs cannot be resolved, the skill must stop before any Jira update and show the preview plus a clear error.

### Approval Gate

Before writing to Jira, the skill must use `question()` to ask for explicit approval:

```
📝 Jira Update Plan (N tickets):
  BE-1: assignee=Andi, start=2026-06-15, end=2026-06-16
  BE-2: assignee=Budi, start=2026-06-15, end=2026-06-15
  BE-4: assignee=Andi, start=2026-06-18, end=2026-06-18 (capacity fill)

Options:
- Approve and update Jira
- Stop without updating
- Revise assignment/schedule
```

No Jira update is allowed without explicit approval.

### Partial Failure Handling

If some Jira updates succeed and others fail:

- Do not rollback successful updates automatically.
- Do not retry automatically without approval.
- Display a result table with success/failure per ticket.
- Show the failed ticket key, field, payload summary, and error message.

## Final Output

```
📋 Timeframe Summary
Story: [KEY] <summary>
Scope: Engineer only (FE + BE parallel)
Start date: 2026-06-15
Working days skipped: weekends + N custom holidays

Classification:
  Heavy tickets: X (→ Senior)
  Supporting tickets: Y (→ Junior, overflow → Senior)

Capacity:
  Seniors: A used / B total (Z%)
  Juniors: C used / D total (E%)

Remaining SP accounted: N entries

Updated tickets: N
Failed tickets: M
```

## Integration with `generate-test-case`

The skill does not always run `generate-test-case` automatically.

At runtime, it asks via `question()` whether to:

- Run `generate-test-case` precheck first, for FE/BE gap and API contract alignment.
- Skip precheck and calculate timeframe directly.

This preserves fast scheduling while allowing quality checks when needed.

## Non-goals

- No QA/UAT timeline calculation.
- No parent Story date update.
- No automatic Jira update without approval.
- No hard FE→BE dependency model.
- No YAML/JSON roster requirement.
- No permanent engineer capacity registry.
- No automatic holiday calendar integration beyond user-provided custom dates.
- No automatic remaining SP detection from Jira (manual input only).

## Common Mistakes

| Mistake | Fix |
|---|---|
| Bertanya lewat chat biasa | Selalu gunakan `question()` |
| Mengupdate parent Story | V1 hanya update Sub-task Engineer |
| Mengupdate Jira tanpa approval | Preview + approval via `question()` wajib |
| Memasukkan QA/UAT timeline | Exclude dari v1 |
| Menganggap FE menunggu BE | FE/BE full parallel by default |
| Memakai 8 SP/day default semua engineer | Senior 6, Junior 5 kecuali user override |
| Tidak handle missing SP | Fallback tanya user via `question()` |
| Tidak resolve Jira field ID | Resolve fields sebelum update |
| Retry gagal otomatis | Minta approval dulu |
| Assign heavy ticket ke junior tanpa overflow rule | Heavy selalu ke senior dulu (Phase 1) |
| Lupa account remaining SP | Kurangkan dari available_sp di day-grid |
| All-junior track tetap classify heavy/supporting | Skip classification, distribute all evenly |

## Design Reference

Source design draft:
`/Users/derikurniawan/Development/Github/.omo/drafts/2026-06-11-timeframe-from-story-design.md`

## Changelog

### v2 (2026-06-12)

- Updated Junior default from 4 to 5 SP/day
- Added weight-aware ticket assignment (heavy → senior, supporting → junior)
- Added auto-detect keyword classification for ticket weight
- Added remaining SP input model (manual lead input)
- Added 3-phase scheduling algorithm (Priority Queue with Capacity Fill)
- Added all-junior squad detection and handling
- Added assignee update to Jira fields
- Added capacity utilization summary to preview
- Added capacity fill phase for senior full utilization

### v1 (2026-06-11)

- Initial design with simple load-balancing
- Senior=6, Junior=4 defaults
- No ticket weight classification
- No remaining SP handling
- No assignee update