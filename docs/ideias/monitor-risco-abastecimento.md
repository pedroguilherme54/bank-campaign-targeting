# CLAUDE.md — Monitor de Risco de Abastecimento (Eurostat Comext)

> Brief do projeto para o Claude Code. Lê este ficheiro no início de cada sessão.

---

## 1. Contexto e objetivo

**Problema:** crises recentes (pandemia, guerra na Ucrânia, disrupções no Mar Vermelho, tarifas) mostraram que muitos países e empresas dependem de poucas origens para bens críticos. Os dados para medir essa dependência existem e são públicos (Eurostat Comext), mas são enormes e técnicos: códigos de produto que mudam todos os anos, unidades heterogéneas, valores confidenciais e fluxos intra/extra-UE misturados.

**Objetivo:** construir uma aplicação que **ingere, harmoniza e modela** as importações de Portugal (com a UE27 como referência). A aplicação calcula **indicadores de risco de abastecimento por produto**: concentração de origens, dependência extra-UE, anomalias de preço unitário, ruturas de origem e volatilidade. Tudo é apresentado num dashboard interativo e acompanhado de previsões para os produtos mais críticos.

**Mensagem para o portefólio:** "Aplico a lógica de procurement (concentração de fornecedores, benchmarking de preço, risco de fornecimento) à escala de um país, com dados públicos complexos transformados num modelo analítico fiável."

**Dados:** estatísticas agregadas de comércio entre países. Não há dados pessoais nem empresas identificadas.

---

## 2. Fontes de dados

| Fonte | O que dá | Acesso |
|---|---|---|
| **Eurostat Comext — DS-045409** "EU trade since 1988 by HS2-4-6 and CN8" | Comércio mensal por país declarante × país parceiro × produto (CN8) × fluxo (importação/exportação): valor em € e quantidade em kg | API SDMX do Comext (`https://ec.europa.eu/eurostat/api/comext/dissemination`), **sempre com filtros**. Os datasets completos não podem ser descarregados pela API |
| **Comext — ficheiros bulk mensais** | O mesmo, todos os produtos, um ficheiro por mês (ex.: `full202305.7z`) | Bulk download do Eurostat. São ficheiros grandes; usar só se a API não chegar |
| **Nomenclatura Combinada (NC/CN)** + tabelas de correspondência | Descrição e hierarquia dos códigos (capítulo → HS4 → HS6 → CN8) e mapeamento das mudanças de código entre anos | Metadados/classificações do Comext (Eurostat) |
| **Lista de países** | Código do parceiro → nome, ISO, UE/extra-UE, região | Code lists do Comext |
| *(Extensão)* World Bank — Worldwide Governance Indicators | Risco de país (estabilidade política, etc.) | API do World Bank. Verificar disponibilidade na fase da extensão |

**A confirmar na Fase 1** (não assumir):
- Nomes e formato das colunas. Esperado algo como `period, declarant, partner, product_nc, flow, value_in_euros, quantity_in_kg`, possivelmente também quantidades suplementares.
- Como a API devolve os dados (SDMX-CSV ou JSON), os limites de pedidos e como filtrar por produto, país e período.
- Como aparecem os **dados confidenciais** e os códigos agregados (ex.: totais, "países não especificados").

### Âmbito inicial
- **Declarante:** Portugal (PT). UE27 como benchmark de preços e concentração.
- **Fluxo:** importações.
- **Período:** 2015 até ao último mês disponível.
- **Produtos:** 4 capítulos com uma boa história para contar. Sugestão (ajustar na Fase 1):
  - **27** — combustíveis e energia
  - **30** — produtos farmacêuticos
  - **03** — peixe (bacalhau: dependência clássica de Portugal)
  - **85** — máquinas e equipamento elétrico (inclui semicondutores)
- Só alargar o âmbito quando o pipeline estiver estável.

---

## 3. Indicadores de risco de abastecimento

Calculados por **produto (HS6 ou CN8 harmonizado) × ano** (e mês quando fizer sentido).

