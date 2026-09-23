# Documentação Técnica — CAMPANHAS DUNORTE DISTRIBUIDORA.QVS

> Complementa `CAMPANHAS.MD` (regras de negócio) e `METRICAS E DIMENSOES.MD`
> (mapeamento de dimensões/métricas) com o que foi levantado a partir da
> leitura do script de carga real. Objetivo: servir de contexto para
> continuar o desenvolvimento sem precisar reler o `.QVS` inteiro (2008
> linhas) a cada conversa.

## 1. O que este script faz

Pipeline de transformação (ETL) que roda mensalmente/trimestralmente e
gera uma série de QVDs intermediários e finais usados pelos dashboards de
campanhas comerciais (P&G / Mega Marcas) da Dunorte Distribuidora. Não
tem UI — é só carga de dados.

Fontes de origem:
- Planilhas Excel em `lib://4_Plan/Metas P&G/` (`CAMPANHAS_AAAA_MM.xlsx`,
  `METAS_AAAA-MMM.xlsx`) — cadastros de metas, prêmios e faixas.
- QVDs já tratados em `lib://Carga_Duno/TESTE/TRANSFORMADOR/MES/COMERCIAL/MEGA MARCAS/`
  (`COMERCIAL_TRATADO_AAAA_MM.QVD`) — fato de vendas do mês.
- QVDs de outras esteiras (sem `/TESTE/`): Escolha Certa e Platinum Point,
  em `lib://Carga_Duno/TRANSFORMADOR/MES/COMERCIAL/MEGA MARCAS/`.
- Cadastros: `CAD_RCA.QVD`, `CAD_CLIENTE.QVD` em
  `lib://Carga_Duno/EXTRATOR/CADASTRO/`.

Destino final dos QVDs transformados:
`lib://Carga_Duno/TESTE/TRANSFORMADOR/MES/COMERCIAL/MEGA MARCAS/CAMPANHAS/`

## 2. Estrutura em 3 camadas (reorganizado em 2026-08-27)

O script tem 3 abas Qlik (`///$tab`), nesta ordem, cada uma rodando
integralmente antes da próxima:

### Aba "Transformação" — extração/transformação, gera QVDs intermediários
1. **Mappings de cadastro** (`MAP_SUP`, `MAP_RCA_NOME`, `MAP_RCA_SUP`,
   `MAP_CLIENTE_PRINC`) — carregados uma vez, usados via `ApplyMap()` em
   quase todo o resto do script.
2. **TRF_BASE_RCA_INDICADORES** — extrai a planilha CAMPANHAS (abas
   `PREM_RCA` + `INDICADORES`) e gera a base "catálogo" de indicadores
   (todo indicador que existe, independente de ter sido batido ou não).
3. **PREM_RANK_GILLETTE_RCA** — ranking de prêmio por posição/grupo
   (Gillette Trimestral, nível RCA) + mapeamento supervisor→grupo de
   rank. Renomeado de `PREM_RANK_GILLETE` (sem `_RCA`) em 2026-08-27
   quando a aba da planilha foi renomeada.
3.1. **PREM_RANK_GILLETTE_SUP** — mesma estrutura do item 3, mas ranking
   a nível de Supervisor (aba nova na planilha). Adicionado em
   2026-08-27, **só extração/transformação por enquanto** — ainda não
   ligado a nenhum cálculo de premiação (uso futuro, conforme pedido do
   usuário). QVDs: `TRF_PREM_RANK_GILLETTE_SUP_AAAA_MM.qvd`,
   `TRF_SUP_GRUPO_GILLETTE_SUP_AAAA_MM.qvd`.
4. **Fato Vendas Trimestre Fixo** (Seção + Departamento) — consolida os 3
   meses do trimestre calendário atual (T1-T4, alinhado ao trimestre
   civil).
4.1. **Fato Vendas Trimestre Móvel** (Seção + Departamento + Devolução
   RCA + Faturamento RCA + Positivação Seção/Departamento) — mesma
   dimensões/medidas do transformador de Vendas Mês Atual, mas somadas
   sobre uma **janela móvel de 3 meses** (mês atual + 2 anteriores,
   recalculada todo mês, diferente do Trimestre Fixo). Usa
   `AddMonths(MonthStart(Today()), offset)` para tratar corretamente a
   virada de ano (ex: Jan/2026 → janela Nov/2025-Jan/2026). Não inclui
   Mix Mínimo/Listing (indicadores de campanha mensal, não agregação de
   vendas).

   A extração bruta (`TMP_VENDAS_TRIMESTRE_MOVEL`) carrega o mesmo
   conjunto completo de campos do `TMP_VENDAS` do Mês Atual (`Cod
   Cliente Principal` via `ApplyMap`, `Cod Produto`, todas as
   quantidades em unidade/caixa de pedido e faturado) - ajustado pelo
   usuário em 2026-08-27 para ficar disponível para uso futuro (ex: um
   Mix Mínimo/Listing ou indicador a nível de produto sobre a janela
   móvel), mesmo que as 6 agregações atuais (itens 3-6 do bloco) só
   usem um subconjunto desses campos por enquanto.

   QVDs: `FATO_VENDAS_SECAO_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_VENDAS_DEPARTAMENTO_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_DEVOLUCAO_RCA_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_FATURAMENTO_RCA_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_POSITIVACAO_SECAO_TRIMESTRE_MOVEL_AAAA_MM.qvd`,
   `FATO_POSITIVACAO_DEPARTAMENTO_TRIMESTRE_MOVEL_AAAA_MM.qvd`.
   Adicionado em 2026-08-27 — **ainda não ligado ao `TRF_BASE_RCA`** (ver
   Pontos em aberto).
5. **Fato Vendas Mês Atual** (Seção + Departamento + RCA) — inclui
   Positivação de Clientes, o cálculo completo de **Mix Mínimo** (seção
   7 do script, ver item 4 abaixo) e de **Listing Iniciativas - 100%
   Carteira** (seção 8, ver item 4.1 abaixo).
6. **Escolha Certa** e **Platinum Point** (mês atual) — QVDs de KPI por
   RCA.
7. **Metas P&G** — lê `METAS_AAAA-MMM.xlsx` (abas `RCA_SEC`, `RCA_DEP`,
   `PLATINUM_POINT`, `ESCOLHA_CERTA`) e gera um QVD de meta por aba.
8. **PREM_RCA** — extrai a aba PREM_RCA (Cod RCA, Indicador, Ganho,
   Faixa) → `TRF_PREM_RCA_AAAA_MM.qvd`.
9. **RCA_DEVOL** — extrai as 4 faixas de % devolução/repasse por RCA →
   `TRF_RCA_DEVOL_AAAA_MM.qvd`.
10. **RCA_DEVOL_RANK** — extrai a faixa máxima de devolução para o
    ranking Gillette → `TRF_RCA_DEVOL_RANK_AAAA_MM.qvd`.

> Os itens 8-10 ficavam antes intercalados dentro da camada de cálculo
> (entre `BASE_RCA_INDICADORES_REALIZADO`, o cálculo de Ganho e o
> Ranking Gillette) — movidos para cá em 2026-08-27, ver Changelog.

### Aba "Modelagem" — onde as bases transformadas se ligam e os cálculos acontecem
1. **TRF_BASE_RCA** (montagem vertical) — uma tabela única
   Data+RCA+Indicador com Meta e Realizado juntos, feita via uma
   sequência de `LEFT JOIN`s com chaves textuais compostas
   (`_Meta`, `_MetaDep`, `_MetaPP`, `_MetaEC`, `_MetaMix`, `_MetaListing`...)
   — cada join usa um nome de campo de valor único para evitar o "bug de
   chave dupla" (ver seção 5). Resultado: `BASE_RCA_REAL_SECAO_DEP_MES_AAAA_MM.qvd`.
2. **BASE_RCA_INDICADORES_REALIZADO** — agrega tudo (Base + Escolha
   Certa + Platinum Point) no grão Data+RCA+Indicador, calcula
   `PercAtingimentoPedido`/`PercAtingimentoFaturado`.
3. **Cálculo do Ganho por RCA/Indicador** — lê o QVD do PREM_RCA (aba
   Transformação), cruza faixa de premiação com o % de atingimento
   realizado → `GanhoPedido`/`GanhoFaturado`.
4. **% Devolução RCA + Faixa de Repasse + Ganho Final** — lê os QVDs do
   RCA_DEVOL (aba Transformação), calcula % devolução, faixa de repasse
   aplicável e `GanhoFinalPedidoRca`/`GanhoFinalFaturadoRca`.
5. **Ranking Gillette Trimestral** — lê os QVDs do RCA_DEVOL_RANK e do
   PREM_RANK_GILLETE (aba Transformação); só entram no ranking RCAs com
   ≥100% de atingimento; RCAs com devolução acima da faixa máxima
   permitida são zerados no ranking (mas não no ganho por indicador).
   Resultado final: `PREMIACAO_GILLETTE_TRI`.

### Aba "Carregamento" — só organização/documentação (sem STORE novo)
Marca `BASE_RCA_INDICADORES_REALIZADO` (com Ganho) e
`PREMIACAO_GILLETTE_TRI` como as duas tabelas finais que carregam no
modelo de dados do app Qlik. Decisão do usuário: essa camada não grava
nada em QVD — as duas tabelas já são o resultado final da Modelagem e
ficam residentes, associadas ao restante do modelo.

## 3. Indicadores implementados no script (x regras de negócio)

| Indicador (CAMPANHAS.MD)              | Implementado no `.QVS`?                                  |
|----------------------------------------|-----------------------------------------------------------|
| Campanha Gillette (Trimestral)         | Sim — seções "Fato Vendas Trimestre Fixo" + Ranking Gillette |
| Campanha Mensal (indicadores gerais)   | Sim — Fato Vendas Mês Atual (Seção/Departamento/RCA)       |
| Regra de devolução (Gillette e Mensal) | Sim — blocos RCA_DEVOL e RCA_DEVOL_RANK                    |
| Mix Mínimo Contrato                    | Sim — seção 7, dentro do bloco Vendas Trimestre Móvel desde 2026-08-27 (ver item 4) |
| Escolha Certa Especial (R$20/positivação, só RCA) | **Parcial** — o script gera o KPI `FAIXA ESCOLHA CERTA` (soma de `QTD_ESCOLHA_CERTA` por faixa), mas o cálculo do valor R$20 por positivação não aparece neste `.QVS`. Verificar se é feito em outra camada (dashboard/expressão) ou está faltando. |
| Indicador Listing (`LISTING INICIATIVAS--100% CARTEIRA`) | **Sim (implementado e ligado ao TRF_BASE_RCA em 2026-08-26)** — seção 8 do script (cálculo) + seções 3.2/8.2 (Realizado/Meta ligados à base unificada), ver item 4.1 abaixo. Regra: cliente compra TODOS os produtos da aba `LISTING_PRODUTOS` do seu `RAMO`, cada produto exige qtd mínima em CAIXAS; RCA só ganha se 100% da carteira completar. Validado rodando no Qlik Sense. |
| Campanha PET (Supervisor Suzy 240+340→240340) | **Não encontrado neste arquivo.** Nem o indicador PET nem o código fictício 240340 aparecem no script lido. Pode estar em outro `.qvs`/tab do projeto Qlik. |
| Supervisor Gerson (73+74→7374)         | **Não encontrado neste arquivo.** Mesma observação acima. |

> Ação sugerida: confirmar com quem mantém o app Qlik se PET e os
> supervisores fictícios (7374/240340) vivem em outro script/tab antes de
> assumir que estão faltando.

## 4. Mix Mínimo — como funciona no script (seção 7, dentro do bloco "Vendas Trimestre Móvel")

