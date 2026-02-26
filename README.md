# README — StratCore

## 🧱 BASELINE v1.0 — VALIDADO (ESTADO CONGELADO)

Este repositório encontra-se no estado **Baseline v1.0 — VALIDADO**.

Isso significa que os seguintes componentes foram **implementados, testados manualmente e validados de ponta a ponta**:

- STOP inicial
- Break-even (1R)
- TP1 (Partial Take Profit)
- TP2 (Final Take Profit)
- Trailing Stop (por R)
- BinanceFuturesAdapter (contrato estável)
- Persistência factual
- Guardrails arquiteturais

📌 **Regra de Ouro**  
Nenhuma funcionalidade nova pode ser adicionada sem:
1. Criar um **novo ciclo isolado**
2. Manter este baseline **inalterado**
3. Validar manualmente via **Tinker**
4. Documentar explicitamente no README

Alterações diretas neste estado são consideradas **regressão arquitetural**.

---

## 🎯 Objetivo do Sistema

StratCore é um sistema de trading automatizado com foco em **controle absoluto de risco, previsibilidade arquitetural e fidelidade à Binance**, oferecendo:

- Binance Futures USDT-M
- Stop Loss **gerenciado internamente**
- Break-even automático
- Trailing Stop (por R)
- Partial Take Profit (TP1 / TP2)
- Controle de risco diário e global
- Banco de dados refletindo **estado factual**, não intenção
- Execução manual via Frontend ou automações internas

---

## 🧨 O QUE ACONTECEU (CONTEXTO REAL)

O sistema **já estava funcionando corretamente**.

Durante uma sequência de ajustes e refatorações, o projeto entrou em instabilidade por causa de:

- múltiplas alterações simultâneas
- perda do contrato original do `BinanceFuturesAdapter`
- sobreposição de responsabilidades
- tentativa de “melhorar” sem isolar comportamento antigo
- alterações que **não respeitaram o estado anterior validado**

### Resultado prático observado
- Stops sendo cancelados sozinhos
- Ordens sumindo da Binance
- Ordens canceladas na Binance mas abertas no banco
- Posições abertas no banco com `positionAmt = 0`
- Erros de runtime mascarando erros arquiteturais
- Perda de previsibilidade e confiança

⚠️ **Nada disso foi falha de lógica de trading.**  
Foi **falha arquitetural e de processo**.

---

## 🚨 PROBLEMA CENTRAL IDENTIFICADO

> **Não houve isolamento do comportamento antigo estável.**

Tentamos:
- corrigir bugs
- melhorar arquitetura
- adicionar segurança

Tudo ao mesmo tempo.

👉 Isso quebrou o sistema.

---

## ❌ O QUE NÃO PODE MAIS ACONTECER

### 1️⃣ Nunca usar `closePosition=true` para STOP gerenciado

**Regra absoluta.**

- A Binance cancela ordens automaticamente
- Quebra:
  - Break-even
  - Trailing Stop
  - TP1 / TP2
- Remove o controle do sistema

✅ Stop gerenciado **sempre** deve ser:
- `STOP_MARKET`
- `reduceOnly = true`

---

### 2️⃣ Mais de um serviço mexendo em STOP / TP / fechamento

📌 **Regra definida:**
> Apenas **UM serviço** pode decidir e executar STOP / BE / TP / Trailing.

Esse serviço é o **`PositionStopManager`**.

Violação desta regra é **erro arquitetural**, não bug.

---

### 3️⃣ BinanceFuturesAdapter sem contrato estável

Erros reais que ocorreram no passado:

- Métodos inexistentes sendo chamados
- Imports incorretos
- Retornos inconsistentes
- Suposição de chaves inexistentes
- Mudança silenciosa de comportamento

👉 O Adapter deixou de ser previsível.

---

## ✅ O QUE FOI FEITO E VALIDADO (ESTADO ATUAL)

### 🔧 BinanceFuturesAdapter — VALIDADO E ISOLADO

- Contrato explícito
- Retornos consistentes
- Erros **não silenciosos**
- Testado em **Binance Demo/Testnet**

