# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## O que é este repositório

Não é um projeto de software (sem build, testes, lint, package manager). É o
script de carga (ETL) Qlik Sense da **Dunorte Distribuidora**, usado para
processar as campanhas comerciais de P&G / Mega Marcas. O conteúdo é:

- `CAMPANHAS DUNORTE DISTRIBUIDORA.QVS` — o script de carga Qlik em si
  (~3300 linhas, 3 abas `///$tab`). Este é o artefato central do repo.
- `DOCUMENTACAO CAMPANHAS DUNORTE.md` — documentação técnica detalhada do
  `.QVS`: pipeline, camadas, indicadores implementados, convenções,
  changelog de mudanças e pontos em aberto. **Leia este arquivo primeiro**
  antes de mexer no `.QVS` — evita reler o script inteiro a cada conversa e
  documenta decisões de negócio já tomadas (não óbvias a partir do código).
- `campanhas/*.MD` — regras de negócio por campanha/mês (ex.
  `CAMPANHAS MENSAL SET 2026.MD`, `CAMPANHAS MENSAL AGO 2026.MD`), definindo
  indicadores, metas, premiação e faixas de devolução para cada ciclo.
- `data/` — planilhas Excel de origem (ex. `CAMPANHAS_2026_08.xlsx`) usadas
  como referência para conferir nomes reais de abas/colunas citados na
  documentação.

Não há UI, app rodável ou testes automatizados aqui — a "execução" é feita
recarregando o `.QVS` no Qlik Sense/Qlik Cloud (fora deste repositório) e
validando manualmente o modelo de dados resultante.

## Workflow ao editar o `.QVS`

1. Ler `DOCUMENTACAO CAMPANHAS DUNORTE.md` (seções 2-5) para entender a
   camada afetada (Transformação, Modelagem ou Carregamento) e as
   convenções vigentes (chaves compostas com nome de campo único por join,
   indicadores que compartilham `TipoIndicador`/`Codigo`, caminhos
   centralizados em variáveis no topo do script).
2. Ao ligar um novo indicador a `TRF_BASE_RCA`, seguir o padrão existente:
   bloco `REALIZADO_<X>`/`META_<X>` com chave composta própria
   (`_Realizado<X>`/`_Meta<X>`), entrada no `Alt()` da etapa de unificação,
   e inclusão no `DROP FIELD` final. Usar o nome exato do `Indicador` como
   aparece na aba `INDICADORES` da planilha de origem (match textual —
   divergência de nome quebra o join silenciosamente, sem erro).
3. Cuidado com referências circulares no modelo de dados Qlik: nunca deixar
   duas tabelas que permanecem no modelo final compartilharem mais de um
   nome de campo em comum além da chave de junção pretendida (renomear
   cópias internas antes de expor/persistir um campo).
4. Depois de editar, atualizar `DOCUMENTACAO CAMPANHAS DUNORTE.md` (seção
   de changelog e, se aplicável, "Pontos em aberto") — é o mecanismo deste
   repo para persistir contexto entre sessões, já que não há testes
   automatizados que capturem intenção.
5. Validação real só acontece recarregando no Qlik Sense/Qlik Cloud
   (fora do Claude Code) — não afirmar que uma mudança "funciona" sem essa
   validação ter sido feita pelo usuário.

## Skill disponível

A skill `qlik-load-script` (`.claude/skills/qlik-load-script/`) cobre
sintaxe e padrões de script de carga Qlik — use-a para completar/estender
trechos `.qvs` ou tirar dúvidas de sintaxe Qlik.