Implementa a regra do `CAMPANHAS.MD`: cliente principal precisa positivar
um número mínimo de **grupos de produto** dentro da sua **Categoria**
(DPP/CC/HFS/NMR), e o RCA só ganha se **todos** os seus clientes
principais baterem (regra tudo-ou-nada).

> **Mudança de regra de negócio em 2026-08-27** (a pedido do usuário,
> depois de validar a nova estrutura da planilha):
> 1. **Realizado agora é medido sobre a janela móvel de 3 meses**, não
>    mais só o mês atual — o cálculo inteiro foi realocado do bloco
>    "Vendas Mês Atual" para o bloco "Vendas Trimestre Móvel", e passou
>    a consumir `TMP_VENDAS_TRIMESTRE_MOVEL` em vez de `TMP_VENDAS`.
> 2. **`QTDMIN` agora vem da aba `MIX_MIN`** (um valor por **cliente**,
>    aplicado igual a todos os grupos que ele precisa comprar) — antes
>    vinha da aba `MIXMIN_GRUPO_PRODUTO` (um valor por **grupo**, igual
>    pra todo cliente da mesma categoria). Confirmado direto na
>    planilha: `MIXMIN_GRUPO_PRODUTO` não tem mais coluna `QTDMIN`, e
>    `NOMEGRUPO`/`CATEGORIA` agora vêm preenchidos em toda linha do
>    grupo (antes só na 1ª linha, efeito de célula mesclada) — por isso
>    a extração desses dois campos trocou de
>    `WHERE Not IsNull(QTDMIN)` para `LOAD DISTINCT`.
> 3. **Chave composta Cliente+Filial** (`_ChaveClienteFilial = CodClientePrincipal & '|' & CodFilial`)
>    em toda comparação de compra — confirmado direto na planilha que o
>    mesmo `CODCLIPRINC` pode representar clientes diferentes em filiais
>    diferentes (ex: cliente `334788` aparece nas filiais `1` e `3` na
>    aba `MIX_MIN`). Usar só `CodClientePrincipal` misturaria as compras
>    desses dois clientes.
>
> **O contrato de saída não mudou**: mesmo nome de QVD
> (`FATO_MIXMINIMO_RCA_MES_AAAA_MM.qvd`), mesmos campos
> (`Indicador='MIX MINIMO'`, `TipoIndicador='DEPARTAMENTO'`,
> `PeriodoIndicador='MESATUAL'`, `ClasseIndicador='MANUAL'`, `Meta=1`).
> A Modelagem (`REALIZADO_MIXMINIMO`/`META_MIXMINIMO`) não precisou de
> nenhuma alteração.

Passo a passo (nomes de tabela no script):
1. `TRF_MIX_MIN` — Cliente Principal × Filial × Categoria × Objetivo ×
   **QtdMin** (agora por cliente), vindo da aba `MIX_MIN`, mais o campo
   `_ChaveClienteFilial`.
2. `TRF_MIXMIN_PRODUTO_GRUPO` / `TRF_MIXMIN_GRUPO` — mapeamento
   Produto→**NomeGrupo** e NomeGrupo→Categoria (sem QtdMin nem
   CodGrupo — a chave de negócio do grupo é o nome, não o código
   numérico, que não é único entre categorias), vindo da aba
   `MIXMIN_GRUPO_PRODUTO`.
3. `MIXMIN_ELEGIVEL` — join Cliente(+Filial)×**NomeGrupo** por Categoria
   (todo cliente pareado com todos os grupos da própria categoria);
   `QtdMin` já vem do próprio cliente, sem precisar de join extra.
4. `TEMP_VENDAS_MIXMIN` → `COMPRA_CLIENTE_GRUPO` — quantidade realmente
   comprada por **Cliente+Filial** + **NomeGrupo**, **na janela móvel de
   3 meses** (só conta valor > 0).
5. `MIXMIN_GRUPO_FLAG` — grupo positivado se
   `QtdComprada >= QtdMin (do cliente)`.
6. `MIXMIN_CLIENTE_FLAG` — cliente (+filial) atingiu objetivo se
   `GruposPositivados >= Objetivo`.
7. `FATO_MIXMINIMO_RCA` — `Min()` do flag por cliente+filial agregado
   por RCA: só fica 1 (100%) se **todos** os clientes (em qualquer
   filial) do RCA bateram.

QVD final: `FATO_MIXMINIMO_RCA_MES_AAAA_MM.qvd`. Esse mesmo QVD alimenta
tanto o Realizado quanto a Meta do indicador "MIX MINIMO" na montagem de
`TRF_BASE_RCA` (é ligado duas vezes, com chaves `_RealizadoMix`/`_MetaMix`
separadas).

## 4.1 Listing Iniciativas - 100% Carteira — como funciona no script (seção 8, logo após o Mix Mínimo)

Regra: cliente principal precisa comprar **todos** os produtos da aba
`LISTING_PRODUTOS` que pertencem ao seu **Ramo** (Ramo do produto vs.
Ramo do cliente, este último vindo da aba `MIX_MIN`), cada produto
exigindo uma quantidade mínima em **caixas** (`QTDMIN_CX`). RCA só ganha
se **100% da carteira** (todos os seus clientes principais) completar —
regra tudo-ou-nada, igual ao Mix Mínimo.

Headers reais confirmados na planilha `CAMPANHAS_AAAA_MM.xlsx`:
- `MIX_MIN`: `DATA, COD_SUP, COD_RCA, CODCLIPRINC, CLASSE, RAMO, CATEGORIA, OBJETIVO`
- `LISTING_PRODUTOS`: `DATA, COD_PRODUTO, NOMEPRODUTO, RAMO, QTDMIN_CX`

Passo a passo (nomes de tabela no script, seção 8.1 a 8.7):
1. `LISTING_PARTICIPANTES` — Cliente Principal × RCA × Ramo, mesma base
   de participantes da aba `MIX_MIN` (reload minimalista — só os 3
   campos necessários, já que `TRF_MIX_MIN` da seção 7 não preserva o
   campo `Ramo` até aqui).
2. `TRF_LISTING_PRODUTOS` — Produto × Ramo × QtdMinCx, vindo da aba
   `LISTING_PRODUTOS`.
3. `LISTING_ELEGIVEL` — join Cliente×Produto por Ramo (todo cliente
   pareado com todos os produtos obrigatórios do próprio ramo).
4. `TEMP_VENDAS_LISTING` → `COMPRA_CLIENTE_PRODUTO_LISTING` — quantidade
   em **caixas** realmente comprada por Cliente Principal + Produto (só
   conta valor > 0). Usa `QtdCaixaPedidoLiquido`/`QtdCaixaFaturadoLiquido`
   de `TMP_VENDAS` (não as quantidades em unidades, usadas pelo Mix
   Mínimo) — decisão confirmada com o usuário, já que `QTDMIN_CX` é
   literalmente "quantidade mínima de caixas".
5. `LISTING_PRODUTO_FLAG` — produto positivado se
   `QtdCaixaComprada >= QtdMinCx`.
6. `LISTING_CLIENTE` — cliente completou a lista se `Min()` de todos os
   flags de produto do seu Ramo = 1 (um produto faltando já derruba).
7. `FATO_LISTING_RCA` — `Min()` do flag por cliente agregado por RCA:
   só fica 1 (100%) se **toda a carteira** do RCA completou.

QVD final: `FATO_LISTING_RCA_MES_AAAA_MM.qvd`, com
`Indicador='LISTING INICIATIVAS--100% CARTEIRA'`,
`TipoIndicador='DEPARTAMENTO'`, `ClasseIndicador='MANUAL'`, `Meta=1`
fixo — mesmo padrão de campos do Mix Mínimo.

> **Atenção ao nome exato do indicador**: a linha de catálogo na aba
> `INDICADORES` da planilha `CAMPANHAS_AAAA_MM.xlsx` usa literalmente
> `LISTING INICIATIVAS--100% CARTEIRA` (dois hífens, sem espaço, "100%"
> colado em "CARTEIRA") — **não** `LISTING INICIATIVAS`. Como a chave
> composta (`_RealizadoListing`/`_MetaListing`) exige match textual
> exato do campo `Indicador` contra essa linha do catálogo, usar o nome
> "limpo" quebraria o join silenciosamente (Meta/Realizado ficariam
> `Null` sem erro nenhum). Confirmado lendo `CAMPANHAS_2026_08.xlsx`
> diretamente (aba `INDICADORES`: `CODIGO=3, TIPO=DEPARTAMENTO,
> CLASSE=MANUAL, PERIODO=MESATUAL`).

**Ligado ao `TRF_BASE_RCA`** (2026-08-26, seções 3.2 e 8.2 do script,
mesmo padrão do Mix Mínimo):
- `REALIZADO_LISTING` — lê `FATO_LISTING_RCA_MES_$(vDataCarg).qvd`,
  chave `_RealizadoListing`, valores renomeados
  `ValorPedidoLiquidoListing`/`ValorFaturadoLiquidoListing`.
- `META_LISTING` — mesma fonte, chave `_MetaListing`, valor
  `MetaListing` (= `Meta`, sempre 1).
- Etapa 9 (UNIFICAÇÃO) atualizada: `MetaListing` entrou no `Alt()` de
  `Meta`, `ValorPedidoLiquidoListing`/`ValorFaturadoLiquidoListing`
  entraram nos `Alt()` de `ValorPedidoLiquidoFinal`/
  `ValorFaturadoLiquidoFinal`, e todos os campos intermediários foram
  incluídos no `DROP FIELD` final.

Resultado: "LISTING INICIATIVAS--100% CARTEIRA" agora aparece
normalmente em `BASE_RCA_REAL_SECAO_DEP_MES_AAAA_MM.qvd`, com Meta e
Realizado, junto dos demais indicadores.

## 5. Convenções importantes do script (para não quebrar nada em manutenção)

- **Chaves compostas com nome de campo único por join**: toda vez que um
  novo bloco de Meta é `LEFT JOIN`ado em `TRF_BASE_RCA`, o campo de valor
  vem renomeado (`MetaSecaoFat`, `MetaDepFat`, `MetaPlatinumPoint`...).
  Isso evita o "bug de chave dupla" do Qlik: se dois LEFT JOINs
  sucessivos compartilhassem o mesmo nome de campo de valor, esse campo
  viraria parte da chave de match do segundo join sem querer. Ao criar
  um novo bloco de Meta/Realizado, sempre dar um nome de campo de valor
  exclusivo antes do join.
- **Indicadores que dividem `TipoIndicador`/`Codigo`**: Platinum Point,
  Escolha Certa e Mix Mínimo compartilham `TipoIndicador='DEPARTAMENTO'`
  e `Codigo='3'` na base de indicadores — por isso o nome do `Indicador`
  entra na fórmula da chave (`_MetaPP`, `_MetaEC`, `_MetaMix`) para não
  colidir.
- **Caminhos**: `vPathMetasPG`, `vPathCampanhas`, `vPathVendasMes` foram
  centralizados no topo do script (ver Changelog). Escolha Certa e
  Platinum Point são exceção — leem de fora de `/TESTE/`
  (`lib://Carga_Duno/TRANSFORMADOR/...`), propositalmente diferente do
  resto.
- **Padrão de bloco opcional**: quase todo bloco de extração é envolvido
  em `IF Not IsNull(FileSize(...)) THEN ... ELSE TRACE aviso ENDIF` —
  se o arquivo de origem do mês/trimestre ainda não existe, o bloco é
  pulado sem quebrar o restante da carga (útil pra rodar o script antes
  do fechamento do mês).
- **Vigência sempre = mês/trimestre atual**: todas as datas são derivadas
  de `Today()`. Não há parâmetro para reprocessar um mês passado sem
  editar o script manualmente — se isso virar necessidade recorrente,
  vale adicionar uma variável de override no topo.

## 6. Changelog de otimizações aplicadas (2026-08-26)

