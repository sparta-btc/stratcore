# 📘 README — StratCore

## 🧱 BASELINE v1.0 — VALIDADO  
## 🚀 CICLO v1.1 — AUTOMATION RUNNER (ESTADO CONTROLADO)

Este repositório encontra-se no estado **Baseline v1.0 — VALIDADO**, com o **Ciclo v1.1 — AutomationRunner** adicionado **sem alterar nenhuma lógica de trading**.

👉 O que muda no v1.1:
- Apenas **quem chama** o motor
- **Nada** muda em **quem decide**

Toda a lógica de decisão permanece **congelada, validada e auditável**.

---

## 📌 REGRA DE OURO (ANTI-REGRESSÃO)

Nenhuma funcionalidade nova pode ser adicionada sem:

1. Criar um **novo ciclo isolado**
2. Manter o baseline **inalterado**
3. Validar manualmente via **Tinker**
4. Documentar explicitamente no README

Qualquer violação é considerada **regressão arquitetural**, não bug.

---

## 🎯 Objetivo do Sistema

StratCore é um sistema de trading automatizado com foco em:

- **Controle absoluto de risco**
- **Previsibilidade arquitetural**
- **Fidelidade à Binance**

Funcionalidades:

- Binance Futures USDT-M
- Stop Loss **gerenciado internamente**
- Break-even automático (1R)
- Trailing Stop (por R)
- Partial Take Profit (TP1 / TP2)
- Controle de risco diário e global
- Banco refletindo **estado factual**, nunca intenção
- Execução manual ou automação **controlada**

---

## 🧨 CONTEXTO REAL (O QUE QUEBROU NO PASSADO)

O sistema já funcionava corretamente.

Quebrou por:
- múltiplas alterações simultâneas
- quebra de contrato do Adapter
- múltiplos writers de STOP / TP
- refatorações sem isolamento
- decisões misturadas com execução

Resultado:
- Stops cancelados sozinhos
- Ordens inconsistentes
- Posições inválidas
- Perda total de previsibilidade

⚠️ Falha **arquitetural e de processo**, não de trading.

---

## ❌ O QUE NÃO PODE MAIS ACONTECER

### 🔒 Single Writer Rule

> **Apenas o `PositionStopManager` pode criar, cancelar ou substituir**
> STOP / Break-even / TP / Trailing.

Violação = **erro arquitetural**.

---

### ❌ STOP com `closePosition=true`

**Proibido.**

- Binance cancela ordens automaticamente
- Quebra BE / Trailing / TP
- Remove controle do sistema

✅ STOP sempre:
- `STOP_MARKET`
- `reduceOnly = true`

---

## ✅ BASELINE v1.0 — O QUE FOI VALIDADO

### 🔧 BinanceFuturesAdapter

- Contrato explícito
- Retornos previsíveis
- Erros não silenciosos
- Testado em Demo/Testnet

Métodos estáveis:
- getAccountInfo
- getPosition
- getOpenOrders
- getMarkPrice
- placeOrder (simulado quando necessário)

📌 Ambiente tratado **somente no Adapter**  
Domínio de trading é **agnóstico ao ambiente**.

---

### ▶️ Abertura de Posição

- Via Frontend
- `TradeGuard → ExecutionEngine`
- Binance como verdade absoluta
- Persistência factual
- Testado manualmente

---

### 🔁 PositionSynchronizer (READ ONLY)

- Apenas sincroniza posições OPEN
- Não cria posição
- Não cancela ordens
- Não cria stop
- Idempotente

---

### 🧠 PositionStopManager (CORE)

**Único writer do sistema.**

Responsável por:
- STOP inicial
- Break-even
- TP1 / TP2
- Trailing Stop

Cada decisão:
- 1 chamada
- 1 ação
- 1 `TradeEvent`

---

## 🗄️ Persistência — CONTRATO

### `positions`
Reflete **somente fatos**.

Campos críticos:
- entry_price
- size / remaining_size
- initial_stop
- stop_price / current_stop
- stop_order_id
- break_even_applied / break_even_at
- tp1_price / tp1_applied / tp1_closed_at
- tp2_price / tp2_applied / tp2_closed_at
- trailing_active / trailing_started_at / last_trailing_stop
- status / state

---

### `trade_events`
Auditoria **imutável**.

- action
- price
- reason
- snapshot
- created_at

📌 Cada decisão gera **exatamente um evento**.

---

## ✅ LÓGICA DE TRADING VALIDADA

### Break-even (1R)
- Cancela stop anterior
- Move para entry
- Evento `BREAK_EVEN_APPLIED`

### TP1
- MARKET + reduceOnly
- 50% do size original
- Evento `TP1_APPLIED`

### TP2
- Fecha 100% do remaining_size
- Evento `TP2_APPLIED`
- Status = CLOSED

### Trailing Stop (por R)
- R = |entry - initial_stop|
- Start: 3R
- Step: 1R
- Nunca regride

Eventos:
- `TRAILING_STARTED`
- `TRAILING_MOVED`

---

## 🚀 CICLO v1.1 — AUTOMATION RUNNER

### Objetivo
Automatizar **quem chama** o motor, não **o que decide**.

### Fluxo
Scheduler / Cron
→ AutomationRunner
→ PositionStopManager::handle(position)


### Regras
AutomationRunner:
- PODE chamar `handle`
- NÃO PODE decidir nada
- NÃO PODE criar lógica
- NÃO PODE escrever estado

---

## ⚙️ Automação — CONTROLADA POR FLAG

```env
TRADING_AUTOMATION_ENABLED=false

### Regras
AutomationRunner:
- PODE chamar `handle`
- NÃO PODE decidir nada
- NÃO PODE criar lógica
- NÃO PODE escrever estado

---

## ⚙️ Automação — CONTROLADA POR FLAG

```env
TRADING_AUTOMATION_ENABLED=false

false (default)

Nenhuma decisão automática
Execução manual via Tinker / Frontend

true

AutomationRunner ativo
Mesma lógica
Mesmos eventos
Mesmo baseline

📌 Desligar a flag restaura o modo manual sem impacto.

## 🧪 MODO DE TRABALHO (OBRIGATÓRIO)

```php
// Executa UMA decisão
app(PositionStopManager::class)->handle($position);

// Verifica estado factual
$position->refresh();
$position->toArray();

// Verifica auditoria
TradeEvent::where('position_id', $position->id)->get();
