---
name: generate-test-case
description: Use when user wants to generate test cases from a Jira Story ticket.
  Fetches Epic context (PRD - mandatory), scans all Sub-task QA to build QA context
  with a checkpoint, then asks user which Sub-task Engineer to analyze (single,
  multiple, or all). Synthesizes test cases that validate engineer coverage against
  QA expectations and product PRD. Posts result as adaptive comment on the engineer
  ticket: test cases only if no gap, or two separate comments (test cases + gap
  analysis) if gaps are detected.
  Triggers on "Buatkan test case skenario dari tiket story: <KEY>",
  "generate test case", or "buat test case dari story".
---

# Generate Test Case

## Overview

8-step interactive workflow: Story → Epic (Wajib) → Sub-tasks → QA Context Checkpoint → API Contract Check → Pilih Engineer → Sintesis test case → Post sebagai comment.

**Approach:** AI Synthesis — Epic PRD + Sub-task QA mendefinisikan ekspektasi sebagai *primary benchmark*; test case memvalidasi apakah Sub-task Engineer meng-cover semua ekspektasi tersebut. Gap antara scope engineer dan ekspektasi QA + Product dicatat otomatis, dan dipost sebagai comment terpisah jika ditemukan.

## When to Use

- User ketik `"Buatkan test case skenario dari tiket story: <STORY-KEY>"`
- User ingin generate test case dari tiket Sub-task Engineer berdasarkan kerangka Sub-task QA dan PRD Epic

**NOT for:** membuat test plan dari scratch tanpa konteks Jira.

## Configuration

```bash
JIRA_USER="sholahuddin.alisyahbana@thelionparcel.com"
JIRA_TOKEN="ATATT3xFfGF0HQvsJ6VtAdyGiSb7d5-fqQoUfLvBq9imCmzh1Kk90QO0kiqNf-UvGfMnTMvn87MrdnofPoeVVV18nrQBrWSL7q3bbFH1N2vYjq8pyutSeun6m7_RNg7gXH3QiYnJ-3-uH2BhrvxF1SZL_wnqqZtdt1M7nH8p2Cjb6Ajf_Xj3h64=2A2A12CF"
JIRA_DOMAIN="lionparcel.atlassian.net"
```

---

## Input Normalization

Sebelum Step 1, ekstrak story key dari input user:

```python
import re

user_input = "..."  # input dari user

# Match URL: https://lionparcel.atlassian.net/browse/LION-1234
url_match = re.search(r'/browse/([A-Z]+-\d+)', user_input)
# Match key langsung: LION-1234
key_match = re.search(r'\b([A-Z]+-\d+)\b', user_input)

story_key = url_match.group(1) if url_match else (key_match.group(1) if key_match else None)
if not story_key:
    print("ERROR: Tidak dapat menemukan story key dari input")
    exit(1)
```

---

## Step 1 — Fetch Story Detail

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$STORY_KEY?fields=summary,description,status,assignee,parent"
```

Tampilkan ke user:
```
📌 Story: [KEY] <summary>
Status : <status>
Assignee: <displayName>
```

Simpan `parent_key` jika ada:
```python
parent_key = story_data['fields'].get('parent', {}).get('key')  # None jika tidak ada parent
```

---

## Step 1.5 — Fetch Epic Context (WAJIB)

Epic context adalah sumber kebenaran product intent. PRD di Epic digunakan sebagai *primary benchmark* untuk memvalidasi coverage engineer — **bukan opsional**.

### Alur:

```
parent_key ada?
  ├─ Ya  → fetch epic, parse deskripsinya → gunakan sebagai `epic_context`
  └─ Tidak → WAJIB tanya user via question()
                ├─ User provide epic key → fetch epic → gunakan sebagai `epic_context`
                └─ User pilih degraded mode → set epic_context = None,
                                               tampilkan WARNING, lanjut tanpa PRD
