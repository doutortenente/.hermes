---
name: sasi-schema-planning
version: 1.0.0
description: Plan SASI DB schema upgrades from clinical notes.
---

# SASI Schema Upgrade Planning

Use when the user wants to plan a schema migration for the SASI ICU tracking system (Airtable mirror ↔ Supabase P0). This covers the end-to-end diagnostic workflow: extract clinical concepts from real notes → introspect current Airtable schema → cross-reference with Supabase P0 contract → produce a phased upgrade plan.

## Core Procedure

### 1. Ingest Clinical Corpus
- Locate the source PDF (usually in Downloads on Tijolão, via SSH bridge)
- Extract full text with pymupdf (`fitz`/`pymupdf`)
- Count recurring clinical markers across all pages (devices, vitals, labs, consults, problems, procedures)
- Identify distinct patients, evolutions, and signed notes

### 2. Introspect Current Airtable Schema
- Pull base list via Airtable Meta API (`/v0/meta/bases`)
- For target base, fetch table schemas (`/v0/meta/bases/{baseId}/tables`)
- Record: field names, types, select options, linked tables, record counts
- Flag empty tables (opportunity for schema change without migration)

### 3. Load Supabase P0 Reference
- Read `references/08-p0-banco-10ago.md` from `sasi-ingest-export` skill (authoritative contract)
- Extract: enum changes, new tables (`dispositivo_episodios`, `janelas_24h`), vocabulary codes, derived-field rules

### 4. Cross-Reference & Gap Analysis
| Source | Check | Output |
|---|---|---|
| Clinical notes | Concept frequency | What MUST be modeled |
| Airtable schema | Field-by-field vs P0 | Enum drift, missing tables, missing fields |
| P0 reference | Vocabulary (79 codes) | Missing codes, `custom` deprecation |
| Clinical notes | Patterns NOT in either | Laudos, problemas ativos, antecedentes, consultorias, infusões |

### 5. Produce Phased Plan
- **Fase 1**: P0 parity fixes (enum corrections, missing P0 fields/tables, vocabulary sync) — Airtable-only, reversible, zero data risk
- **Fase 2**: Clinical gaps requiring Supabase migration first (laudos, problemas, antecedentes, consultorias, infusões) — write SQL migrations for Supabase, then mirror to Airtable
- **Fase 3**: Explicit non-modeling decisions (profilaxias, HMA longa) — document why excluded

## Output Format (ADHD-adapted)
1. **Diagnóstico** — 3 findings max, numbered
2. **Plano** — 3 phases, each with concrete steps
3. **Opinião** — 1 line on priority/sequencing
4. **Próxima ação** — single executable token (e.g. "F1")

## Pitfalls
- **Enum drift is silent killer** — Airtable `singleSelect` options diverge from Supabase CHECK constraints; always diff enums first. Fixing enum with data inside costs 10× more.
- **Vocabulary `custom` is technical debt** — P0 killed it (79 codes). Every `custom` event in Airtable is a future migration. Sync vocab 1:1 before any data ingest.
- **Derived fields ≠ editable fields** — `pacientes.dispositivos` became derived in P0. Writing to it in Airtable creates phantom state. Mark read-only in schema or drop.
- **Empty base = golden window** — If record counts are 0, do ALL schema changes now. Never wait for data to accumulate.
- **Laudos (imaging) are highest ROI** — Eco FE/PSAP series change clinical decisions. Model as separate table with structured achados_json, not prose.
- **Problemas ativos = clinical truth** — The numbered problem list opening every note is the source of truth. Text in `impressao` is lossy; extract to `problemas` table with `data_resolucao`.
- **Antecedentes + MUC appear in 100% of admissions** — Pacientes table needs `antecedentes` + `medicacoes_domiciliares` fields. Current `alergias` alone is insufficient.
- **Consultorias are lightweight** — 9 distinct specialties in corpus. Multiselect on Paciente beats a join table for this volume.
- **Infusões contínuas não-vasoativas need vocab** — Insulina BIC, heparina plena appear repeatedly. Add `ins_dose`, `heparina_dose` to vocabulary following `nor_dose` pattern.

## Tool Commands Reference
```bash
# SSH to Tijolão for PDF access
ssh -i ~/.ssh/id_ed25519 dr@localhost 'cat /home/dr/Downloads/file.pdf' > /tmp/file.pdf

# Extract text
python -c "import fitz; doc=fitz.open('/tmp/file.pdf'); print('\n'.join(p.get_text() for p in doc))" > /tmp/file.txt

# Airtable token from cofre
TOKEN=$(ssh -i ~/.ssh/id_ed25519 dr@localhost 'grep AIRTABLE_API_KEY /home/dr/projetos/.env | cut -d= -f2')

# List bases
curl -s -H "Authorization: Bearer $TOKEN" https://api.airtable.com/v0/meta/bases

# Schema for base
curl -s -H "Authorization: Bearer $TOKEN" https://api.airtable.com/v0/meta/bases/{BASE_ID}/tables

# Count records per table
curl -s -H "Authorization: Bearer $TOKEN" "https://api.airtable.com/v0/{BASE_ID}/{TABLE_ID}?maxRecords=100"
```

## Reference Files
- `references/sasi-p0-contract.md` → symlink/copy of `sasi-ingest-export/references/08-p0-banco-10ago.md` (loaded at step 3)
- `references/clinical-marker-catalog.md` → reusable list of clinical concepts to count (devices, vitals, labs, scores, procedures, consults)

## Related Skills
- `sasi-ingest-export` (provides P0 reference, vocab, ingest rules — read-only)
- `plan` (general plan writing format)
- `tijolao-ssh-bridge` (access Tijolão files)