1. **Eliminado round-trip de disco no bloco Mix Mínimo.** `TRF_MIX_MIN`,
   `TRF_MIXMIN_PRODUTO_GRUPO` e `TRF_MIXMIN_GRUPO` deixaram de ser
   gravadas em QVD e imediatamente relidas do disco — agora permanecem
   residentes em memória e são reaproveitadas direto pelas seções
   seguintes. Os QVDs entregáveis continuam sendo gerados normalmente.
2. **Caminhos de origem/destino centralizados** em três constantes no
   topo do script (`vPathMetasPG`, `vPathCampanhas`, `vPathVendasMes`),
   substituindo ~10 declarações `SET` idênticas espalhadas pelo arquivo.
   Os blocos de Escolha Certa/Platinum Point mantiveram seu path próprio
   (origem diferente, sem `/TESTE/`).
3. **Corrigido comentário desatualizado** na seção "BASE DE INDICADORES
   (bruto)" que descrevia uma conversão texto→número (vírgula decimal)
   que não existe em nenhum ponto do código — os valores já chegam
   numéricos (produzidos por `Sum()` mais acima no próprio script).
   Substituído por nota + `// TODO: verify` para o caso da fonte mudar.

Nenhuma lógica de negócio, nome de campo ou QVD de saída foi alterado —
mudanças puramente estruturais/de I/O e de documentação. **Ainda não
validado rodando no Qlik Cloud** — validar antes de subir para produção.

**2026-08-26 (2ª leva):** Implementado o cálculo do Realizado da campanha
**Listing Iniciativas--100% Carteira** (seção 8, novo bloco — ver item
4.1). QVD gerado: `FATO_LISTING_RCA_MES_AAAA_MM.qvd`. Validado rodando
no Qlik Sense (script completo, sem erros).

**2026-08-26 (3ª leva):** Ligado o `FATO_LISTING_RCA` ao encadeamento de
`LEFT JOIN`s de `TRF_BASE_RCA` (novas seções 3.2 `REALIZADO_LISTING` e
8.2 `META_LISTING`, mesmo padrão do Mix Mínimo) — Listing Iniciativas
agora aparece na tabela unificada `BASE_RCA_REAL_SECAO_DEP_MES_AAAA_MM.qvd`
junto com Meta e Realizado. Corrigido também um mismatch de nome:
o valor correto do campo `Indicador` é `LISTING INICIATIVAS--100%
CARTEIRA` (conforme a linha de catálogo real na aba `INDICADORES`), não
`LISTING INICIATIVAS` — usar o nome errado quebraria o join
silenciosamente. **Validado rodando no Qlik Sense em 2026-08-27
(junto com a reorganização abaixo) — valores batendo.**

**2026-08-27:** Reorganizado o script inteiro em 3 abas Qlik (`///$tab`)
— **Transformação**, **Modelagem**, **Carregamento** — ver seção 2.
Os blocos de extração PREM_RCA, RCA_DEVOL e RCA_DEVOL_RANK, que antes
ficavam intercalados dentro da camada de cálculo, foram movidos
(cut+paste puro, sem alterar nenhuma linha de lógica) para o final da
aba Transformação. Mudança é 100% estrutural — confirmado que os 3
blocos são autocontidos (variáveis próprias baseadas em `Today()`, sem
depender de estado deixado por outros blocos) e que seus consumidores
na Modelagem continuam funcionando porque variáveis Qlik são globais e
persistem independente de onde o bloco está fisicamente no arquivo,
desde que Transformação rode inteira antes de Modelagem. Aba
Carregamento é só documentação (nenhum STORE novo, por decisão do
usuário). **Validado rodando no Qlik Sense — script completo, sem
erros, valores batendo com a versão anterior.**

**2026-08-27 (2ª leva):** Criado o transformador **Vendas Trimestre
Móvel** (aba Transformação, logo após o Trimestre Fixo) — mesmas 6
saídas do transformador de Vendas Mês Atual (Seção, Departamento,
Devolução RCA, Faturamento RCA, Positivação Seção, Positivação
Departamento), somadas sobre a janela móvel de 3 meses (mês atual + 2
anteriores). Ver item 4.1 da seção 2. A extração bruta foi depois
ajustada pelo usuário para carregar o conjunto completo de campos do
Mês Atual (`Cod Cliente Principal`, `Cod Produto`, quantidades
unidade/caixa) para uso futuro. **Validado rodando no Qlik Sense —
script editor carregou normalmente, sem erros, com as 3 abas
(Transformação/Modelagem/Carregamento) reconhecidas corretamente.**
Ainda não ligado ao `TRF_BASE_RCA` — ver Pontos em aberto.

**2026-08-27 (3ª leva):** A planilha `CAMPANHAS_AAAA_MM.xlsx` mudou de
estrutura: a aba `PREM_RANK_GILLETE` foi renomeada para
`PREM_RANK_GILLETTE_RCA`, e uma aba nova `PREM_RANK_GILLETTE_SUP` foi
adicionada (mesmo layout, ranking a nível de Supervisor em vez de RCA).
Script atualizado: `table is PREM_RANK_GILLETE` → `PREM_RANK_GILLETTE_RCA`
nos dois LOADs existentes (senão o reload quebraria por não achar a
aba antiga), e criado um novo bloco espelhado para
`PREM_RANK_GILLETTE_SUP` (`TRF_PREM_RANK_GILLETTE_SUP`/
`TRF_SUP_GRUPO_GILLETTE_SUP`) — só extração por enquanto, sem ligação
com cálculo de premiação (uso futuro, a pedido do usuário). Ver item
3.1 da seção 2.

> `COD_FILIAL`/`QTDMIN` novos na aba `MIX_MIN` (mencionados acima):
> **uso confirmado e implementado em 2026-08-27 (4ª leva)**, ver abaixo.

**2026-08-27 (4ª leva):** Mudança de regra de negócio no indicador
**Mix Mínimo**, a pedido do usuário — ver detalhes completos na seção 4:
(1) o cálculo inteiro foi **movido do bloco "Vendas Mês Atual" para
"Vendas Trimestre Móvel"** (Realizado agora soma 3 meses, não 1);
(2) `QTDMIN` passou a vir da aba `MIX_MIN` (por cliente) em vez de
`MIXMIN_GRUPO_PRODUTO` (por grupo) — confirmado direto na planilha que
essa coluna foi removida de `MIXMIN_GRUPO_PRODUTO`; (3) introduzida a
chave composta `_ChaveClienteFilial` em toda comparação de compra, já
que o mesmo `CODCLIPRINC` pode repetir em filiais diferentes
(confirmado: cliente `334788` aparece nas filiais `1` e `3`). O
contrato de saída (nome do QVD, campos) não mudou, então a Modelagem
não precisou de nenhum ajuste.

**2026-08-27 (5ª leva) — bug corrigido, achado durante validação pelo
usuário:** RCA `6026` aparecia com `PercAtingimentoFaturado` somando
**2** em vez de 1 para o indicador MIX MINIMO (sinal de duplicidade).
Causa raiz confirmada direto na planilha: **`COD_GRUPO` na aba
`MIXMIN_GRUPO_PRODUTO` não é um id único global — ele se repete por
Categoria** (Grupo 1 existe em DPP, HFS, C&C e NRM, cada um um produto
diferente). 258 dos 332 produtos estão associados a mais de um par
(Grupo, Categoria) — ex: produto `219481` é `(Grupo 1, DPP)` e também
`(Grupo 1, HFS)`. `TRF_MIXMIN_PRODUTO_GRUPO` não carregava `Categoria`,
então o cruzamento de vendas→grupo (seção 7.5) batia em **todos** os
grupos com aquele número, de categorias não relacionadas, inflando a
quantidade comprada. Corrigido: `TRF_MIXMIN_PRODUTO_GRUPO` agora carrega
`Categoria`; criado `MAP_CLIENTE_CATEGORIA` (Cliente+Filial→Categoria,
a partir de `TRF_MIX_MIN`) para trazer a categoria do próprio cliente
para `TEMP_VENDAS_MIXMIN`; o cruzamento agora usa a chave composta
`CodProduto + Categoria`.

> **Ajuste adicional no mesmo dia, a pedido do usuário**: mesmo com
> `Categoria` no cruzamento, agregar por `CodGrupo` ainda dependia de um
> código numérico que **não é o identificador de negócio real do grupo**
> — confirmado na planilha que o mesmo `NOMEGRUPO` pode ter `CodGrupo`
> diferente em categorias diferentes (103 dos 186 nomes de grupo caem
> nesse caso). Como `NOMEGRUPO` já vem preenchido em toda linha de
> produto da aba (não só na primeira linha do grupo), a agregação inteira
> foi trocada de `CodGrupo` para `NomeGrupo` — `TRF_MIXMIN_PRODUTO_GRUPO`
> e `TRF_MIXMIN_GRUPO` não carregam mais `COD_GRUPO`, e todas as chaves
> de join/agregação da seção 7 (7.4 `MIXMIN_ELEGIVEL`, 7.5
> `COMPRA_CLIENTE_GRUPO`, 7.6 `LEFT JOIN`) usam `NomeGrupo` no lugar de
> `CodGrupo`. Confirmado que, dentro da mesma Categoria, `CodGrupo` e
> `NomeGrupo` já eram 1:1 (0 inconsistências), então essa troca não muda
> o resultado matemático da versão anterior — só remove a dependência de
> um código cuja unicidade só valia por acidente do agrupamento por
> Categoria, deixando o identificador de negócio real como chave.
> **Ainda não validado no Qlik Sense após esta correção.**

> **Segundo problema encontrado durante a mesma investigação, NÃO
> corrigido no script** (usuário optou por corrigir na planilha): a
> categoria de cliente aparece grafada como `NMR` na aba `MIX_MIN` mas
> como `NRM` na aba `MIXMIN_GRUPO_PRODUTO` (letras trocadas). Como a
> seção 7.4 faz um `JOIN` (inner join) por `Categoria`, clientes com
> categoria `NMR` não encontram nenhum grupo correspondente e **somem
> inteiramente** de `MIXMIN_ELEGIVEL` — o que pode fazer o RCA inteiro
> sumir do indicador Mix Mínimo (se todos os clientes dele forem `NMR`)
> ou dar resultado incorreto (se o RCA tiver clientes mistos, o cliente
> `NMR` é ignorado na regra tudo-ou-nada). **Ação: corrigir a grafia na
> planilha** (unificar `NMR`/`NRM` para o mesmo valor nas duas abas)
> antes da próxima carga.

**2026-08-27 (6ª leva) — bug corrigido, achado após o usuário confirmar
que o "2" persistia mesmo depois da correção acima e do ajuste
NMR/NRM na planilha:** o problema não estava mais no cálculo do
Mix Mínimo em si (dimensão simples já mostrava `PercAtingimentoFaturado
= 1` corretamente para o RCA `6026`) — era uma **referência circular na
Modelagem**, visível no diagrama do modelo de dados do Qlik: `Sum()`
sobre o campo duplicava o valor, mas o campo como dimensão não mudava
(sintoma clássico de loop de associação). Duas causas encontradas na
seção "PREMIAÇÃO GILLETTE TRIMESTRAL":
1. **`BASE_GILLETTE_TRI` (que vira `PREMIACAO_GILLETTE_TRI`) carregava
   `CodSupervisor` e `PercAtingimentoFaturado`** com os mesmos nomes que
   já existem em `BASE_RCA_INDICADORES_REALIZADO`** — como as duas
   tabelas permanecem no modelo final, isso as ligava por **três campos
   simultâneos** (`CodRca` + `CodSupervisor` + `PercAtingimentoFaturado`),
   o que o Qlik também trata como referência circular entre duas
   tabelas. **Corrigido**: renomeados para `CodSupervisorGillette` e
   `PercAtingimentoFaturadoGillette` dentro de `BASE_GILLETTE_TRI`
   (e a referência em `RANKING_GILLETTE` atualizada) — agora `CodRca` é
   o único campo em comum entre as duas tabelas, como deveria ser.