| # | Indicador | Lógica (resumo) | Paralelo em procurement |
|---|---|---|---|
| I1 | **Concentração de origens (HHI)** | Σ (quota de cada país parceiro)² sobre o valor importado | Concentração de fornecedores |
| I2 | **Dependência do 1.º fornecedor** | Quota do top-1 parceiro e a sua evolução | Single sourcing |
| I3 | **Dependência extra-UE** | % do valor importado de fora da UE27 | Risco geográfico/logístico |
| I4 | **Preço unitário anómalo** | €/kg de Portugal vs. a própria série histórica (robust z-score, mediana/MAD) e vs. o preço médio da UE27 | Benchmarking de preço |
| I5 | **Rutura de origem** | Mudança brusca da quota de um parceiro entre períodos (variação acima de um limiar; deteção de change point como extensão) | Substituição forçada de fornecedor |
| I6 | **Volatilidade** | Coeficiente de variação mensal de volume e de preço unitário | Instabilidade de fornecimento |
| I7 | *(Extensão)* **Risco de país** | Quota importada ponderada pelo risco de país (WGI) | Avaliação de risco do fornecedor |

**Índice de Vulnerabilidade de Abastecimento (IVA-produto):** combinação ponderada e transparente de I1–I6 (e I7, se existir), normalizada de 0 a 100. Os pesos estão em `config/pesos.yaml`. Cada produto mostra a contribuição de cada indicador para o índice, sem caixas negras.

**Previsão:** para os 10 produtos mais vulneráveis, prever o volume importado e o preço unitário a 6–12 meses. Começar com um modelo de base (sazonal naïve), depois LightGBM ou similar, sempre com validação temporal (sem fuga de dados do futuro).

Cada indicador devolve: `produto`, `periodo`, `indicador`, `valor`, `referencia` (ex.: média UE ou histórico), `severidade` (baixa/média/alta) e `explicacao` (texto legível).

---

## 4. Desafios de dados (o "coração" técnico do projeto)

Documentar no README como cada um foi resolvido:

1. **Códigos CN que mudam todos os anos.** Criar um `produto_key` estável a partir das tabelas de correspondência. Quando há divisões ou fusões de códigos, agregar ao nível comum (normalmente HS6). Medir e reportar a % de valor afetada.
2. **Quantidades em falta ou zero:** sem kg não há preço unitário. Excluir esses casos do I4 e reportar a cobertura.
3. **Dados confidenciais:** parte do comércio aparece só em agregados. Identificar os casos e não os tratar como "zero".
4. **Agregados vs. países:** remover os códigos de agregado (totais, zonas) antes de calcular quotas, para não contar em duplicado.
5. **Intra vs. extra-UE:** o comércio intra-UE pode esconder a origem real (ex.: produto asiático que entra via Países Baixos). Documentar como limitação. Uma possível extensão é usar mirror data.
6. **Dados provisórios:** os meses recentes são revistos pelo Eurostat. A ingestão tem de ser idempotente e capaz de reprocessar os últimos N meses.

---

## 5. Arquitetura

```
 Eurostat Comext API (DS-045409)    Nomenclatura CN + correspondências    Code lists países
            │                                   │                               │
            ▼                                   ▼                               ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │ 1. INGESTÃO      src/ingest/   pedidos filtrados por capítulo/período; cache  │
 │                                em data/raw/ (originais, nunca alterados)      │
 └────────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │ 2. LIMPEZA       src/clean/    tipos, períodos, remoção de agregados,         │
 │                                flags de confidencial/sem quantidade           │
 │                                → data/staging/ (parquet)                      │
 └────────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │ 3. HARMONIZAÇÃO  src/harmonize/  cadeia de correspondências CN → produto_key  │
 │                                  estável; relatório de cobertura              │
 └────────────────────────────────────────────────────────────────────────────────┘
            │
            ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │ 4. MODELO        src/model/ + DuckDB   esquema em estrela                     │
 │                                        → data/warehouse/comext.duckdb         │
 └────────────────────────────────────────────────────────────────────────────────┘
            │
            ├──────────────────────────────┐
            ▼                              ▼
 ┌──────────────────────────┐   ┌──────────────────────────┐
 │ 5. INDICADORES           │   │ 6. PREVISÃO              │
 │ src/indicators/          │   │ src/forecast/            │
 │ I1–I6 + índice IVA       │   │ baseline → LightGBM      │
 └──────────────────────────┘   └──────────────────────────┘
            │                              │
            └──────────────┬───────────────┘
                           ▼
 ┌────────────────────────────────────────────────────────────────────────────────┐
 │ 7. APP           app/ (Streamlit)                                              │
 │  Visão geral · Produto · País parceiro · Comparação PT vs UE · Previsões       │
 └────────────────────────────────────────────────────────────────────────────────┘
```

