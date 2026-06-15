---
name: jira-bug-triage-analysis
description: Use when given a Jira QA bug ticket to identify root cause
  in codebase, find responsible engineer sub-tasks filtered by Jira
  assignee accountIds, map code ownership via git history, and create
  new engineer sub-tasks with proper assignee and issue link relations.
  Triggers on "analisis tiket bug", "cari subtask engineer", "siapa yang
  handle bug ini", "buat subtask engineer", or "create tiket engineer".
---

# Jira Bug Triage Analysis

## Overview

5-step process: QA bug ticket → root cause in code → git owner → Jira
sub-task. Combines Jira API graph traversal, codebase grep/trace, and
git log to produce a structured engineer assignment recommendation.

## When to Use

- Diberikan URL atau key Jira QA bug ticket untuk dianalisis
- Perlu menemukan sub-task engineer yang bertanggung jawab atas bug
- Perlu filter assignment berdasarkan Jira accountId tertentu
- Perlu mapping code ownership ke Jira work item

**NOT for:** debugging tanpa konteks Jira.

## Configuration

Jira credentials untuk project ini:

```bash
JIRA_USER="sholahuddin.alisyahbana@thelionparcel.com"
JIRA_TOKEN="ATATT3xFfGF0HQvsJ6VtAdyGiSb7d5-fqQoUfLvBq9imCmzh1Kk90QO0kiqNf-UvGfMnTMvn87MrdnofPoeVVV18nrQBrWSL7q3bbFH1N2vYjq8pyutSeun6m7_RNg7gXH3QiYnJ-3-uH2BhrvxF1SZL_wnqqZtdt1M7nH8p2Cjb6Ajf_Xj3h64=2A2A12CF"
JIRA_DOMAIN="lionparcel.atlassian.net"
```

Repos lokal:
- Hydra: `/Users/sholahuddinalisyahbana/go/src/github.com/Lionparcel/hydra`
- Gober: `/Users/sholahuddinalisyahbana/go/src/github.com/Lionparcel/gober`
- Horde: `/Users/sholahuddinalisyahbana/go/src/github.com/Lionparcel/horde`
- Pegasus: `/Users/sholahuddinalisyahbana/go/src/github.com/Lionparcel/pegasus`

## Core Workflow

### Step 1 — Fetch Bug Ticket

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$KEY?fields=summary,description,status,assignee,issuelinks,comment,parent"
```

Parse dengan Python helper (ADF → plain text):

```python
def extract_text(node):
    if not node: return ''
    if node.get('type') == 'text': return node.get('text', '')
    result = ''.join(extract_text(c) for c in node.get('content', []))
    if node.get('type') in ('paragraph', 'listItem', 'heading'): result += '\n'
    return result
```

Extract: actual result, expected result, steps to reproduce, parent key,
semua issuelinks.

---

### Step 2 — Traverse Issue Graph

1. Ambil **parent** epic/story → list semua child issues
2. Fetch tiap child: summary, status, assignee (accountId)
3. Follow `issuelinks` (Relates, Clones, Blocks) → cari engineer tickets

```bash
# Fetch semua child issues dari parent epic/story
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$PARENT_KEY?fields=subtasks,issuelinks"
```

**FALLBACK jika JQL `parent=` return 0** — ini common Jira quirk, fetch langsung per key:

```bash
for key in KEY-123 KEY-124 KEY-125; do
  curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
    "https://$JIRA_DOMAIN/rest/api/3/issue/$key?fields=summary,status,assignee"
done
```

Focus pada subtask dengan prefix: `[BE]`, `[FE]`, `[Handle]`, `CLONE -`.

---

### Step 3 — Filter by Assignee AccountId

Match assignees dari subtasks dengan accountId list yang diberikan user:

```python
filter_ids = {'id1', 'id2', 'id3', ...}   # dari parameter user

for iss in subtasks:
    aid = iss['fields'].get('assignee', {}).get('accountId', '')
    if aid in filter_ids:
        print(f"MATCH: {iss['key']} — {iss['fields']['summary']} — {assignee_name}")
```

JQL alternatif (kadang tidak bekerja untuk sub-task):
```
assignee in (id1,id2) AND parent=KEY-XXX
```

---

### Step 4 — Root Cause Investigation

**Prinsip penting:** Bug sering ada di **write path** (saat data dibuat/disimpan),
bukan di read/display path. Selalu trace ke source.

```bash
# 1. Cari field yang disebutkan di bug description
grep -rn "field_name\|FieldName" src/usecase/ --include="*.go" | grep -v "test\|mock"

# 2. Baca fungsi yang relevan
sed -n 'LINE_START,LINE_ENDp' src/usecase/relevant_file.go

# 3. Jika field berasal dari DB, cari di write path
grep -rn "field_name" src/usecase/ --include="*.go" \
  | grep -i "create\|generate\|insert\|consumer\|scheduler"
