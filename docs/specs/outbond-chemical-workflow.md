
# OBC v5.x — Chemical-Aware Workflow Specification

**Modul:** Outbound Cargo (OBC)  
**Target Versi:** v5.0 (planned)  
**Versi Saat Ini:** OBC v4.1 (`outbound.html`)  
**Referensi:** TKO B07-009/KPI48540/2025-S9  
**Status:** DRAFT — Belum diimplementasikan  

---

## Latar Belakang

OBC v4.1 saat ini menangani semua jenis material (general, chemical, B3) dengan workflow
yang seragam — 4 step: Creator → Function Head → Legality → Gate.

Spesifikasi ini mendefinisikan **workflow chemical-aware** untuk OBC v5.x, dengan penambahan:

- Peran **HSSE/OH** sebagai otoritas clearance untuk material chemical/B3
- Peran **Section Head** yang terpisah dari Function Head
- Validasi **SDS vs label** di level Gate Security
- **17 status** transisi (vs 5 di v4.1) untuk traceability penuh
- Fokus material: **Aluminum Sulphate, Liquid N₂, Molasses, Caustic Soda**

### Prinsip Utama

| Peran | Otoritas |
|---|---|
| HSSE/OH | Otoritas chemical: SDS, CoA, hazard clearance |
| Security Gate | SDS match label + pemeriksaan fisik |
| Legality | Admin dokumen + konfirmasi HSSE clearance sudah ada |

> **Catatan Implementasi:** Pseudocode ini memerlukan back-end server (`persist()`, `NOTIFY()`,
> `GENERATE_QR_TOKEN()`). Tidak kompatibel dengan arsitektur localStorage-only OBC v4.1.

---

## Enum & Struktur Data

### Role

```
ENUM Role {
  USER_AREA,
  SECTION_HEAD,
  HSSE_OH,
  LEGALITY,
  GATE_SECURITY,
  SS_SUPERINTENDENT
}
```

### Status (17 State)

```
ENUM Status {
  DRAFT,
  SUBMITTED,
  REJECTED_SH,
  APPROVED_SH,
  PENDING_HSSE,
  HOLD_HSSE,
  REJECTED_HSSE,
  APPROVED_HSSE,
  PENDING_LEGALITY,
  HOLD_LEGALITY,
  REJECTED_LEGALITY,
  STAMPED_LEGALITY,
  GATE_READY,
  HOLD_GATE,
  REJECTED_GATE,
  RELEASED,
  CLOSED
}
```

### ChemKey

```
ENUM ChemKey {
  ALUM_SULPHATE,
  LIQUID_N2,
  MOLASSES,
  CAUSTIC_SODA,
  OTHER_CHEM
}
```

### Manifest

```
STRUCT Manifest {
  id
  status
  created_by_user_id
  section_head_id
  hsse_approver_id
  legality_id
  gate_officer_id

  gate_target            // Pintu 1 / 4 / 5
  schedule_time
  vehicle_plate
  driver_name
  driver_id_number

  items[]                // list of ManifestItem
  attachments[]          // list of Attachment (SDS/CoA/DO/PO/etc)
  audit_events[]         // append-only

  hsse_clearance_status  // NONE / APPROVED / HOLD / REJECTED
  hsse_clearance_note
  legality_stamp_id
  qr_token               // manifest_id + checksum (tanpa PII/data sensitif mentah)
}
```

### ManifestItem

```
STRUCT ManifestItem {
  line_id
  description_raw
  qty
  unit
  classification          // GENERAL / CHEMICAL / B3 / SCRAP / TOOLS / etc
  chem_key               // derived jika classification CHEMICAL/B3

  requires_coa_flag      // derived dari requirement PO/SPK/receiver (bukan oleh Security)
  packaging_type         // drum / jerrycan / IBC / dewar / cylinder / etc
  label_photo_id         // evidence untuk Gate "match"
  sds_id                 // attachment id
  coa_id                 // attachment id (opsional/kondisional)
  sds_extracted          // key fields dari SDS (manual atau OCR)
}
```

