# Clinical Marker Catalog — Reusable Concepts for SASI Note Census

Use this list when scanning clinical notes to quantify what must be modeled. Count occurrences per concept across the corpus.

## Device & Access Markers
- IOT / intubação / traqueostomia
- CVC / PICC / PowerPICC / arterial line
- SVD / SNE / SNG
- Dreno (pigtail, pleural, peritoneal)
- NPT / TNE
- Marca-passo / LVAD / ECMO / IABP / Swan

## Vasoactive & Infusion Markers
- Noradrenalina (nor_dose)
- Dobutamina (dobuta_dose)
- Dopamina (dopa_dose)
- Vasopressina (vaso_dose)
- Adrenalina (adr_dose)
- Fentanil (fent_dose) — sedation/analgesia
- Midazolam (midaz_dose)
- Propofol (propofol_dose)
- Dexmedetomidina (precedex_dose)
- **Insulina BIC (ins_dose)** — continuous non-vasoactive
- **Heparina plena (heparina_dose)** — continuous non-vasoactive

## Vital Signs (with thresholds)
- PA sys / PA dia / PAM (PAM<65, PAM>110)
- FC (FC>100)
- FR (FR>20)
- SpO2 (SpO2<92)
- Temp (TAX<35.5)
- Glicemia capilar (Dx>180)
- GCS / RASS / CAM-ICU
- P/F ratio (pf_ratio)

## Labs (LOINC-mapped where possible)
- Hb / Ht / Plaq / Leuco
- UR / Cr / K / Na / Mg / Ca / Ca iônico
- Lactato / pH / pCO2 / HCO3 / SBE / PO2
- INR / TP / TTPA / Fibrinogênio / D-dímero
- Troponina / ProBNP / PCR / Procalcitonina
- TGO / TGP / GGT / FA / BD / BI / Amilase / Lipase / DHL / VHS
- TSH / T4L
- Gasometria (pH, pCO2, PO2, HCO3, SBE, lactato, FiO2, SatO2)

## Scores
- SOFA (total + 6 sub-scores)
- PESI
- APACHE (if present)

## Imaging / Procedures
- TC (tórax, abdome, pelve, crânio)
- EcoTT / Eco TE (FE, PSAP, PAD, VTR, VCI, valvular)
- Doppler (venoso MMII, arterial, carotídeo)
- USG (rins, vias urinárias, abdominal)
- Rx tórax
- Broncoscopia / Endoscopia / Cápsula endoscópica
- Cirurgia / PO / Drenagem
- Biópsia
- Traqueostomia / Percutânea / Cirúrgica
- Diálise / TRS / HD / UF dialise

## Microbiology
- Hemocultura / Urocultura / Aspirado traqueal / Lavado BAL
- LCR / Secreção ferida / Líquido peritoneal / Líquido pleural
- Antibiograma (S/I/R + CIM)
- Cultura parcial (72h, 24h)

## Antibiotics
- Drogas + dose + via + frequência + data_início/fim
- Intenção: empírica / dirigida / profilática
- Foco infeccioso
- Escalonamento (fechar curso A, abrir curso B)

## Problems List (numbered at note opening)
- Extract each numbered item as active problem with resolution date

## Consults / Specialty Follow-up
- Cardiologia / Infectologia / Nefrologia / Cirurgia Geral
- GO / Neurocirurgia / Pneumologia / Endocrino / Hematologia

## Background / Admission Data
- Antecedentes pessoais (AP)
- Medicações uso domiciliar (MUC)
- Alergias
- CID atual
- Readmissão / Reinternação

## Balance / Fluids
- Diurese (total, por cateter, espontânea)
- Balanço hídrico (BH)
- Evacuações
- Dieta / Infusões / Ingestão hídrica

## Sedation / Analgesia / Delirium
- Sedação (sim/não + drogas)
- Analgesia adequada
- CAM-ICU
- Quetiapina / Haloperidol / Benzodiazepínicos

## Prophylaxis
- HBPM (enoxaparina / clexane dose)
- PPI (pantoprazol)
- Outros

## Outcomes / Disposition
- Alta UTI / andar / óbito / transferência
- Óbito
- Reinternação

## Usage Notes
- Count each marker per page / per note
- Note when a concept appears but has NO target field in Airtable or Supabase
- Flag concepts with >5 occurrences that are NOT modeled — these are migration candidates
- Flag concepts modeled in Supabase P0 but MISSING in Airtable mirror — these are Fase 1 parity fixes