```

Untuk setiap bug, dokumentasikan:
- `file.go:line` yang mengandung logic salah
- Nilai aktual (kode saat ini) vs nilai expected
- Rumus/logic yang seharusnya dipakai

---

### Step 5 — Git Ownership Mapping

```bash
cd /path/to/repo
git log --follow -5 --pretty=format:"%h %an %ad %s" src/usecase/file.go
```

Cross-reference `%an` (git author name) dengan Jira assignee `displayName`.
Jika berbeda, gunakan **git log sebagai source of truth** untuk ownership kode.

---

## Output Format

### Bagian 1: Analisis Bug

Satu blok per bug:
```
**Bug N — <nama field>**
Root cause: `path/file.go:line`
Aktual:   <kode/logic saat ini>
Expected: <kode/logic yang benar>
Contoh:   input X → output Y (salah), seharusnya Z
```

### Bagian 2: Engineer Sub-Task Table

| # | Tiket | Judul | Assignee | Match Filter | Relasi ke Bug |
|---|-------|-------|----------|-------------|---------------|
| 1 | KEY-XXX | [BE][svc] Judul task | Nama (accountId) | ✅ | Owner `file.go` (Bug 1 & 2) |
| 2 | KEY-YYY | [BE][svc] Judul task | Nama (accountId) | ✅ | Display `field` di list/detail API |

### Bagian 3: Rekomendasi Fix Assignment

| Bug | Engineer | File | Deskripsi Fix |
|-----|----------|------|---------------|
| Bug 1 — field X salah | Nama | `svc/file.go:line` | Ganti logic A → B |
| Bug 2 — field Y salah | Nama | `svc/file.go:line` | Gunakan sumber data yang tepat |

---

## Common Mistakes

| Kesalahan | Fix |
|-----------|-----|
| Hanya lihat subtask dari bug ticket itu sendiri | Traverse ke **parent epic/story** — engineer subtask ada di sana |
| JQL `parent=` return 0 | Common Jira quirk — fetch tiap subtask langsung by key |
| Asumsi bug ada di read/API path | Trace ke write path: scheduler, consumer, create usecase |
| Asumsi assignee subtask = penulis kode | Selalu `git log` untuk konfirmasi actual code author |
| Berhenti di API/usecase level untuk read | Kalau field berasal dari DB, cek bagaimana data itu di-insert |
| Lupa cek bug sekunder | Bug sering berpasangan di file yang sama — scan semua field terkait |
| Asumsi 1 bug = 1 subtask | 1 engineer bisa memiliki N subtask yang masing-masing berkontribusi pada bug yang sama |

---

## Step 6 — Create Engineer Sub-Tasks & Issue Relations

Setelah Step 5 selesai dan engineer + fix sudah diidentifikasi, buat tiket subtask baru dan relasikan ke tiket-tiket terkait.

### 6a. Fetch Project & Issue Type Metadata

```bash
# Cari issue type ID untuk Sub-task di project
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/project/$PROJECT_KEY" \
  | python3 -c "import json,sys; d=json.load(sys.stdin); [print(it['id'], it['name']) for it in d.get('issueTypes',[])]"
```

Catat `id` untuk issue type `Sub-task` (biasanya berbeda per project).

### 6b. Konfirmasi Sebelum Membuat Sub-Task (WAJIB)

**STOP — Jangan buat tiket apapun sebelum mendapat persetujuan eksplisit dari user.**

Sebelum memanggil `question` tool, tampilkan terlebih dahulu ringkasan subtask yang akan dibuat dalam format teks:

```
📋 Subtask yang akan dibuat untuk [KEY-BUG]:
  1. [BE][<svc>] <judul task>
     Assignee : <Nama Engineer> (<accountId>)
     Parent   : <KEY-PARENT>
     Fix      : <deskripsi singkat fix>

  2. [BE][<svc>] <judul task>
     Assignee : <Nama Engineer> (<accountId>)
     Parent   : <KEY-PARENT>
     Fix      : <deskripsi singkat fix>