```

### Fetch Epic (jika `parent_key` ada):

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$PARENT_KEY?fields=summary,description,issuetype"
```

```python
parent_issuetype = parent_data['fields']['issuetype']['name']  # 'Epic', 'Story', dll
parent_summary   = parent_data['fields']['summary']
parent_desc_raw  = parent_data['fields'].get('description')
parent_desc_text = extract_text(parent_desc_raw) if parent_desc_raw else ''

if parent_issuetype == 'Epic':
    epic_context = f"Epic: [{parent_key}] {parent_summary}\n\n{parent_desc_text}"
    print(f"🗂️  Epic ditemukan: [{parent_key}] {parent_summary}")
else:
    # Parent bukan Epic (mis. Feature/Initiative) — tetap pakai sebagai konteks
    epic_context = f"Parent [{parent_key}] ({parent_issuetype}): {parent_summary}\n\n{parent_desc_text}"
    print(f"🗂️  Parent ticket: [{parent_key}] ({parent_issuetype}) {parent_summary}")
```

### Jika Tidak Ada Parent (WAJIB tanya user):

```python
answer = question([{
    "header": "Epic Key (Wajib)",
    "question": (
        f"Story {story_key} tidak memiliki parent Epic yang terdeteksi. "
        "Epic diperlukan untuk membaca PRD dan memvalidasi coverage engineer terhadap ekspektasi product. "
        "Masukkan nomor tiket Epic-nya, atau pilih degraded mode:"
    ),
    "options": [
        {
            "label": "Lanjut tanpa Epic (degraded mode)",
            "description": "⚠️ Test case dibuat hanya dari Sub-task Engineer + QA. Validasi coverage terhadap PRD tidak tersedia."
        }
    ]
}])

# Jika user mengetik epic key secara manual:
user_input_val = answer[0] if answer else ""
epic_key_input = re.search(r'\b([A-Z]+-\d+)\b', user_input_val)
if epic_key_input:
    epic_key = epic_key_input.group(1)
    # [fetch + parse sama seperti di atas, set epic_context]
else:
    # User memilih degraded mode
    epic_context = None
    print("⚠️  Melanjutkan tanpa konteks Epic. Validasi PRD tidak tersedia.")
```

### Gunakan `epic_context` di Step 5 sebagai primary benchmark:

```
epic_context → PRD product intent → validasi apakah engineer meng-cover ekspektasi product
```

---

## Step 2 — Fetch Semua Sub-tasks

**⚠️ Gunakan POST `/rest/api/3/search/jql` — GET /search sudah deprecated di Jira Cloud.**

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  -X POST -H "Content-Type: application/json" \
  "https://$JIRA_DOMAIN/rest/api/3/search/jql" \
  -d '{
    "jql": "parent = '"$STORY_KEY"' ORDER BY created ASC",
    "fields": ["summary","status","issuetype","assignee","description","priority"],
    "maxResults": 100
  }'
```

Pisahkan hasil berdasarkan issuetype:

```python
def extract_text(node):
    """Parse ADF node → plain text."""
    if not node: return ''
    if node.get('type') == 'text': return node.get('text', '')
    result = ''.join(extract_text(c) for c in node.get('content', []))
    if node.get('type') in ('paragraph', 'listItem', 'heading', 'tableCell'):
        result += '\n'
    return result

qa_tasks       = [i for i in issues if i['fields']['issuetype']['name'] == 'Sub-task QA']
engineer_tasks = [i for i in issues if i['fields']['issuetype']['name'] == 'Sub-task Engineer']
```

Tampilkan ringkasan:
```
📋 Sub-tasks di story [KEY]:
  Sub-task QA       : N tiket
  Sub-task Engineer : M tiket