2. **`BASE_GILLETTE_TRI` também copiava (via `ApplyMap`) o valor de
   devolução de `GANHO_FINAL_RCA` usando o MESMO nome de campo**
   `PercDevolucaoRcaFaturado` que já existe em `GANHO_FINAL_RCA` — isso
   ligava `GANHO_FINAL_RCA` e `PREMIACAO_GILLETTE_TRI` diretamente, e
   como as duas também se ligam a `BASE_RCA_INDICADORES_REALIZADO` via
   `CodRca`, fechava um loop de 3 tabelas (exatamente o mostrado no
   diagrama do modelo de dados que o usuário enviou).

   **Correção original errada (2026-08-27, corrigida na hora):** cheguei
   a colocar um `DROP TABLE GANHO_FINAL_RCA;` achando que era uma tabela
   intermediária esquecida — **estava errado**: `GANHO_FINAL_RCA` é a
   base de premiação final do RCA, usada pelo usuário para validar
   contra o total de devolução dele, e precisa continuar no modelo.
   Revertido o `DROP` imediatamente após o usuário apontar o problema.

   **Correção final**: renomeada a cópia dentro de `BASE_GILLETTE_TRI`
   para `PercDevolucaoRcaFaturadoGillette` (só usada internamente para o
   cálculo do ranking Gillette) — `GANHO_FINAL_RCA` continua intacta no
   modelo, com todos os seus campos (incluindo o `PercDevolucaoRcaFaturado`
   original), ligada a `BASE_RCA_INDICADORES_REALIZADO` e a
   `PREMIACAO_GILLETTE_TRI` **só por `CodRca`** (formato estrela, sem
   ciclo). **Ainda não validado no Qlik Sense.**

## 7.1 Setembro 2026 — Premiação do Supervisor (implementada)

Confirmado lendo `data/CAMPANHAS_2026_09.xlsx` que o pipeline de
Supervisor (blocos `PREM_SUP`/`SUP_DEVOL`/`SUP_DEVOL_RANK` na
Transformação + `BASE_SUP_INDICADORES_REALIZADO`/`GANHO_FINAL_SUP`/
`RANKING_GILLETTE_SUP`/`PREMIACAO_GILLETTE_TRI_SUP` na Modelagem, ver
seção 2) já estava largamente implementado, ao contrário do que a
versão anterior desta seção dizia.

**Correção importante (2026-09-14, achado pelo usuário revendo o
`.QVS`)**: a afirmação original desta seção — de que "qualquer
indicador novo sobe pro Supervisor automaticamente" — estava **errada**
para indicadores que só existem na aba `PREM_SUP`, sem nenhuma linha
correspondente em `PREM_RCA`. O motivo: `BASE_SUP_INDICADORES_REALIZADO`
(PARTE 1) reagrega `BASE_RCA_INDICADORES_REALIZADO` por `CodSupervisor`
— mas essa tabela só tem linhas para indicadores que passaram pelo
catálogo `TRF_BASE_RCA_INDICADORES`, que por sua vez só é gerado a
partir da aba `PREM_RCA`. Se um indicador **nunca** aparece em
`PREM_RCA` (só em `PREM_SUP`), nenhum RCA nunca teve uma linha desse
indicador — e reagregar "o que o RCA tem" por Supervisor não produz
nada para reagregar. Confirmado comparando as duas abas de
`CAMPANHAS_2026_09.xlsx`: `FATURAMENTO TOTAL RR`, `FATURAMENTO TOTAL
AM` e `RENTABILIDADE 9%` (todos Classe `FATURAMENTO`, Tipo
`DEPARTAMENTO`, Período `MESATUAL`) só existem em `PREM_SUP` — os
demais indicadores de Supervisor (`PANTENE*`, `HEAD E SHOULDERS`,
`ORAL B*`, `DESODORANTE`, `NEXGARD*`, `FROTLINE`, `TERAPEUTICOS`,
`POSITIVAÇÃO BRFOODS`/`PROCTER`, `FAIXA ESCOLHA CERTA`, `PLATINUM
POINTS`, `GILLETTE TRIMESTRAL`) também existem em `PREM_RCA`, então
esses sim já funcionavam via reagregação.

**Correção aplicada no `.QVS`** (aba Transformação + Modelagem):
- Novo catálogo `TRF_BASE_SUP_INDICADORES` (Transformação, logo após o
  catálogo `TRF_BASE_RCA_INDICADORES` existente) — mesma lógica, mas lê
  a aba `PREM_SUP` em vez de `PREM_RCA`, então cobre TODO indicador de
  Supervisor, inclusive os exclusivos.
- Novo bloco **PARTE 1.1** na Modelagem (logo após a PARTE 1 de
  `BASE_SUP_INDICADORES_REALIZADO`): usa `Not Exists(IndicadorSup,
  Indicador)` para achar dinamicamente quais indicadores do catálogo de
  Supervisor ainda não têm nenhuma linha vinda do RCA, e calcula o
  Realizado/Meta desses diretamente — reagregando
  `FATO_VENDAS_DEPARTAMENTO_MES` (Realizado, via `ApplyMap('MAP_RCA_SUP', ...)`)
  e `TRF_METAS_DEPARTAMENTO_MES` (Meta, usando o campo `CodSupervisor`
  próprio da aba `RCA_DEP`) até o nível de Supervisor, depois
  `CONCATENATE`ado em `BASE_SUP_INDICADORES_REALIZADO` antes do cálculo
  de Ganho (PARTE 2) — que já funciona sem alteração, pois
  `TRF_PREM_SUP` cobre esses indicadores normalmente.
- **Limitação atual, documentada no comentário do bloco**: só cobre
  `TipoIndicador='DEPARTAMENTO'` + `PeriodoIndicador='MESATUAL'` (único
  caso que aparece hoje para indicadores exclusivos de Supervisor). Se
  aparecer um indicador exclusivo de outro Tipo/Período, o bloco precisa
  ganhar mais uma fonte (mesmo padrão do `CONCATENATE` em
  `REALIZADO_SECAO` do RCA). **Ainda não validado no Qlik Sense.**

Confirmado direto na aba `INDICADORES` (catálogo, 50 linhas) e
`PREM_SUP` (premiação por supervisor, 87 linhas) de
`CAMPANHAS_2026_09.xlsx` que os seguintes indicadores **já funcionam**
(uns via reagregação — PARTE 1 —, outros via o novo bloco — PARTE
1.1), assim que a planilha e os QVDs de Metas do mês estiverem
disponíveis: `FATURAMENTO TOTAL RR`, `FATURAMENTO TOTAL AM`,
`RENTABILIDADE 9%` (PARTE 1.1, novos), `PANTENE SHAMPOO E
CONDICIONADOR`, `PANTENE TRATAMENTO`, `PANTENE KIT`, `HEAD E
SHOULDERS`, `ORAL B CREME`, `ORAL B ESCOVAS`, `DESODORANTE`, `NEXGARD`,
`NEXGARD SPECTRA`, `NEXGARD COMBO`, `FROTLINE`, `TERAPEUTICOS`,
`POSITIVAÇÃO BRFOODS`, `POSITIVAÇÃO PROCTER`, `GILLETTE TRIMESTRAL`
(PARTE 1, já existentes).

Os supervisores fictícios **Suzy (COD_SUP `240340`, campanha PET:
NEXGARD/NEXGARD SPECTRA/NEXGARD COMBO/FROTLINE/TERAPEUTICOS/POSITIVAÇÃO
BRFOODS)** e **Gerson (COD_SUP `7374`)** — que a seção 7 (antiga)
listava como "não encontrados" — **já aparecem em `PREM_SUP` de
setembro**. Isso resolve esse ponto em aberto, desde que o cadastro
`CAD_RCA` associe os RCAs certos a esses 2 códigos de supervisor (dado
de cadastro, fora deste `.QVS` — não verificável aqui).

`CATFOCO ALWAYS` e `CATFOCO PAMPERS` (Classe `KPI`, igual Escolha
Certa/Platinum Point) não tinham pipeline pronto nesta data — a regra de
"cliente positivado numa categoria foco" não existia em nenhuma aba de
`CAMPANHAS_2026_09.xlsx`. **Implementado em 2026-09-15, ver seção 7.3.**

## 7.2 Setembro 2026 — Base de Realizado do Supervisor reescrita no grão do Supervisor

**Sintoma (achado pelo usuário, 2026-09-15, comparando `PREM_SUP` de
`CAMPANHAS_2026_09.xlsx` com a tabela `REALIZADO SUPERVISOR` no Qlik)**:
o supervisor `29` tem 7 indicadores na planilha (`CATFOCO ALWAYS`,
`CATFOCO PAMPERS`, `POSITIVAÇÃO PROCTER`, `FAIXA ESCOLHA CERTA`,
`PLATINUM POINTS`, `GILLETTE TRIMESTRAL`, `FATURAMENTO TOTAL AM`), mas o
Qlik mostrava 9 linhas para ele — 7 delas erradas (`DESODORANTE`, `HEAD E
SHOULDERS`, `ORAL B CREME`, `ORAL B ESCOVAS`, `PANTENE KIT`, `PANTENE
SHAMPOO E CONDICIONADOR`, `PANTENE TRATAMENTO`, todas com `FaixaSup`/
`GanhoSup` nulos) e 5 dos 7 verdadeiros ausentes.

**Causa raiz (única, com dois sintomas opostos)**: a `PARTE 1` da
Modelagem construía a base do Supervisor **reagregando
`BASE_RCA_INDICADORES_REALIZADO` por `CodSupervisor`** — ou seja, o
Supervisor só enxergava o que os RCAs dele já apuravam:

- **Sobra** — herdava todo indicador que qualquer RCA do time dele apura,
  mesmo os que não estão no `PREM_SUP` dele. O `LEFT JOIN` com
  `TRF_PREM_SUP` na `PARTE 2` preserva as linhas da esquerda, então elas
  sobreviviam com Ganho/Faixa nulos.
- **Falta** — quando **nenhum RCA do supervisor participa da campanha que
  ele mesmo precisa bater** (exatamente o caso de `POSITIVAÇÃO PROCTER`,
  `FAIXA ESCOLHA CERTA` e `PLATINUM POINTS` para o sup 29), não havia
  nada para reagregar e o indicador simplesmente não existia — mesmo com
  venda/positivação acontecendo na carteira do time. O bloco `PARTE 1.1`
  (Setembro/2026) tratava só um recorte disso (`DEPARTAMENTO` +
  `MESATUAL` + `FATURAMENTO`), e com um `Not Exists(IndicadorSup,
  Indicador)` **global por nome de indicador**, sem supervisor — se o sup
  A já tivesse `X` vindo dos RCAs, o sup B que tem `X` como exclusivo
  ficava sem Realizado.

**Correção estrutural aplicada (Modelagem, `PARTE 1` inteira reescrita;
`PARTE 1.1` eliminada e absorvida)**: a base do Supervisor agora parte do
**catálogo dele** (`TRF_BASE_SUP_INDICADORES`, gerado da aba `PREM_SUP`)
e busca Realizado/Meta **direto nas fontes de fato**, agregando de RCA
para Supervisor. É a mesma montagem vertical do `TRF_BASE_RCA` (chaves
textuais compostas + cadeia de `LEFT JOIN`s + `Alt()` na unificação), só
que com `CodSupervisor` no lugar de `CodRca`:

