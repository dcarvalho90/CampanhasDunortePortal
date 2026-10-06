# Campanha Mensal Outubro 2026
    Acompanhamento dos indicadores de vendedores e supervisores da campanha
    mensal de outubro de 2026.

> Levantado em 2026-10-06 direto de `CAMPANHAS_2026_10.xlsx` (cópia local
> em `data/` e versão do servidor em `lib://4_Plan/Metas P&G/` — conteúdo
> idêntico) e `METAS_2026-OUT.xlsx` (servidor), comparando com setembro.
> Itens marcados **[A CONFIRMAR]** dependem de resposta do usuário.

## O que mudou em relação a setembro

### 1. Gillette Trimestral sai, entra Ranking de Faturamento
- Abas removidas: `PREM_RANK_GILLETTE_RCA`, `PREM_RANK_GILLETTE_SUP`,
  `RCA_DEVOL_RANK`, `SUP_DEVOL_RANK`. Seções Gillette (93/94/96/97/98,
  `TRIMESTRE FIXO`) saíram da aba `INDICADORES`.
- Abas novas: `PREM_RANK_RCA` e `PREM_RANK_SUP` ("CAMPANHA FATURAMENTO
  TOTAL - RANKING").
  - **RCA**: 32 RCAs em 6 grupos por supervisor (ADRIANO, GERSON - TOP,
    HAYLEN, IGOR, RAFAELA, WILLIAN). 1º lugar R$ 2.500, 2º R$ 1.500. A aba
    traz o RCA (`COD_RCA`) — em setembro trazia supervisor (`COD_SUP`) — e
    repete cada RCA nas posições 1 e 2.
  - **Supervisor**: um grupo só (FATURAMENTO TOTAL): 1º R$ 3.000, 2º
    R$ 2.000, 3º R$ 1.500, 4º R$ 1.000. Participantes: 79, 33, 60, 105, 73,
    34. A coluna `DATA` está 01/10, 02/10, 03/10, 04/10 (autopreenchimento
    do Excel).
  - Em `PREM_SUP` existe `RANKING FATURAMENTO` (sem ganho) para 10
    supervisores: 39, 75, 76, 29, 79, 33, 60, 105, 73, 34.
- **[A CONFIRMAR]** métrica do ranking (faturamento de quais
  departamentos — mesmo de `FATURAMENTO TOTAL AM/RR`?), período (mês de
  outubro ou trimestre T4?), se há regra de devolução/atingimento mínimo
  para entrar no pódio, e quem participa do ranking de supervisor (6 da
  aba `PREM_RANK_SUP` ou 10 do `PREM_SUP`).

### 2. Nomes de indicador divergentes entre abas (quebra o join sem erro)
O script liga `PREM_SUP`/`PREM_RCA` ao catálogo `INDICADORES` por nome
exato. Nomes em outubro sem correspondência no catálogo:

| Aba | Nome usado | Nome no catálogo `INDICADORES` |
|---|---|---|
| PREM_SUP | DEO | DESODORANTE |
| PREM_SUP | H&S | HEAD E SHOULDERS |
| PREM_SUP | PANTENE Shampoo e Cond | PANTENE SHAMPOO E CONDICIONADOR |
| PREM_SUP | PANTENE tratamento | PANTENE TRATAMENTO |
| PREM_SUP | BRF | POSITIVAÇÃO BRFOODS |
| PREM_SUP | SPECTRA / COMBO | NEXGARD SPECTRA / NEXGARD COMBO |
| PREM_SUP | FAIXA EC | FAIXA ESCOLHA CERTA |
| PREM_SUP | Platinum points | PLATINUM POINTS |
| PREM_SUP | CAT FOCO P3 / CAT FOCO PANTENE / CAT FOCO VENUS | CATFOCO PRESTO 3 / CATFOCO PANTENE BAMBU / CATFOCO VENUS |
| PREM_SUP e PREM_RCA | FATURAMENTO | FATURAMENTO TOTAL AM / RR? **[A CONFIRMAR]** |
| PREM_SUP e PREM_RCA | MARGEM MINIMA 9% | RENTABILIDADE 9%? **[A CONFIRMAR]** |

**Decisão (2026-10-06)**: corrigir os nomes na planilha — o script não
trata apelidos. **Feito em 2026-10-06 na cópia `data/CAMPANHAS_2026_10.xlsx`**
(79 células da coluna INDICADOR: 59 em `PREM_SUP`, 20 em `PREM_RCA`; o
restante da planilha não foi alterado). Decisões inferidas de setembro:
- `FATURAMENTO` em `PREM_SUP` → `FATURAMENTO TOTAL RR` para o 7374 (Gerson)
  e `FATURAMENTO TOTAL AM` para 79, 33, 60, 105, 34 (mesma divisão de
  setembro). No `PREM_RCA` o usuário já tinha marcado AM/RR (RR = RCAs do
  75).
- `MARGEM MINIMA 9%` → `RENTABILIDADE 9%` (sup 34 e RCAs 3407/3460; em
  setembro o 34 tinha `RENTABILIDADE 9%`).
- `RANKING FATURAMENTO` mantido — não é indicador do catálogo, será
  tratado na implementação do ranking.
- **A planilha da rede precisa ser substituída por esta cópia** — é a da
  rede que o script lê.

(`ORAL B ESCOVAS` com/sem espaço final não é problema — o Qlik remove o
espaço ao ler o Excel.)

### 3. Categoria Foco
- Saem `CATFOCO ALWAYS` e `CATFOCO PAMPERS`; entram `CATFOCO PRESTO 3`,
  `CATFOCO VENUS` e `CATFOCO PANTENE BAMBU` (supervisores 39, 75, 76, 29,
  7374; ganhos R$ 150–600).
- A regra de produto de cada categoria fica fixa no script (seção 6.1 da
  Transformação). **[A CONFIRMAR]** produtos/seções/ramos e quantidade
  mínima de cada nova categoria.

### 4. Indicadores de RCA
- `PREM_RCA`: 291 linhas e 36 RCAs (setembro: 585 linhas, 75 RCAs).
- Novos no RCA: `FATURAMENTO` (18 RCAs, R$ 750), `MARGEM MINIMA 9%`
  (3407 e 3460, R$ 750), `MIX MINIMO` e `LISTING INICIATIVAS--100%
  CARTEIRA` (11 RCAs dos supervisores 60, 105 e 73, R$ 500 cada).
- Mantidos: Pantene (3), Head e Shoulders, Oral B (2), Desodorante —
  R$ 300 (R$ 150 para 3407, 3460 e RCAs 7403, 7437, 7440, 7557 do Gerson
  Interior) — e PET (Nexgard, Spectra, Combo, Frotline, Terapêuticos,
  Positivação BRFoods).
- Sai `GILLETTE TRIMESTRAL`.

### 5. Indicadores de Supervisor
- Sai `POSITIVAÇÃO PROCTER`.
- Supervisores 39, 75, 76, 29: só Cat Foco (3), Faixa EC, Platinum
  Points e Ranking.
- Supervisores 79, 33, 60, 105, 34: Faturamento, P&G por seção e
  Ranking; 60 e 105 também Mix Mínimo e Listing; 34 também Margem
  Mínima 9%.
- 7374 (Gerson fictício): todos menos Ranking; 73 só Ranking.
- 240340 (PET): BRF, Combo, Frotline, Nexgard, Spectra, Terapêuticos.

### 6. Supervisores e faixas de devolução
- Supervisor novo **79 – RAFAELA**; o 29 passou a ser **HUDSON** (em
  setembro o 29 era "P_RAFAELA LIMA"). `SUP_DEVOL` agora com 10
  supervisores (entrou o 79), mesmas faixas.
- `RCA_DEVOL`: 78 RCAs, mesmas 4 tabelas de faixa. Entram 3420, 3903,
  3907, 7587; saem 2402, 2421.

### 7. Mix Mínimo e Listing
- `MIX_MIN`: 14 clientes (60, 73, 105); categorias NMR, HFS, C&C e **DPP**
  (nova, Farma).
- **`MIXMIN_GRUPO_PRODUTO` e `LISTING_PRODUTOS` estão vazias** — sem elas
  não há Realizado de Mix Mínimo nem de Listing.
- `LISTING_PRODUTOS` traz a nota "APENAS 1 SKU DE 5 QUE DEPENDEM NA REGRA
  P&G". **[A CONFIRMAR]** se a regra do Listing mudou (antes: cliente
  precisa comprar TODOS os produtos do Ramo).