### Modelo de dados (esquema em estrela, DuckDB)

- `fact_comercio`: periodo, declarante, id_parceiro, cn8_original, produto_key, fluxo, valor_eur, quantidade_kg, flag_confidencial, flag_sem_quantidade
- `dim_produto`: produto_key, cn8, hs6, hs4, capitulo, descrição (PT/EN), anos_validade
- `map_cn_correspondencia`: ano, cn8_antigo, cn8_novo, tipo (1:1, divisão, fusão)
- `dim_pais`: id_parceiro, iso2, nome, ue27 (s/n), região
- `dim_tempo`: periodo, ano, trimestre, mês
- `fact_indicador`: saída da camada 5
- `fact_previsao`: produto_key, periodo, variavel, previsto, intervalo_inf, intervalo_sup, modelo

### Stack

| Camada | Ferramenta | Porquê |
|---|---|---|
| Linguagem | Python 3.12 | — |
| Ambiente | `uv` (ou venv + pip) | reprodutível |
| HTTP/API | `requests` (+ `pandasdmx` ou parsing de SDMX-CSV) | decidir na Fase 1 depois de testar a API |
| Transformação | pandas (polars opcional) | familiar |
| Armazenamento | Parquet + DuckDB | SQL analítico local, sem servidor |
| Estatística/ML | scipy, statsmodels, scikit-learn, LightGBM | z-score robusto, sazonalidade, previsão |
| App | Streamlit + Plotly | deploy fácil no Streamlit Community Cloud |
| Qualidade | pytest, ruff, pre-commit | |
| CI | GitHub Actions | lint e testes em cada push; extensão: atualização mensal agendada |
| Config | YAML (`config/`) | âmbito, limiares e pesos fora do código |

### Estrutura de pastas

```
monitor-risco-abastecimento/
├── CLAUDE.md
├── README.md
├── pyproject.toml
├── config/
│   ├── ambito.yaml          # declarante, capítulos, período
│   ├── limiares.yaml        # limiares de severidade dos indicadores
│   └── pesos.yaml           # pesos do índice IVA
├── data/                    # (no .gitignore, exceto amostras)
│   ├── raw/  staging/  warehouse/
│   └── sample/              # amostra pequena para testes e CI
├── notebooks/               # só exploração; lógica final vai para src/
├── src/monitor/
│   ├── ingest/  clean/  harmonize/  model/  indicators/  forecast/
│   └── pipeline.py          # corre tudo de ponta a ponta
├── app/
│   ├── Home.py
│   └── pages/
├── tests/
│   ├── test_clean.py  test_harmonize.py  test_indicators.py  test_forecast.py
│   └── fixtures/            # casos pequenos construídos à mão
└── .github/workflows/ci.yml
```

---

## 6. Roadmap por fases

Estimativa para um ritmo de 1–3 h/dia. Cada fase termina com um commit/PR e um critério de "concluído".

