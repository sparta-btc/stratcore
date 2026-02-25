# README — StratCore

## 🎯 Objetivo do Sistema

StratCore é um sistema de trading automatizado com foco em **controle absoluto de risco, previsibilidade arquitetural e fidelidade à Binance**, oferecendo:

- Binance Futures USDT-M
- Stop Loss **gerenciado internamente**
- Break-even automático
- Trailing Stop
- Partial Take Profit (TP1 / TP2)
- Controle de risco diário e global
- Banco de dados refletindo **apenas fatos reais da Binance**
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
- `BreakEvenManager` criando stop
- `TrailingStopManager` criando stop
- `PositionFailSafe` recriando stop
- `PartialCloseManager` fechando parcial
- `Tp2Manager` ativando trailing
- `OrderSynchronizer` cancelando stop

👉 Resultado: **loops, cancelamentos e comportamento imprevisível.**

📌 **Regra definida:**  
> Apenas **UM serviço** pode decidir e executar STOP / BE / TP / Trailing.

---

### 3️⃣ BinanceFuturesAdapter sem contrato estável

Erros reais que ocorreram:

- Métodos chamados que não existiam (`getMarkPrice`)
- Imports incorretos
- Adapter retornando formatos diferentes
- Código assumindo chaves inexistentes
- Mudança silenciosa de comportamento

👉 O Adapter deixou de ser previsível.

---

## ✅ O QUE FOI FEITO (VALIDAÇÕES REAIS)

### 🔧 BinanceFuturesAdapter — VALIDADO
- Contrato explícito
- Retornos consistentes
- Erros não silenciosos
- Testado em **Binance Testnet**
- Métodos validados:
  - `getAccountInfo`
  - `getPosition`
  - `getOpenOrders`
  - `getMarkPrice`

---

### 🔁 PositionSynchronizer — READ ONLY
- Sincroniza **apenas posições OPEN já existentes no banco**
- Não cria posição
- Não cancela ordens
- Não cria stop
- Idempotente
- Testado manualmente em infra real

---

### ▶️ Abertura de posição — ESTÁVEL
- Abertura via **Frontend**
- Execução via `TradeGuard` + `ExecutionEngine`
- Registro correto no banco
- Binance como verdade absoluta
- Testado com:
  - abertura
  - sincronização
  - idempotência

---

### 🔒 Automação — DESLIGADA
Durante a fase de estabilização:

- Cron do sistema **desligado**
- `schedule:run` **comentado**
- Nenhum job, listener ou loop automático ativo
- Execução **100% manual e observável**

---

## 📌 Regra de Ouro — Controle de Posições

O StratCore é um sistema de trading com **controle fechado de domínio**.

- Apenas posições **abertas pelo StratCore** são gerenciadas.
- Isso inclui posições abertas:
  - via Frontend
  - via automações internas
  - via serviços do próprio sistema

Posições abertas fora do StratCore (painel da Binance, app mobile ou APIs externas)
**não são importadas nem gerenciadas**, mesmo que existam na exchange.

📌 Isso garante:
- previsibilidade
- ausência de loops
- controle total de risco
- consistência entre banco e Binance

---

## 🧱 ESTADO ATUAL DO SISTEMA

- ✔️ Core estável  
- ✔️ Adapter confiável  
- ✔️ Abertura de posição validada  
- ✔️ Sincronização idempotente  
- ✔️ Nenhuma automação oculta  
- ✔️ Nenhum serviço concorrente ativo  

👉 **Sistema pronto para evolução segura.**

---

## 📂 Arquivos Críticos Validados (Não Regredir)

Os arquivos abaixo foram **validados manualmente em infra real (Binance Testnet)** e constituem o **núcleo estável do StratCore**.  
Qualquer alteração nesses arquivos **exige novo ciclo completo de validação**.

### 🔧 Camada Exchange
- `app/Services/Exchange/BinanceFuturesAdapter.php`  
  - Contrato estável e explícito
  - Métodos validados:
    - `getAccountInfo`
    - `getPosition`
    - `getOpenOrders`
    - `getMarkPrice`
  - Binance tratada como **verdade absoluta**

---

### ▶️ Execução e Abertura de Posição
- `app/Services/Trading/TradeGuard.php`  
  - Validação de risco antes da execução
  - Contrato de preço explícito (markPrice como float)
- `app/Services/Trading/ExecutionEngine.php`  
  - Executor “burro”
  - Não decide estratégia
  - Apenas envia ordens para a Binance

---

### 🔁 Sincronização
- `app/Services/Trading/PositionSynchronizer.php`  
  - **Read-only**
  - Sincroniza apenas posições OPEN já existentes no banco
  - Não cria posição
  - Não cancela ordens
  - Idempotente

---

### 🧠 Gestão de Posição (Estado Atual)
- **Nenhum serviço automático de STOP / TP / Trailing ativo**
- Serviços antigos (`DynamicPositionManager`, `PartialCloseManager`, `Tp2Manager`)
  estão **desativados e fora do fluxo**

---

### 🗄️ Persistência
- `app/Models/Position.php`
- `app/Models/TradeEvent.php`

Esses modelos refletem **estado factual**, não intenção de trading.

---

### ⛔ Automação
- `routes/console.php`  
  - Scheduler **comentado**
- Cron do sistema **desligado**
- Nenhum job, listener ou loop automático ativo

---

📌 **Regra prática:**  
Se um bug aparecer, **o primeiro passo é verificar se algum desses arquivos foi alterado** sem validação completa.

## ⚠️ Regra de Transição (Anti-Regressão)

Durante a fase atual de estabilização arquitetural, **nenhuma funcionalidade nova pode ser adicionada** enquanto **TODAS** as condições abaixo não forem atendidas:

- O `PositionStopManager` **não estiver isolado** como serviço único de STOP / BE / TP / Trailing  
- **Não existir teste manual validado em Binance Testnet**, cobrindo:
  - SL inicial
  - Break-even
  - TP1 / TP2
  - Trailing
- **Não houver log explícito e auditável** para **cada decisão de stop**, incluindo:
  - motivo
  - preço
  - estado da posição
  - timestamp

📌 Esta regra existe para impedir regressões causadas por  
“apenas mais um ajuste rápido” fora de um ciclo completo de validação.

## ▶️ PRÓXIMO PASSO

### 🔥 Criar o `PositionStopManager`

Será criado um **serviço único**, responsável por:

- Criar SL inicial
- Aplicar Break-even
- Executar TP1 / TP2
- Ativar Trailing Stop
- Atualizar estado da posição
- Chamar **apenas** o `ExecutionEngine`

📌 Regras obrigatórias:
- Um único ponto de decisão
- Ordem fixa de execução
- Nenhum loop
- Nenhum outro serviço pode tocar em STOP / TP
- Testado primeiro na Binance Testnet
- Automação só será reativada após validação completa

---



Este README representa o **estado real e validado do sistema**  
e deve ser usado como **base obrigatória** para qualquer novo desenvolvimento.

🧠 Decisão Arquitetural do StratCore

Essa diferença é tratada exclusivamente no BinanceFuturesAdapter.

Nenhuma regra de ambiente vaza para:

PositionStopManager

ExecutionEngine

Frontend

O domínio de trading permanece agnóstico ao ambiente.

📌 Regra de Ouro

Se uma ordem condicional não aparece na Binance Demo/Testnet,
verifique primeiro o endpoint (/order vs /algo/order)
antes de suspeitar de lógica, cache ou sincronização.