```

Jika `engineer_tasks` kosong → tampilkan error dan stop:
```
❌ Tidak ada Sub-task Engineer di story ini.
```

---

## Step 2.5 — Scan QA Context & Checkpoint

Setelah semua Sub-task QA di-fetch, scan deskripsinya untuk membangun **QA context** sebelum analisa engineer dimulai. QA context adalah salah satu dari dua *primary benchmark* (bersama Epic PRD).

### Ekstraksi QA Context

Untuk setiap Sub-task QA, parse deskripsi ADF → plain text, lalu ekstrak:

```python
def extract_qa_context(qa_tasks: list) -> dict:
    """
    Scan semua Sub-task QA dan ekstrak context per tiket.
    Returns: {qa_key: {summary, scenarios, impact, expected_results, raw_text}}
    """
    qa_context = {}
    for task in qa_tasks:
        key     = task['key']
        summary = task['fields']['summary']
        raw     = task['fields'].get('description')
        text    = extract_text(raw) if raw else ''

        lines = [l.strip() for l in text.split('\n') if l.strip()]

        # Ekstrak skenario
        scenarios = [l for l in lines if any(
            kw in l.lower() for kw in ['skenario', 'scenario', 'test case', 'testcase', 'tc', 'case']
        )]
        # Ekstrak impact
        impact_lines = [l for l in lines if 'impact' in l.lower() or 'dampak' in l.lower()]
        # Ekstrak expected result
        expected_lines = [l for l in lines if any(
            kw in l.lower() for kw in ['expected', 'ekspektasi', 'hasil yang diharapkan', 'should', 'harus']
        )]

        qa_context[key] = {
            'summary'         : summary,
            'scenarios'       : scenarios or lines[:5],  # fallback: 5 baris pertama
            'impact'          : impact_lines,
            'expected_results': expected_lines,
            'raw_text'        : text
        }

    return qa_context

qa_context = extract_qa_context(qa_tasks)
```

### Tampilkan QA Context Checkpoint ke User

```
📋 QA Context berhasil di-extract dari {len(qa_tasks)} Sub-task QA:

├── [QA-KEY1] <summary>
│     Skenario ditemukan : N
│     Impact             : Ya / Tidak
│     Expected result    : M item
│
├── [QA-KEY2] <summary>
│     Skenario ditemukan : P
│     Impact             : Ya / Tidak
│     Expected result    : Q item
│
└── Total: X skenario sebagai referensi analisa
```

Kemudian konfirmasi via `question()`:

```python
question([{
    "header": "Lanjut ke analisa Engineer?",
    "question": (
        f"QA context dari {len(qa_tasks)} Sub-task QA sudah di-extract. "
        "Lanjut ke pemilihan Sub-task Engineer untuk dianalisa?"
    ),
    "options": [
        {
            "label": "Ya, lanjut pilih Engineer",
            "description": "QA context sudah benar, lanjutkan ke pemilihan engineer"
        },
        {
            "label": "Ada yang perlu dicek ulang",
            "description": "Hentikan proses — saya akan cek tiket QA terlebih dahulu"
        }
    ]
}])
```

Jika user pilih "Ada yang perlu dicek ulang" → stop proses tanpa perubahan apapun.

Jika tidak ada Sub-task QA sama sekali → lanjut dengan `qa_context = {}` dan tampilkan:
```
⚠️  Tidak ada Sub-task QA di story ini. Analisa hanya menggunakan Epic PRD.
```

---

## Step 2.8 — API Contract Check (Auto-continue)

Analisa tambahan yang berjalan **otomatis** setelah QA Context Checkpoint. Tidak ada konfirmasi — hasil ditampilkan di chat lalu langsung lanjut ke Step 3.

### Deteksi FE / BE / Skip

```python
# Pisahkan berdasarkan prefix judul
be_tasks = [t for t in engineer_tasks
            if t['fields']['summary'].upper().startswith('[BE]')]
fe_tasks = [t for t in engineer_tasks
            if t['fields']['summary'].upper().startswith('[FE]')]
