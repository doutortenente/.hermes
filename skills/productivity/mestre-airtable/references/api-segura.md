# API segura do Airtable

## Autenticação

Use Personal Access Token para automação da própria conta; OAuth para produto em que terceiros conectam suas contas. Chaves legadas `key...` morreram em 2024.

Escopos mínimos usuais:

- `data.records:read`
- `data.records:write`
- `schema.bases:read`
- `schema.bases:write` somente quando o agente altera estrutura

Além do escopo, conceda a base ou o workspace na área Access do token. Teste `GET https://api.airtable.com/v0/meta/bases`.

Criar base requer `POST /v0/meta/bases`, `schema.bases:write`, papel Creator no workspace e `workspaceId`. A API pública não lista workspaces de conta comum quando não há base acessível; obtenha `wsp...` na interface/token ou conceda o workspace inteiro ao token. Não adivinhe.

## Leituras

- Bases: `GET /v0/meta/bases`
- Esquema: `GET /v0/meta/bases/{baseId}/tables`
- Registros: `GET /v0/{baseId}/{tableIdOrName}`
- Registro: `GET /v0/{baseId}/{tableId}/{recordId}`

Use IDs estáveis em código. Campos vazios são omitidos na resposta; ausência do valor não prova ausência do campo.

## Escritas

- Criar: `POST /v0/{baseId}/{tableId}`
- Atualizar parcial: `PATCH /v0/{baseId}/{tableId}/{recordId}`
- Upsert: `PATCH /v0/{baseId}/{tableId}` com `performUpsert.fieldsToMergeOn`
- Excluir: `DELETE /v0/{baseId}/{tableId}/{recordId}`

Até 10 registros por chamada em lote. Limite documentado: 5 requisições/segundo por base e 50/segundo por usuário/service account com PAT. Em 429, respeite `Retry-After`; não faça rajada cega.

Listagem de registros pagina até 100 por página; siga `offset` até ele desaparecer. Listagem de bases pode devolver até 1000 por página.

## Idempotência

Crie campo texto `Chave externa` com valor determinístico, por exemplo `tarefa:<uuid-da-origem>`. Faça upsert por esse campo. Antes de usar múltiplos campos para merge, teste duplicidade; combinação ambígua falha ou produz resultado errado.

Após escrita:

1. Capture `recordId` retornado.
2. Faça GET desse registro.
3. Compare campos enviados, considerando coerção e campos calculados.
4. Em importação, releia por chave externa e confirme exatamente um registro.

## Erros

- 401: credencial ausente/inválida/regenerada.
- 403: escopo, recurso ou papel insuficiente.
- 404/`MODEL_ID_NOT_FOUND`: ID errado ou invisível ao token.
- 422: corpo, tipo de campo, fórmula ou opção incompatível.
- 429: limite; aguardar e repetir com backoff.

Leia o objeto `error`; não diagnostique apenas pelo código HTTP.

## Segurança

- PAT somente em servidor/cofre, nunca em `VITE_*`, `NEXT_PUBLIC_*`, bundle ou log.
- Permissão mínima e tokens separados por serviço quando a blast radius justificar.
- Preserve credenciais exatamente como fornecidas. Não rotacione, revogue, mova nem apague sem ordem explícita do operador.
- Não confie em view como controle de acesso; API autorizada pode ler a tabela.

Fontes:
- https://airtable.com/developers/web/api/authentication
- https://airtable.com/developers/web/api/rate-limits
- https://airtable.com/developers/web/api/list-records
- https://airtable.com/developers/web/api/create-base