| Bloco | Conteúdo | Chave |
|---|---|---|
| `TRF_BASE_SUP` | catálogo `PREM_SUP` × `INDICADORES` | — |
| 1.1 `REALIZADO_SUP` | 6 fontes de venda/positivação (Seção e Departamento, Trimestre Fixo e Mês Atual) | `_RealizadoSup` |
| 1.2 `REALIZADO_KPI_SUP` | Escolha Certa + Platinum Points | `_RealizadoKpiSup` |
| 1.3 Mix Mínimo / Listing | Realizado e Meta (mesma fonte, ligada 2×) | `_RealizadoMixSup`/`_MetaMixSup`, `_RealizadoListingSup`/`_MetaListingSup` |
| 1.4 Metas | Seção, Gillette Trimestral (×3 nas seções 93/94/96/97/98), Departamento Fat/Pos, Platinum Point, Escolha Certa | `_MetaSup`, `_MetaDepSup`, `_MetaDepPosSup`, `_MetaPPSup`, `_MetaECSup` |
| 1.5 Unificação | `Alt()` → `MetaSup`/`ValorPedidoLiquidoSup`/`ValorFaturadoLiquidoSup` + `GROUP BY` final | — |

Pontos de projeto que valem lembrar ao mexer nesse bloco:

- **Toda fonte é agregada (`GROUP BY` pela chave de supervisor) ANTES do
  `LEFT JOIN`** — diferença crítica em relação ao `TRF_BASE_RCA`: vários
  RCAs colapsam no mesmo supervisor, e `LEFT JOIN` não soma linhas
  repetidas, só as replica. Todos os blocos usam o padrão de duas etapas
  (`<TABELA>_TEMP` com a chave → `<TABELA>` com `GROUP BY <chave>`), que
  também evita depender de `GROUP BY` sobre expressão.
- **De onde vem o `CodSupervisor` em cada fonte**: fatos de
  venda/positivação, Mix Mínimo, Listing, Escolha Certa e Platinum Point
  só têm `Cod RCA` → `ApplyMap('MAP_RCA_SUP', ...)`; as metas
  (`RCA_SEC`/`RCA_DEP`/`PLATINUM_POINT`/`ESCOLHA_CERTA`) já trazem a
  coluna de supervisor da própria planilha (`SV`/`G`/`F14`) e usam o
  campo direto, sem `ApplyMap`.
- **Indicadores do `PREM_SUP` sem fonte de dado** (hoje `CATFOCO ALWAYS`
  e `CATFOCO PAMPERS`, ver 7.1) passam a **aparecer com Meta/Realizado
  nulos**, em vez de sumir — é o que o supervisor tem que bater, só falta
  a fonte.
- **Nomes de campo de saída não mudaram** (`DataSup`, `CodSupervisor`,
  `IndicadorSup`, `ClasseIndicadorSup`, `MetaSup`,
  `ValorPedidoLiquidoSup`, `ValorFaturadoLiquidoSup`,
  `PercAtingimentoPedidoSup`, `PercAtingimentoFaturadoSup`), então as
  PARTES 2 a 5 (Ganho, % Devolução, Faixa de Repasse, Ganho Final) e o
  Ranking Gillette de Supervisor seguem sem alteração.
- A regra de "único campo em comum com `BASE_RCA_INDICADORES_REALIZADO` é
  `CodSupervisor`" continua valendo — por isso todo campo tem sufixo
  `Sup` e `NomeSupervisor`/`Supervisor` não são trazidos.

**Bug de recarga corrigido em seguida (mesmo dia, achado pelo usuário
rodando no Qlik Sense)**: `TRF_BASE_SUP` (bloco `1.` acima) lia o QVD do
catálogo com `FROM [...TRF_BASE_SUP_INDICADORES_$(vAno)_$(vMesAtual).QVD]`
e falhava com `Cannot open file` — o nome real do arquivo em disco é
`..._2026_09.QVD`, mas o erro mostrava `..._2026_set.QVD`. Causa: dentro
da aba Modelagem, `vMesAtual` é redefinido (poucas linhas antes, junto de
`vDataCarg`/`vTrimestre`) como `Month(Today())` — um valor dual que, ao
ser interpolado com `$(...)`, usa a representação textual do mês (`set`,
abreviação de setembro) em vez do número (na Transformação, onde o QVD é
gravado, `vMesAtual` é `Num(Month(Today()),'00')` = `"09"`, daí a
divergência). **Correção**: trocado para `$(vDataCarg)`
(`Date(MonthStart(Today()),'YYYY_MM')` = `"2026_09"`), a mesma variável
que `TRF_BASE_RCA_INDICADORES` já usa para ler seu catálogo nesta mesma
aba — nunca usar `$(vAno)_$(vMesAtual)` na Modelagem, só na
Transformação.

**Validado no Qlik Sense em 2026-09-15 (confirmado pelo usuário): recarga
concluída sem erros, base de Supervisor funcionando.**

## 7.3 Setembro 2026 — CATFOCO ALWAYS / CATFOCO PAMPERS implementados

Indicadores exclusivos de Supervisor (Classe `KPI`, Tipo `DEPARTAMENTO`,
Período `MESATUAL`, `Codigo='3'` — mesmo `Codigo` de Platinum
Point/Escolha Certa). Fonte de dado e regra de negócio confirmadas pelo
usuário em 2026-09-15, reproduzindo o mesmo cálculo já usado nos objetos
de set analysis do app Qlik Sense (contagem de clientes distintos
positivados dentro de um recorte específico de produto/seção/ramo).

**Regra de Realizado** (nova seção "6.1 CATEGORIA FOCO" na Transformação,
dentro de "Vendas Mês Atual", gera `FATO_CATFOCO_RCA_MES_AAAA_MM.qvd`):

- **CATFOCO ALWAYS**: cliente positivado = `Sum(ValorFaturadoLiquido) > 0`
  (Faturado) / `Sum(ValorVendaLiquida) > 0` (Pedido) restrito aos 12
  `Cod Produto` da família Always (`216404,216312,220114,211834,17504,
  219481,17734,217921,211833,219480,17505,17156`) e `Cod Departamento=3`.
- **CATFOCO PAMPERS**: cliente positivado = a mesma condição de
  Faturado/Pedido **E** quantidade >= 10 — `Sum(QtdPedidoLiquido) >= 10`
  ("Qtd Vendida Liquida") no lado Pedido, `Sum(QtdFaturadoLiquido) >= 10`
  ("Qtd Faturada Liquida") no lado Faturado (**corrigido em 2026-09-15**
  a pedido do usuário — antes o lado Faturado também checava
  `QtdPedidoLiquido`, igual ao Pedido) — restrito a `Cod Ramo P&G` em
  `{169,168,112,113,114,115,127,171,170,38,126,8,160,48,47}`, `Cod Secao`
  em `{4382,3691,4164,4383,3542,4083,30}` e `CodPlataformaPG = 176`.
- O lado "Pedido" não existe na definição de negócio original (dada só
  em termos de Faturado) — **decisão do usuário (2026-09-15)**: espelhar
  a mesma regra trocando `Vlr Tot Faturado Liquido Venda` por
  `Vlr Venda Liquida`, mesmo padrão das Positivações de Seção/Departamento
  já existentes.
- **Novo campo/mapping**: `CodPlataformaPG` (via nova `MAPPING
  MAP_CLIENTE_PLATAFORMA`, `CAD_CLIENTE.QVD`) e `"Cod Ramo P&G"`
  adicionados ao `LOAD` de `TMP_VENDAS` (Vendas Mês Atual). **TODO:
  verify** — nome exato do campo `CODPLATAFORMA_PG` em `CAD_CLIENTE.QVD`
  assumido igual ao usado no Qlik Sense hoje.

**Regra de Meta** (nova seção "2.5 META CATEGORIA FOCO" no transformador
de Metas P&G, lê a aba `METACAT_FOCO` de `METAS_2026-SET.xlsx`, gera
`TRF_METAS_CATFOCO_MES_AAAA_MM.qvd`): Meta por RCA (`METACAT_FOCO`),
agregada por Supervisor na Modelagem como soma das metas dos RCAs dele
(confirmado pelo usuário). **TODO: verify** — a coluna do Cod Supervisor
nesta aba não tem texto de cabeçalho (só `DTREF`/`CODRCA`/`CAT_FOCO`/
`METACAT_FOCO` têm); assumido que o Qlik nomeia automaticamente pela
letra da coluna (`F`) quando o cabeçalho está em branco, mesmo padrão já
confirmado em uso pelo campo `G` do bloco de Meta Platinum Point (idêntica
posição relativa: header só até a coluna anterior, RCA nomeado, Sup sem
nome).

**Modelagem**: o Realizado entra na PARTE 1.2 (bloco `REALIZADO_KPI_SUP`,
que já reagrega Escolha Certa/Platinum Point de RCA para Supervisor) —
`FATO_CATFOCO_RCA` virou uma terceira fonte no mesmo `CONCATENATE`,
usando `'$(vDataAux)'` literal na chave (não tem campo `DATA` nem
`"Cod RCA"`, só `CodRca` já pronto pra `ApplyMap('MAP_RCA_SUP', ...)`,
mesmo padrão do bloco 1.1). A Meta entra na PARTE 1.4 como um novo bloco
`META_CATFOCO_SUP`, com uma chave nova (`_MetaCatFocoSup`, adicionada ao
catálogo `TRF_BASE_SUP`) — os dois indicadores (ALWAYS/PAMPERS)
compartilham o mesmo bloco porque o `Indicador` já vem do próprio QVD de
Meta. `MetaCatFocoSup` entrou no `Alt()` da unificação (PARTE 1.5).

**Correção adicional (2026-09-15, a pedido do usuário)**: o lado Faturado
de CATFOCO PAMPERS checava `Sum(QtdPedidoLiquido) >= 10` (a mesma
quantidade usada no lado Pedido) em vez de `Sum(QtdFaturadoLiquido) >=
10` — cada trilha agora usa sua própria quantidade líquida
(`QtdPedidoLiquido`="Qtd Vendida Liquida" no Pedido,
`QtdFaturadoLiquido`="Qtd Faturada Liquida" no Faturado, ambas já
carregadas em `TMP_VENDAS`).

**Validado no Qlik Sense em 2026-09-15 (confirmado pelo usuário):
recarga concluída sem erros** — os dois `TODO: verify` (campo
`CODPLATAFORMA_PG` em `CAD_CLIENTE.QVD` e nome auto-gerado `F` da coluna
de supervisor em `METACAT_FOCO`) resolveram corretamente.

## 7.4 Setembro 2026 — Histórico das tabelas finais (aba Carregamento)

**Problema (achado pelo usuário, 2026-09-15)**: "se eu quiser consultar
meses anteriores não vou ter acesso" — cada recarga calculava as 6
tabelas finais só para o mês/trimestre **atual** (`Today()`) e elas só
existiam na memória do Qlik; a recarga seguinte substituía tudo, sem
nenhum jeito de olhar um mês fechado depois que o mês seguinte começava
a ser processado.

**Diagnóstico importante**: os QVDs **intermediários** da Transformação
(`FATO_VENDAS_*`, `TRF_PREM_*`, `TRF_BASE_*_INDICADORES` etc.) **já são
históricos** — o nome de cada arquivo leva o ano/mês
(`$(vAno)_$(vMesAtual)` ou `$(vDataCarg)`), então a recarga de um mês
novo nunca sobrescreve o QVD do mês anterior, só cria um arquivo com nome
diferente. O buraco estava só na Modelagem/Carregamento: liam e
recalculavam sempre o mês atual, mas nunca acumulavam o resultado final
de volta em disco.

**Correção aplicada** (aba Carregamento, antes só comentário/sem lógica —
agora com o padrão clássico de QVD incremental do Qlik para cada uma das
6 tabelas finais):

1. Lê o QVD histórico da tabela (`HISTORICO_<TABELA>.qvd`, mesma pasta
   `$(vPathCampanhas)`), se existir, **excluindo** as linhas do período
   atual (evita duplicar se o mesmo período for recarregado de novo —
   decisão do usuário: **substituir**, não somar).
