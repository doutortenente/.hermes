---
name: sasi-cockpit-pipeline
description: "Pipeline SASI: OCR -> motor -> ponte Airtable -> Supabase."
version: 0.1.0
author: Nícholas Nagaita (doutortenente), Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [SASI, UTI, Airtable, Supabase, OCR, pipeline]
    related_skills: [sasi-ingest-export, mestre-airtable]
---

# SASI Cockpit Pipeline

Pipeline operacional do Dr. Nicolas (R2 Intensivista, plantao 24h a cada 15 dias) para gestao de ate 12 pacientes/UTI. Transforma insumos brutos (fotos, PDFs, texto) em cockpit Airtable operacional + cofre clinico Supabase.

## Quando usar

- Preparar/operar plantao: ingest de folhas, exames, laudos -> JSON auditado -> ponte -> cockpit
- Evoluir schema do cockpit (novas tabelas, campos, enums) e sincronizar com Supabase P0
- Depurar idempotencia da ponte, dedup de janelas, migracao de vocabulario
- Entregar passagem de plantao (1 pagina) e evolucao medica (SOAP SASI)

Nao usar para: decisoes clinicas (e ferramenta, nao medico), ingest sem fonte legivel.

## Arquitetura (fixa por decisao de custo)

| Camada | Tech | Responsabilidade |
|---|---|---|
| Extracao + Auditoria | sasi-ingest-export (skill) | Foto/PDF/texto -> JSON validado, zero alucinacao |
| Motor deterministico | scripts/build_passagem.py | Max/Min vitais, balanco hidrico, flags, SOFA insumos -- LLM NAO calcula |
| Ponte cockpit | scripts/sasi_ponte.py | JSON -> Airtable (Pacientes, Evolucoes, Janelas24h, DispositivoEpisodios, PlantaoDia, Laudos, Vocabulario) |
| Cofre clinico | Supabase (30 migrations P0) | Tabelas ATB, Culturas, DispositivoEpisodios, Janelas24h, SOFA diario, RLS |
| Frontend | Morto (Next.js cancelado com Claude Code R$500/mes) | Nao ressuscitar |

Fonte da verdade: Supabase = cofre; Airtable = cockpit operacional do plantao. Escrita concorrente so com regra explicita (hoje: ponte so escreve no Airtable; Supabase via MCP sasi_deploy_ingest sob comando deploy).

## Procedimento de plantao (ordem de execucao)

1. Ingestao paralela -- subagentes por leito (skill sasi-ingest-export) extraem JSON cru por paciente.
2. Motor -- python3 scripts/build_passagem.py --file leitos.json calcula max/min, balanco, flags, SOFA insumos, emite .md de passagem + JSON consolidado.
3. Ponte -- python3 scripts/sasi_ponte.py --input leitos.json --token-file /tmp/at_tok.json faz upsert idempotente no Airtable:
   - Pacientes (upsert por leito)
   - Evolucoes (data_plantao + turno + illness_severity)
   - Janelas24h (dedup por paciente+tipo+janela_fim = mesmo dia)
   - DispositivoEpisodios (abre/fecha por data da fonte)
   - PlantaoDia (snapshot do dia)
   - Laudos (PDFs processados)
   - Vocabulario (77 tipos canonicos -- custom morreu)
4. Verificacao -- releitura por rec... + contagens: 1 paciente/leito, 8 janelas/tipo/dia, zero duplicata.
5. Deploy Supabase -- so quando Dr. disser deploy / salvar no Supabase -> MCP sasi_deploy_ingest grava eventos/evolucao/pendencias/ATB/Culturas/DispositivoEpisodios/Janelas24h.
6. Passagem -- script ja emitiu .md (1 pagina A4 por paciente ou bloco). Copiar/colar no prontuario.

## Idempotencia da ponte (regras duras)

- Paciente: upsert por leito (chave natural do plantao). Reprocessar = atualiza, nao duplica.
- Janelas24h: dedup por (paciente, tipo, janela_fim[:10]) -- mesma data de plantao = mesmo registro. NAO usar filterByFormula com FIND em linked record (array nao concatena como string); listar tudo e filtrar em Python (base <= 200 regs/plantao).
- DispositivoEpisodios: INSERT se nao houver episodio aberto igual; UPDATE para fechar (data_fim + motivo_fim). Nunca escrever em pacientes.dispositivos (campo derivado).
- Vocabulario: 77 codigos ativos fixos. Valor sem tipo -> warning, nao gravar como custom.

## Pitfalls conhecidos (custaram tempo real)

- POST /meta/bases/{}/tables/{}/fields espera UM objeto (name/type/options), NAO {"fields":[...]}.
- PATCH de choices em singleSelect retorna Changing a field's type... mesmo sem mudar tipo -- Meta API nao edita/remove choices. Workaround: renomear campo legado + criar novo com enum correto.
- DELETE de tabela nao existe na Web API (404). Tabela residual -> renomear zzz DELETAR + avisar operador.
- CREATE-VIEW nao existe na Web API (POST /views da 422 enganoso; doc oficial so lista GET/DELETE, delete e Enterprise). Entregar roteiro de cliques UI.
- Endpoint records: /v0/{baseId}/{tableId} (sem /records em GET/PATCH/DELETE por id); POST criacao e /v0/{baseId}/{tableId} com {"records":[...]}.
- filterByFormula + linked record nao funciona (array != string). Filtrar em Python apos listagem completa.
- janela_fim no JSON = data do PLANTAO (ex: 2026-08-24), nao data de gravacao. Dedup compara data da janela, nao now().
- Motor build_passagem.py: diurese/SNG/dreno/evacuacao = sempre PERDA (CANON do script). Celula ilegivel = ? (incerteza, nao soma). Conferencia enfermagem (ganhos/perdas/bh) flaga divergencia automatica.
- SOFA 6 componentes: 3 ficam zerados se faltar insumo (P/F precisa FiO2; plaquetas precisa hemograma; BB precisa bioquimica; PAM+nor_dose precisa peso; GCS precisa folha neuro; Cr+diurese precisa bioquimica/folha). Declarar faltante, nunca chutar.

## Verificacao de alta

- 7 pacientes reais no cockpit + 0 janelas teste
- 3 runs da ponte: run1=gravados, run2/3=atualizados (0 novos)
- Contagem exata: 1 registro Pacientes/leito, 8 Janelas24h/tipo/dia
- Vocabulario 77 ativos, 0 custom
- Supabase migrations 30/30 aplicadas (P0 de 10-ago incluido)
- sasi_deploy_ingest grava sem erro quando acionado
- Passagem .md abre no editor e cabe em 1 pagina A4

## Referencias ligadas

- sasi-ingest-export (extracao + auditoria + export templates)
- mestre-airtable (operacao segura de bases, API, migracao)
- references/08-p0-banco-10ago.md (contrato Supabase P0)
- scripts/build_passagem.py (motor deterministico)
- scripts/sasi_ponte.py (ponte idempotente)
- EXTRACAO-CLINICA-SASI/BRIEFING.md (regras clinicas de extracao)
- EXTRACAO-CLINICA-SASI/docs/SOFA-RULESET.md (ruleset SOFA v1.0)