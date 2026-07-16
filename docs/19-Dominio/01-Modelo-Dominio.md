# Modelo de Domínio (Core Concepts)

Este documento abstrai a implementação do AdvancedBot, focando exclusivamente nos **conceitos do domínio**. O objetivo é servir de fundação para a reescrita (ex: em Java), definindo *o que* o sistema é, ignorando detalhes de rede (TCP, `NetworkStream`) ou plataforma (.NET).

---

## 1. Sessão (Session)

### Definição e Objetivo
O agregado raiz do bot. Representa a existência de um "cliente Minecraft virtual" ativo. Uma Sessão agrupa todas as informações e o estado de um jogador específico logado em um servidor específico.

### Responsabilidades
- Gerenciar o ciclo de vida da conexão com o servidor.
- Coordenar a comunicação entre o Tradutor de Protocolo, o Mundo e a Automação.
- Orquestrar as rotinas de tick ("batimento cardíaco" do jogo).

### Ciclo de vida e Estados possíveis
Nasce quando o usuário solicita o início de um bot.
*Estados*: `Offline` -> `Authenticating` -> `Connecting` -> `Handshaking` -> `LoggingIn` -> `Playing` -> `Disconnected`.

### Relações com outros conceitos
- **Possui 1** `Player` (estado do próprio avatar).
- **Possui 1** `World` (visão local do mapa).
- **Possui 1** `ProtocolTranslator` (ponte de comunicação).
- **Possui N** `Agents` (automações ativas).

### Eventos (Pub/Sub)
- **Produz**: `SessionStarted`, `SessionConnected`, `SessionDisconnected`, `TickEmitted`.
- **Consome**: Nenhum (ela é o emissor raiz de tempo/tick).

### Regras de negócio, Invariantes e Restrições
- *Invariante*: O estado de `Playing` só é atingido após a validação criptográfica (se exigida) e o pacote de `JoinGame`.
- *Restrição*: Duas sessões nunca podem compartilhar estado de Mundo ou Inventário.
- *Dependências*: Proxy Checker, Account Manager.

### Possível representação em Java
```java
public interface Session {
    SessionId getId();
    SessionState getState();
    Player getPlayer();
    World getWorld();
    void tick();
    void connect();
    void disconnect();
}
```

---

## 2. Mundo (World)

### Definição e Objetivo
A representação estritamente local (client-side) do universo ao redor do jogador. É um mapa tridimensional esparso composto de Chunks e Entidades.

### Responsabilidades
- Armazenar blocos, sinais (signs), e biomas ao alcance da visão (render distance).
- Indexar Entidades ativas que o servidor notificou.
- Fornecer resolução de colisões e busca de caminhos (raycasting) para a IA.

### Ciclo de vida e Estados possíveis
Nasce vazio no estado `Playing`. É limpo totalmente quando a sessão muda de dimensão (Nether/End) ou desconecta.

### Relações com outros conceitos
- **Contém N** `Chunks`.
- **Contém N** `Entities`.
- **É lido por** `Agents` (IA) para tomada de decisão.

### Eventos (Pub/Sub)
- **Produz**: `BlockChanged`, `EntitySpawned`, `EntityDespawned`, `EntityMoved`.
- **Consome**: `ChunkDataReceived`, `BlockUpdateReceived`, `UnloadChunkReceived` (vindas do Protocolo).

### Regras de negócio, Invariantes e Restrições
- *Invariante*: Um bloco fora dos chunks carregados é considerado *Air* ou Indeterminado (dependendo do comportamento de pathfinding).
- *Restrição*: O Mundo não envia pacotes para o servidor. Ele é um modelo de *leitura e reação*.

### Possível representação em Java
```java
public class World {
    private final Map<ChunkCoordinates, Chunk> chunks;
    private final Map<EntityId, Entity> entities;
    
    public Block getBlockAt(Vector3i position);
    public List<AABB> getCollisionBoxes(AABB area);
}
```

---

## 3. Jogador (Player / Self)

### Definição e Objetivo
O estado específico do bot no mundo. Diferencia-se das outras entidades por ser a única capaz de sofrer ações deliberadas de input (mover, clicar, falar).

### Responsabilidades
- Manter posição (X, Y, Z), rotação (Yaw, Pitch) e status vitais (Vida, Fome).
- Gerenciar o `Inventory` (Inventário do jogador).
- Aplicar gravidade e colisões (Física Client-Side) sobre si mesmo.

### Eventos (Pub/Sub)
- **Produz**: `PlayerMoved`, `PlayerHealthChanged`, `PlayerDied`, `InventoryUpdated`.
- **Consome**: `PositionLookPushedFromServer`, `HealthUpdateReceived`.

### Possível representação em Java
```java
public class Player extends Entity {
    private Inventory inventory;
    private float health;
    private int foodLevel;
    
    public void move(Vector3d offset);
    public void jump();
}
```

---

## 4. Agente / Automação (Agent / Macro)

### Definição e Objetivo
O cérebro reativo e proativo que substitui a interação humana. Pode ser desde um comando simples ("ande até X") até rotinas complexas ("minere madeira, guarde no baú e repita").

### Responsabilidades
- Observar o `World` e o `Player`.
- Decidir a próxima ação (Quebrar, Andar, Bater, Comer) em um dado `Tick`.
- Emitir inputs simulados (Movimento, Interações de Bloco, Interações de Entidade).

### Ciclo de vida e Estados possíveis
- *Estados*: `Idle` -> `Running` -> `Suspended` (ex: quando morre ou volta pro hub) -> `Finished`.

### Relações com outros conceitos
- **Inspeciona** `World` e `Player`.
- **Emite comandos via** `ProtocolTranslator` (implicitamente, alterando estado que será serializado).

### Regras de negócio, Invariantes e Restrições
- *Restrição*: Agentes não podem trapacear limites da física do jogo (ex: minerar a 20 blocos de distância), sob pena de expulsão pelo servidor.

### Possível representação em Java
```java
public interface Agent {
    void onTick(SessionContext ctx);
    void start();
    void stop();
    AgentState getState();
}
```

---

## 5. Tradutor de Protocolo (Protocol Translator)

### Definição e Objetivo
A camada de isolamento (Anti-Corruption Layer) entre o Domínio do Bot e os bytes da rede.

### Responsabilidades
- Converter a intenção do jogador (ex: "Clique no Bloco X") para o formato específico da versão (ex: 1.5.2 vs 1.8).
- Converter bytes do servidor em mutações no `World` e `Player`.

### Ciclo de vida e Estados possíveis
Criado logo após a Sessão descobrir a versão correta. Seu estado reflete o aperto de mãos da rede (`Handshake`, `Login`, `Play`).

### Eventos (Pub/Sub)
- **Produz**: Eventos de Domínio (`BlockChanged`, `PlayerMoved`, `ChatMessageReceived`).
- **Consome**: Comandos da Sessão (`SendChatMessage`, `SendPlayerMove`).

### Possível representação em Java
```java
public interface ProtocolTranslator {
    void handleIncomingBytes(ByteBuffer buffer);
    void sendAction(DomainAction action);
}
```
