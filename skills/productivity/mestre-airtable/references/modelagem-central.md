# Modelagem da Central do Trampo

## Escopo mínimo recomendado

Não transforme a base num prontuário universal. Comece com cinco tabelas:

| Tabela | Campo principal | Campos essenciais |
|---|---|---|
| Áreas | Nome | Chave externa, Cor, Ativa, Ordem |
| Projetos | Nome | Chave externa, Área, Estado, Próximo passo, Revisão, Arquivado |
| Tarefas | Tarefa | Chave externa, Projeto, Área, Estado, Prioridade, Prazo, Esperando por, Próxima ação, Feita em, Arquivada |
| Stack | Ferramenta | Chave externa, Área, Para quê, Onde mora, Saúde, Custo, Ciclo, Próxima revisão |
| Pessoas | Nome/apelido | Chave externa, Contexto, Área, Canal, Último contato, Arquivada |

`Chave externa` é texto estável para sincronização. O `rec...` do Airtable continua sendo o endereço interno.

## Relações

- Projeto → uma Área.
- Tarefa → zero/um Projeto e uma Área; Área pode ser lookup do Projeto quando houver projeto.
- Stack → uma ou mais Áreas apenas se isso for realmente útil.
- Pessoa → Áreas/projetos somente quando houver uso operacional.
- Subtarefa → ligação opcional da Tarefa com outra Tarefa; não abra sexta tabela antes da necessidade.

## Estados

Tarefa: `Entrada`, `Próxima`, `Fazendo`, `Esperando`, `Agendada`, `Algum dia`, `Feita`, `Cancelada`.

Projeto: `Ativo`, `Em espera`, `Concluído`, `Cancelado`.

Stack: `Ativo`, `Subutilizado`, `Avaliar`, `Abandonar`, `Duplicado`.

Mantenha poucos estados. Estado não é prioridade, nem área, nem etiqueta.

## Views para TDAH/dislexia

1. `Entrada`: tudo sem triagem.
2. `Hoje`: Próxima/Fazendo, vencida ou com prazo hoje; no máximo itens executáveis.
3. `Esperando`: bloqueadas, com pessoa e data de cobrança.
4. `Atrasadas`: prazo anterior a hoje e não concluídas.
5. `Projetos ativos`: próximo passo não vazio.
6. `Revisão semanal`: projetos/stack com revisão vencida.
7. `Arquivo`: feitas/canceladas/arquivadas.

A interface deve mostrar ação, não todos os campos. Cor é reforço, nunca único significado.

## Dados clínicos proibidos

Não inserir nome, iniciais, leito combinado com data, prontuário, CPF, exame, imagem, diagnóstico identificável, evolução, prescrição ou áudio de paciente. Tarefa clínica deve ser genérica: `revisar protocolo SOFA`, não `rever SOFA do João, leito 7`.

## Ordem de criação

1. Áreas e seus registros-semente.
2. Projetos, Stack e Pessoas.
3. Tarefas.
4. Campos de ligação/lookup/rollup.
5. Views.
6. Automação somente após o fluxo manual provar valor.

Sementes permitidas sem inventar pessoa/custo: áreas `Medicina`, `Dev`, `Estudo`, `Gestão`, `Sistema`; projetos e stack podem vir da origem existente. Campo desconhecido permanece vazio.