# Tiket dengan prefix "CLONE - " tidak masuk be_tasks/fe_tasks (prefix tidak match)

# Skip jika tidak ada pasangan lengkap FE + BE
if not be_tasks or not fe_tasks:
    pass  # Lanjut ke Step 3 tanpa notifikasi apapun
```

### Ekstraksi API References (Free-form AI)

Untuk setiap tiket FE dan BE, parse deskripsi ADF → plain text, lalu identifikasi referensi API dari teks bebas menggunakan pattern matching:

```python
import re

def extract_api_refs(text: str) -> dict:
    # HTTP method + URL: GET /api/v1/xxx, POST /booking/create, dll
    endpoint_pattern = re.compile(
        r'\b(GET|POST|PUT|DELETE|PATCH|HEAD|OPTIONS)\s+(/[\w/{}:.-]+)',
        re.IGNORECASE
    )
    endpoints = list({
        f"{m.group(1).upper()} {m.group(2)}"
        for m in endpoint_pattern.finditer(text)
    })

    # Query/path params: ?key=, :paramName, parameter: xxx
    param_pattern = re.compile(r'[?&:]([a-zA-Z_][\w]*)(?:\s*=|\b)', re.IGNORECASE)
    params = list({m.group(1) for m in param_pattern.finditer(text)
                   if len(m.group(1)) > 2})

    # Payload/response field keys: "key": atau key:
    field_pattern = re.compile(r'"([a-zA-Z_][\w]*)"\s*:', re.IGNORECASE)
    fields = list({m.group(1) for m in field_pattern.finditer(text)})

    # Status codes yang disebutkan
    status_pattern = re.compile(r'\b(200|201|400|401|403|404|422|500)\b')
    status_codes = list({m.group(1) for m in status_pattern.finditer(text)})

    return {
        "endpoints"   : endpoints,
        "params"      : params,
        "fields"      : fields,
        "status_codes": status_codes
    }
```

### Komparasi & Deteksi Mismatch

```python
def compare_contracts(fe_refs: dict, be_refs: dict) -> dict:
    fe_endpoints = set(fe_refs['endpoints'])
    be_endpoints = set(be_refs['endpoints'])
    fe_fields    = set(fe_refs['fields'])
    be_fields    = set(be_refs['fields'])

    return {
        "match"         : list(fe_endpoints & be_endpoints),
        "fe_only_ep"    : list(fe_endpoints - be_endpoints),  # FE expects, BE tidak define
        "be_only_ep"    : list(be_endpoints - fe_endpoints),  # BE define, FE tidak sebut
        "fe_only_fields": list(fe_fields - be_fields),
        "be_only_fields": list(be_fields - fe_fields),
        "has_mismatch"  : bool(
            (fe_endpoints - be_endpoints) or
            (be_endpoints - fe_endpoints) or
            (fe_fields - be_fields) or
            (be_fields - fe_fields)
        )
    }
```

### Tampilan di Chat (informatif, auto-continue)

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
🔌 API Contract Check
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FE : [FE-KEY] <summary>
BE : [BE-KEY] <summary>

✅ Match        : GET /api/v1/shipment/tracking

⚠️  FE expects  : POST /api/v1/booking/create
                  (tidak ditemukan di deskripsi BE)

⚠️  BE defines  : field "createdAt" di response
                  (tidak disebutkan di deskripsi FE)

→ Mismatch ditemukan. Comment dipost ke [FE-KEY] dan [BE-KEY].
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

Kondisi khusus:

```
# Semua sesuai → satu baris saja
🔌 API Contract Check — ✅ Tidak ditemukan mismatch.

# Tidak ada referensi API terdeteksi di salah satu tiket
🔌 API Contract Check — ℹ️ Tidak ditemukan referensi API
   eksplisit di [FE-KEY / BE-KEY]. Dilewati.

