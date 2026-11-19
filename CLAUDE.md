# CLAUDE.md — Otimização de Campanhas de Marketing Bancário (ML + App)

> Brief do projeto para o Claude Code. Lê este ficheiro no início de cada sessão.
>
> **Estado atual:** Fase 0 em adaptação (setup, uv, testes e CI já funcionam).

---

## 1. Contexto e objetivo

**Problema de negócio:** um banco português faz campanhas de telemarketing para vender depósitos a prazo. Cada chamada custa tempo e dinheiro, e só uma pequena parte dos clientes contactados subscreve. Ligar a toda a gente é caro e pouco eficaz.

**Objetivo:** treinar um modelo de machine learning que estima a **probabilidade de cada cliente subscrever** e pô-lo numa app Streamlit que permite:
1. prever a probabilidade para um perfil de cliente;
2. **simular uma campanha:** "se ligarmos só aos X% com maior probabilidade, quantas subscrições conseguimos, quanto custa e qual o resultado líquido?".

**Mensagem para o portefólio:** "Não fico pela accuracy: traduzo um modelo de classificação numa decisão de negócio (quem contactar, quanto se poupa) e evito armadilhas como a fuga de informação."

---

## 2. Dados

- **Dataset:** UCI Bank Marketing, ficheiro `bank-additional-full.csv` (versão com contexto socioeconómico).
- **Origem:** campanhas de marketing direto (telefónicas) de uma instituição bancária portuguesa, de maio de 2008 a novembro de 2010. Os registos estão **ordenados por data**.
- **Dimensão:** 41 188 linhas, 20 variáveis de entrada + variável alvo `y` (subscreveu: yes/no).
- **Download:** manual, a partir do UCI Machine Learning Repository, para `data/raw/`. O ficheiro original nunca é alterado.
- **Citação obrigatória no README:** Moro, S., Cortez, P. & Rita, P. (2014). *A Data-Driven Approach to Predict the Success of Bank Telemarketing.* Decision Support Systems.

### Regra crítica: excluir `duration`
A variável `duration` (duração da última chamada) só é conhecida **depois** de a chamada acontecer, e nessa altura o resultado já se sabe. Usá-la é **fuga de informação (data leakage)**. Fica excluída do modelo final. Pode ser usada só num modelo de comparação, para mostrar o efeito no README.

### A confirmar na Fase 1 (não assumir)
- Separador do CSV (provavelmente `;`) e tipos de cada coluna.
- Proporção de "yes" na variável alvo (esperado: perto de 11%, ou seja, classes desequilibradas).
- Valores `"unknown"` nas variáveis categóricas: são dados em falta ou uma categoria com significado?
- `pdays = 999` significa "nunca contactado antes", não é um número real de dias.
- Variáveis socioeconómicas (`emp.var.rate`, `euribor3m`, etc.) dependem da data. Avaliar o risco de o modelo "aprender o período" em vez do cliente.

---

## 3. Abordagem de machine learning

### Divisão dos dados
- Separar logo no início um **conjunto de teste** que só se usa no fim.
- **Decisão a tomar na Fase 2 (com justificação no `docs/decisoes.md`):**
  - divisão **estratificada aleatória** (mais simples), ou
  - divisão **temporal** (treino nos meses mais antigos, teste nos mais recentes), que é mais realista porque os dados estão ordenados por data.

### Pré-processamento
- Tudo dentro de um `sklearn.Pipeline` com `ColumnTransformer`, para não haver fugas entre treino e teste:
  - categóricas → `OneHotEncoder(handle_unknown="ignore")`
  - numéricas → `StandardScaler` (para modelos lineares)
  - variável derivada: `contactado_antes` (a partir de `pdays`)

### Modelos (por esta ordem)
1. `DummyClassifier`: a referência mínima a bater.
2. `LogisticRegression` (com `class_weight="balanced"`): simples e interpretável.
3. `RandomForestClassifier` e `HistGradientBoostingClassifier`.

Usar validação cruzada estratificada no treino e só depois avaliar no teste.

### Métricas
A **accuracy não serve**: um modelo que diz sempre "no" acerta cerca de 89%. Usar:
- **PR-AUC** (principal) e ROC-AUC
- precisão e recall no limiar escolhido
- **curva de ganhos/lift:** % de subscrições captadas ao contactar os top X% de clientes

### Limiar de decisão
Não usar 0,5 por defeito. Escolher o limiar a partir da camada de negócio (secção 4).

### Explicabilidade
- Permutation importance (scikit-learn).
- SHAP como extensão opcional.

---

## 4. Camada de negócio — simulador de campanha

Parâmetros editáveis na app, com valores por defeito **hipotéticos e assumidos como tal**:
- custo por chamada (€)
- valor esperado de cada subscrição (€)
- capacidade de chamadas (nº máximo)

Para cada % de clientes contactados (ordenados por probabilidade):
- chamadas feitas, subscrições esperadas, custo, receita esperada, **resultado líquido**
- comparação com "ligar a todos" e com "ligar ao acaso"

Resultado: gráfico do resultado líquido vs. % contactados, com o ponto ótimo assinalado.

---

## 5. Arquitetura e estrutura

