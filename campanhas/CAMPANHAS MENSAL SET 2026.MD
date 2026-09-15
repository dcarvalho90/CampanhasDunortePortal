# Campanha Mensal Setembro 2026
    Criar um acompanhamento de todos os indicadores dos vendedores e supervisores referente a campanha mensal valida para setembro de 2026.

## Indicadores
    Cada indicador tera um objetivo, realizado, percentual de atigimento do objetivo e valor que sera pago como premiação caso o objetivo seja alcançado.

## Premiação Final
   Para definir a premiação final do colaborador é preciso encontrar o percentual de devolucao total do colaborador:
   - Percentual de Devolução = Total de devolução / Total Faturado

   Cada colaborador tem faixas devolucao maxima estipuladas. Cada faixa de devolução paga um percentual do total de premiação dos indicadores.

## Indicadores Supervisores

> Lista conferida direto em `data/CAMPANHAS_2026_09.xlsx` (abas
> `INDICADORES` e `PREM_SUP`) em 2026-09-14 — substitui o rascunho
> anterior (que tinha só 7 itens). Ver `DOCUMENTACAO CAMPANHAS
> DUNORTE.md`, seção 7.1, para o detalhe técnico de cada um.

    1. Faturamento Total RR
        Soma do faturamento do supervisor nos departamentos de RR (Cod Departamento 3, 139, 178, 161, 221, 222, 165)

    2. Faturamento Total AM
        Soma do faturamento do supervisor nos departamentos de AM (Cod Departamento 3, 139, 178, 161, 221, 222, 165, 162)

    3. Rentabilidade 9%
        Faturamento do supervisor nos departamentos 161, 165, 178, 221, 222, 3

    4. Positivação BRFOODS
        Quantidade de clientes positivados no departamento 221

    5. Positivação Procter
        Quantidade de clientes positivados no departamento 3

    6. Faixa Escolha Certa
        Quantidade de escolhas certas feitas (mesma fonte do indicador de RCA)

    7. Platinum Points
        Quantidade de platinum points feitos (mesma fonte do indicador de RCA)

    8. Mix Mínimo / Listing Iniciativas--100% Carteira
        Mesma regra tudo-ou-nada já implementada a nível de RCA, reagregada por supervisor

    9. Categoria Foco Pampers / Categoria Foco Always
        Quantidade de clientes positivados na categoria foco Pampers/Always.
        **Implementado em 2026-09-15** (fonte de dado e regra de negócio
        confirmadas pelo usuário, ver DOCUMENTACAO CAMPANHAS DUNORTE.md,
        seção 7.3): Always = 12 produtos fixos + Departamento 3; Pampers =
        Ramo P&G + Seção + Plataforma P&G, com quantidade mínima de 10
        unidades. Meta por RCA vem da aba METACAT_FOCO de
        METAS_2026-SET.xlsx, somada por supervisor. Validado no Qlik
        Sense em 2026-09-15.

    10. Pantene Shampoo e Condicionador / Pantene Tratamento / Pantene Kit
        Faturamento por seção (Cod Secao 33 / 4363 / 4381)

    11. Head e Shoulders / Oral B Creme / Oral B Escovas / Desodorante
        Faturamento por seção correspondente

    12. Nexgard / Nexgard Spectra / Nexgard Combo / Frotline / Terapeuticos
        Campanha PET (supervisor fictício "Susy", Cod Supervisor 240340) — faturamento por seção

    13. Gillette Trimestral
        Ranking trimestral entre supervisores do mesmo grupo (aba PREM_RANK_GILLETTE_SUP), mesma lógica do ranking de RCA

    