# Tidak ada pasangan FE + BE
🔌 API Contract Check — ℹ️ Tidak ada pasangan FE+BE. Dilewati.
```

### Post Comment ke FE dan BE (jika ada mismatch)

Comment yang **identik** dipost ke FE ticket dan BE ticket. Tidak ada konfirmasi terpisah — langsung dieksekusi setelah analisa selesai.

```python
def build_contract_comment(fe_key: str, be_key: str, comparison: dict) -> str:
    rows = []
    for ep in comparison['match']:
        rows.append(f"| ✅ | `{ep}` | FE + BE | Sesuai |")
    for ep in comparison['fe_only_ep']:
        rows.append(f"| ⚠️ | `{ep}` | FE only | Tidak ditemukan di deskripsi BE |")
    for ep in comparison['be_only_ep']:
        rows.append(f"| ⚠️ | `{ep}` | BE only | Tidak disebutkan di deskripsi FE |")
    for f in comparison['fe_only_fields']:
        rows.append(f"| ⚠️ | field `{f}` | FE only | BE tidak menyebutkan field ini |")
    for f in comparison['be_only_fields']:
        rows.append(f"| ⚠️ | field `{f}` | BE only | FE tidak menyebutkan penggunaan field ini |")

    table = "\n".join(rows)
    return f"""### 🔌 API Contract Analysis

**Cross-check:** [{fe_key}] ↔ [{be_key}]

| Status | Item | Sisi | Catatan |
|--------|------|------|---------|
{table}

**Rekomendasi:** FE dan BE perlu align kontrak ini sebelum implementasi — konfirmasi ke tiket masing-masing."""


if comparison['has_mismatch']:
    for ticket_key in [fe_key, be_key]:
        response = post_comment(ticket_key, build_contract_comment(fe_key, be_key, comparison))
        if 'id' in response:
            print(f"✅ Contract analysis dipost ke [{ticket_key}]")
        else:
            print(f"❌ Gagal post ke [{ticket_key}]: {response}")
```


---

## Step 3 — Pilih Sub-task Engineer (Interaktif)

Gunakan `question` tool dengan opsi **"Semua"** sebagai pilihan pertama:

```python
question([{
    "header": "Pilih Sub-task Engineer",
    "question": f"Story {story_key} memiliki {len(engineer_tasks)} Sub-task Engineer. Mana yang ingin dibuatkan test case-nya?",
    "multiple": True,
    "options": [
        {
            "label": "Semua Sub-task Engineer",
            "description": f"Proses semua {len(engineer_tasks)} tiket secara berurutan"
        },
        *[
            {
                "label": f"{t['key']} — {t['fields']['summary'][:70]}",
                "description": f"Status: {t['fields']['status']['name']} | Assignee: {t['fields'].get('assignee', {}).get('displayName', 'Unassigned')}"
            }
            for t in engineer_tasks
        ]
    ]
}])

# Jika user memilih "Semua Sub-task Engineer" → selected_tasks = engineer_tasks
# Jika user memilih tiket spesifik → filter engineer_tasks sesuai pilihan user
```

Lanjutkan dengan tiket-tiket yang dipilih user.

---

## Step 4 — Baca Deskripsi Sub-task Engineer yang Dipilih

Fetch detail tiket engineer (description = sumber scope & AC):

```bash
curl -s -u "$JIRA_USER:$JIRA_TOKEN" \
  "https://$JIRA_DOMAIN/rest/api/3/issue/$ENGINEER_KEY?fields=description,summary,status,assignee"