```
 bank-additional-full.csv
          │
          ▼
 src/bank_marketing/data.py        carregar + validar + limpar
          │
          ▼
 src/bank_marketing/features.py    variáveis derivadas + ColumnTransformer
          │
          ▼
 src/bank_marketing/train.py       treino, validação cruzada, escolha do modelo
          │                        → models/modelo.joblib + models/metricas.json
          ▼
 src/bank_marketing/evaluate.py    métricas, curva de ganhos, importância
          │
          ▼
 src/bank_marketing/business.py    simulador de campanha (funções puras, testadas)
          │
          ▼
 app/ (Streamlit)                  Visão geral · Prever cliente · Simulador · Sobre o modelo
```

### Estrutura de pastas

```
bank-campaign-targeting/
├── CLAUDE.md
├── README.md
├── pyproject.toml
├── config/
│   └── config.yaml          # caminhos, seed, variáveis excluídas, parâmetros do simulador
├── data/
│   ├── raw/                 # CSV original (no .gitignore)
│   └── processed/           # (no .gitignore)
├── models/                  # modelo treinado + métricas (o .joblib final pode ir para o Git se for pequeno)
├── notebooks/
│   ├── 01_eda.ipynb
│   └── 02_modelacao.ipynb   # só exploração; lógica final vai para src/
├── docs/
│   ├── decisoes.md          # registo de decisões (uma linha por decisão)
│   └── ideias/              # projetos futuros (ex.: monitor de risco de abastecimento)
├── src/bank_marketing/
│   ├── __init__.py
│   ├── data.py  features.py  train.py  evaluate.py  business.py
├── app/
│   ├── Home.py
│   └── pages/
└── tests/
    ├── test_data.py  test_features.py  test_business.py
```

### Stack
- Python 3.12 · uv · pandas · scikit-learn · joblib · Streamlit · Plotly · pyyaml
- pytest · ruff · GitHub Actions (já configurado)

---

## 6. Roadmap por fases

Estimativa para um ritmo de 1–3 h/dia (total: cerca de 3–4 semanas).

| Fase | Entregável | Concluído quando… | Estim. |
|---|---|---|---|
| **0. Adaptação do setup** | Repo renomeado, pacote `bank_marketing`, dependências de ML, CLAUDE.md novo | CI verde com a nova estrutura | 1 sessão |
| **1. EDA** | `01_eda.ipynb`: distribuição do alvo, variáveis vs. taxa de subscrição, `unknown`, `pdays`, efeito do tempo | 5–7 insights escritos em linguagem de negócio; pontos de atenção anotados em `docs/decisoes.md` | 3–5 dias |
| **2. Dados + pipeline + baseline** | `data.py`, `features.py` com testes; Dummy + Regressão Logística | Pipeline corre de ponta a ponta; baseline com PR-AUC registada; decisão sobre a divisão (aleatória vs. temporal) documentada | 4–6 dias |
| **3. Modelos + avaliação** | `train.py`, `evaluate.py`; comparação de modelos; curva de ganhos; importância das variáveis | Modelo escolhido com justificação; métricas no teste guardadas em `models/metricas.json`; comparação com/sem `duration` documentada | 4–6 dias |
| **4. Camada de negócio** | `business.py` com testes | Para parâmetros dados, o simulador devolve o % ótimo de clientes a contactar; testes com casos calculados à mão | 2–3 dias |
| **5. App Streamlit** | 4 páginas: Visão geral · Prever cliente · Simulador · Sobre o modelo | App corre localmente com o modelo guardado; parâmetros do simulador editáveis | 4–6 dias |
| **6. Portefólio** | README de consultor, deploy no Streamlit Community Cloud, screenshots | Link público funcional; README explica problema → abordagem → resultados → limitações | 2–3 dias |

---

## 7. Regras de trabalho para o Claude

- **O Pedro escreve o código.** Está a reaprender a programar. O Claude explica o conceito, propõe a abordagem e a assinatura das funções, revê o código escrito e aponta erros. Só gera código completo quando o Pedro pedir explicitamente. A configuração (YAML, CI, `.gitignore`) pode ser gerada.
- Avançar **uma fase de cada vez** e confirmar o critério de "concluído" antes de passar à seguinte.
- Responder em **português de Portugal**. Nomes de código em inglês ou português, mas consistentes.
- Alterações cirúrgicas: indicar ficheiro, linha/bloco e o antes → depois. Não reescrever ficheiros inteiros sem necessidade.
- **Nunca assumir colunas ou valores.** Verificar nos dados primeiro.
- Fixar a `random_state` (seed em `config.yaml`) para os resultados serem reprodutíveis.
- Toda a lógica que entra em `src/` tem teste. Os notebooks são só para exploração.
- Quando houver uma escolha de modelação, explicar o trade-off e deixar o Pedro decidir. Cada decisão fica numa linha em `docs/decisoes.md`.
- Correr scripts com `uv run`, nunca com o Code Runner.

---

## 8. Limitações conhecidas (para o README)

- Dados de 2008–2010 (período de crise financeira), de um só banco: o modelo não se generaliza automaticamente para hoje.
- Alguns atributos foram omitidos pelos autores por questões de privacidade.
- Os valores do simulador (custo por chamada, valor da subscrição) são hipotéticos.
- O modelo estima uma propensão, não uma causa: não diz porque é que o cliente subscreve.