Relasi: subtask baru → blocks <KEY-BUG>; relates to existing subtasks
```

Kemudian gunakan `question` tool dalam **satu panggilan** dengan 3 pertanyaan sekaligus:

**Pertanyaan 1 — Konfirmasi Assignee (per subtask)**

Untuk setiap subtask, tanyakan assignee. Opsi yang disediakan:
- Assignee hasil analisa (rekomendasi dari git log + issue graph) sebagai opsi pertama dengan label `<Nama Engineer> (Rekomendasi)`
- Opsi manual jika user ingin engineer lain (gunakan `custom: true` agar user bisa ketik accountId)

**Pertanyaan 2 — Story Point**

Estimasi story point berdasarkan kompleksitas fix yang ditemukan di Step 4:
- **1 SP**: perubahan 1 kondisi/baris, tidak ada side effect, tidak ada migration
- **2 SP**: perubahan 1–2 fungsi, logic sederhana, mungkin ada unit test
- **3 SP**: perubahan multi-fungsi atau multi-service (hydra + gober), perlu tracing alur
- **5 SP**: perubahan signifikan di core flow, ada risk regression, perlu integration test
- **8 SP**: refactor besar, multiple service, perlu design ulang alur data

Sertakan alasan estimasi di field `description` tiap opsi.

**Pertanyaan 3 — Assign Bug Ticket**

Tanyakan apakah tiket bug utama perlu di-assign juga ke engineer yang menangani subtask.
- "Ya, assign bug tiket ke engineer tersebut"
- "Tidak (Rekomendasi)"

**Contoh pemanggilan `question` tool:**

```
question([
  {
    header: "Assignee subtask 1",
    question: "Siapa assignee untuk subtask '[BE][hydra] Fix: ...'?",
    options: [
      {
        label: "<Nama Engineer> (Rekomendasi)",
        description: "Author commit [GENESIS-XXXXX] di file terkait (git log: arifsefrianto)"
      }
    ]
    // custom: true otomatis ditambahkan — user bisa ketik accountId manual
  },
  // ulangi per subtask jika ada lebih dari 1
  {
    header: "Story Point",
    question: "Berapa story point untuk subtask ini? (estimasi dari analisa root cause)",
    options: [
      { label: "1 SP", description: "Perubahan 1 kondisi/baris, tidak ada side effect" },
      { label: "2 SP", description: "Perubahan 1-2 fungsi, logic sederhana" },
      { label: "3 SP (Rekomendasi)", description: "Perubahan multi-fungsi/multi-service — sesuai root cause di hydra + gober" },
      { label: "5 SP", description: "Perubahan core flow, risk regression tinggi" },
      { label: "8 SP", description: "Refactor besar, multiple service, perlu design ulang" }
    ]
  },
  {
    header: "Assign Bug Tiket",
    question: "Apakah tiket bug utama ([KEY-BUG]) juga ingin di-assign ke engineer yang menangani subtask ini?",
    options: [
      { label: "Ya, assign bug tiket ke engineer tersebut", description: "Bug tiket akan di-assign secara otomatis ke accountId engineer yang dipilih" },
      { label: "Tidak (Rekomendasi)", description: "Biarkan assignee bug tiket tidak berubah" }
    ]
  }
])
```

**Aturan estimasi story point rekomendasi:**
- Hitung jumlah file yang perlu diubah dari Step 4
- Hitung jumlah service yang terlibat (hydra, gober, horde = +1 SP per tambahan service)
- Jika ada risk regression pada core flow (booking, payment, commission) → tambah 1 SP
- Gunakan nilai tersebut sebagai opsi `(Rekomendasi)` di label

**Lanjutkan ke 6c hanya jika user sudah menjawab semua pertanyaan.**
Gunakan jawaban user sebagai nilai final untuk `assignee.accountId` dan `story_points` saat create tiket.
Jika user memilih opsi custom untuk assignee → gunakan accountId yang diketik user.
Jika user menjawab "Ya" untuk Assign Bug Tiket, catat keputusan ini untuk dijalankan setelah pembuatan subtask.
Jika user tidak menjawab / menutup dialog → STOP, jangan buat tiket apapun.

### 6c. Create Sub-Task untuk Setiap Engineer

Buat satu sub-task per engineer menggunakan nilai assignee dan story point dari jawaban user di Step 6b:

```bash
# $ENGINEER_ACCOUNT_ID = accountId dari jawaban user (rekomendasi atau manual)
# $STORY_POINTS = angka dari pilihan user (1/2/3/5/8)

curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST \
  -H "Content-Type: application/json" \
  "https://$JIRA_DOMAIN/rest/api/3/issue" \
  -d '{
    "fields": {
      "project": { "key": "'"$PROJECT_KEY"'" },
      "parent": { "key": "'"$PARENT_KEY"'" },
      "summary": "[BE]['"$SERVICE"'] '"$JUDUL_TASK"'",
      "issuetype": { "id": "'"$SUBTASK_TYPE_ID"'" },
      "assignee": { "accountId": "'"$ENGINEER_ACCOUNT_ID"'" },
      "story_points": '"$STORY_POINTS"',
      "description": {
        "type": "doc",
        "version": 1,
        "content": [{
          "type": "paragraph",
          "content": [{ "type": "text", "text": "'"$DESKRIPSI_FIX"'" }]
        }]
      }
    }
  }'
```

**Catatan story_points field name:** Jira Cloud sering menggunakan custom field ID untuk story points (biasanya `customfield_10016` atau `customfield_10028`). Jika `story_points` gagal, cek dengan:

```bash
# Fetch field metadata untuk tahu custom field ID story points
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/field" \
  | python3 -c "