### 8. Metas (`METAS_2026-OUT.xlsx`)
- **É cópia exata de setembro** com a data trocada para 01/10/2026: mesmas
  linhas e mesmos totais em todas as abas (`RCA_SEC`, `RCA_DEP`,
  `PLATINUM_POINT`, `ESCOLHA_CERTA`, `METACAT_FOCO`). `METACAT_FOCO` ainda
  traz ALWAYS/PAMPERS. **[A CONFIRMAR]** se as metas de outubro ainda vão
  ser preenchidas.

## Respostas do usuário (2026-10-06)
- **Ranking de Faturamento**: **mensal**, olhando o MESATUAL (não é
  trimestral como o Gillette).
- **Categoria Foco nova** (Presto 3, Venus, Pantene Bambu): definição de
  produtos ainda vai ser enviada pelo usuário.
- **Mix Mínimo e Listing**: não haverá em outubro — por isso as abas de
  produtos estão em branco.
- **Metas**: vêm de `METAS_2026-OUT.xlsx` na rede (mesmo caminho de
  setembro, `lib://4_Plan/Metas P&G/`) — o script já lê esse arquivo
  pelo mês. Em 2026-10-06 o arquivo ainda era idêntico ao de setembro
  (ver item 8).

### Regras do Ranking de Faturamento (definidas pelo usuário, 2026-10-06)
- **Período**: mensal (MESATUAL).
- **Posição**: maior % de atingimento da meta, **só a partir de 100%**.
- **Devolução**: usa a **FAIXA3** da aba de devolução — `RCA_DEVOL` para
  RCA, `SUP_DEVOL` para Supervisor. Se o % de devolução atingir a FAIXA3,
  o realizado do ranking é **zerado** (sai do pódio).
