# README — StratCore

## 🎯 Objetivo do Sistema

StratCore é um sistema de trading automatizado com foco em **controle absoluto de risco, previsibilidade arquitetural e fidelidade à Binance**, oferecendo:

- Binance Futures USDT-M
- Stop Loss **gerenciado internamente**
- Break-even automático
- Trailing Stop
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

Essa foi a **principal causa do caos**.

📌 **Regra definida:**
> Apenas **UM serviço** pode decidir e executar STOP / BE / TP / Trailing.

Esse serviço é o **`PositionStopManager`**.

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

#### 📌 Decisão Arquitetural Importante

Diferença entre ambientes é tratada **exclusivamente no Adapter**:

| Ambiente | STOP / TP |
|--------|-----------|
| Mainnet | `/fapi/v1/order` |
| Demo/Testnet | **Simulação controlada** |

Nenhuma regra de ambiente vaza para:
- `PositionStopManager`
- `ExecutionEngine`
- Frontend

📌 **Domínio de trading é 100% agnóstico ao ambiente.**

---

### ▶️ Abertura de posição — VALIDADA

- Abertura via **Frontend**
- Execução via `TradeGuard` + `ExecutionEngine`
- Registro correto no banco
- Binance como verdade absoluta
- Testado manualmente:
  - abertura
  - sincronização
  - idempotência

---

### 🔁 PositionSynchronizer — READ ONLY

- Sincroniza **apenas posições OPEN já existentes no banco**
- Não cria posição
- Não cancela ordens
- Não cria stop
- Idempotente
- Testado manualmente

---

### 🧠 Gestão de Posição — ISOLADA E CONTROLADA

- `PositionStopManager` criado como **serviço único**
- Responsável por:
  - SL inicial
  - Break-even (**VALIDADO**)
  - (futuro) TP1 / TP2
  - (futuro) Trailing
- Execução manual via Tinker durante validação
- Cada decisão gera `TradeEvent` **auditável**

⚠️ Em Demo/Testnet:
- Ordens condicionais **não aparecem na Binance**
- Banco + eventos representam o **estado lógico validado**
- Comportamento esperado e documentado

---

## 🗄️ Persistência — CONTRATO VALIDADO

### Position
- `stop_order_id` persistido corretamente
- Reflete **estado factual**, não intenção

### TradeEvent
Cada decisão persiste:
- `action`
- `price`
- `reason`
- `snapshot`

📌 **Auditabilidade confirmada.**

---

## ✅ BREAK-EVEN (1R) — VALIDADO DE PONTA A PONTA

### Validação manual concluída

- SL inicial criado corretamente
- Movimento ≥ 1R dispara Break-even
- STOP anterior cancelado
- Novo STOP criado em `entry_price`
- `break_even_applied = true`
- `break_even_at` preenchido
- `stop_order_id` atualizado
- `TradeEvent::BREAK_EVEN_APPLIED` gerado
- Nenhum loop
- Nenhuma ação dupla

📌 Estado final validado no banco:
- `current_stop = entry_price`
- `stop_order_id = sim_...`
- Estado consistente e previsível

---

## 🧠 Guardrails Arquiteturais

### 🔒 Single Writer Rule (Regra do Escritor Único)

> **Apenas o `PositionStopManager` pode criar, cancelar ou substituir ordens condicionais
(STOP, Break-even, TP, Trailing) de uma posição.**

Implicações:
- `ExecutionEngine` **não decide**
- `PositionSynchronizer` é **read-only**
- Frontend **nunca cria ordens condicionais**

Violação desta regra é considerada **erro arquitetural**, não bug.

---

## ⛔ Automação — DESLIGADA

Durante estabilização:

- Cron desligado
- `schedule:run` comentado
- Nenhum job ativo
- Execução **manual, previsível e auditável**

---

## 📂 Arquivos Críticos (ANTI-REGRESSÃO)

Qualquer alteração exige **novo ciclo completo de validação**:

- `BinanceFuturesAdapter.php`
- `ExecutionEngine.php`
- `TradeGuard.php`
- `PositionStopManager.php`
- `PositionSynchronizer.php`
- `Position.php`
- `TradeEvent.php`

📌 **Regra prática:**
Se algo quebrar, **verifique primeiro esses arquivos**.

---

## 🧪 MODO DE TRABALHO (IMPORTANTE PARA NOVOS CHATS)

Este projeto **não evolui por tentativa e erro**.

### Forma oficial de trabalhar

- Um passo por vez
- Execução sempre manual
- Via **Tinker**
- Sempre com:
  - **1 comando**
  - **1 verificação**
  - **1 conclusão**

Padrão utilizado em todas as validações:

```php
// Executa UMA decisão
app(PositionStopManager::class)->handle($position);

// Verifica estado factual
$position->refresh();
$position->toArray();

// Verifica auditoria
TradeEvent::where('position_id', $position->id)->get();
