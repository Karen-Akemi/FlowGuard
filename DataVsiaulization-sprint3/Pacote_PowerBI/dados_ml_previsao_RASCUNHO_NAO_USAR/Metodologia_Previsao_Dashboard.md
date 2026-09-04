# Pacote de Previsão — ML → Dashboard (FlowGuard / Chronos AI)

Este pacote **não substitui** o notebook `ml.ipynb` entregue na Sprint 3 de Machine
Learning — ele é derivado dele. O notebook original faz a AED, a engenharia de
features e a **validação** dos modelos (comparação Ridge / Random Forest / Extra
Trees / HistGradientBoosting / Regressão Linear em holdout temporal), mas só
exporta métricas de erro retrospectivas (MAE/RMSE/R²), sem gerar uma previsão
prospectiva real para os dias seguintes ao fim do dataset (31/12/2025).

Este pacote fecha essa lacuna: treina os modelos finais (mesma metodologia e
mesmas 21 features do notebook) com **100% do período de regime operacional
atual (01/09/2025 a 31/12/2025, 122 dias)** e gera a previsão real de
**D+1 a D+7 (01/01/2026 a 07/01/2026)**, em formato pronto para ser importado
como fonte de dados no Power BI.

## Modelos utilizados

- **Previsão diária recursiva (D+1 a D+7): Ridge (alpha=10, com padronização)**,
  o modelo com menor MAE em D+1 no holdout do notebook original (144,18,
  praticamente empatado com Extra Trees 144,37) e o mais simples/interpretável
  entre os dois — critério também pedido pela disciplina de ML para o notebook
  principal. Previsão feita passo a passo: a previsão de cada dia vira "lag"
  para prever o dia seguinte (walk-forward recursivo), por isso o erro tende a
  crescer de D+1 para D+7.
- **Previsão pontual D+7 (ponto de checagem): Random Forest (300 árvores,
  max_depth=6)**, o modelo com menor MAE em D+7 no holdout original (146,59),
  aplicado diretamente sobre o último dia conhecido (31/12/2025) para estimar o
  volume de 07/01/2026 sem acumular erro de 7 passos recursivos.

## Limitações — importante deixar explícito no relatório/dashboard

- Amostra de treino pequena (114 obs. para D+1, 108 para D+7) porque o corte
  de regime (01/09/2025) descarta os dois anos anteriores, que operavam sob um
  padrão de volume ~14x menor (ver AED do notebook).
- No holdout do notebook original, o **R² foi negativo em todos os modelos**
  testados (pior do que simplesmente prever a média) — os modelos batem o
  baseline ingênuo em MAE, mas a confiabilidade estatística ainda é baixa.
  Isso deve ser comunicado no dashboard como "estimativa exploratória", não
  como previsão de alta confiança.
- As séries `volume_p2`, `volume_p3`, `volume_kpi` e `volume_manual` dos dias
  futuros **não são modeladas individualmente** — são estimadas por
  participação média dos últimos 14 dias sobre o `volume_total` previsto,
  apenas para alimentar as features recursivas do próprio modelo.
- Quanto mais distante o horizonte (D+2 a D+7), maior o acúmulo de incerteza,
  pois cada passo usa a previsão do passo anterior como entrada.

## Arquivos deste pacote

| Arquivo | Conteúdo | Uso sugerido no Power BI |
|---|---|---|
| `serie_historica_com_previsao.csv` | `data, volume, tipo (Histórico/Previsão)` — série diária completa (01/09/2025 a 07/01/2026) | Gráfico de linha único "Histórico + Previsão" (igual ao mockup `flowguard-dashboard.html`), colorindo por `tipo` |
| `previsao_d1_a_d7_jan2026.csv` | `data, horizonte (D+1..D+7), volume_previsto, volume_baseline_ingenuo` | Cards de "Previsão D+1" / "Previsão D+7 (acum.)" e comparação com baseline |
| `previsao_d7_pontual.csv` | Previsão direta de 07/01/2026 pelo modelo Random Forest (ponto de checagem) | Nota de rodapé/tooltip de validação cruzada do card de Previsão D+7 |
| `comparativo_modelos_final.csv` | Métricas MAE/RMSE/R²/sMAPE de todos os modelos testados, D+1 e D+7 (holdout original) | Página/seção de transparência sobre a confiabilidade do modelo |
| `validacao_temporal_d1.csv` / `validacao_temporal_d7.csv` | MAE/RMSE médios da validação cruzada temporal (TimeSeriesSplit) | Idem |
| `importancia_random_forest.csv` / `importancia_extra_trees.csv` | Importância das 21 features nos dois modelos de ensemble | Gráfico de barras de "principais fatores que explicam o volume previsto" |
| `metodologia_previsao.json` | Resumo estruturado desta metodologia + valores da previsão | Referência/documentação |

## Próximo passo (fora do escopo deste pacote)

Para produção real (não apenas a entrega da Sprint 3), o ideal é reexecutar
este pipeline periodicamente (ex.: diariamente, via Azure Data Factory —
conforme arquitetura proposta na Sprint 2) para que a previsão D+1/D+7 se
mantenha sempre relativa ao último dia realmente fechado no Azure SQL DB.