### SDSExtract

```
STRUCT SDSExtract {
  product_name
  cas_number_optional
  un_number_optional
  hazard_class_optional
  revision_date_optional
  section14_present      // boolean
}
```

### Attachment

```
STRUCT Attachment {
  id
  type    // SDS / COA / SURAT_JALAN / DO / PO / SPK / PHOTO_LABEL / PHOTO_ITEM / OTHER
  hash    // server-side hash untuk integritas (opsional tapi direkomendasikan)
  url_or_storage_ref
}
```

### GateCheckResult

```
STRUCT GateCheckResult {
  physical_qty_ok
  packaging_ok
  label_present
  sds_present_in_system
  label_vs_sds_match
  un_class_match_if_present
  notes
}
```

---

## Alur Status

```
DRAFT
  └─► SUBMITTED
        ├─► REJECTED_SH
        └─► APPROVED_SH
              ├─► PENDING_HSSE (jika ada item chemical/B3)
              │     ├─► HOLD_HSSE
              │     ├─► REJECTED_HSSE
              │     └─► APPROVED_HSSE ──┐
              │                         │
              └─► PENDING_LEGALITY ◄────┘ (jika tidak ada chemical)
                    ├─► HOLD_LEGALITY
                    ├─► REJECTED_LEGALITY
                    └─► STAMPED_LEGALITY
                          └─► GATE_READY
                                ├─► HOLD_GATE
                                ├─► REJECTED_GATE
                                └─► RELEASED
                                      └─► CLOSED
```

---

## Fungsi Utilitas

### Audit Log (Append-Only)

```pseudocode
FUNCTION AUDIT_APPEND(manifest, actor_role, actor_id, action, payload):
  event = {ts_server_now(), actor_role, actor_id, action, payload}
  manifest.audit_events.append(event)   // harus append-only di backend
  persist(event)
```

### Normalisasi Chem Key

```pseudocode
FUNCTION NORMALIZE_CHEM_KEY(desc_raw):
  t = LOWER(desc_raw)
  IF t contains ["liquid n2","ln2","liquid nitrogen","nitrogen refrigerated"]
    RETURN LIQUID_N2
  IF t contains ["caustic soda","sodium hydroxide","naoh"]
    RETURN CAUSTIC_SODA
  IF t contains ["aluminum sulphate","aluminium sulphate","alum sulphate","al2(so4)3"]
    RETURN ALUM_SULPHATE
  IF t contains ["molasses","tetes tebu","molase"]
    RETURN MOLASSES
  RETURN OTHER_CHEM

FUNCTION IS_CHEMICAL(item):
  RETURN item.classification IN ["CHEMICAL", "B3"]
```

### Policy: Lampiran Wajib

```pseudocode
FUNCTION REQUIRE_SDS(item):
  // Security policy RU: semua chemical wajib SDS
  RETURN IS_CHEMICAL(item)

FUNCTION REQUIRE_LABEL_PHOTO(item):
  // Untuk Gate match. Minimal 1 foto label packaging.
  RETURN IS_CHEMICAL(item)

FUNCTION REQUIRE_HSSE_CLEARANCE(manifest):
  // HSSE clearance wajib bila ada item chemical/B3
  RETURN EXISTS(manifest.items WHERE IS_CHEMICAL(item))

FUNCTION REQUIRE_COA(item):
  // CoA bukan kewajiban universal;
  // mengikuti requirement PO/SPK/receiver/HSSE policy
  RETURN item.requires_coa_flag == TRUE
```

### Validasi SDS vs Label (Gate-Focused)

