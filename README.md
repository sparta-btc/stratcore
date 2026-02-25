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

Houve situações como:
- `BreakEvenManager`  
- `TrailingStopManager`  
- `PositionFailSafe`  
- `PartialCloseManager`  
- `Tp2Manager`  
- `OrderSynchronizer`  

👉 Resultado: **loops, cancelamentos e comportamento imprevisível.**

📌 **Regra definida:**  
> Apenas **UM serviço** pode decidir e executar STOP / BE / TP / Trailing.

---

### 3️⃣ BinanceFuturesAdapter sem contrato estável

Erros reais que ocorreram:

- Métodos inexistentes sendo chamados  
- Imports incorretos  
- Retornos inconsistentes  
- Suposição de chaves inexistentes  
- Mudança silenciosa de comportamento  

👉 O Adapter deixou de ser previsível.

---

## ✅ O QUE FOI FEITO (ESTADO ATUAL VALIDADO)

### 🔧 BinanceFuturesAdapter — VALIDADO E ISOLADO

- Contrato explícito  
- Retornos consistentes  
- Erros **não silenciosos**  
- Testado em **Binance Demo/Testnet**  
- Métodos estáveis:
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

### ▶️ Abertura de posição — ESTÁVEL

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
- Testado manualmente em infra real  

---

### 🧠 Gestão de Posição — EM ISOLAMENTO CONTROLADO

- `PositionStopManager` criado como **serviço único**  
- Responsável por:
  - SL inicial  
  - Break-even  
  - (futuro) TP1 / TP2  
  - (futuro) Trailing  
- Execução manual via Tinker durante validação  
- Gera `TradeEvent` **auditável**

⚠️ Em Demo/Testnet:
- Ordens condicionais **não aparecem na Binance**
- Banco + eventos representam o **estado lógico validado**
- Comportamento esperado e documentado

---

## 🧠 Guardrails Arquiteturais

### 🔒 Single Writer Rule (Regra do Escritor Único)

**Regra explícita e obrigatória do StratCore:**

> **Apenas o `PositionStopManager` pode criar, cancelar ou substituir ordens condicionais
(STOP, Break-even, TP, Trailing) de uma posição.**

Implicações diretas:

- Nenhum outro serviço pode:
  - criar STOP
  - mover STOP
  - cancelar STOP
  - executar TP parcial ou total
- `ExecutionEngine` **não decide** nada — apenas executa ordens quando solicitado.
- `PositionSynchronizer` é **read-only**.
- Frontend **nunca** cria ordens condicionais diretamente.

📌 Esta regra existe para:
- evitar loops de cancelamento/criação
- impedir estados divergentes entre banco e Binance
- garantir previsibilidade e auditabilidade
- proteger o sistema contra “ajustes rápidos” fora do fluxo correto

Qualquer violação desta regra é considerada **erro arquitetural**, não bug pontual.

---

### 🗄️ Persistência

- `Position.php`  
- `TradeEvent.php`  

Refletem **estado factual do sistema**, não intenção de trading.

---

### ⛔ Automação — DESLIGADA

Durante estabilização:

- Cron **desligado**  
- `schedule:run` **comentado**  
- Nenhum job, listener ou loop ativo  
- Execução **manual, previsível e auditável**

---

## 📂 Arquivos Críticos (Anti-Regressão)

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

## ⚠️ Regra de Transição (Anti-Regressão)

Nenhuma funcionalidade nova pode ser adicionada enquanto:

- `PositionStopManager` não estiver **totalmente validado**
- Não houver teste manual cobrindo:
  - SL inicial
  - Break-even
  - Cancelamento e recriação de stop
- Não existir `TradeEvent` explícito para **cada decisão**

---

📌 **Regra de Ouro Final**

Se uma ordem condicional **não aparece na Binance Demo/Testnet**:

> Verifique **primeiro o endpoint e o ambiente**  
> antes de suspeitar de lógica, cache ou sincronização.

---

## ▶️ PRÓXIMO PASSO (ATUAL)

### 🔥 Validar Break-Even (1R) de ponta a ponta

Objetivo imediato:

- Validar **exclusivamente** o Break-even  
- Sem TP  
- Sem Trailing  
- Sem automação  

Checklist obrigatório:

- [ ] SL inicial criado corretamente  
- [ ] Movimento ≥ 1R dispara BE  
- [ ] Stop anterior cancelado  
- [ ] Novo STOP no `entry_price`  
- [ ] `break_even_applied = true`  
- [ ] `TradeEvent::BREAK_EVEN_APPLIED` gerado  
- [ ] Nenhum loop  
- [ ] Nenhuma ação dupla  

📌 **Somente após isso**:
- TP1 será introduzido  
- Depois TP2  
- Por último Trailing  
- E **só então** automação será reativada  