Métodos estáveis:
- `getAccountInfo`
- `getPosition`
- `getOpenOrders`
- `getMarkPrice`
- `placeOrder` (com simulação controlada)

📌 Diferença entre ambientes é tratada **exclusivamente no Adapter**.  
O domínio de trading é **100% agnóstico ao ambiente**.

---

### ▶️ Abertura de posição — VALIDADA

- Abertura via **Frontend**
- Execução via `TradeGuard` → `ExecutionEngine`
- Binance como verdade absoluta
- Registro correto no banco
- Testado manualmente (idempotente)

---

### 🔁 PositionSynchronizer — READ ONLY

- Sincroniza **apenas posições OPEN**
- Não cria posição
- Não cancela ordens
- Não cria stop
- Idempotente

---

### 🧠 Gestão de Posição — ISOLADA E CONTROLADA

- `PositionStopManager` como **writer único**
- Responsável por:
  - SL inicial
  - Break-even
  - TP1 / TP2
  - Trailing Stop
- Execução manual via **Tinker**
- Cada decisão gera um `TradeEvent` auditável

⚠️ Em Demo/Testnet:
- Ordens condicionais **não aparecem na Binance**
- Banco + eventos representam o **estado lógico validado**

---

## 🗄️ Persistência — CONTRATO VALIDADO

### `positions`
Tabela de **estado factual da posição**.

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
- last_stop_recreated_at

📌 **Nunca reflete intenção, apenas fatos ocorridos.**

---

### `trade_events`
Tabela de auditoria **imutável**.

Campos críticos:
- position_id
- action
- price
- reason
- snapshot
- meta_json
- created_at

📌 Cada decisão gera **exatamente um evento**.

---

## ✅ BREAK-EVEN (1R) — VALIDADO
- Dispara após 1R
- Cancela stop anterior
- Move stop para `entry_price`
- Evento `BREAK_EVEN_APPLIED`
- Sem loops ou ações duplicadas

---

## ✅ TP1 — PARTIAL TAKE PROFIT — VALIDADO
- MARKET + reduceOnly
- 50% do tamanho original
- Atualiza `remaining_size`
- Evento `TP1_APPLIED`
- Não altera stop ou BE

---

## ✅ TP2 — FINAL TAKE PROFIT — VALIDADO
- Executa somente após TP1
- Fecha 100% do `remaining_size`
- Marca posição como CLOSED
- Evento `TP2_APPLIED`

---

## ✅ TRAILING STOP (POR R) — VALIDADO

### Modelo
- Trailing discreto por R
- R = |entry_price - initial_stop|

### Parâmetros
- trailing_start_r = 3
- trailing_step_r = 1

### Regras
- Só inicia após Break-even
- Só com posição OPEN
- Evento `TRAILING_STARTED`
- Não move stop no START
- Move stop em degraus de 1R
- Nunca regride
- Evento `TRAILING_MOVED`

📌 Trailing não interfere em BE / TP1 / TP2.

---

## 🧱 Stack Técnica — Baseline v1.0

- PHP: 8.3.6
- Laravel: 12.51.0
- Livewire: 4.1.4
- Banco de Dados: MySQL / MariaDB
- Exchange: Binance Futures USDT-M

📌 Versões fazem parte do contrato.

---

## ⛔ Automação — DESLIGADA

- Cron desligado
- Scheduler inativo
- Nenhum Job ativo
- Execução manual via **Tinker**

---

## 📂 Arquivos Críticos (ANTI-REGRESSÃO)

Qualquer alteração exige **novo ciclo completo de validação**:

- BinanceFuturesAdapter.php
- ExecutionEngine.php
- TradeGuard.php
- PositionStopManager.php
- PositionSynchronizer.php
- Position.php
- TradeEvent.php

---

## 🧪 MODO DE TRABALHO (OBRIGATÓRIO)

```php
// Executa UMA decisão
app(PositionStopManager::class)->handle($position);

// Verifica estado factual
$position->refresh();
$position->toArray();

// Verifica auditoria
TradeEvent::where('position_id', $position->id)->get();