```pseudocode
FUNCTION VALIDATE_SDS_MATCH_LABEL(item):
  IF item.label_photo_id is NULL  RETURN FAIL("No label evidence")
  IF item.sds_id is NULL          RETURN FAIL("No SDS attached")

  // label_text bisa manual entry ATAU OCR dari foto label
  label_text = GET_LABEL_TEXT(item.label_photo_id)
  sds        = item.sds_extracted

  IF NOT FUZZY_MATCH(label_text, sds.product_name)
    RETURN FAIL("Label vs SDS product mismatch")

  // Cek UN/Class hanya jika label dan SDS sama-sama mencantumkannya
  label_un    = PARSE_UN_FROM_TEXT(label_text)      // e.g., "UN 1977"
  label_class = PARSE_CLASS_FROM_TEXT(label_text)   // e.g., "Class 8"

  IF label_un exists AND sds.un_number_optional exists
      AND label_un != sds.un_number_optional:
    RETURN FAIL("UN mismatch label vs SDS")

  IF label_class exists AND sds.hazard_class_optional exists
      AND label_class != sds.hazard_class_optional:
    RETURN FAIL("Class mismatch label vs SDS")

  RETURN PASS
```

---

## Alur Kerja Per Step

### Step 0 — Buat Draft (User Area)

```pseudocode
FUNCTION CREATE_MANIFEST(user_id, role, manifest_payload):
  REQUIRE role == USER_AREA
  m = new Manifest()
  m.status              = DRAFT
  m.created_by_user_id  = user_id
  m.items               = manifest_payload.items

  FOR each item IN m.items:
    IF IS_CHEMICAL(item):
      item.chem_key = NORMALIZE_CHEM_KEY(item.description_raw)

  AUDIT_APPEND(m, role, user_id, "CREATE_DRAFT", {summary_fields})
  persist(m)
  RETURN m
```

### Step 1 — Submit ke Section Head

```pseudocode
FUNCTION SUBMIT_MANIFEST(manifest, user_id, role):
  REQUIRE role == USER_AREA
  REQUIRE manifest.status == DRAFT
  REQUIRE manifest.vehicle_plate not empty
  REQUIRE manifest.driver_name not empty
  REQUIRE manifest.gate_target not empty
  REQUIRE manifest.items.length > 0

  // Kunci field kritis setelah submit (revisi butuh alur tersendiri)
  LOCK_FIELDS(manifest, [
    "items.qty", "items.description_raw",
    "vehicle_plate", "gate_target", "driver_id_number"
  ])

  manifest.status = SUBMITTED
  AUDIT_APPEND(manifest, role, user_id, "SUBMIT_TO_SH", {})
  NOTIFY(manifest.section_head_id, "Manifest waiting approval", manifest.id)
  persist(manifest)
```

### Step 2 — Section Head Approval

```pseudocode
FUNCTION SH_DECISION(manifest, sh_id, role, decision, note):
  REQUIRE role == SECTION_HEAD
  REQUIRE manifest.status == SUBMITTED

  IF decision == "REJECT":
    manifest.status = REJECTED_SH
    AUDIT_APPEND(manifest, role, sh_id, "SH_REJECT", {note})
    NOTIFY(manifest.created_by_user_id, "Rejected by Section Head", {manifest.id, note})
    persist(manifest)
    RETURN

  IF decision == "APPROVE":
    manifest.status = APPROVED_SH
    AUDIT_APPEND(manifest, role, sh_id, "SH_APPROVE", {note})

    IF REQUIRE_HSSE_CLEARANCE(manifest):
      manifest.status = PENDING_HSSE
      AUDIT_APPEND(manifest, role, sh_id, "ROUTE_TO_HSSE", {})
      NOTIFY(HSSE_QUEUE, "Chemical manifest needs clearance", manifest.id)
    ELSE:
      manifest.status = PENDING_LEGALITY
      NOTIFY(LEGALITY_QUEUE, "Manifest needs legality verification", manifest.id)

    persist(manifest)
```

### Step 3 — HSSE/OH Clearance (Otoritas Chemical)

