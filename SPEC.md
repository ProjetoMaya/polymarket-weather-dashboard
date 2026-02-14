# SPEC — Motor de Detecção de Edge Mercados Meteorológicos Polymarket

## 1. Objetivo
Sistema determinístico para:
- Interpretar corretamente mercados meteorológicos do Polymarket
- Estimar probabilidade real do evento
- Calcular edge estatístico
- Determinar elegibilidade de trade
- Aplicar sizing com controle de risco

Prioridade: robustez estatística > agressividade.

---

## 2. Requisitos Funcionais

### 2.1 Market Rule Parser

#### 2.1.1 Extração de Campos
Extrair e validar:
- Data completa (YYYY-MM-DD)
- Ano explícito
- Local (cidade + estação)
- Métrica exata (max, min, mean)
- Fonte oficial de resolução
- Timezone oficial de resolução
- Tipo de intervalo:
  - Exato: "7°C"
  - Range: "6-8°C"
  - Threshold: "9°C or higher"

#### 2.1.2 Validação
Bloquear trade se:
- Ano não corresponder ao contrato
- Data estiver no passado
- Fonte de resolução não identificável
- Timezone indefinido
- Métrica ambígua

Flag: `market_rule_confidence = HIGH | MEDIUM | LOW`
- Trade permitido apenas se HIGH.

---

## 3. Data Integrity Layer

### 3.1 Timezone
- Nunca usar timezone automático
- Converter forecast para timezone explícito do contrato
- Garantir que dia meteorológico = dia de resolução
- **Erro fatal** se mismatch detectado

### 3.2 Date Matching
- Data do contrato = fonte primária
- Não assumir ano corrente

---

## 4. Modelo Estatístico

### 4.1 Inputs
- `μ_raw` = média dos modelos
- `bias_hist` = erro médio histórico
- `σ_hist` = desvio padrão histórico
- `σ_model_disp` = desvio entre modelos
- `horizon_days`

### 4.2 Correção de Média
```
μ_corrigido = μ_raw + bias_hist
```

### 4.3 Cálculo de Sigma Final
```
sigma_final = sqrt(σ_hist² + σ_model_disp² + σ_horizon_penalty²)
```
Onde:
- `σ_horizon_penalty = base_sigma * horizon_days * k`
- `k` calibrado via backtest
- **Não usar piso fixo arbitrário**

---

## 5. Probabilidade

### 5.1 Distribuição
- D+1 e D+2: Normal
- Horizontes >2: Requer Monte Carlo

### 5.2 Conversão de Contrato para Intervalo

| Tipo | Tratamento |
|------|------------|
| "X°C" | P(X) = P(X-0.5 ≤ T < X+0.5) |
| "X-Y°C" | P = P(X ≤ T ≤ Y) — sem expansão ±0.5 |
| "X°C or higher" | P = P(T ≥ X) |
| "X°C or below" | P = P(T ≤ X) |

---

## 6. Edge Calculation

### 6.1 Probabilidade Implícita
```
prob_market = preço_yes
```

### 6.2 Edge
```
edge = prob_real - prob_market
```

### 6.3 Valor Esperado
```
EV = (prob_real × 1) - preço_pago
```

---

## 7. Critério de Trade

Permitir trade apenas se:
- `edge ≥ 0.10` (10%)
- `market_rule_confidence == HIGH`
- `model_alignment == TRUE`
- `σ_model_disp ≤ limite_definido`

---

## 8. Position Sizing

Kelly fracionado:
```
f = edge / odds
position_size = f × 0.25
```
Limite máximo: 2% do capital total.

---

## 9. Backtesting (Obrigatório)

- Mínimo 60 dias históricos
- Separar treino e validação
- Calcular: EV médio, Sharpe, Max drawdown, Taxa de acerto

**Ativar apenas se:**
- `EV_validacao ≥ 5%`
- `MaxDrawdown ≤ limite_definido`

---

## 10. Testes Unitários (Obrigatórios)

- Parsing correto de ranges
- Conversão correta de timezone
- Cálculo correto de CDF
- Cálculo correto de EV
- Bloqueio de mercados ambíguos

**Nenhum ajuste estatístico pode ser commitado sem teste associado.**

---

## 11. Estrutura Recomendada

```
/core
  probability_engine.ts
  sigma_engine.ts
  rule_parser.ts
  edge_calculator.ts
/data
  historical_error.json
/backtest
  simulate.ts
/tests
  rule_parser.test.ts
  probability_engine.test.ts
  edge_calculator.test.ts
/docs
  statistical_assumptions.md
```

---

## 12. Princípios de Engenharia

1. **Nunca assumir regra implícita**
2. **Nunca usar sigma fixo arbitrário**
3. **Nunca permitir trade sem validação semântica**
4. **Nunca ajustar modelo apenas para melhorar backtest**
5. Separar: Estimativa | Decisão | Execução

---

## 13. Filosofia Operacional

- Ser conservador na estimativa
- Ser seletivo nos trades
- Priorizar repetibilidade
- Aceitar perder operações individuais
- Maximizar EV no longo prazo