```

Parse deskripsi ADF → plain text menggunakan `extract_text()` dari Step 2.

`qa_context` sudah tersedia dari Step 2.5. `epic_context` sudah tersedia dari Step 1.5.

---

## Step 5 — Sintesis Test Case (AI Synthesis)

**Epic PRD + QA context = primary benchmark. Engineer AC = yang divalidasi.**

Analisis dengan reasoning berikut:

1. **Bangun ekspektasi:** Gunakan `epic_context` (PRD) + `qa_context` (skenario, impact, expected result dari semua QA) sebagai daftar lengkap apa yang diharapkan oleh product dan QA
2. **Identifikasi scope engineer:** Baca deskripsi Sub-task Engineer → petakan fitur/behaviour yang dikerjakan
3. **Buat test case:** Hasilkan skenario yang:
   - Memvalidasi bahwa scope engineer meng-cover ekspektasi dari QA context dan PRD Epic
   - Mencakup happy path + edge case sesuai AC engineer
   - Mengikuti pola/gaya penulisan dari kerangka Sub-task QA
4. **Deteksi gap:**
   - Ekspektasi dari Sub-task QA atau PRD Epic yang **tidak ter-cover** oleh scope engineer ini → catat sebagai gap
   - Scope di AC engineer yang **tidak selaras** dengan ekspektasi QA/PRD → catat sebagai potensi mismatch

---

## Step 6 — Format Output

**Test case section:**

```markdown
### Test Case: [ENGINEER-KEY] <summary>

| No | Skenario | Expected Result |
|----|----------|-----------------|
| 1  | <skenario> | <expected result> |
| 2  | <skenario> | <expected result> |

**Referensi Sub-task QA:** [QA-KEY1], [QA-KEY2]
```

**Gap section (hanya render jika ada gap):**

```markdown
### ⚠️ Gap Analysis: [ENGINEER-KEY]

**Ekspektasi yang tidak ter-cover oleh scope engineer ini:**
- [QA-KEY1] Skenario "X" — tidak ada AC di engineer yang meng-cover ini
- PRD Epic: Requirement "Y" — belum terlihat di deskripsi engineer

**Rekomendasi:**
Engineer perlu konfirmasi apakah skenario/requirement di atas masuk scope tiket ini
atau perlu didelegasikan ke tiket lain.
```

Jika tidak ada Sub-task QA sama sekali:
```
> ⚠️ Tidak ada Sub-task QA sebagai referensi. Test case dibuat hanya dari deskripsi Sub-task Engineer dan Epic PRD.
```

---

## Step 7 — Konfirmasi sebelum Post

Tampilkan preview lengkap di chat (test case + gap analysis jika ada), lalu gunakan `question` tool:

```python
question([{
    "header": "Post ke Jira?",
    "question": f"Test case untuk {engineer_key} sudah siap. Lanjut post sebagai comment di tiket ini?",
    "options": [
        {
            "label": "Ya, post sekarang",
            "description": "Ditambahkan sebagai comment di tiket Sub-task Engineer"
        },
        {
            "label": "Tidak, revisi dulu",
            "description": "Tampilkan saja di chat — saya copy manual ke Jira"
        }
    ]
}])
```

Jika user pilih "Tidak" → stop, tampilkan output di chat saja.

---

## Step 8 — Post sebagai Comment (Adaptive)

**Adaptive posting:**
- Tidak ada gap → **1 comment**: test cases saja
- Ada gap → **2 comment terpisah**: Comment 1 = test cases, Comment 2 = gap analysis

```python
import json, subprocess

def markdown_to_adf(text: str) -> dict:
    """
    Konversi sederhana: setiap baris non-kosong menjadi paragraph node ADF.
    Tabel markdown di-wrap sebagai codeBlock agar tidak rusak formatting.
    """
    lines = text.strip().split('\n')
    nodes = []
    in_table = False
    table_lines = []

    for line in lines:
        if line.startswith('|'):
            in_table = True
            table_lines.append(line)
        else:
            if in_table:
                nodes.append({
                    "type": "codeBlock",
                    "attrs": {"language": "markdown"},
                    "content": [{"type": "text", "text": '\n'.join(table_lines)}]
                })
                table_lines = []
                in_table = False
            if line.strip():
                nodes.append({
                    "type": "paragraph",
                    "content": [{"type": "text", "text": line}]
                })

    if table_lines:
        nodes.append({
            "type": "codeBlock",
            "attrs": {"language": "markdown"},
            "content": [{"type": "text", "text": '\n'.join(table_lines)}]
        })

    return {"type": "doc", "version": 1, "content": nodes}