```pseudocode
FUNCTION HSSE_REVIEW(manifest, hsse_id, role, decision, note):
  REQUIRE role == HSSE_OH
  REQUIRE manifest.status IN [PENDING_HSSE, HOLD_HSSE]

  FOR each item IN manifest.items WHERE IS_CHEMICAL(item):

    IF REQUIRE_SDS(item) AND item.sds_id is NULL:
      manifest.status                = HOLD_HSSE
      manifest.hsse_clearance_status = "HOLD"
      AUDIT_APPEND(manifest, role, hsse_id, "HSSE_HOLD",
        {reason: "SDS missing", item: item.line_id})
      NOTIFY(manifest.created_by_user_id, "HSSE HOLD: SDS missing",
        {manifest.id, item.line_id})
      persist(manifest)
      RETURN

    IF REQUIRE_LABEL_PHOTO(item) AND item.label_photo_id is NULL:
      manifest.status                = HOLD_HSSE
      manifest.hsse_clearance_status = "HOLD"
      AUDIT_APPEND(manifest, role, hsse_id, "HSSE_HOLD",
        {reason: "Label photo missing", item: item.line_id})
      NOTIFY(manifest.created_by_user_id, "HSSE HOLD: label evidence missing",
        {manifest.id, item.line_id})
      persist(manifest)
      RETURN

    IF REQUIRE_COA(item) AND item.coa_id is NULL:
      manifest.status                = HOLD_HSSE
      manifest.hsse_clearance_status = "HOLD"
      AUDIT_APPEND(manifest, role, hsse_id, "HSSE_HOLD",
        {reason: "CoA required but missing", item: item.line_id})
      NOTIFY(manifest.created_by_user_id, "HSSE HOLD: CoA missing",
        {manifest.id, item.line_id})
      persist(manifest)
      RETURN

    // Opsional: HSSE bisa jalankan SDS-vs-label precheck untuk kurangi masalah di gate
    r = VALIDATE_SDS_MATCH_LABEL(item)
    IF r is FAIL:
      manifest.status                = HOLD_HSSE
      manifest.hsse_clearance_status = "HOLD"
      AUDIT_APPEND(manifest, role, hsse_id, "HSSE_HOLD",
        {reason: r.message, item: item.line_id})
      NOTIFY(manifest.created_by_user_id, "HSSE HOLD: SDS/label mismatch",
        {manifest.id, item.line_id, r.message})
      persist(manifest)
      RETURN

  IF decision == "REJECT":
    manifest.status                = REJECTED_HSSE
    manifest.hsse_clearance_status = "REJECTED"
    manifest.hsse_approver_id      = hsse_id
    manifest.hsse_clearance_note   = note
    AUDIT_APPEND(manifest, role, hsse_id, "HSSE_REJECT", {note})
    NOTIFY([manifest.created_by_user_id, manifest.section_head_id],
      "HSSE REJECT", {manifest.id, note})
    persist(manifest)
    RETURN

  IF decision == "APPROVE":
    manifest.status                = PENDING_LEGALITY
    manifest.hsse_clearance_status = "APPROVED"
    manifest.hsse_approver_id      = hsse_id
    manifest.hsse_clearance_note   = note
    AUDIT_APPEND(manifest, role, hsse_id, "HSSE_APPROVE", {note})
    NOTIFY(LEGALITY_QUEUE,
      "HSSE cleared chemical manifest; legality verify needed", manifest.id)
    persist(manifest)
```

### Step 4 — Legality Verification & Stamp

