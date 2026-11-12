---
name: mestre-airtable
description: Use when designing or operating Airtable systems.
version: 0.1.0
author: Nícholas Nagaita, Hermes Agent
license: MIT
platforms: [linux, macos, windows]
metadata:
  hermes:
    tags: [Airtable, API, Bases, Automation, Data]
    related_skills: [airtable]
---

# Mestre Airtable

Projeta e opera bases Airtable como sistemas de dados, não como planilhas improvisadas. A skill comunitária `airtable` cobre comandos básicos; esta skill acrescenta arquitetura, segurança, migração, idempotência, automação e verificação.

## Quando usar

- Criar ou evoluir base, tabela, campo, view, interface ou automação.
- Integrar Airtable com app, Hermes, n8n ou webhook.
- Migrar dados para Airtable ou exportar backup.
- Diagnosticar autenticação, escopos, acesso a recursos, paginação e limites.
- Modelar Central do Trampo, tarefas, projetos, áreas, stack e pessoas.

Não usar para: guardar dado identificável de paciente, prescrição, exame, laudo ou qualquer PHI.

## Regras obrigatórias

1. Consulte Context7: `ctx7 library airtable "<tema>"`; depois `ctx7 docs /websites/airtable_developers_web_api "<um conceito>"`.
2. Leia o esquema antes de toda mutação. Use IDs de base/tabela/campo em automação; nomes são para humanos.
3. Token tem duas travas: escopo define o que faz; recurso define onde faz. HTTP 200 com `bases: []` significa token válido sem base visível.
4. Nunca imprima token. Guarde no cofre do ambiente com modo 0600; aplicativo recebe segredo apenas no servidor.
5. Use `PATCH`, operações em lotes de até 10 registros, paginação até sumir `offset`, espera em 429 e releitura pós-escrita.
6. Toda sincronização precisa ser idempotente: repetir não duplica. Defina chave externa única antes de importar.
7. Exclusão, troca destrutiva de tipo ou migração em massa exige contagem, plano de reversão e confirmação.
8. Sem fonte legível, campo/ID/limite é `null`; nunca invente.

## Procedimento

1. Leia `references/api-segura.md` e a referência do domínio.
2. Defina fonte da verdade. Um fluxo não pode ter Airtable e outro banco aceitando escrita concorrente sem regra explícita.
3. Modele uma entidade por tabela e ligações por registros relacionados. Evite texto duplicado.
4. Crie primeiro tabelas independentes; depois campos de ligação, lookup e rollup.
5. Crie views para execução humana: Hoje, Entrada, Esperando, Atrasadas, Projetos ativos e Arquivo.
6. Configure API e app somente no servidor. Navegador nunca recebe PAT.
7. Migre em modo sombra: exportar origem, transformar, upsert, comparar contagens e amostras, congelar escrita antiga, virar leitura/escrita, manter rollback.
8. Releia esquema e registros exatos. Registre IDs sem segredos.

## Referências

- `references/api-segura.md`: autenticação, API, limites, erros e verificação.
- `references/modelagem-central.md`: esquema recomendado da Central do Trampo.
- `references/integracoes-migracao.md`: app, n8n, webhooks, backup e virada.

## Erros comuns

- PATCH de choices em singleSelect/multipleSelects retorna "Changing a field's type or number precision is not currently supported" mesmo sem mudar tipo: a Meta API não aceita editar/remover choices existentes via API. Workaround: renomear o campo legado (PATCH name + description) e criar campo novo com o enum correto.
- POST /meta/bases/{}/tables/{}/fields espera UM objeto de campo (name/type/options) no corpo, NÃO {"fields":[...]} — array gera "parameter validation failed".
- POST de tabela aceita "fields":[...] no corpo (diferente do POST de campo); checkbox exige options {color,icon}; dateTime exige options.timeZone como string (ex. "America/Sao_Paulo").
- DELETE de tabela NÃO existe na Web API (404 NOT_FOUND na rota): tabela residual só se remove pela UI; renomear para "zzz DELETAR" e avisar o operador.
- CREATE-VIEW NÃO existe na Web API (POST .../tables/{}/views dá 422 "parameter validation failed" enganoso; llms.txt oficial só tem list-views/get-view-metadata/delete-view, e delete é Enterprise-only). View é artefato de UI: entregar roteiro de cliques com nome+filtros exatos ao operador.
- Endpoint de records é /v0/{baseId}/{tableId} (sem sufixo /records em GET/PATCH/DELETE por id); POST de criação é /v0/{baseId}/{tableId} com {"records":[...]}.
- Usar campo autonumérico como chave externa e descobrir que importação/recriação mudou tudo.
- Fazer tabela separada para subtarefa sem necessidade; relação consigo mesma costuma bastar.
- Esconder item em view e achar que a API não o vê.
- Fazer chamada Airtable direto do React e vazar PAT no pacote enviado ao navegador.
- Confiar em nome de campo e quebrar integração após renomear.
- Importar com `POST` sem chave de deduplicação.
- Tratar URL temporária de anexo como endereço permanente.
- Usar Airtable para dado clínico identificável sem contrato, base legal, controle e política aprovados.

## Verificação de alta

- Token testado sem exposição; base e esquema relidos.
- Todas as páginas consumidas; totais medidos.
- Criação repetida não duplica registros.
- Escrita relida por `rec...` e comparada campo a campo.
- App não contém PAT no cliente nem no repositório.
- Erros 401/403/422/429 têm tratamento explícito.
- Backup exportado e restauração ensaiada antes da virada.
- Nenhum dado de paciente entrou na base.