- **Participantes**: RCA = aba `PREM_RANK_RCA` (grupo = `GRUPO_RANK`);
  Supervisor = aba `PREM_RANK_SUP` (79, 33, 60, 105, 73, 34).
  `RANKING FATURAMENTO` no `PREM_SUP` é só marcação — não define quem
  participa.
- **Mix Mínimo e Listing**: ficam nas abas de prêmio; listas de produtos
  serão preenchidas depois. Até lá aparecem sem realizado.

### Respostas complementares (2026-10-06)
- Realizado do ranking = **faturamento de tudo** (todos os departamentos),
  não só os do indicador `FATURAMENTO TOTAL`. Meta = META_FAT de todos os
  departamentos da aba `RCA_DEP`.
- Gerson TOP no ranking de supervisor: código real **73** (só os RCAs
  dele), com a FAIXA3 da `SUP_DEVOL` do **7374**.
- Devolução **≥ FAIXA3 zera**; abaixo da FAIXA3 não penaliza.

### Categoria Foco (definida pelo usuário, 2026-10-06)
- 3 categorias: **PRESTO 3**, **VENUS** (Venus Total) e **PANTENE BAMBU**.
- **Produtos**: planilha `PRODUTOS CATEGORIA FOCO - OUTUBRO'26.xls` na
  pasta das metas (`lib://4_Plan/Metas P&G/`), uma aba por categoria
  (coluna `CODPROD`): Pantene Bambu 11, Venus 21, Presto 3 3 produtos.
- **Clientes** (iguais para as 3): Cod Plataforma **116, 176** e Cod Ramo
  P&G **169, 168, 112, 113, 114, 162, 115, 127, 171, 170, 38, 126, 8, 160,
  48, 47, 172, 141, 140, 139, 138**.
- **Positivação**: cliente com valor líquido > 0 nos produtos da
  categoria conta 1. Sem quantidade mínima e sem filtro de departamento.
  Pedido (valor pedido líquido) e Faturado (valor faturado líquido)
  contados em separado.
- **Meta**: aba `METACAT_FOCO` do `METAS_2026-OUT.xlsx`, coluna
  `CAT_FOCO` = `PRESTO 3` / `VENUS` / `PANTENE BAMBU`. Em 2026-10-06 o
  arquivo ainda trazia ALWAYS/PAMPERS (cópia de setembro).

### Metas de outubro (`METAS_2026-OUT.xlsx`, versão de 2026-10-06 20:44)
- `RCA_SEC` (metas por seção) e `METACAT_FOCO` atualizadas para outubro.
- `PLATINUM_POINT`, `ESCOLHA_CERTA` e `RCA_DEP` (META_FAT/META_POS por
  departamento — base do Ranking de Faturamento e de `FATURAMENTO TOTAL
  AM/RR`) iguais a setembro — **intencional para outubro** (usuário,
  2026-10-06): não é regra fixa, as metas normalmente mudam a cada mês;
  coincidiu de repetirem de setembro para outubro. Nos próximos meses,
  metas idênticas ao mês anterior continuam merecendo conferência.

### Mudança de supervisores (out/2026) — EM ABERTO
- Primeira carga de outubro (2026-10-06, 19:07Z): o `CAD_RCA.QVD` (ERP)
  ainda traz os 15 RCAs da Rafaela (7661, 7602, 7604, 7656, 2929, 2902,
  2935, 2932, 7692, 7658, 7687, 2946, 2533, 2961, 7600) sob o supervisor
  **29 "P_RAFAELA LIMA"**; a planilha de outubro usa **79 = Rafaela** e
  **29 = Hudson**.
- Efeito: sup 79 sem RCAs (realizado 0 nos indicadores de supervisor e no
  ranking, meta do ranking 0); sup 29 recebe o realizado do time da
  Rafaela com a premiação do Hudson (Cat Foco, Faixa EC, Platinum).
- Usuário (2026-10-06): houve mudança interna de supervisores a partir de
  outubro; o cadastro é feito no ERP — vai validar se dá para ajustar na
  planilha. Alternativa no script: mapa 29 → 79 a partir de out/2026
  (mesmo padrão do `MAP_SUP_FICTICIO`).

## Impacto no script (antes de recarregar outubro)
- ~~A recarga de outubro vai falhar (abas do Gillette removidas)~~ —
  **resolvido no script em 2026-10-06** (seção 7.15 da DOCUMENTACAO):
  blocos do Gillette só rodam se a aba existe; Ranking de Faturamento
  implementado.
- ~~Nomes divergentes~~ — corrigidos na planilha (item 2).
- ~~Regra da Categoria Foco nova~~ — implementada em 2026-10-06 (seção
  7.15 da DOCUMENTACAO). Teste com as vendas de outubro até 06/10:
  clientes positivados no Pedido/Faturado — Presto 3 52/34, Venus 46/38,
  Pantene Bambu 32/27.
- **Pendente**: `METAS_2026-OUT.xlsx` ainda igual a setembro — ranking e
  indicadores saem com as metas de setembro até ele ser atualizado.
