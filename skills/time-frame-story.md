---
name: timeframe-from-story
description: >
  Use when user wants to calculate engineering timeframe from a Jira Story ticket,
  allocate FE/BE Sub-task Engineer work across engineers by SP/day capacity with
  weight-aware assignment (heavy tickets → senior, supporting → junior), handle
  remaining SP from other stories, preview the schedule, and optionally update
  Jira Expected Start Date / Expected End Date / assignee fields.
  Triggers on "Buat timeframe dari story <KEY/URL>", "Generate timeline
  story <KEY/URL>", "Hitung jadwal pengerjaan story <KEY>", or "buat jadwal
  pengerjaan story".
---

# Timeframe From Story

## Overview

End-to-end workflow untuk menghitung timeframe pengerjaan **Sub-task Engineer FE/BE** dari Jira Story dengan **weight-aware assignment**.

Output utama:
- Klasifikasi tiket: Heavy (→ Senior) vs Supporting (→ Junior)
- Preview schedule per Sub-task Engineer dengan capacity utilization
- Optional update Jira fields setelah approval:
    - `Expected Start Date[Date]`
    - `Expected End Date[Date]`
    - `assignee`

Semua input user dan semua approval wajib menggunakan OpenCode `question()`.

## When to Use

Gunakan skill ini ketika user meminta:
- `Buat timeframe dari story CORB-71`
- `Generate timeline story https://lionparcel.atlassian.net/browse/CORB-71`
- `Hitung jadwal pengerjaan story <KEY>`
- `Buat jadwal pengerjaan tiket story`

**NOT for:**
- Timeline QA/UAT
- Estimasi tanpa tiket Jira Story
- Pengerjaan implementasi tiket
- Membuat test case; gunakan `generate-test-case` untuk itu

## Core Rules

- 1 SP = 1 jam pengerjaan.
- Jam kerja nominal = 8 jam/hari.
- Kapasitas delivery engineer menggunakan SP/day.
- Default kapasitas jika user hanya memberi level:
    - Senior = 6 SP/day
    - Junior = 5 SP/day
- **Weight-aware assignment:**
    - Tiket berat (high SP + keyword analisis) → Senior pertama
    - Tiket pendukung → Junior pertama
    - Senior yang masih under-capacity → dapat tiket pendukung (capacity fill)
- **All-junior squad:** Jika semua engineer di track adalah junior, skip klasifikasi, distribute semua evenly.
- FE dan BE berjalan **full parallel** by default.
- QA/UAT tidak masuk timeline.
- Weekend dilewati.
- Custom holidays/non-working dates dari user juga dilewati.
- Semua input user memakai `question()`.
- Tidak boleh update Jira sebelum preview + approval via `question()`.

## Step 0 — Normalize Story Key

Ekstrak Story key dari input user:

```python
import re

user_input = "..."
url_match = re.search(r'/browse/([A-Z]+-\d+)', user_input)
key_match = re.search(r'\b([A-Z]+-\d+)\b', user_input)
story_key = url_match.group(1) if url_match else (key_match.group(1) if key_match else None)
```

Jika tidak ada key, stop dan minta user provide Jira Story key/URL via `question()`.

## Step 1 — Fetch Story Detail

Gunakan Jira MCP jika tersedia:

```text
atlassian_jira_get_issue(story_key, fields="summary,status,assignee,parent,description,issuetype")
```

Tampilkan ringkasan:

```text
📌 Story : [KEY] <summary>
Status  : <status>
Assignee: <displayName / Unassigned>
```

## Step 2 — Fetch Child Issues

Cari semua child issue:

```jql
parent = STORY_KEY ORDER BY created ASC
```

Gunakan:

```text
atlassian_jira_search(jql, fields="summary,status,assignee,issuetype,description,parent")
```

Pisahkan:
- `Sub-task Engineer`
- `Sub-task QA` (ignored for timeline)
- others (ignored unless user maps manually)

Jika tidak ada Sub-task Engineer, stop dengan pesan jelas.

## Step 3 — Classify FE / BE

Klasifikasi berdasarkan prefix summary:

- `[FE]` → FE track
- `[BE]` → BE track

Jika ada Sub-task Engineer tanpa prefix `[FE]` atau `[BE]`, tampilkan via `question()`:
- Map ke FE
- Map ke BE
- Exclude dari schedule

## Step 4 — Optional `generate-test-case` Precheck

Tanya user via `question()`:

```text
Mau run generate-test-case precheck dulu untuk cek gap/API contract, atau langsung hitung timeline?
```

Options:
- Run precheck dulu
- Skip, langsung hitung timeline

Jika user memilih precheck, invoke/ikuti skill `generate-test-case` untuk story yang sama. Setelah selesai, kembali ke timeframe flow.

## Step 5 — Resolve Jira Fields

Resolve field IDs untuk:
- `Story Point[Number]` / `Story Point`
- `Story Point Dev[Number]` / `Story Point Dev`
- `Expected Start Date[Date]`
- `Expected End Date[Date]`

Gunakan Jira field search tools jika tersedia.

Jika date fields tidak ditemukan, tetap tampilkan preview schedule, tapi jangan update Jira.

## Step 6 — Resolve SP Per Ticket

Untuk setiap Sub-task Engineer:
1. Ambil SP dari `Story Point[Number]` / `Story Point`.
2. Jika kosong, ambil dari `Story Point Dev[Number]` / `Story Point Dev`.
3. Jika tetap kosong/unreadable, tanya user via `question()` untuk tiket tersebut.

Pertanyaan harus spesifik:

```text
[BE][Horde] Update client approval — berapa SP?
```

## Step 7 — Collect Schedule Inputs

Semua memakai `question()`.

### Start Date

Tanya tanggal mulai dalam format `YYYY-MM-DD`.

Jika jatuh weekend/custom holiday, tanya:
- Geser ke working day berikutnya
- Tetap pakai tanggal itu

### Custom Holidays / Non-working Dates

Format simple:

```text
2026-06-17, 2026-06-18
```

Boleh kosong.

### Engineer Roster

**Step 7a — Konfirmasi kapasitas default** via `question()`:

```text
Kapasitas default engineer:
  Senior = 6 SP/day
  Junior = 5 SP/day

Apakah mau pakai default ini, atau ada perubahan kapasitas?
```

Options:
- Pakai default (Senior 6, Junior 5)
- Ada perubahan kapasitas

Jika user memilih "Ada perubahan kapasitas", tanya via `question()`:

```text
Berapa kapasitas SP/day untuk masing-masing level?
Format: Senior <angka> SP/day, Junior <angka> SP/day
Contoh: Senior 8 SP/day, Junior 6 SP/day
```

**Step 7b — Input roster engineer** via `question()`:

Format simple list:

```text
BE: Andi senior, Budi junior
FE: Cici senior, Dedi junior
```

Atau dengan custom capacity per engineer:

```text
BE: Andi senior 8SP/day, Budi junior 6SP/day
FE: Cici senior, Dedi junior
```

Jika angka kapasitas kosong di roster, gunakan kapasitas default yang sudah dikonfirmasi di Step 7a.

**All-junior detection:** Jika semua engineer di satu track adalah junior, track tersebut otomatis masuk mode "distribute all evenly" — skip heavy/supporting classification.

Jika track punya tiket tapi roster kosong, stop dan minta roster untuk track tersebut via `question()`.

### Remaining SP from Other Stories

Tanya lead apakah ada engineer yang sudah punya sisa SP dari story lain:

```text
Apakah ada engineer yang sudah punya sisa SP dari story lain?
Format: <Nama> <tanggal> <sisa SP>
Contoh: Andi 2026-06-16 3SP, Budi 2026-06-17 2SP
(Kosongkan jika tidak ada)
```

Parse menjadi map:

```json
{
  "Andi": { "2026-06-16": 3 },
  "Budi": { "2026-06-17": 2 }
}
```

Saat scheduling: `available_sp[engineer][date] = capacity[engineer] - remaining_sp[engineer][date]`

## Step 8 — Classify Ticket Weight

### Auto-Detection

Ticket dianggap **HEAVY** jika:
- SP >= 5 (threshold default, bisa di-override lead) **DAN**
- Deskripsi mengandung keyword: `analisa`, `analysis`, `investigate`, `complex`, `migration`, `refactor`, `architecture`, `integration`, `debug`, `performance`, `optimization`

Ticket dianggap **SUPPORTING** jika tidak memenuhi kedua kriteria di atas.

Jika hanya salah satu kondisi terpenuhi (SP tinggi tapi no keyword, atau keyword tapi SP rendah), tampilkan ke lead via `question()` untuk klasifikasi manual.

### Classification Preview

Tampilkan hasil klasifikasi ke lead:

```text
📋 Ticket Classification
[BE][Horde] Update client approval (12 SP) → 🔴 HEAVY (SP≥5 + keyword: 'analysis')
[BE][Horde] Add logging middleware (3 SP) → 🟢 SUPPORTING
[BE][Horde] Refactor auth module (8 SP) → 🔴 HEAVY (SP≥5 + keyword: 'refactor')

Konfirmasi klasifikasi atau override?
```

Lead bisa override per tiket via `question()`.

### All-Junior Track

Jika semua engineer di track adalah junior:
- Skip klasifikasi heavy/supporting untuk track tersebut
- Semua tiket treated equally

## Step 9 — Schedule Calculation (Priority Queue with Capacity Fill)

### Pre-Processing

1. Build day-grid per engineer:
    - Untuk setiap working day mulai dari start date
    - `available_sp[engineer][date] = capacity[engineer] - remaining_sp[engineer][date]`

2. Pisah tiket per track ke:
    - `heavy_queue[]` — tiket HEAVY, sorted by Jira created order
    - `supporting_queue[]` — tiket SUPPORTING, sorted by Jira created order

### Phase 1: Heavy Tickets → Senior

Untuk setiap heavy ticket di `heavy_queue`:
1. Cari **senior** di track yang sama dengan earliest available slot.
2. Tie-breaker: senior dengan total assigned SP paling kecil.
3. Allocate SP across working days:
    - Day 1: `min(ticket_remaining_sp, available_sp[senior][day1])`
    - Day 2: `min(sisa, available_sp[senior][day2])`
    - Lanjut sampai ticket fully allocated.
4. Update `available_sp` grid.
5. Record: ticket → assignee, start_date, end_date.

### Phase 2: Supporting Tickets → Junior

Untuk setiap supporting ticket di `supporting_queue`:
1. Cari **junior** di track yang sama dengan earliest available slot.
2. Tie-breaker: junior dengan total assigned SP paling kecil.
3. Allocate SP across working days (same logic as Phase 1).
4. Update `available_sp` grid.
5. Record: ticket → assignee, start_date, end_date.

### Phase 3: Capacity Fill (Senior ← Supporting Overflow)

Untuk setiap senior di track:
1. Hitung `total_assigned` = sum semua SP yang sudah di-assign ke senior ini.
2. Hitung `total_capacity` = sum `available_sp` across semua scheduled days.
3. Jika `total_assigned < total_capacity` DAN ada supporting tickets yang belum ter-assign:
    - Ambil supporting tickets yang belum ter-assign.
    - Assign ke senior ini sampai capacity penuh.
    - Update `available_sp` grid.

### Edge Cases

| Scenario | Handling |
|---|---|
| Senior habis tapi masih ada heavy ticket | Heavy ticket overflow ke junior (warning di preview) |
| Junior habis tapi masih ada supporting ticket | Supporting ticket spill ke senior (Phase 3 handle ini) |
| All-junior track | Skip Phase 1, semua tiket masuk Phase 2 langsung |
| Engineer dengan remaining SP di start date | Available SP di hari itu = capacity − remaining |
| Single engineer di track | Semua tiket ke engineer itu, spread across days |
| No seniors in track | Semua tiket treated as supporting, assigned ke juniors |

### Date Granularity

Hitung internal dengan jam/SP, tapi output Jira date-only.

Contoh:

```text
Capacity: 6 SP/day
Ticket  : 9 SP
Start   : Monday

Monday  : 6 SP
Tuesday : 3 SP

Expected Start Date = Monday
Expected End Date   = Tuesday
```

## Step 10 — Preview Schedule

Tampilkan preview sebelum update Jira:

### Schedule Table

| Ticket | Track | SP | Type | Assignee | Level | Expected Start | Expected End | Notes |
|---|---|---:|---|---|---|---|---|---|
| BE-1 | BE | 12 | 🔴 Heavy | Andi | Senior | 2026-06-15 | 2026-06-16 | - |
| BE-2 | BE | 3 | 🟢 Support | Budi | Junior | 2026-06-15 | 2026-06-15 | - |
| BE-3 | BE | 8 | 🔴 Heavy | Andi | Senior | 2026-06-17 | 2026-06-17 | - |
| BE-4 | BE | 5 | 🟢 Support | Andi | Senior | 2026-06-18 | 2026-06-18 | ⚠️ Capacity fill |
| FE-1 | FE | 6 | 🟢 Support | Cici | Junior | 2026-06-15 | 2026-06-16 | - |

### Capacity Summary