2. `CONCATENATE` com a tabela recém-calculada (sempre referente ao
   período atual).
3. `STORE` do resultado (períodos antigos + atual) de volta no mesmo QVD
   histórico — o arquivo cresce um período por recarga.

**Campo de período por tabela** (usado para o filtro de exclusão acima e
para o dashboard poder filtrar por mês/trimestre):

| Tabela | Campo de período | Grão do período | Observação |
|---|---|---|---|
| `BASE_RCA_INDICADORES_REALIZADO` | `DATA` (já existia) | Mensal | — |
| `BASE_SUP_INDICADORES_REALIZADO` | `DataSup` (já existia) | Mensal | — |
| `GANHO_FINAL_RCA` | `DataRef` (novo) | Mensal | — |
| `GANHO_FINAL_SUP` | `DataRefSup` (novo) | Mensal | Sufixo `Sup` — sem ele, `DataRef` viraria campo em comum **novo** com `GANHO_FINAL_RCA` (hoje as duas não compartilham nenhum campo direto), criando uma chave sintética indesejada. |
| `PREMIACAO_GILLETTE_TRI` | `TrimestreRef` (novo, ex: `"2026-T3"`) | **Trimestral** | O indicador é uma apuração do trimestre corrente, recalculada a cada mês do mesmo trimestre — o registro do trimestre é **substituído** a cada recarga (ranking mais atualizado), não vira 3 linhas por trimestre. |
| `PREMIACAO_GILLETTE_TRI_SUP` | `TrimestreRefSup` (novo) | Trimestral | Mesma lógica, sufixo `Sup` pelo mesmo motivo de `GANHO_FINAL_SUP`. |

**Schema evoluindo**: se um campo novo for adicionado a alguma destas 6
tabelas no futuro, os períodos antigos do QVD histórico simplesmente
carregam `Null` nesse campo novo (`CONCATENATE` por nome de campo,
comportamento padrão do Qlik) — não precisa de migração manual.

**Impacto no dashboard**: a partir de agora essas 6 tabelas trazem
**múltiplos períodos simultaneamente** no modelo (não só o mês/trimestre
atual). Qualquer gráfico/KPI que hoje usa `CodRca`/`CodSupervisor` como
dimensão **sem filtrar por período** (`DATA`/`DataSup`/`DataRef`/
`DataRefSup`/`TrimestreRef`/`TrimestreRefSup`) passa a **somar todos os
períodos já processados juntos** — o usuário está ciente e vai ajustar os
objetos do app para filtrar/selecionar o período (aceito explicitamente
ao decidir adicionar o campo de período às 4 tabelas que não tinham).

**Primeira carga**: como os arquivos `HISTORICO_*.qvd` ainda não existem,
a primeira recarga após esta mudança começa o histórico só com o mês/
trimestre atual — meses **anteriores** a essa mudança não são
recuperados retroativamente (o histórico só passa a acumular a partir de
agora).

**Validado no Qlik Sense em 2026-09-15 (confirmado pelo usuário)**: os 6
QVDs `HISTORICO_*.qvd` foram gravados com sucesso na primeira recarga.
**Ainda por confirmar**: uma segunda recarga no mesmo mês/trimestre
substitui (em vez de duplicar) o período atual, e o comportamento de
`CodRca`/`CodSupervisor` sem filtro de período nos objetos do app (ver
"Impacto no dashboard" acima) — ver item correspondente na seção 7.

## 7.5 Setembro 2026 — % Devolução do ranking Gillette Trimestral trocado para o Trimestre Fixo

**Dúvida do usuário (2026-09-15)**: a devolução usada para decidir se um
RCA é zerado no ranking Gillette Trimestral era do Mês Atual ou do
Trimestre Fixo (mesma janela do Realizado/Meta do indicador)?

**Resposta encontrada no código (antes da correção)**: **Mês Atual**, nos
dois usos que existiam — tanto a Faixa de Repasse geral (todos os
indicadores) quanto a regra de zerar o RCA no ranking Gillette usavam o
mesmo campo `PercDevolucaoRcaFaturado` de `GANHO_FINAL_RCA`, calculado a
partir de `FATO_DEVOLUCAO_RCA_MES`/`FATO_FATURAMENTO_RCA_MES` (só o mês
corrente). Isso é correto para os indicadores mensais (cuja Meta/
Realizado também são do mês), mas **inconsistente para o GILLETTE
TRIMESTRAL**, cujo Realizado/Meta são apurados sobre os 3 meses do
Trimestre Fixo — um RCA podia ter devolução baixa em 2 dos 3 meses do
trimestre e ainda assim ser zerado no ranking só por estourar a faixa no
mês da apuração (ou vice-versa).

**Decisão do usuário**: a devolução do ranking Gillette deve ser a do
**mesmo Trimestre Fixo** apurado pelo indicador.

**Correção aplicada**:

- **Transformação** (seção "Vendas Trimestre Fixo", bloco novo "5.
  DEVOLUCAO E FATURAMENTO POR RCA... NO TRIMESTRE FIXO"): dois QVDs
  novos, `FATO_DEVOLUCAO_RCA_TRIMESTRE_FIXO_AAAA_TN.qvd` e
  `FATO_FATURAMENTO_RCA_TRIMESTRE_FIXO_AAAA_TN.qvd`, agregando a mesma
  `TMP_VENDAS` do trimestre (já usada para `FATO_VENDAS_TRIMESTRE_FIXO_
  SECAO`/`DEPARTAMENTO`) só por `Cod RCA` (sem quebra por Seção/
  Departamento) — mesmo padrão de `FATO_DEVOLUCAO_RCA`/
  `FATO_FATURAMENTO_RCA` (Mês Atual), só que trimestral.
- **Modelagem** (bloco `PREMIAÇÃO GILLETTE TRIMESTRAL`, antes de
  `BASE_GILLETTE_TRI`): novo `MAP_PERC_DEVOL_FAT_RCA_TRI`, calculado a
  partir dos 2 QVDs acima (`TotalDevolucaoRcaTri / TotalFaturadoRcaTri`),
  substituindo o antigo `MAP_PERC_DEVOL_FAT_RCA` (que lia
  `GANHO_FINAL_RCA.PercDevolucaoRcaFaturado`, mês atual) nas 3 fórmulas
  de `BASE_GILLETTE_TRI` (`PercDevolucaoRcaFaturadoGillette`,
  `PercAtingimentoFaturadoRank`, `FlagZeradoPorDevolucao`).
- **O que NÃO mudou**: a Faixa de Repasse dos demais indicadores
  (mensais, `GANHO_FINAL_RCA` / PARTE 4) continua usando
  `PercDevolucaoRcaFaturado` do **Mês Atual** — só o ranking Gillette
  Trimestral passou a usar a janela trimestral. `GANHO_FINAL_RCA`
  continua no modelo de dados (não foi alterado nem dropado).

**Correção adicional no ranking do Supervisor (mesmo dia, achado pelo
usuário revisando o script)**: o ranking Gillette Trimestral do
**Supervisor** tinha o mesmo problema de janela (usava
`GANHO_FINAL_SUP.PercDevolucaoSupFaturado`, Mês Atual) **e mais um
segundo problema**: mesmo trocando a janela, a devolução do Supervisor
precisa ser a soma de **todos os RCAs do time dele** — a campanha
Gillette Trimestral do Supervisor é o resultado do time inteiro, não
apenas dos RCAs que individualmente têm o indicador cadastrado em
`PREM_RCA`. O lado Realizado (`PercAtingimentoFaturadoGilletteSup`,
vindo de `BASE_SUP_INDICADORES_REALIZADO`) já fazia isso certo, porque é
calculado direto da `FATO_VENDAS_SECAO_TRIMESTRE_FIXO` para todos os
RCAs (ver seção 7.2) — só a devolução do ranking ainda dependia de uma
reagregação que, por si só, já cobria todos os RCAs (via `MAP_RCA_SUP`
em `GANHO_FINAL_SUP`), mas na janela errada.

**Correção aplicada**: novo `MAP_PERC_DEVOL_FAT_SUP_TRI`, calculado
agregando `FATO_DEVOLUCAO_RCA_TRIMESTRE_FIXO`/
`FATO_FATURAMENTO_RCA_TRIMESTRE_FIXO` (as mesmas 2 fontes RCA/Trimestre
Fixo criadas acima) direto por `CodSupervisor` via
`ApplyMap('MAP_RCA_SUP', ...)` — soma **todos** os RCAs do supervisor,
independente de cada um ter ou não `GILLETTE TRIMESTRAL` em `PREM_RCA`
— substituindo `MAP_PERC_DEVOL_FAT_SUP` nas 3 fórmulas de
`BASE_GILLETTE_TRI_SUP`. Mesmo cuidado de sempre: agregação (`GROUP BY
CodSupervisor`) feita **antes** do `LEFT JOIN`, porque vários RCAs
colapsam no mesmo supervisor.

**Ainda não validado no Qlik Sense** (2026-09-15).

## 7.6 Setembro 2026 — Supervisor fictício Gerson (73+74 → 7374): código correto por indicador

**Achado do usuário (2026-09-15, revisando o grupo de ranking Gillette no
Qlik Sense)**: comparando `data/grupo_rank_sup.png` (aba
`PREM_RANK_GILLETTE_SUP` da planilha, grupo "ADRIANO/GERSON TOP/WILLIAM"
com 4 supervisores: 60, 105, 73, 34) com `data/qlik_rank_sup.png` (Qlik
mostrando só 3: 105, 34, 60) — faltava o Gerson.

**Causa raiz**: a planilha `CAMPANHAS_2026_09.xlsx` usa **códigos
diferentes** para o supervisor fictício "Gerson" em abas diferentes:

| Aba | Código do Gerson |
|---|---|
| `PREM_SUP` (premiação) | `7374` |
| `SUP_DEVOL` (faixa de repasse mensal) | `7374` |
| `PREM_RANK_GILLETTE_SUP` (grupo de ranking) | `73` |
| `SUP_DEVOL_RANK` (faixa máxima do ranking) | `73` |

Confirmado também em `data/METAS_2026-SET.xlsx` (abas `RCA_SEC`/
`RCA_DEP`, coluna `SV`) que o cadastro trata os RCAs do Gerson com os
códigos **reais e separados** `73` (TOPCONTAS) e `74` (INTERIOR) — nunca
`7374` diretamente; `7374` é só uma convenção de relatório usada em
`PREM_SUP`/`SUP_DEVOL` pra reportar a soma dos dois times como se fosse
um supervisor só.

**Regra de negócio confirmada pelo usuário**: `GILLETTE TRIMESTRAL`
apura **só pelo código real 73** (o usuário vai corrigir `PREM_SUP` para
usar `73` em vez de `7374` nessa linha). **Todos os demais indicadores**
do Gerson precisam somar o Realizado de `73` **e** `74` juntos, sob o
código fictício `7374` (como já é `PREM_SUP`/`SUP_DEVOL` hoje).

**Correção aplicada**: nova `MAPPING MAP_SUP_FICTICIO` (73→7374, 74→7374,
default = o próprio código pra qualquer outro supervisor).

**Escopo corrigido pelo usuário (2026-09-15, depois da primeira versão
desta correção ter aplicado o mapa a tudo)**: `MAP_SUP_FICTICIO` só se
aplica aos **indicadores mensais em geral** — bloco 1.1 (Vendas/
Positivação Mês Atual, Seção e Departamento), 1.2 (Escolha Certa/
Platinum Point/CatFoco), 1.4 (Metas de Seção/Departamento/Platinum
Point/Escolha Certa/CatFoco) e a devolução mensal usada na Faixa de
Repasse (PARTE 3 da Premiação Final do Supervisor). **Ficam de fora**
(mantêm o código **real** do supervisor, igual já estava antes de
qualquer correção do Gerson):

- **GILLETTE TRIMESTRAL** — já era a exceção original (fontes TRIMESTRE
  FIXO no bloco 1.1, e a devolução trimestral da seção 7.5).
  `META_GILLETTE_TRI_SUP_TEMP` mistura as duas regras **na mesma
  tabela**: linhas das seções 93/94/96/97/98 (Gillette) usam o código
  real, as demais seções (Faturamento Seção mensal) usam o fictício.
- **MIX MÍNIMO** e **LISTING INICIATIVAS--100% CARTEIRA** (bloco 1.3) —
  usam o código real do supervisor, sem passar pelo mapa.

**Efeito colateral conhecido (cosmético, não financeiro)**: como
`GANHO_FINAL_SUP` tem grão só `CodSupervisor` (sem quebra por
indicador), o Gerson pode aparecer como **linhas distintas** nessa
tabela — `73`/`74` (Gillette, Mix Mínimo, Listing — Ganho de Gillette
hoje é sempre ~0 porque `PREM_SUP` deixa `GANHO`/`FAIXA` em branco pra
esse indicador, igual ao RCA) e `7374` (indicadores mensais gerais). Não
afeta valores pagos, só a granularidade de exibição nessa tabela
específica — se isso incomodar visualmente no dashboard, vale revisitar.

**Ainda não validado no Qlik Sense** — depende também da correção da
planilha (mudar `PREM_SUP` de `7374` para `73` na linha `GILLETTE
TRIMESTRAL` do Gerson) que o usuário ainda vai aplicar.

## 7.7 Setembro 2026 — `FAIXA ESCOLHA CERTA` ausente para o supervisor 39 (Matheus): dado de cadastro, não bug

**Sintoma (achado pelo usuário, 2026-09-15)**: na tabela "REALIZADO
SUPERVISOR" do Qlik Sense, o indicador `FAIXA ESCOLHA CERTA` aparecia só
para os supervisores 29, 75 e 76 — faltava o supervisor 39 (Matheus),
mesmo ele estando cadastrado em `PREM_SUP` (`CAMPANHAS_2026_09.xlsx`)
com `GANHO=1500`/`FAIXA=1`.

**Investigação**: confirmado que a linha `(CodSupervisor=39, IndicadorSup
='FAIXA ESCOLHA CERTA')` não tem nenhum `WHERE`/filtro no script que a
excluiria (diferente do bug do Gillette/CATFOCO das seções 7.2/7.3) — o
catálogo `TRF_BASE_SUP_INDICADORES` (de `PREM_SUP`) já garante que ela
existe. Checando `METAS_2026-SET.xlsx` (aba `ESCOLHA_CERTA`, coluna do
supervisor real, não `META_EC`) confirmou-se que **nenhum RCA do
supervisor 39 tinha linha nessa aba** — sem Meta cadastrada, `MetaSup`
ficava 0 e `PercAtingimentoFaturadoSup` nulo, e o objeto do Qlik
aparentemente suprime linhas com todas as métricas zeradas/nulas.

**Resolução**: o usuário ajustou a aba `ESCOLHA_CERTA` de
`METAS_2026-SET.xlsx` incluindo os RCAs do supervisor 39. Após a
recarga, a linha passou a aparecer normalmente (`MetaSup=480`,
`PercAtingimentoFaturadoSup=9,38%`, junto dos outros 3 supervisores).
**Confirmado no Qlik Sense em 2026-09-15**: não era bug de script, era
dado de cadastro faltante na planilha de Metas.

## 7.8 Setembro 2026 — Listing Iniciativas não aparecia para o supervisor 60: rollup RCA→Sup trocado para usar `COD_SUP` da própria `MIX_MIN`

**Sintoma (achado pelo usuário, 2026-09-23)**: na tabela "REALIZADO
SUPERVISOR" do Qlik Sense, o indicador `LISTING INICIATIVAS--100%
CARTEIRA` não aparecia para o supervisor 60, mesmo ele estando
cadastrado em `PREM_SUP` (`CAMPANHAS_2026_09.xlsx`, `GANHO=700`/
`FAIXA=1`) e mesmo o `MIX MINIMO` (mesmo padrão de indicador exclusivo
de supervisor) aparecendo normalmente pra esse mesmo supervisor.

**Investigação**: catálogo (`TRF_BASE_SUP_INDICADORES`, de `PREM_SUP`) e
`INDICADORES` batiam certinho (`CODIGO=3, TIPO=DEPARTAMENTO,
CLASSE=MANUAL, PERIODO=MESATUAL`, texto do `Indicador` idêntico) — não
era problema de catálogo nem de nome divergente. `PREM_RCA` **não tem**
`LISTING INICIATIVAS--100% CARTEIRA` nem `MIX MINIMO` — confirmando que
os dois são indicadores exclusivos de supervisor (igual ao padrão de
CATFOCO ALWAYS/PAMPERS antes de terem fonte própria, ver seção 7.3).

**Causa**: mesmo sendo exclusivo de supervisor, o Realizado do Listing
era calculado no grão de **RCA** (`FATO_LISTING_RCA`, a partir de
`COD_RCA` da aba `MIX_MIN`) e só depois somado pro supervisor via
`ApplyMap('MAP_RCA_SUP', CodRca)` — que usa o cadastro "oficial"
`CAD_RCA.QVD`, não o `COD_SUP` que a própria aba `MIX_MIN` já traz pronto
pra carteira daquele mês. Se o(s) RCA(s) da carteira de Listing do
supervisor 60 não estiverem cadastrados sob o supervisor 60 no
`CAD_RCA` (podem estar desatualizados/diferentes do que a planilha de
campanha usa), o rollup não encontra o supervisor 60 e a linha some —
mesmo mecanismo de supressão de linha com métricas nulas já confirmado
na seção 7.7 (`FAIXA ESCOLHA CERTA` do supervisor 39).

**Resolução (decisão do usuário, 2026-09-23)**: `CodSupervisor` agora é
trazido direto do `COD_SUP` da aba `MIX_MIN` e carregado através de todo
o pipeline do Listing (`LISTING_PARTICIPANTES` → `LISTING_ELEGIVEL` →
`LISTING_CLIENTE` → `FATO_LISTING_RCA`, seção 8 da Transformação), em vez
de ser derivado por `ApplyMap('MAP_RCA_SUP', CodRca)` na Modelagem. Os
blocos `REALIZADO_LISTING_SUP_TEMP`/`META_LISTING_SUP_TEMP` (seção 1.3 da
Modelagem) agora leem `CodSupervisor` direto do QVD
`FATO_LISTING_RCA_MES_AAAA_MM.qvd` em vez de recalculá-lo. **Mix Mínimo
não foi alterado** (continua usando `ApplyMap('MAP_RCA_SUP', CodRca)`) —
ele já aparecia corretamente para o supervisor 60, então a mudança ficou
restrita ao indicador com o problema reportado; se o mesmo sintoma
aparecer no Mix Mínimo, aplicar o mesmo padrão (`COD_SUP` de `MIX_MIN`
carregado pela seção 7 da Transformação até `FATO_MIXMINIMO_RCA`).

Validado no Qlik pelo usuário em seguida: o indicador continuava
ausente, e não só para o supervisor 60 — para **nenhum** supervisor. Ver
causa real na seção 7.9.

## 7.9 Setembro 2026 — Listing Iniciativas nunca tinha Realizado pra ninguém: `Ramo` com capitalização diferente entre `MIX_MIN` e `LISTING_PRODUTOS` quebrava o `JOIN` (case-sensitive)

**Sintoma**: após a correção da seção 7.8 (rollup RCA→Sup), o indicador
`LISTING INICIATIVAS--100% CARTEIRA` continuou sem aparecer — não só
pro supervisor 60, pra nenhum supervisor. Isso descartou de vez a
hipótese de mapeamento de supervisor: o problema estava antes disso, na
geração do próprio `FATO_LISTING_RCA` (Transformação, seção 8).

**Causa confirmada** lendo `CAMPANHAS_2026_09.xlsx` diretamente: a coluna
`RAMO` vem com capitalização **diferente** em cada aba —
`MIX_MIN.RAMO` = `"Alimentar"` / `"Farma"` (capitalizado), enquanto
`LISTING_PRODUTOS.RAMO` = `"ALIMENTAR"` / `"FARMA"` (tudo maiúsculo). O
`JOIN` da seção 8.3 (`LISTING_ELEGIVEL` × `TRF_LISTING_PRODUTOS`) é um
**inner join** por `Ramo`, e comparação de texto no Qlik é
case-sensitive — `"Alimentar"` ≠ `"ALIMENTAR"` — então **nenhum** cliente
sobrevivia ao join, `LISTING_ELEGIVEL` ficava vazia, e
`FATO_LISTING_RCA` nunca tinha Realizado real pra ninguém (RCA ou
Supervisor). Mix Mínimo não sofre disso porque seu join equivalente
(seção 7.4, `MIXMIN_ELEGIVEL` × `TRF_MIXMIN_GRUPO`) usa `Categoria`, não
`Ramo`.

**Resolução**: `RAMO` agora é normalizado com `Upper(Trim(...))` nos dois
lados do join — `LISTING_PARTICIPANTES` (de `MIX_MIN`, seção 8.1) e
`TRF_LISTING_PRODUTOS` (de `LISTING_PRODUTOS`, seção 8.2) — para não
depender de digitação consistente entre as duas abas da planilha.

**Não validado no Qlik Sense ainda** — precisa reload completo (as 3
abas, na ordem) e conferência visual das tabelas "REALIZADO RCA" e
"REALIZADO SUPERVISOR" pro indicador Listing.

Validado pelo usuário em seguida: o indicador passou a aparecer, mas só
para o supervisor 105 — sumido pro 60 e pro 73 (Gerson). Ver seção 7.10.

## 7.10 Setembro 2026 — Listing só aparecia pro supervisor 105: `PREM_SUP` do Gerson usa o código fictício (7374) em vez do real (73); supervisor 60 ainda em investigação

**Sintoma**: depois da correção do `Ramo` (seção 7.9), Listing passou a
ter Realizado de verdade, mas na tabela "REALIZADO SUPERVISOR" só
aparecia para o supervisor 105 — sumido pros supervisores 60 e 73.

**Causa confirmada pro supervisor 73 (Gerson)**: lendo `PREM_SUP` de
`CAMPANHAS_2026_09.xlsx` direto, a linha `MIX MINIMO` e a linha
`LISTING INICIATIVAS--100% CARTEIRA` do Gerson estão cadastradas sob
`COD_SUP=7374` (o código fictício que junta TOPCONTAS+INTERIOR) — ao
contrário da própria regra já documentada no script (comentário no
`.QVS`, bloco `MAP_SUP_FICTICIO`): "MIX MINIMO e LISTING INICIATIVAS
usam o código REAL do supervisor", mesma exceção que `GILLETTE
TRIMESTRAL` já segue corretamente (essa está sob `COD_SUP=73` na mesma
planilha). Como o cálculo do Realizado usa o código REAL (73, vindo de
`COD_SUP` em `MIX_MIN`), ele nunca encontra a linha de catálogo (que
está em 7374) — chave de junção diferente, linha some. **É dado da
planilha, não bug do script** (mesmo padrão do item já pendente do
Gillette, ver seção 7 abaixo). Resolução: o usuário vai corrigir
`COD_SUP` de `7374` para `73` nas linhas `MIX MINIMO` e `LISTING
INICIATIVAS--100% CARTEIRA` do Gerson no `PREM_SUP`.

**Supervisor 60 — resolvido, não é bug (confirmado com as tabelas de
debug, 2026-09-23)**: usuário conferiu `DEBUG_LISTING_PRODUTO_CLIENTE_
MES_2026_09.qvd` e `DEBUG_LISTING_CLIENTE_MES_2026_09.qvd` no Qlik.
Resultado: os 5 clientes da carteira do supervisor 60 (17367, 31536,
111078, 318518, 334788, nos RCAs 6011/222/6050) têm
`QtdCaixaPedidaProduto`/`QtdCaixaFaturadaProduto` nulas para os 2
produtos exigidos do Ramo Alimentar (223254, 223275) —
`FlagProdutoPositivado=0` em todas as linhas, logo
`FlagClienteCompletouPedido`/`FlagClienteCompletouFaturado=0` para todos
os 5 clientes. **Realizado genuinamente 0%** — nenhum cliente da
carteira comprou os produtos exigidos esse mês, não é ausência de linha
em `FATO_LISTING_RCA` (a linha existe, com valor 0, igual pros RCAs
6011/222/6050). Com `MetaListingSup=3` (soma de `Meta=1` dos 3 RCAs),
`PercAtingimentoFaturadoSup=0%` e `GanhoFaturadoSup=R$0` são valores
reais calculados, não nulos.

A linha sumir da tabela "REALIZADO SUPERVISOR" do Qlik Sense mesmo com
métricas reais (zeradas) é comportamento de apresentação do **objeto**
(provável "Suprimir valores zero" ligado no gráfico/tabela), não do
script de carga — fora do escopo do `.QVS`. Se o usuário quiser ver a
linha com 0%/R$0 em vez de ausente, precisa desligar essa opção nas
propriedades de apresentação do objeto no Qlik Sense.

## 7.11 Setembro 2026 — Meta de Listing pro Supervisor trocada de "qtd de RCAs" pra "qtd de Clientes"

**Sintoma (achado pelo usuário, 2026-09-23)**: depois das correções das
seções 7.9/7.10, o Listing passou a aparecer pro supervisor 60, mas
`MetaSup=3` — o usuário esperava `5`, o número de Clientes Principais
que ele tem cadastrados na aba `MIX_MIN` (participantes do Listing).

**Causa**: a Meta de Listing pro Supervisor vinha de `FATO_LISTING_RCA`
(grão RCA) somando `Meta=1` fixo POR RCA — para o supervisor 60, 3 RCAs
(6011, 222, 6050) davam `MetaSup=3`, não relacionado à quantidade de
clientes. Isso também distorcia o Realizado: um RCA com 1 cliente na
carteira pesava igual a outro com 4 (cada RCA valia "1" na Meta,
independente do tamanho da carteira dele).

**Resolução (decisão do usuário, 2026-09-23)**: nova tabela
`FATO_LISTING_SUP` (seção 8.8 da Transformação, gerada logo depois de
`FATO_LISTING_RCA`, a partir da mesma `LISTING_CLIENTE`), calculada
DIRETO no grão do Supervisor, ignorando RCA:
- `Meta` = `Count(CodClientePrincipal)` — quantidade de Clientes
  Principais participantes da carteira toda do supervisor.
- `ValorPedidoLiquido`/`ValorFaturadoLiquido` = `Sum(FlagClienteCompletou...)`
  — quantos desses clientes completaram individualmente a própria lista
  de produtos obrigatórios do Ramo.

QVD final: `FATO_LISTING_SUP_MES_AAAA_MM.qvd`. Na Modelagem (seção 1.3),
`REALIZADO_LISTING_SUP_TEMP`/`META_LISTING_SUP_TEMP` agora leem esse QVD
direto (`FATO_LISTING_RCA` continua existindo e alimentando
`TRF_BASE_RCA`, mas não é mais fonte do Supervisor) — como o QVD já vem
com `CodSupervisor` único por linha, não precisa mais de
`GROUP BY`/reagregação antes do `LEFT JOIN` em `TRF_BASE_SUP`.

Resultado esperado pro supervisor 60: `MetaSup=5`,
`ValorFaturadoSupListing`/`ValorPedidoSupListing` = quantidade dos 5
clientes que efetivamente compraram os produtos exigidos (confirmado
pelas tabelas `DEBUG_LISTING_*` da seção 7.10: hoje é 0 dos 5).

**Não validado no Qlik Sense ainda** — precisa reload completo (as 3
abas, na ordem).

## 7. Pontos em aberto para continuar o projeto

- **Validar a mudança de grão da Meta de Listing pro Supervisor** (ver
  seção 7.11) — conferir no Qlik que `MetaSup` do Listing agora reflete
  a quantidade de Clientes Principais da carteira (5 pro supervisor 60),
  não mais quantidade de RCAs.
- **Corrigir a planilha `CAMPANHAS_2026_09.xlsx`**: mudar `COD_SUP` da
  linha `GILLETTE TRIMESTRAL` do Gerson na aba `PREM_SUP` de `7374` para
  `73` (ver seção 7.6) — sem essa correção na planilha, o Gerson continua
  de fora do ranking Gillette Trimestral mesmo com o script já ajustado
  (a catalogação em `TRF_BASE_SUP_INDICADORES` ainda viria como `7374`,
  que não bate com `MAP_GRUPO_GILLETTE_SUP`/`MAP_PERC_DEVOL_FAT_SUP_TRI`,
  ambos chaveados por `73`).
- **Corrigir a planilha `CAMPANHAS_2026_09.xlsx`**: mudar `COD_SUP` da
  linha `MIX MINIMO` do Gerson na aba `PREM_SUP` de `7374` para `73`
  (ver seção 7.10 — o `LISTING INICIATIVAS--100% CARTEIRA` do Gerson já
  foi corrigido) — sem essa correção, o Mix Mínimo continua sumido pro
  Gerson mesmo com o Realizado calculado certo (código real 73).
- **Conferir a opção de "Suprimir valores zero" no objeto "REALIZADO
  SUPERVISOR" do Qlik Sense** (ver seção 7.10) — o supervisor 60 tem
  Realizado 0% genuíno no Listing (confirmado via `DEBUG_LISTING_*`),
  mas a linha some da tabela em vez de aparecer com R$0/0%, igual o
  Gillette aparece com traço. Não é ajuste de script, é propriedade de
  apresentação do objeto.
- **Validar no Qlik Sense a correção da seção 7.6** (código correto do
  Gerson por indicador — `MAP_SUP_FICTICIO`), depois da correção da
  planilha acima: conferir que o Gerson aparece no grupo "ADRIANO/GERSON
  TOP/WILLIAM" do ranking Gillette (4 supervisores) sob o código real
  (73/74); que os indicadores mensais gerais dele (`PANTENE*`, `HEAD E
  SHOULDERS`, `DESODORANTE`, `FATURAMENTO TOTAL AM`, Escolha Certa,
  Platinum Point, CatFoco) somam RCAs 73+74 sob o código fictício
  `7374`; e que `MIX MINIMO`/`LISTING INICIATIVAS--100% CARTEIRA`
  continuam com o código real (73/74), **sem** passar pelo
  `MAP_SUP_FICTICIO`.
- **Validar no Qlik Sense a correção da seção 7.5** (% devolução do
  ranking Gillette Trimestral trocado de Mês Atual para Trimestre Fixo,
  tanto no RCA quanto no Supervisor): confirmar que os 2 QVDs novos
  (`FATO_DEVOLUCAO_RCA_TRIMESTRE_FIXO_*`/
  `FATO_FATURAMENTO_RCA_TRIMESTRE_FIXO_*`) são gerados; comparar
  `FlagZeradoPorDevolucao` (RCA) antes/depois para pelo menos um RCA com
  devolução alta em só 1 dos 3 meses do trimestre; comparar
  `FlagZeradoPorDevolucaoSup` (Supervisor) para um supervisor que tenha
  algum RCA sem `GILLETTE TRIMESTRAL` individual em `PREM_RCA` mas com
  vendas nas seções do indicador (caso em que a correção da agregação
  por `MAP_RCA_SUP` deveria mudar o resultado).
- Historização das 6 tabelas finais (seção 7.4): **STORE dos 6 QVDs
  `HISTORICO_*.qvd` já validado** (2026-09-15) na primeira recarga. Ainda
  falta confirmar (1) que uma segunda recarga no mesmo mês/trimestre
  **substitui** (não duplica) o período atual, e (2) ajustar os objetos
  do app que hoje usam `CodRca`/`CodSupervisor` sem filtrar por período
  (`DATA`/`DataSup`/`DataRef`/`DataRefSup`/`TrimestreRef`/
  `TrimestreRefSup`) — passam a somar todos os períodos acumulados
  juntos, não só o atual.
- Confirmar se o valor de R$20 por positivação do **Escolha Certa
  Especial** é calculado em algum lugar (script ou app Qlik) — não
  localizado aqui.
- **Corrigir na planilha** a grafia divergente `NMR` (aba `MIX_MIN`) vs
  `NRM` (aba `MIXMIN_GRUPO_PRODUTO`) — enquanto não for corrigido,
  clientes dessa categoria ficam de fora do cálculo de Mix Mínimo.
- Validar no Qlik Sense as três correções do bug de duplicidade do **Mix
  Mínimo** (cruzamento Produto+Categoria na seção 4, e as duas
  referências circulares na Modelagem/Gillette) — conferir que
  `Sum([PercAtingimentoFaturado])` não soma mais valores > 1 num gráfico
  (ex: RCA 6026) e que o diagrama do modelo de dados mostra
  `GANHO_FINAL_RCA` **ainda presente** (não deve sumir - é a base de
  premiação final do RCA), mas ligada a `BASE_RCA_INDICADORES_REALIZADO`
  e `PREMIACAO_GILLETTE_TRI` só por `CodRca`, sem loop entre as 3.
- **CATFOCO ALWAYS / CATFOCO PAMPERS** implementados e **validados no
  Qlik Sense em 2026-09-15** (ver seção 7.3).
- Reescrita da seção 7.2 **validada no Qlik Sense em 2026-09-15**
  (recarga concluída sem erros, base de Supervisor funcionando). Ainda
  vale, numa próxima conferência: (a) checar se as metas de Supervisor
  batem com a soma das metas dos RCAs do time (nenhum valor multiplicado
  — sinal de `LEFT JOIN` sem agregação prévia); (b) confirmar que nenhum
  supervisor perdeu indicador que ele deveria apurar, além do sup 29 já
  conferido.
- A base do Supervisor não cobre `PeriodoIndicador='TRIMESTRE MOVEL'`
  (nem a do RCA cobre) — se entrar um indicador de Supervisor nesse
  período, falta mais um `CONCATENATE` no bloco 1.1.
- Não foi possível confirmar os QVDs de **Metas de Setembro**
  (`METAS_2026-SET.xlsx` → `TRF_METAS_MES`/`TRF_METAS_DEPARTAMENTO_MES`)
  — só `CAMPANHAS_2026_09.xlsx` foi conferida em `data/`. Sem a planilha
  de Metas, os indicadores novos de Faturamento (ex: `FATURAMENTO TOTAL
  RR/AM`, `RENTABILIDADE 9%`) rodam com `Meta` nula até ela existir.
- **Vendas Trimestre Móvel ainda não está ligado ao `TRF_BASE_RCA`** —
  hoje só existe como transformador (aba Transformação). Para aparecer
  na tabela unificada do dashboard, precisaria: (1) uma linha de
  catálogo com `PERIODO='TRIMESTRE MOVEL'` na aba `INDICADORES` da
  planilha CAMPANHAS (mesmo cuidado que tivemos com o nome exato do
  Listing Iniciativas), e (2) mais um `CONCATENATE` na montagem de
  `REALIZADO_SECAO` (Modelagem), igual ao que já existe para Trimestre
  Fixo e Mês Atual.
- O script assume `Today()` como referência de mês/trimestre em ~10
  pontos diferentes — se for necessário reprocessar meses fechados,
  vale adicionar parametrização.
