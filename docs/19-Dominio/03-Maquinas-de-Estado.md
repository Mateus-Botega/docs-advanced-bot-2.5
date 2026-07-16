# Máquinas de Estado

Este documento mapeia os diferentes estados das peças centrais do domínio. A transição estrita entre estes estados é vital para evitar bugs de concorrência ou ações impossíveis (como tentar andar estando desconectado).

## 1. Máquina de Estado da Sessão (Conexão e Jogo)

O ciclo de vida fundamental de um bot se conectando ao servidor.

```mermaid
stateDiagram-v2
    [*] --> Offline
    
    Offline --> Authenticating : Start() [Tem conta]
    Offline --> Connecting : Start() [Cracked]
    
    Authenticating --> Connecting : Success
    Authenticating --> ErroCritico : InvalidCredentials
    
    Connecting --> Handshaking : Socket Open
    Connecting --> Offline : Timeout/Refused
    
    Handshaking --> LoggingIn : Send Handshake
    
    LoggingIn --> Encryption : EncryptRequest
    LoggingIn --> Playing : JoinGame
    Encryption --> Playing : Encryption done + JoinGame
    
    Playing --> Disconnected : Kicked / Error / Logout
    Disconnected --> Connecting : Reconnect()
    Disconnected --> [*] : Stop()
    ErroCritico --> [*] : Stop()
```

- **Estados Inválidos/Proibidos**: É impossível passar de `Handshaking` direto para `Playing` sem o estado `LoggingIn`. O estado `Encryption` é opcional (depende do servidor/online-mode).
- **Impacto**: Enquanto a Sessão não estiver em `Playing`, a IA, os Comandos e o Inventário estão completamente bloqueados.

## 2. Máquina de Estado do Pathfinding (A*)

Máquina que rege a movimentação de um ponto A a um ponto B (como AutoMiner ou comando de Walk).

```mermaid
stateDiagram-v2
    [*] --> Idle
    
    Idle --> Calculating : SetDestination(target)
    
    Calculating --> FollowingPath : PathFound
    Calculating --> Error_NoPath : PathNotFound
    
    FollowingPath --> FollowingPath : ArrivedAtNode (move to next)
    FollowingPath --> ArrivedAtTarget : FinalNodeReached
    FollowingPath --> Calculating : PathBlocked / EntityCollision
    
    ArrivedAtTarget --> Idle : Clear()
    Error_NoPath --> Idle : Clear()
```

- **Invariante**: Se um bloco for alterado no Mundo (`BlockChanged` event) e esse bloco estiver no trajeto atual (`FollowingPath`), o sistema DEVE forçar a transição para `Calculating` para repensar a rota.

## 3. Máquina de Estado da Mineração (AutoMiner / QuebrarMadeira)

Lógica reativa ao ato de destruir um bloco no mundo.

```mermaid
stateDiagram-v2
    [*] --> Searching
    
    Searching --> WalkingToBlock : Ore Found
    Searching --> Idle : No Ores / End
    
    WalkingToBlock --> Breaking : In Range
    WalkingToBlock --> Searching : Block Unloaded
    
    Breaking --> WaitingBreakFinish : DiggingStart Sent
    
    WaitingBreakFinish --> Searching : Block Broken (Air)
    WaitingBreakFinish --> Searching : Break Timeout (15s)
    
    Idle --> [*]
```

- **Invariante**: É impossível iniciar `Breaking` se a distância entre o jogador e o bloco for superior ao alcance legítimo (4.5 a 5 blocos).

## 4. Máquina de Estado do Macro de Pesca (Solk)

A automação de pesca requer sincronia exata de estado.

```mermaid
stateDiagram-v2
    [*] --> EquipFishingRod
    
    EquipFishingRod --> CastingRod : Rod Equipped
    
    CastingRod --> WaitingForFish : Send UseItem (Throw)
    
    WaitingForFish --> ReelingIn : SplashSound / Velocity Drop
    WaitingForFish --> CastingRod : Timeout / Entity Destroyed
    
    ReelingIn --> Collecting : Send UseItem (Pull)
    
    Collecting --> EquipFishingRod : Pause (1-2s)
```

- **Falha Truncada**: Se em `WaitingForFish` o servidor enviar pacote para descarregar o Chunk da boia, a máquina volta imediatamente para o estado inicial para arremessar novamente.

---

## 5. Máquina de Estado do Farm de Mobs (`CommandMob` / Solk)

Automatiza farm de mobs com troca de ferramenta, venda de drops e reação a eventos de servidor.