```text
📊 Capacity Utilization
BE Track:
  Andi (Senior, 6 SP/day): 25/30 SP used (83%) — Days: Jun 15-18
  Budi (Junior, 5 SP/day):  8/10 SP used (80%) — Days: Jun 15-16

FE Track:
  Cici (Junior, 5 SP/day):  10/10 SP used (100%) — Days: Jun 15-16
```

### Warnings

Preview harus menyertakan warning untuk:
- SP yang diinput manual
- Tiket excluded / unmapped
- Weekend/holiday skipped
- Missing Jira fields
- Engineer overload jika ada
- Tiket yang di-assign sebagai capacity fill (senior dapat supporting ticket)
- Under-capacity engineer

### Remaining SP Transparency

```text
📌 Remaining SP from Other Stories:
  Andi: Jun 15 = 3SP remaining (available: 3/6)
  (No other remaining SP reported)
```

## Step 11 — Approval Before Jira Update

Wajib `question()`:

```text
📝 Jira Update Plan (N tickets):
  BE-1: assignee=Andi, start=2026-06-15, end=2026-06-16
  BE-2: assignee=Budi, start=2026-06-15, end=2026-06-15
  BE-4: assignee=Andi, start=2026-06-18, end=2026-06-18 (capacity fill)
```

Options:
- Approve and update Jira
- Stop without updating Jira
- Revise assignment/schedule

Jika user tidak approve, stop tanpa update.

## Step 12 — Update Jira Fields

Update hanya FE/BE Sub-task Engineer.

Tidak update parent Story.

Untuk setiap scheduled ticket, update 3 fields:

```json
{
  "customfield_expected_start": "YYYY-MM-DD",
  "customfield_expected_end": "YYYY-MM-DD",
  "assignee": "accountId atau username engineer"
}
```

### Assignee Resolution

Resolve engineer name ke Jira accountId:
1. Match nama dari roster input ke Jira user search.
2. Jika ambiguous, tampilkan ke lead via `question()` untuk konfirmasi.

Gunakan Jira update tool setelah mapping field ID berhasil.

## Step 13 — Partial Failure Handling

Jika sebagian update gagal:
- Jangan rollback otomatis.
- Jangan retry otomatis tanpa approval.
- Tampilkan tabel hasil:

| Ticket | Status | Assignee | Start | End | Error |
|---|---|---|---|---|---|
| BE-1 | ✅ Updated | Andi | 2026-06-15 | 2026-06-16 | - |
| BE-2 | ❌ Failed | Budi | 2026-06-16 | 2026-06-17 | permission denied |

Jika semua sukses:

```text
✅ Jira date fields and assignees updated.
```

## Output Summary

Akhiri dengan:

```text
📋 Timeframe Summary
Story: [KEY] <summary>
Scope: Engineer only
Tracks: FE and BE parallel
Start date: YYYY-MM-DD
Working days skipped: weekends + <N custom holidays>

Classification:
  Heavy tickets: X (→ Senior)
  Supporting tickets: Y (→ Junior, overflow → Senior)

Capacity:
  Seniors: A used / B total (Z%)
  Juniors: C used / D total (E%)

Remaining SP accounted: N entries

Updated tickets: N
Failed tickets : M
```

## Common Mistakes

| Mistake | Fix |
|---|---|
| Bertanya lewat chat biasa | Selalu gunakan `question()` |
| Mengupdate parent Story | Hanya update Sub-task Engineer |
| Mengupdate Jira tanpa approval | Preview + approval via `question()` wajib |
| Memasukkan QA/UAT timeline | Exclude dari timeline |
| Menganggap FE menunggu BE | FE/BE full parallel by default |
| Memakai 8 SP/day default semua engineer | Senior 6, Junior 5 kecuali user override |
| Tidak handle missing SP | Fallback tanya user via `question()` |
| Tidak resolve Jira field ID | Resolve fields sebelum update |
| Retry gagal otomatis | Minta approval dulu |
| Assign heavy ticket ke junior tanpa overflow | Heavy selalu ke senior dulu (Phase 1) |
| Lupa account remaining SP | Kurangkan dari available_sp di day-grid |
| All-junior track tetap classify heavy/supporting | Skip classification, distribute all evenly |
| Tidak update assignee | Update assignee bersamaan dengan date fields |

## Design Reference

Source design draft:
`/Users/sholahuddinalisyahbana/go/src/github.com/Banalp/Notes/skills/time-frame-story-draft.md`