```pseudocode
FUNCTION LEGALITY_VERIFY_AND_STAMP(manifest, legality_id, role, decision, note):
  REQUIRE role == LEGALITY
  REQUIRE manifest.status IN [PENDING_LEGALITY, HOLD_LEGALITY]

  REQUIRE HAS_ATTACHMENT(manifest, "SURAT_JALAN") OR HAS_ATTACHMENT(manifest, "DO")
    ELSE HOLD("Missing DO/SJ")

  // Legality konfirmasi HSSE clearance — tidak melakukan pekerjaan HSSE
  IF REQUIRE_HSSE_CLEARANCE(manifest):
    IF manifest.hsse_clearance_status != "APPROVED":
      manifest.status = HOLD_LEGALITY
      AUDIT_APPEND(manifest, role, legality_id, "LEGALITY_HOLD",
        {reason: "HSSE clearance not approved"})
      NOTIFY(HSSE_QUEUE, "Need HSSE clearance", manifest.id)
      persist(manifest)
      RETURN

  IF decision == "REJECT":
    manifest.status = REJECTED_LEGALITY
    AUDIT_APPEND(manifest, role, legality_id, "LEGALITY_REJECT", {note})
    NOTIFY([manifest.created_by_user_id, manifest.section_head_id],
      "Rejected by Legality", {manifest.id, note})
    persist(manifest)
    RETURN

  IF decision == "STAMP":
    manifest.legality_stamp_id = GENERATE_STAMP_ID()
    manifest.qr_token          = GENERATE_QR_TOKEN(manifest.id, checksum_server(manifest))
    manifest.legality_id       = legality_id
    manifest.status            = STAMPED_LEGALITY

    LOCK_FIELDS(manifest, ["items", "vehicle_plate", "driver_id_number", "attachments"])
    AUDIT_APPEND(manifest, role, legality_id, "LEGALITY_STAMP",
      {stamp_id: manifest.legality_stamp_id, note})

    manifest.status = GATE_READY
    PUSH_TO_GATE_MONITOR(manifest.id, summary_for_gate(manifest))
    NOTIFY(GATE_QUEUE, "Gate-ready manifest", manifest.id)
    persist(manifest)
```

### Step 5 — Gate Security (Release / Reject / Hold)

```pseudocode
FUNCTION GATE_CHECK_AND_DECIDE(manifest, gate_id, role, decision, gate_notes):
  REQUIRE role == GATE_SECURITY
  REQUIRE manifest.status IN [GATE_READY, HOLD_GATE]

  result = new GateCheckResult()
  result.physical_qty_ok = PHYSICAL_MATCH_QTY(manifest)
  result.packaging_ok    = PHYSICAL_PACKAGING_OK(manifest)

  FOR each item IN manifest.items WHERE IS_CHEMICAL(item):

    // Gate tidak boleh bypass HSSE
    IF manifest.hsse_clearance_status != "APPROVED":
      manifest.status = HOLD_GATE
      AUDIT_APPEND(manifest, role, gate_id, "GATE_HOLD",
        {reason: "HSSE clearance not approved"})
      NOTIFY([HSSE_QUEUE, LEGALITY_QUEUE],
        "Gate HOLD: HSSE clearance missing", manifest.id)
      persist(manifest)
      RETURN

    r = VALIDATE_SDS_MATCH_LABEL(item)
    IF r is FAIL:
      manifest.status = HOLD_GATE
      AUDIT_APPEND(manifest, role, gate_id, "GATE_HOLD",
        {reason: r.message, item: item.line_id})
      NOTIFY([HSSE_QUEUE, LEGALITY_QUEUE],
        "Gate HOLD: SDS/Label mismatch", {manifest.id, item.line_id, r.message})
      persist(manifest)
      RETURN

  // Deteksi bypass: label tampak chemical tapi manifest tidak diflag
  IF DETECT_CHEMICAL_KEYWORDS_FROM_LABELS(manifest) == TRUE
      AND NOT REQUIRE_HSSE_CLEARANCE(manifest):
    manifest.status = HOLD_GATE
    AUDIT_APPEND(manifest, role, gate_id, "GATE_HOLD",
      {reason: "Possible chemical bypass (label indicates chemical)"})
    NOTIFY([HSSE_QUEUE, LEGALITY_QUEUE],
      "Gate HOLD: possible chemical bypass", manifest.id)
    persist(manifest)
    RETURN

  IF decision == "REJECT":
    manifest.status          = REJECTED_GATE
    manifest.gate_officer_id = gate_id
    REQUIRE ATTACH_EVIDENCE_PHOTO(manifest,
      ["PHOTO_LABEL", "PHOTO_ITEM", "PHOTO_PLATE"])
    AUDIT_APPEND(manifest, role, gate_id, "GATE_REJECT", {gate_notes})
    NOTIFY([
      manifest.created_by_user_id,
      manifest.section_head_id,
      LEGALITY_QUEUE,
      HSSE_QUEUE
    ], "Gate REJECT", {manifest.id, gate_notes})
    persist(manifest)
    RETURN

  IF decision == "HOLD":
    manifest.status          = HOLD_GATE
    manifest.gate_officer_id = gate_id
    AUDIT_APPEND(manifest, role, gate_id, "GATE_HOLD", {gate_notes})
    NOTIFY([LEGALITY_QUEUE, HSSE_QUEUE], "Gate HOLD", {manifest.id, gate_notes})
    persist(manifest)
    RETURN

  IF decision == "RELEASE":
    REQUIRE result.physical_qty_ok == TRUE
    REQUIRE result.packaging_ok    == TRUE
    manifest.status          = RELEASED
    manifest.gate_officer_id = gate_id
    AUDIT_APPEND(manifest, role, gate_id, "GATE_RELEASE", {gate_notes})
    NOTIFY([manifest.created_by_user_id, LEGALITY_QUEUE],
      "Released at Gate", manifest.id)
    manifest.status = CLOSED
    AUDIT_APPEND(manifest, "SYSTEM", "SYSTEM", "CLOSE_CASE", {})
    persist(manifest)
```