def post_comment(engineer_key: str, markdown: str) -> dict:
    payload = {"body": markdown_to_adf(markdown)}
    result = subprocess.run([
        "curl", "-s", "-u", f"{JIRA_USER}:{JIRA_TOKEN}",
        "-X", "POST", "-H", "Content-Type: application/json",
        f"https://{JIRA_DOMAIN}/rest/api/3/issue/{engineer_key}/comment",
        "-d", json.dumps(payload)
    ], capture_output=True, text=True)
    return json.loads(result.stdout)


# Post test cases (selalu)
response1 = post_comment(engineer_key, test_case_markdown)
if 'id' in response1:
    print(f"✅ Test case comment berhasil dipost. ID: {response1['id']}")
else:
    print(f"❌ Gagal post test case comment: {response1}")

# Post gap analysis (hanya jika ada gap)
if has_gap:
    response2 = post_comment(engineer_key, gap_analysis_markdown)
    if 'id' in response2:
        print(f"✅ Gap analysis comment berhasil dipost. ID: {response2['id']}")
    else:
        print(f"❌ Gagal post gap analysis comment: {response2}")
```

Jika gagal → tampilkan output markdown di chat + instruksi:
```
❌ Gagal post comment ke Jira. Copy berikut secara manual ke tiket [KEY]:

<test case dan/atau gap analysis markdown>
```

---

## Output In-Chat (setelah berhasil)

```
✅ Test case untuk [ENGINEER-KEY] berhasil diposting sebagai comment.

📋 Ringkasan:
  - Jumlah skenario  : N
  - Referensi QA     : QA-KEY1, QA-KEY2
  - Gap ditemukan    : Ya (X item) / Tidak

🔗 https://lionparcel.atlassian.net/browse/ENGINEER-KEY
```

Jika ada beberapa engineer dipilih → proses sekuensial, ulangi Step 4–8 per tiket.

---

## Common Mistakes

| Kesalahan | Fix |
|-----------|-----|
| Pakai GET `/rest/api/3/search` | **Gunakan POST `/rest/api/3/search/jql`** — deprecated di Jira Cloud |
| Skip epic context | **Epic wajib** — PRD adalah primary benchmark untuk validasi coverage engineer |
| Pakai QA hanya sebagai referensi gaya | QA context adalah **primary benchmark**, bukan style guide |
| Test case scope seluruh story | Scope **hanya** dari AC Sub-task Engineer yang dipilih |
| Edit description tiket | **Selalu post sebagai comment**, jangan sentuh description |
| Tidak tampilkan preview sebelum post | **Wajib konfirmasi** via `question` sebelum post ke Jira |
| Lupa deteksi gap | Selalu analisa gap antara scope engineer vs ekspektasi QA + PRD |
| Tabel markdown rusak di Jira ADF | Gunakan `codeBlock` untuk wrap tabel markdown |
| Post gap + test case dalam 1 comment | **Adaptive posting**: pisahkan jadi 2 comment terpisah jika ada gap |
| Analisa tiket CLONE - xxx di contract check | Tiket dengan prefix **CLONE -** bukan tiket pengerjaan utama — **skip** dari API contract check |
| Tambah konfirmasi di Step 2.8 | API contract check **auto-continue** — tidak ada `question()`, hasil langsung tampil dan flow lanjut |
| Post contract analysis ke 1 tiket saja | Mismatch ditemukan → **post ke FE dan BE** — keduanya perlu aware karena bisa kerja paralel |
| Expect contract check akurat 100% dari free-form | Ekstraksi dari teks bebas adalah **best-effort** — engineer tetap harus review hasilnya |