| Fase | Entregável | Concluído quando… | Estim. |
|---|---|---|---|
| **0. Setup** | Repo, ambiente, estrutura de pastas, ruff, pytest, CI vazio a passar | `pytest` e `ruff` correm verdes no GitHub Actions | 2–3 dias |
| **1. Exploração da API** | Notebook que faz pedidos filtrados (1 capítulo, PT, 1 ano), documenta colunas, formatos, agregados, confidenciais e limites; amostra em `data/sample/` | Lista escrita dos problemas de dados encontrados, que vira o backlog das Fases 2–3; âmbito final em `ambito.yaml` | 1 semana |
| **2. Ingestão + limpeza** | `ingest/` com cache e reprocessamento dos últimos N meses; `clean/` com testes | Pipeline reprodutível do raw para staging; correr duas vezes dá o mesmo resultado (idempotente) | 1–2 semanas |
| **3. Harmonização CN** | Tabela de correspondências, `produto_key`, relatório de cobertura | Séries de 2015 até hoje contínuas para os produtos do âmbito; % de valor afetado por mudanças de código reportada | 1–2 semanas |
| **4. Modelo em DuckDB** | Esquema em estrela + queries de validação | Totais por ano e capítulo no modelo batem com o staging (teste automático) e com os totais publicados pelo Eurostat | 1 semana |
| **5. Indicadores I1–I6 + IVA** | Indicadores, config de limiares e pesos, testes com casos construídos à mão | Cada indicador tem teste com resultado conhecido (ex.: HHI de 2 países a 50/50 = 0,5); os 10 produtos com IVA mais alto são revistos manualmente e fazem sentido | 1–2 semanas |
| **6. Previsão** | Baseline sazonal + LightGBM, backtesting temporal | O modelo supera o baseline no backtest (métrica: MAPE ou sMAPE) ou fica documentado porque não supera | 1–2 semanas |
| **7. App Streamlit** | 5 páginas: Visão geral · Produto · País parceiro · PT vs UE · Previsões | App corre localmente sobre o DuckDB; filtros por capítulo, período e parceiro; cada indicador tem explicação visível | 1–2 semanas |
| **8. Portefólio** | README de consultor, deploy público, screenshots, secção de limitações, 3 "insights" escritos | Link público funcional; o README explica problema → abordagem → resultados → limitações | 3–5 dias |

**Extensões opcionais (depois da Fase 8):**
- Risco de país (I7) com os Worldwide Governance Indicators.
- Atualização mensal automática via GitHub Actions agendado.
- Mirror data para perceber a origem real do comércio intra-UE.
- Simulador de cenários ("e se a origem X cortar 50%?").
- Resumo em linguagem natural por produto, gerado por um LLM.

---

## 7. Regras de trabalho para o Claude

- **O Pedro escreve o código.** Está a reaprender a programar. O Claude explica o conceito, propõe a abordagem e a assinatura das funções, revê o código escrito e aponta erros. Só gera código completo quando o Pedro pedir explicitamente.
- Avançar **uma fase de cada vez** e confirmar o critério de "concluído" antes de passar à seguinte.
- Responder em **português de Portugal**. Nomes de código em inglês ou português, mas consistentes (decidir na Fase 0).
- Alterações cirúrgicas: indicar ficheiro, linha/bloco e o antes → depois. Não reescrever ficheiros inteiros sem necessidade.
- **Nunca assumir colunas, formatos ou códigos.** Verificar nos dados ou na documentação do Eurostat primeiro.
- Âmbito, limiares e pesos vivem em `config/`, nunca escritos à mão no código.
- Toda a lógica que entra no pipeline tem teste. Os notebooks são só para exploração.
- Ser simpático com a API do Eurostat: cache local, pedidos filtrados e pausas entre pedidos.
- Quando houver uma escolha de modelação (ex.: nível HS6 vs CN8), explicar o trade-off e deixar o Pedro decidir.

---

## 8. Limitações conhecidas (para o README)

- **Origem vs. proveniência:** o comércio intra-UE pode mascarar a origem real dos bens.
- **Dados confidenciais e provisórios:** os meses recentes são revistos.
- **Preço unitário (€/kg):** é uma aproximação. Mistura qualidades e variantes dentro do mesmo código.
- **Harmonização CN:** as mudanças de código introduzem descontinuidades que são tratadas, mas não eliminadas.
- **Âmbito do índice:** os indicadores medem exposição e concentração, não a probabilidade real de disrupção.