---

## Prioritas Material Chemical

```pseudocode
FUNCTION CHEM_PRIORITY(item):
  IF item.chem_key IN [LIQUID_N2, CAUSTIC_SODA]  RETURN "HIGH"
  IF item.chem_key IN [ALUM_SULPHATE]             RETURN "MEDIUM"
  IF item.chem_key IN [MOLASSES]                  RETURN "LOW"
  RETURN "MEDIUM"
```

| Material | chem_key | Prioritas | Alasan |
|---|---|---|---|
| Liquid N₂ | `LIQUID_N2` | HIGH | Cryogenic, asphyxiation risk |
| Caustic Soda | `CAUSTIC_SODA` | HIGH | Corrosive, Class 8 |
| Aluminum Sulphate | `ALUM_SULPHATE` | MEDIUM | Irritant, volume besar |
| Molasses | `MOLASSES` | LOW | Bukan B3, tapi perlu tracking |

---

## Catatan Implementasi

> Poin-poin ini bukan bagian dari logika pseudocode, tapi krusial untuk validitas sistem di produksi.

- **Semua `AUDIT_APPEND` harus server-side append-only.** Jangan izinkan update/delete pada tabel audit.
- **Jangan simpan data sensitif di localStorage untuk produksi.** Arsitektur ini membutuhkan back-end server.
- **QR token jangan berisi PII.** Cukup `manifest_id` + `checksum` server-side.
- **`LOCK_FIELDS` harus di-enforce di server**, bukan hanya di UI.
- **`FUZZY_MATCH`** untuk validasi SDS vs label disarankan toleransi ~80% untuk mengakomodasi singkatan nama produk.
- **Offline fallback** (SS Superintendent) dari OBC v4.1 tetap berlaku untuk kasus emergency, tapi chemical manifest tetap wajib HSSE clearance setelah rekonsiliasi.

---

## Perbandingan dengan OBC v4.1

| Aspek | OBC v4.1 (Saat Ini) | OBC v5.x (Spec Ini) |
|---|---|---|
| Jumlah status | 5 | 17 |
| Peran | 6 | 6 + HSSE_OH + SECTION_HEAD |
| Chemical handling | Sama dengan general | Dedicated HSSE clearance path |
| SDS validation | Tidak ada | HSSE precheck + Gate enforcement |
| Storage | localStorage | Back-end server (required) |
| Audit log | Tidak ada | Append-only per event |
| QR token | Tidak ada | Setelah Legality stamp |

---

*Dokumen ini adalah spesifikasi teknis untuk OBC v5.x. Implementasi memerlukan back-end server, database persisten, dan sistem notifikasi yang belum tersedia di platform v4.1.0.*
