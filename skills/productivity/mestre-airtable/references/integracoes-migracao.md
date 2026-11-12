# Integrações e migração

## Aplicativo

O navegador chama uma rota de servidor do próprio app; essa rota chama Airtable com PAT. Em Vite/TanStack, nunca exponha PAT como variável `VITE_*`. Valide entrada no servidor, aplique timeout, serialize erro seguro e releia após escrita.

Contrato mínimo da camada de dados:

- `listTasks(cursor?, filters?)`
- `getTask(id)`
- `upsertTask(input, externalKey)`
- `updateTask(id, patch)`
- `archiveTask(id)`

Exclusão lógica (`Arquivada`) é padrão no app; exclusão física fica para manutenção confirmada.

## n8n

Credencial fica no cofre criptografado do n8n, não no JSON exportado do workflow. Use node Airtable quando ele cobre a operação; HTTP Request para Meta API, upsert ou recurso ausente. Controle 5 req/s/base, lote de 10 e repetição em 429.

Webhook do Airtable pode notificar alteração, mas o consumidor deve buscar payload/página, deduplicar e persistir cursor. Se não houver receptor público seguro, use polling incremental por `Last modified time` e chave externa. Não abra o n8n local para a internet sem autenticação/TLS.

## Migração de MongoDB para Airtable

1. Congelar o formato da origem e exportar JSON versionado.
2. Criar mapa explícito origem → campo Airtable; desconhecido = null.
3. Gerar chave externa estável para cada entidade.
4. Importar em lotes de até 10 com upsert.
5. Comparar contagem total por entidade, chaves ausentes/duplicadas e amostra campo a campo.
6. Ligar leitura do app ao Airtable mantendo escrita antiga bloqueada ou espelhada de modo controlado.
7. Testar criar, editar, concluir, arquivar e recarregar.
8. Virar Airtable para única fonte de escrita.
9. Manter export da origem para rollback durante janela definida; não apagar MongoDB no mesmo ato.

Duas fontes graváveis sem reconciliação é prontuário duplo: inevitavelmente diverge.

## Backup

- Exporte todas as páginas via API para JSON com esquema e data.
- Guarde IDs e chaves externas.
- Não trate CSV isolado como backup relacional completo.
- Ensaie restauração numa base descartável antes de depender do backup.
- URLs de anexo podem expirar; faça cópia do conteúdo quando anexos forem essenciais e permitido.

## Verificação final

- Token ausente do bundle (`dist`, `.output`, `.next`) e do Git.
- Chamadas do cliente atingem somente a rota do app.
- Uma repetição do mesmo pedido não duplica.
- Erro Airtable aparece ao usuário sem segredo e sem falsa confirmação.
- Após recarregar, estado persistido coincide com a API.

## Lições de modelagem e migração (classe SASI e afins)

1. **Interface móvel não é navegador.** Interfaces do Airtable em navegador móvel não são suportadas; uso móvel depende do aplicativo oficial. Impressão/PDF roda no computador. Projete views de trabalho para desktop; no celular aceite o app ou tela web clássica (grid/kanban).

2. **Campo primário em criação via API deve ser texto.** `singleLineText` (ou outro tipo aceito como primário) preenchido na importação. Fórmula como primário só funciona após a base existir; não passe fórmula no payload de `POST /v0/meta/bases` ou a criação falha.

3. **“Exatamente uma internação em curso” é regra errada para pacientes de alta.** Use “no máximo uma internação em curso por paciente”; pacientes sem episódio aberto são válidos. A regra vale para vigência, não para histórico.

4. **Relação histórica nula não se adivinha.** Se `internacao_id` (ou qualquer FK) vem nulo da origem:
   - Primeiro: use o episódio explícito quando houver.
   - Segundo: cruze paciente + data dentro do intervalo da internação; se houver **exatamente um** resultado, ligue e marque `Vínculo derivado = verdadeiro`.
   - Terceiro: zero ou múltiplos resultados → quarentena técnica para revisão humana. Nunca preencha por chute.

5. **Chave externa padronizada impede duplicação silenciosa.** Formato: `<tabela_supabase>:<uuid_origem>` (ex.: `evolucoes:550e8400-e29b...`). Upsert em `PATCH /v0/{baseId}/{tableId}` com `performUpsert.fieldsToMergeOn = ["Chave externa"]`. Antes de importar, gere a chave para cada linha da origem.

6. **Cobertura explícita de blocos clínicos críticos.** Em `evolucoes`, nove blocos JSON devem ter destino auditável individual, não apenas menção grupal:
   - `plantao` → Evoluções.Plantão
   - `riscos` → Evoluções.Riscos e contingências JSON + passagem quando estruturado
   - `problemas_ativos` → Problemas e Planos.Problema ativo + JSON legado
   - `impressao` → Evoluções.Impressão legada + fallback de Problemas e Planos
   - `conduta` → Evoluções.Conduta legada + fallback de Problemas e Planos
   - `condutas_sistemas` → Problemas e Planos.Ação 12–24 h (pareamento explícito)
   - `dvas` → Terapias classe Vasoativo + JSON legado
   - `sedativos` → Terapias classe Sedação/analgesia + JSON legado
   - `prescricao` → Evoluções.Prescrição legada + Terapias itens reconhecidos
   Tabela de cobertura no spec é porta de saída da modelagem.

7. **Script de verificação executável fecha o ciclo.** Um `.mjs` que lê o esquema real do Supabase (`src/types/supabase.ts`) e o spec Markdown, e falha (exit 1) quando:
   - Alguma tabela de origem sem destino
   - Algum bloco crítico sem mapeamento
   - Contagem de tabelas/Interfaces do Airtable diferente do esperado
   - Regras obrigatórias ausentes (chave externa, política de nulo, sem escrita simultânea, limitação móvel, sem dado real)
   Rode o script antes de declarar a especificação completa.

8. **Supabase e Airtable nunca escrevem juntos.** Regra de transição: escrita bloqueada no antigo antes de virar no novo; leitura espelhada se necessário, mas **não haverá escrita simultânea nos dois bancos**. Virada é atômica por domínio.

9. **Tabelas de catálogo não migram como editáveis.** `evento_tipo_ref`, `alert_rules`, `trend_rules`, `avisos_agentes`, `ingest_audit_log`, `memorias`, `auth.users` → backup técnico, automação, ou ignoradas. Não criam Interface clínica.

10. **Campos derivados/fotografias legados não viram edição paralela.** `dispositivos`, `severidade_visual`, `out_of_range_count`, `data_adm`, `status_leito`, leito/UTI/HD/gravidade/isolamento de `pacientes` → preservados no backup JSON e, quando o vínculo for inequívoco, entram na fotografia da Internação correspondente. Nunca duplicados como campos editáveis.