```mermaid
stateDiagram-v2
    [*] --> RECEM_LOGOU

    RECEM_LOGOU --> VERIFICAR_INVENTARIO : após teleportes

    VERIFICAR_INVENTARIO --> TROCAR_FERRAMENTA : Durabilidade >= 1300 ou sem ferramenta
    VERIFICAR_INVENTARIO --> VENDER : < 5 slots vagos
    VERIFICAR_INVENTARIO --> MOB_NAO_ENCONTRADO : > 20 min sem venda
    VERIFICAR_INVENTARIO --> BUSCAR_MOB : estado normal

    BUSCAR_MOB --> VERIFICAR_POCAO : mob encontrado
    BUSCAR_MOB --> MOB_NAO_ENCONTRADO : 15 tentativas falhas

    VERIFICAR_POCAO --> MATAR_MOB : poção ok ou não aplicável

    MATAR_MOB --> VERIFICAR_INVENTARIO : após 850 hits
    MATAR_MOB --> TROCAR_FERRAMENTA : durabilidade estourou durante combate

    TROCAR_FERRAMENTA --> BUSCAR_MOB : troca bem-sucedida
    TROCAR_FERRAMENTA --> TROCAR_FERRAMENTA : baú cheio ou falha

    VENDER --> BUSCAR_MOB : após venda em todas as homes

    MOB_NAO_ENCONTRADO --> RECEM_LOGOU : após desbug (spawn + espera 15s)

    SERVIDOR_REINICIANDO --> RECEM_LOGOU : após Task.Delay(300000ms)
    OUTRA_CONTA_IDENTIFICOU_MOB_BUGOU --> RECEM_LOGOU : após spawn + espera 15s

    FINALIZANDO --> [*]
```

- **Estados de Bloqueio**: `SERVIDOR_REINICIANDO` e `OUTRA_CONTA_IDENTIFICOU_MOB_BUGOU` só transitam para `RECEM_LOGOU`. Tentativas de transição para outros estados são ignoradas em `alterarEstado()`.
- **Trigger externo**: transições de bloqueio são disparadas por `onReceiveChat()`, não pelo loop de tick.
- **Auto-reconnect**: se `!IsBeingTicked()` e `disconnectDelay + 10000 < agora`, chama `StartClient()` com espera de 10s.

---

## 6. Máquina de Estado da Pesca V2 (`CommandPescaV2` / Solk)

Versão refatorada da pesca com fase de inicialização de mapeamento de blocos e baús.

```mermaid
stateDiagram-v2
    [*] --> RECEM_LOGOU

    RECEM_LOGOU --> INICIANDO : após teleporte e pulos

    INICIANDO --> VERIFICAR_INVENTARIO : mapas de blocos e baús prontos

    VERIFICAR_INVENTARIO --> PESCAR : vara ok, linha ok, slots ok
    VERIFICAR_INVENTARIO --> REPARAR : vara com durabilidade >= 45
    VERIFICAR_INVENTARIO --> VERIFICAR_LINHA : falta de linha

    VERIFICAR_LINHA --> VERIFICAR_INVENTARIO : linha comprada

    PESCAR --> VERIFICAR_INVENTARIO : 1000 ciclos completados
    PESCAR --> REPARAR : durabilidade >= 45 durante pesca
    PESCAR --> VERIFICAR_INVENTARIO : vara nula ou não é vara

    REPARAR --> VERIFICAR_INVENTARIO : após 2 cliques no bloco de ferro

    ARMAZENAR --> VERIFICAR_INVENTARIO : itens guardados

    FINALIZANDO --> [*]
```

- **Fase de mapeamento**: A fase `INICIANDO` mapeia placas de linha, bloco d'água para pesca, bloco de ferro para reparo e todos os baús por categoria.
- **Bug documentado**: `buscarBausPorArea()` tem loop com condição invertida; as áreas de mapeamento nunca são varridas.

---

## 7. Máquina de Estado do Corner Glitch (`CommandMobTeleport` / Solk)

Macro experimental para atravessar paredes e atacar mobs através de colisões.

```mermaid
stateDiagram-v2
    [*] --> INICIANDO

    INICIANDO --> INICIANDO : mob não encontrado (loop com Task.Delay 5s)
    INICIANDO --> INICIANDO : Corner Glitch falhou (150 tentativas)
    INICIANDO --> FINALIZANDO : \n(após tentativa, com ou sem sucesso)

    FINALIZANDO --> [*]
```

- **Estado incompleto**: O ataque após sucesso do glitch está comentado no código. O estado `FINALIZANDO` é vazio (`async Task finalizando() {}`), efetivamente inoperante.
- **Sem persistência**: A macro reinicia do `INICIANDO` a cada tick (5s delay), sem memória de tentativas anteriores.