import json, sys
fields = json.load(sys.stdin)
for f in fields:
    if 'story' in f.get('name','').lower() or 'point' in f.get('name','').lower():
        print(f['id'], f['name'])
"
```

Gunakan field ID yang ditemukan (misal `customfield_10016`) sebagai pengganti `story_points` di payload.

**Naming convention subtask:**
- `[BE][<service>] Fix: <deskripsi singkat bug yang difix>`
- Contoh: `[BE][horde] Fix: kalkulasi berat salah pada proses reroute`

**Parent key:** gunakan parent epic/story dari bug ticket (sama dengan yang di-traverse di Step 2).

### 6d. Buat Relasi "is related to" — Antar Subtask Engineer

Setelah semua subtask baru dibuat, relasikan satu sama lain dengan `is related to`:

```bash
# Fetch dulu link type ID untuk "Relates"
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/issueLinkType" \
  | python3 -c "import json,sys; [print(lt['id'], lt['name'], lt.get('inward'), lt.get('outward')) for lt in json.load(sys.stdin)['issueLinkTypes']]"
```

Cari entry dengan `name = "Relates"` → catat `id`-nya (biasanya `"10003"`).

```bash
# Buat relasi "is related to" dari NEW_SUBTASK ke setiap subtask lain yang terkait
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST \
  -H "Content-Type: application/json" \
  "https://$JIRA_DOMAIN/rest/api/3/issueLink" \
  -d '{
    "type": { "id": "'"$RELATES_LINK_TYPE_ID"'" },
    "inwardIssue": { "key": "'"$NEW_SUBTASK_KEY"'" },
    "outwardIssue": { "key": "'"$RELATED_SUBTASK_KEY"'" }
  }'
```

Ulangi untuk setiap pasangan subtask (baru ↔ existing engineer subtask lain).

### 6e. Buat Relasi "is blocked by" — Bug Ticket ke Subtask Baru

Bug ticket harus direlasikan ke setiap subtask baru dengan `is blocked by`:

```bash
# Fetch link type untuk "Blocks" / "is blocked by"
# Cari entry dengan outward = "is blocked by" atau name = "Blocks"
# inward = "blocks", outward = "is blocked by"

curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST \
  -H "Content-Type: application/json" \
  "https://$JIRA_DOMAIN/rest/api/3/issueLink" \
  -d '{
    "type": { "id": "'"$BLOCKS_LINK_TYPE_ID"'" },
    "inwardIssue": { "key": "'"$NEW_SUBTASK_KEY"'" },
    "outwardIssue": { "key": "'"$BUG_TICKET_KEY"'" }
  }'
```

Semantik yang dihasilkan: **Bug `$BUG_TICKET_KEY` is blocked by subtask `$NEW_SUBTASK_KEY`**.

Ulangi untuk setiap subtask baru yang dibuat.

### 6f. Assign Bug Ticket ke Engineer (Jika Dipilih)

Jika pada Step 6b user memilih untuk meng-assign tiket bug utama ke engineer, jalankan API berikut ke tiket bug:

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  -X PUT \
  -H "Content-Type: application/json" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$BUG_TICKET_KEY/assignee" \
  -d '{
    "accountId": "'"$ENGINEER_ACCOUNT_ID"'"
  }'
```

### 6g. Verifikasi Hasil

```bash
# Verifikasi subtask baru dan semua relasi-nya
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$NEW_SUBTASK_KEY?fields=summary,assignee,parent,issuelinks,status" \
  | python3 -c "
import json, sys
d = json.load(sys.stdin)['fields']
print('Summary:', d['summary'])
print('Assignee:', d.get('assignee', {}).get('displayName', 'unassigned'))
print('Parent:', d.get('parent', {}).get('key', '-'))
print('Links:')
for l in d.get('issuelinks', []):
    lt = l['type']['name']
    target = l.get('inwardIssue', l.get('outwardIssue', {}))
    print(f'  {lt}: {target.get(\"key\")} — {target.get(\"fields\",{}).get(\"summary\",\"\")}')
"
```

---

## Output Tambahan (setelah Step 6)

### Bagian 4: Tiket Baru yang Dibuat

| # | Tiket Baru | Judul | Assignee | Parent | Relasi |
|---|-----------|-------|----------|--------|--------|
| 1 | KEY-NEW1 | [BE][svc] Fix: ... | Nama Engineer | KEY-EPIC | related to KEY-XXX, KEY-YYY; bug KEY-BUG is blocked by this |
| 2 | KEY-NEW2 | [BE][svc] Fix: ... | Nama Engineer | KEY-EPIC | related to KEY-NEW1; bug KEY-BUG is blocked by this |
