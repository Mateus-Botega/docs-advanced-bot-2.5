# Mapa de Eventos e Mensageria (Reatividade Arquitetural)

Este documento exaustivo cataloga e define a taxonomia de todos os eventos arquiteturais que transitavam de forma rígida (síncrona) no bot original em C# e prescreve como eles devem operar assincronamente na arquitetura Hexagonal em Java (Spring Boot/Reactor). 

O `AdvancedBot` passará de um "Bot Procedimental de Varredura (Polling)" para um "Sistema Orientado a Eventos (Event-Driven)". O motor da IA ficará dormente e despertará exclusivamente através da emissão dos eventos catalogados neste mapa.

---

## 1. O Paradigma: Acoplamento (C#) vs Reatividade (Java)

A principal causa de lentidão e falhas no bot legado residia na forma como o C# lidava com os *Triggers* (Gatilhos) do servidor Minecraft.

### 1.1 O Erro Fatal C#: Mutação Direta de Estado
No legado (`Handler18.cs`), a rotina de recebimento de pacote assumia a responsabilidade não apenas de decodificar o byte, mas de *processá-lo visualmente e fisicamente*.
```csharp
// Exemplo do legado C# (Acoplamento Crítico)
public void HandleBlockChange(Packet0x23 packet) {
    // 1. Mutação Imediata na Thread da Rede!
    Bot.World.SetBlock(packet.X, packet.Y, packet.Z, packet.ID);
    
    // 2. Chamada de UI (Travamento do Socket)
    Bot.MainForm.Log("O bloco " + packet.ID + " quebrou!");
    
    // 3. Despertar da Inteligência
    Bot.AutoMiner.OnBlockBroke(packet.X, packet.Y, packet.Z);
}
```
**O Problema**: Se o `AutoMiner` demorasse 20 milissegundos calculando um AStar novo, a Thread TCP do C# ficava travada, não lendo o próximo pacote (ex: o pacote da vida caindo), levando a mortes injustas ou Kick por `ReadTimeout`.

### 1.2 O Padrão Ouro Java: EventBus Lock-Free
No ecossistema Java, o *Network Pipeline* (Netty) tem apenas UMA função: Cuspir DTOs formatados num tubo (Bus).
```java
// O Padrão Java
public class BlockChangeDecoder extends MessageToMessageDecoder<ByteBuf> {
    @Override
    protected void decode(ChannelHandlerContext ctx, ByteBuf in, List<Object> out) {
        // 1. Lê os bytes rápido e cai fora
        BlockChangedEvent event = new BlockChangedEvent(x, y, z, id);
        
        // 2. Joga no tubo assíncrono (Fire and Forget)
        eventBus.publish(event);
    }
}
```
Todos os Agentes, Gerenciadores de Mundo e Consoles Websockets escutarão esse barramento em suas próprias *Virtual Threads* (Loom) ou via Schedulers.

---

## 2. Taxonomia 1: Eventos de Ciclo de Vida da Sessão

Eventos gerados pelo orquestrador de rede. Determinam se a máquina de estados principal (Tick) do bot deve rodar ou se deve entrar em suspensão (Hibernação).

### 2.1 `SessionConnectingEvent`
- **Gatilho**: Acionado pela API Rest quando o usuário pede `/api/bot/connect` ou pelo Auto-Reconnect.
- **Payload DTO**: `String ip`, `int port`, `String username`.
- **Comportamento C# Legado**: O `MinecraftClient.Connect()` bloqueava a thread principal com `socket.Connect()`. A UI freezava até dar erro ou sucesso.
- **Assinantes Java (Subscribers)**: 
  - `SessionLogListener`: Imprime "[INFO] Conectando a IP:Porta...".
  - `AuthManager`: Prepara o token Yggdrasil na memória caso o server peça.

### 2.2 `SessionHandshakeCompletedEvent`
- **Gatilho**: Quando o bot responde ao `SetCompression` com sucesso, envia a *ClientSettings* e recebe o pacote `0x01 (Join Game)`.
- **Payload DTO**: `int entityId` (ID do bot no mapa), `byte gamemode`, `byte dimension`.
- **Comportamento C# Legado**: Um emaranhado de booleanos (`IsLogged = true`) modificados na raça.
- **Assinantes Java**:
  - `TickOrchestrator`: Acorda o loop de 50ms (`onTick`). O Bot oficialmente abre os olhos.
  - `PlayerTracker`: Instancia o `PlayerVO` e assimila o `entityId` enviado pelo servidor.

### 2.3 `SessionDisconnectedEvent` (Morte da Conexão)
- **Gatilho**: Disparado pelo `ChannelInactive` do Netty ou pelo pacote explícito `0x40 (Disconnect/Kick)`.
- **Payload DTO**: `DisconnectReason reason`, `String kickMessage`.
- **Comportamento C# Legado**: Fechava as Threads na força bruta (`Thread.Abort()`), limpava matrizes, tentava salvar configurações, gerando as vezes uma `InvalidOperationException` se o socket fechar no meio do salvamento.
- **Assinantes Java**:
  - `TickOrchestrator`: Mata o loop principal, colocando todos os Agentes para dormir (`Agent.onDisable()`).
  - `WorldManager`: Desaloca os milhares de Chunks do `ConcurrentHashMap` da RAM para evitar Memory Leaks após DC.
  - `SessionLifecycleManager`: Avalia o `reason`. Se for um banimento temporário, agenda um Auto-Reconnect para dali a 5 minutos. Se for um ban permanente, notifica o usuário via Socket Web.

### 2.4 `KeepAlivePingEvent` e `KeepAlivePongEvent`
- **Gatilho**: O Minecraft envia um VarInt aleatório `0x00 (KeepAlive)`. O Bot é obrigado a devolver o mesmo id exato.
- **Payload DTO**: `long randomId`.
- **Comportamento C# Legado**: Respondia dentro do `Handler18.cs` e ignorava.
- **Assinantes Java**:
  - `PingTrackerService`: Se o tempo entre o Ping chegar e o servidor mandar outro for maior que 10.000ms, o Bot assume "Socket Timeout Silencioso" (Zombie Connection) e joga um `SessionDisconnectedEvent` para forçar o reinício limpo.


---

## 3. Taxonomia 2: Eventos de Mundo e Meio Ambiente

Tratam das mudanças físicas ao redor do Bot (Chunks) ou do próprio estado vital (Health). Estes são os eventos mais massivos e que ditam as ações baseadas na física (Gravity, Pathfinding).

### 3.1 `MapChunkReceivedEvent` e `MapChunkBulkEvent`
- **Gatilho**: Pacote `0x21` (Chunk Data) ou `0x26` (Map Chunk Bulk). Envia colunas colossais de blocos (16x256x16) compactadas em Zlib.
- **Payload DTO**: `int chunkX`, `int chunkZ`, `byte[] compressedData`, `int primaryBitMask`.
- **Comportamento C# Legado**: O `PacketStream` descompactava o Zlib *Síncronamente*. Se viesse um Chunk Bulk pesado, a latência do Bot congelava.
- **Assinantes Java**:
  - `WorldManager`: Descompacta o Zlib numa Thread Separada (Worker Thread) para não travar a IO do Netty. Quando a descompressão termina, salva o `ChunkVO` na matriz Concurrent.
  - `NavigationAgent`: Se o bot estava aguardando um Chunk carregar para saber se poderia pular num penhasco ou não, o `NavigationAgent` retoma o cálculo do AStar.

### 3.2 `BlockChangeEvent` e `MultiBlockChangeEvent`
- **Gatilho**: Pacote `0x23` (Block Change) ou `0x22` (Multi Block). Um jogador ou a redstone alterou um bloco.
- **Payload DTO**: `int x`, `int y`, `int z`, `short blockId`, `byte metadata`.
- **Comportamento C# Legado**: `World.SetBlock(x,y,z,id)` direto.
- **Assinantes Java**:
  - `WorldManager`: Atualiza sua matriz em `O(1)`.
  - `MiningAgent`: Avalia se as coordenadas do evento batem com o bloco que ele está minerando no momento. Se baterem e o `blockId == 0` (Ar), significa que o bot terminou de quebrar.

### 3.3 `UpdateHealthEvent`
- **Gatilho**: O servidor informa a vida do Bot, sua fome (Food) e saturação. Pacote `0x08`.
- **Payload DTO**: `float health`, `short food`, `float foodSaturation`.
- **Comportamento C# Legado**: Atualizava a interface WinForms com a barrinha vermelha. O Killaura analisava se `health < 10` para fugir.
- **Assinantes Java**:
  - `PlayerTracker`: Armazena o estado imutável.
  - `SurvivalAgent` (Nova IA sugerida): Observa a fome (`food <= 18`). Se for, dispara uma Intenção (`ActionIntent`) para o `TickOrchestrator` equipar uma sopa/maçã no slot principal e enviar pacote de *Consume Item*.

### 3.4 `TimeUpdateEvent`
- **Gatilho**: Pacote `0x03`. O servidor informa a idade do mundo e o relógio atual (Ciclo de Dia/Noite).
- **Payload DTO**: `long worldAge`, `long timeOfDay`.
- **Comportamento C# Legado**: O C# guardava na classe `World`, mas raramente usava.
- **Assinantes Java**:
  - `CombatAgent`: Zumbis e Esqueletos pegam fogo de dia. O Agente de Combate deve assinar o `TimeUpdateEvent` e, se `timeOfDay < 12000`, ajustar a prioridade de alvos (ignorar zumbis expostos ao Sol, pois morrerão sozinhos).


---

## 4. Taxonomia 3: Eventos de Entidades (Entity Tracking)

O ecossistema do Minecraft 1.8 é superpovoado. Uma *Mob Trap* (Fazenda de monstros) pode gerar milhares de eventos de entidades por segundo. O tratamento destes eventos no Java deve ser cirúrgico para evitar esgotamento de CPU.

### 4.1 `SpawnPlayerEvent` e `SpawnMobEvent`
- **Gatilho**: Pacotes `0x0C` (Player) e `0x0F` (Mob). O servidor informa que um novo ator entrou na Render Distance do Bot.
- **Payload DTO**: `int entityId`, `UUID uuid`, `int type`, `double x, y, z`, `byte pitch, yaw`, `EntityMetadata metadata`.
- **Comportamento C# Legado**: Adicionava no `Dictionary<int, Entity>`. O C# também possuía um problema onde Mobs podiam spawnar sem dados e crashar o bot.
- **Assinantes Java**:
  - `EntityTrackerService`: Valida o DTO, converte as coordenadas de *Fixed Point* (X / 32.0D) para Double padrão, instancia o `EntityVO` e adiciona ao mapa concorrente.
  - `CombatAgent`: Analisa instantaneamente o tipo (`type`). Se for um monstro hostil e estiver dentro de 4 blocos de distância, aciona a intenção `ActionIntent.AttackEntity`.

### 4.2 `EntityDestroyEvent` (Despawn)
- **Gatilho**: Pacote `0x13`. Um jogador desconectou, um zumbi morreu, ou um item sumiu do chão.
- **Payload DTO**: `int[] entityIds`. (Atenção: é um Array, o servidor pode pedir para matar 100 entidades num pacote só).
- **Comportamento C# Legado**: Fazia um `foreach` apagando do Dicionário. 
- **Assinantes Java**:
  - `EntityTrackerService`: Remove do mapa de rastreio para o Garbage Collector limpar a memória.
  - `CombatAgent`: Se a entidade alvo atual do Killaura estiver neste array, a IA limpa o alvo (Target = null) imediatamente, prevenindo que o Bot continue batendo no vazio (e tome banimento por cheat).

### 4.3 `EntityRelativeMoveEvent` e `EntityTeleportEvent`
- **Gatilho**: Pacotes `0x15`, `0x16`, `0x17` (Move/Look) e `0x18` (Teleport).
- **Payload DTO**: `int entityId`, `double dx, dy, dz` (Movimento relativo) ou `double x, y, z` (Movimento Absoluto).
- **A Fraqueza C#**: O AdvancedBot 2.4.5 atualizava a posição visual, mas a Física (`MPPlayer.ApplyPhysics`) ignorava colisão com outras entidades (Pushing).
- **Assinantes Java**:
  - `EntityTrackerService`: Atualiza o vetor de posição.
  - `PathfindingAgent` (Nova Inteligência): Mobs se movendo podem tapar a passagem. Se o evento informar que um Player (Adversário) pulou na frente do Bot, o Pathfinding deve recalcular a rota (Re-pathing).

### 4.4 `EntityVelocityEvent` (Knockback)
- **Gatilho**: Pacote `0x12`. Um zumbi bateu no bot, ou o bot bateu num zumbi.
- **Payload DTO**: `int entityId`, `short velX, velY, velZ`.
- **Comportamento C# Legado**: Se o `entityId` fosse igual ao ID do Bot, aplicava a velocidade no vetor do jogador. Se não fosse, ignorava.
- **Aprimoramento Java**:
  - `PhysicsEngine`: Recebe o Knockback (Repulsão) e soma no `MotionX/Y/Z` do Bot. Se o `AntiKnockback` (Hacker) estiver ativado no painel, o pacote é ativamente silenciado no Pipeline do Netty (não emite o Evento), forçando a Física a ignorar o golpe.

### 4.5 `EntityMetadataEvent`
- **Gatilho**: Pacote `0x1C`. O servidor avisa que uma entidade pegou fogo, agachou (Sneak), tomou poção de invisibilidade ou que um Creeper começou a inflar.
- **Payload DTO**: `int entityId`, `List<MetadataEntry> entries`.
- **Comportamento C# Legado**: O `DataWatcher.cs` era uma classe de 800 linhas confusa que lia Tipos primitivos baseados em índices não documentados.
- **Assinantes Java**:
  - `CombatAgent`: Evento Crítico. Observa a MetaData de "Vida da Entidade" (Índice 7 para o Ender Dragon/Wither) ou a flag "Está Invisível". Se o jogador inimigo sumir, o Killaura pausa.


---

## 5. Taxonomia 4: Eventos de UI, Chat e Application Level

Estes eventos não se originam do servidor de Minecraft, mas sim da própria aplicação (Agentes, Painel Web ou Motor JS). Eles gerenciam o fluxo de controle humano sobre a máquina.

### 5.1 `ChatMessageReceivedEvent` e `SystemMessageEvent`
- **Gatilho**: Pacote `0x02` (Chat Message). O servidor envia uma string JSON contendo o texto, a cor (ex: `§a` para verde) e a posição (Chat, System, ou Action Bar acima do inventário).
- **Payload DTO**: `String rawJson`, `String unformattedText`, `byte position`.
- **Comportamento C# Legado**: Chamava o `Invoke` da Thread de UI e exibia na TextBox. Também possuía um Regex rudimentar para Anti-AFK (respondendo a matemáticas no chat).
- **Assinantes Java**:
  - `WebConsoleService`: Envia via WebSocket para o FrontEnd (Vue.js).
  - `MacroEngine` (GraalVM): Exporá um hook `onChat(msg)`. Antigos scripts `.js` poderão assinar esse evento para responder comandos de donos (Ex: `!bot vem aqui`).
  - `AntiAfkAgent`: Captura testes de capcha do chat (Ex: "Qual a soma de 2+2?") e injeta um `SendChatIntent`.

### 5.2 `MacroToggleEvent` (Ativação/Desativação de IAs)
- **Gatilho**: O usuário aperta um botão no painel Web para ligar o "AutoMiner".
- **Payload DTO**: `String macroName`, `boolean state`.
- **Comportamento C# Legado**: O WinForms alterava a propriedade `command.Enabled = true`.
- **Assinantes Java**:
  - `AgentDispatcher`: Adiciona ou remove o Agente da lista ativa do `TickOrchestrator`. Se desativado, o dispatcher imediatamente chama `agent.onDisable()` garantindo que o AutoMiner pare de minerar, solte teclas virtuais e limpe alvos.

### 5.3 `PlayerIntentEvent` (Intenções do Orquestrador)
- **Gatilho**: A grande sacada arquitetural. Nenhum Agente no Java tem permissão para enviar pacotes de rede. Quando o Killaura quer bater em alguém, ele emite uma **Intenção** (`AttackEntityIntent`).
- **Payload DTO**: Depende da intenção (ex: `MoveIntent`, `BreakBlockIntent`).
- **Comportamento C# Legado**: Qualquer classe chamava `Bot.SendPacket()` a qualquer momento, causando o erro clássico de "Spam" e kick por *Too Many Packets*.
- **Assinantes Java**:
  - `IntentProcessorService`: Um funil (Queue). Ele recebe as intenções de todas as 15 IAs rodando ao mesmo tempo. Ele filtra redundâncias (ex: dois agentes querendo pular ao mesmo tempo) e converte a Intenção final aprovada em um `MinecraftPacket` despachado para a camada Netty.


---

## 6. Engenharia de Performance: O Motor de Eventos (Event Loop)

Migrar de chamadas síncronas (`Método()`) para Mensageria (`EventBus.publish()`) introduz um novo desafio: *Overhead* de processamento. Se uma *Mob Trap* gerar 5.000 pacotes de velocidade por segundo, instanciar 5.000 `EntityVelocityEvent` no Heap do Java forçará o Garbage Collector. 

### 6.1 A Arquitetura LMAX Disruptor (Zero-Allocation)
Para garantir que o Bot consiga manter 100 sessões ativas com menos de 300MB de RAM, o barramento de eventos principal da arquitetura Java usará os princípios do **Ring Buffer** (ex: LMAX Disruptor ou implementação otimizada do Reactor).

1. **Pré-Alocação**: O motor de eventos não instancia um `new BlockChangeEvent()` a cada pacote. Ele pré-aloca um array circular de 1024 slots de eventos genéricos vazios quando o Bot inicia.
2. **Mutação de Payload (Zero-GC)**: Quando o Netty decodifica um pacote, ele solicita um Evento livre no Ring Buffer, popula as variáveis (X, Y, Z, ID) através de Setters, e publica a sequência.
3. **Consumo Rápido**: Os agentes leem as variáveis e o evento retorna para a piscina livre. Nenhum objeto morre, o GC nunca aciona.

### 6.2 O Efeito Borboleta do Netty Threads
- **A Praga do Legado**: No C#, usar `NetworkStream.BeginRead()` disparava invocações no ThreadPool do Windows. Se a rede fosse rápida, o ThreadPool ficava exaurido (*Thread Starvation*) e os pacotes atrasavam.
- **A Solução Java**: O Netty possui o `EventLoopGroup` (geralmente fixado em 1 ou 2 threads por núcleo de processador). Essa thread minúscula fica girando em Loop Infinito processando Socket IO (NIO). 
  - **LEI DE OURO**: Nenhum `Agent` ou rotina do `WorldManager` tem permissão para rodar código pesado na Thread que publica o evento. Se o `BlockChangeEvent` for disparado, o ouvinte (`Subscriber`) DEVE repassar a carga para um *Worker Thread* próprio (ex: `Schedulers.parallel()`), caso contrário a Thread de IO do Netty será bloqueada, derrubando todas as conexões da JVM.

---

## 7. Apêndice A: Mapa de Roteamento de Pacotes Especiais (Exceções)

Existem pacotes de rede do Minecraft que não geram Eventos de Domínio, pois são rotinas intrínsecas à criptografia ou compressão de rede.

### 7.1 Pacote `0x46` (Set Compression)
- **Ação**: O servidor envia este pacote no meio do Handshake. 
- **Destino Java**: Não gera evento no EventBus. O Handler responsável dentro do Pipeline intercepta este pacote, extrai o *Threshold* inteiro, e silenciosamente adiciona os decodificadores `ZlibEncoder/ZlibDecoder` dinamicamente ao Pipeline do Netty para todos os pacotes subsequentes.

### 7.2 Pacote `0x00` (Login Disconnect) vs `0x40` (Play Disconnect)
- **Ação C#**: O C# os tratava com o mesmo painel de erro visual.
- **Destino Java**: 
  - `Login Disconnect`: Dispara `SessionLoginFailedEvent`. Causas comuns: Servidor lotado, Conta Yggdrasil banida, Manutenção, Nick inválido. O Bot *desiste* da sessão.
  - `Play Disconnect`: Dispara `SessionDisconnectedEvent`. Causas comuns: Reinício de servidor (Server Restart), Kick por Anti-AFK. O Bot *agenda auto-reconnect*.


---

## 8. Taxonomia 5: Eventos de Inventário e Transações (Apêndice B)

O sistema de inventário do Minecraft é um dos mais complexos do protocolo. Ele funciona como um banco de dados sincronizado entre Cliente e Servidor. Qualquer falha de sincronia ("Dessync") resulta no servidor ejetando os itens do bot no chão ou rejeitando cliques (Rubberbanding de inventário).

### 8.1 `WindowItemsEvent`
- **Gatilho**: Pacote `0x30`. Enviado logo após o login ou quando o bot abre um baú. Ele traz o estado completo da janela (todos os slots de uma vez).
- **Payload DTO**: `byte windowId`, `List<ItemStackVO> slots`.
- **Comportamento C# Legado**: O `Inventory.cs` sobrescrevia o array `Item[] slots` na raça. Se o bot estivesse executando um clique de macro no exato momento, ocorria uma *Race Condition*.
- **Assinantes Java**:
  - `InventoryManager`: Sobrescreve o inventário interno com um *Lock* de leitura rápida (Optimistic Lock). Emite imediatamente a flag `InventorySynced = true`.
  - `AutoArmorAgent`: Assina esse evento para, no momento em que o inventário estiver carregado e livre, avaliar se há armaduras melhores nos slots e iniciar a equipagem.

### 8.2 `SetSlotEvent`
- **Gatilho**: Pacote `0x2F`. O servidor atualiza um único slot (ex: o bot pegou um item do chão e ele foi parar no slot 36).
- **Payload DTO**: `byte windowId`, `short slot`, `ItemStackVO item`.
- **A Fraqueza C#**: Muitos scripts antigos (`.js`) tentavam ler o inventário em um *loop* `while(true)`. O C# não possuía um sistema reativo forte para avisar a Macro que o slot mudou, gerando loops famintos (CPU Spikes).
- **Assinantes Java**:
  - `InventoryManager`: Atualiza o slot específico.
  - `CraftingAgent`: Se a IA estava esperando madeira chegar no inventário para *craftar* gravetos, ela deve "dormir" até o `SetSlotEvent` ser disparado contendo `item.id == WOOD`. Quando acordada, prossegue o algoritmo de *craft*.

### 8.3 `ConfirmTransactionEvent`
- **Gatilho**: Pacote `0x32`. O **Santo Graal** da anti-detecção. Sempre que o Bot clica num slot (enviando o pacote `0x0E Click Window`), o servidor *avalia* se o clique foi legal. Se for, ele responde com `ConfirmTransaction (accepted = true)`.
- **Payload DTO**: `byte windowId`, `short actionNumber`, `boolean accepted`.
- **Comportamento C# Legado**: O Bot original era impaciente. Ele clicava e, sem esperar a confirmação, já movia o item para outro lugar. Isso criava *Phantom Items* e o Anti-Cheat acusava "FastClick".
- **Assinantes Java**:
  - `InventoryTransactionService`: Esta classe atua como um semáforo (Semaphore). Quando uma Macro emite um `InventoryClickIntent`, o serviço bloqueia cliques subsequentes para aquele `windowId`. Ele guarda o `actionNumber` gerado. Quando o `ConfirmTransactionEvent` chega, o serviço verifica se o `actionNumber` bate. 
    - Se `accepted == true`: Libera o semáforo para o próximo clique da IA.
    - Se `accepted == false`: Dispara um `InventoryDesyncEvent` alertando a IA de que o plano deu errado (o item não estava lá).

### 8.4 `OpenWindowEvent` e `CloseWindowEvent`
- **Gatilho**: Pacotes `0x2D` (Open) e `0x2E` (Close). O bot clicou num Baú (Chest), Fornalha (Furnace) ou Bancada (Crafting Table).
- **Payload DTO**: `byte windowId`, `String windowType`, `String windowTitle`, `byte numberOfSlots`.
- **Assinantes Java**:
  - `InventoryManager`: Expande o array de slots padrão (45 slots do jogador) adicionando o `numberOfSlots` do baú (ex: +27). 
  - `ChestStealerAgent`: IA Clássica de Minigames (SkyWars/HungerGames). Assim que o evento `OpenWindowEvent` de um baú é disparado, o Agente de Roubo inicia um loop de envio de intenções `InventoryClickIntent` (respeitando o `ConfirmTransaction` acima) para mover todos os itens de valor (Diamantes, Espadas) para o inventário do Bot.
  - `SessionLogListener`: Registra no painel web "Baú '§cTesouro' aberto (ID: 15)".


---

## 9. Apêndice C: Dicionário Exaustivo de Classes de Eventos (Código Java Proposto)

Para guiar o desenvolvedor Java na criação das classes do `EventBus`, a taxonomia acima deve ser traduzida para as abstrações exatas de código. Abaixo, fornecemos os *Records* (Java 16+) imutáveis que substituirão a passagem informal de parâmetros do antigo C#.

### 9.1 Base do Barramento (Interface Pai)
Todo evento no barramento herda desta interface de marcação.
```java
package com.advancedbot.app.events;

/**
 * Interface raiz para todos os eventos da arquitetura Hexagonal.
 * Qualquer classe que implemente isso pode ser postada no GlobalEventBus.
 */
public interface DomainEvent {
    // Retorna o timestamp de criação exato (para profiling de latência de fila)
    default long getTimestamp() {
        return System.currentTimeMillis();
    }
}
```

### 9.2 Rede e Conexão (Network Layer)
```java
package com.advancedbot.app.events.network;

import com.advancedbot.app.events.DomainEvent;
import java.util.UUID;

public record SessionConnectingEvent(
    String targetIp,
    int targetPort,
    String username
) implements DomainEvent {}

public record SessionHandshakeCompletedEvent(
    int entityId,
    UUID uuid,
    byte gamemode,
    byte dimension,
    byte difficulty,
    byte maxPlayers,
    String levelType,
    boolean reducedDebugInfo
) implements DomainEvent {}

public record SessionDisconnectedEvent(
    DisconnectReason reason,
    String kickMessageRawJson
) implements DomainEvent {
    public enum DisconnectReason {
        KICKED_BY_SERVER,
        SOCKET_TIMEOUT,
        PROTOCOL_ERROR,
        USER_ABORT
    }
}

public record KeepAlivePingEvent(
    long randomId
) implements DomainEvent {}
```

### 9.3 Física e Ambiente (World Layer)
```java
package com.advancedbot.app.events.world;

import com.advancedbot.app.events.DomainEvent;
import com.advancedbot.domain.world.ChunkVO;

public record MapChunkReceivedEvent(
    int chunkX,
    int chunkZ,
    boolean groundUpContinuous,
    int primaryBitMask,
    byte[] compressedData
) implements DomainEvent {}

public record BlockChangedEvent(
    int x,
    int y,
    int z,
    short newBlockId,
    byte newMetadata
) implements DomainEvent {}

public record MultiBlockChangeEvent(
    int chunkX,
    int chunkZ,
    int[] packedCoordinates,
    short[] blockIds,
    byte[] metadatas
) implements DomainEvent {}

public record TimeUpdateEvent(
    long worldAge,
    long timeOfDay
) implements DomainEvent {}

public record ExplosionEvent(
    float x,
    float y,
    float z,
    float radius,
    int[] destroyedBlockPositions,
    float playerVelocityX,
    float playerVelocityY,
    float playerVelocityZ
) implements DomainEvent {}
```

### 9.4 Entidades e Combate (Entity Layer)
```java
package com.advancedbot.app.events.entity;

import com.advancedbot.app.events.DomainEvent;
import com.advancedbot.domain.entity.EntityMetadata;
import java.util.List;
import java.util.UUID;

public record SpawnPlayerEvent(
    int entityId,
    UUID playerUuid,
    double x,
    double y,
    double z,
    byte yaw,
    byte pitch,
    short currentItem,
    List<EntityMetadata> metadata
) implements DomainEvent {}

public record SpawnMobEvent(
    int entityId,
    int type,
    double x,
    double y,
    double z,
    byte yaw,
    byte pitch,
    byte headPitch,
    short velocityX,
    short velocityY,
    short velocityZ,
    List<EntityMetadata> metadata
) implements DomainEvent {}

public record EntityDestroyEvent(
    int[] entityIds
) implements DomainEvent {}

public record EntityRelativeMoveEvent(
    int entityId,
    double dx,
    double dy,
    double dz,
    boolean onGround
) implements DomainEvent {}

public record EntityTeleportEvent(
    int entityId,
    double x,
    double y,
    double z,
    byte yaw,
    byte pitch,
    boolean onGround
) implements DomainEvent {}

public record EntityVelocityEvent(
    int entityId,
    short velX,
    short velY,
    short velZ
) implements DomainEvent {}

public record EntityMetadataEvent(
    int entityId,
    List<EntityMetadata> metadataEntries
) implements DomainEvent {}
```


---

## 10. Apêndice D: Hierarquia de Execução e Prioridade (`@Order`)

Quando o `EventBus` publica um `BlockChangedEvent`, cinco ou seis classes diferentes podem estar interessadas nessa informação. Se a macro de mineração processar o bloco quebrado *antes* da classe de física (Domínio) atualizar a matriz 3D, a macro tentará calcular uma rota passando por um bloco que a física ainda acha que é sólido.

Para prevenir *Race Conditions* lógicas na mesma Thread, o Spring Boot provê a anotação `@Order`. Abaixo está a hierarquia **obrigatória** de processamento de eventos do novo Bot.

### Nível 0: Camada de Domínio (Highest Precedence)
- **Quem**: `WorldManager`, `InventoryManager`, `EntityTrackerService`, `PlayerTracker`.
- **Anotação**: `@Order(Ordered.HIGHEST_PRECEDENCE)` ou `@Order(0)`.
- **Justificativa**: A Física e o Estado do Mundo têm o direito absoluto de atualizar seus arrays internos primeiro. Nenhuma IA tem permissão de tomar decisões antes do mundo estar perfeitamente refletido na RAM.

### Nível 1: Camada de Filtros e Semáforos
- **Quem**: `InventoryTransactionService`, `AntiKnockbackService`.
- **Anotação**: `@Order(100)`.
- **Justificativa**: Eles avaliam o estado já mutado pelo Nível 0 e liberam "Locks" (como o `actionNumber` do inventário) para que as macros saibam que estão autorizadas a agir.

### Nível 2: Camada de Navegação e Pathfinding
- **Quem**: `NavigationAgent`, `AStarPathfinder`.
- **Anotação**: `@Order(200)`.
- **Justificativa**: Se um bloco na frente do Bot mudou, o `NavigationAgent` deve recalcular a rota *antes* do Agente de Combate mandar o Bot andar pra frente.

### Nível 3: Camada de Inteligência (Agents)
- **Quem**: `CombatAgent` (Killaura), `MiningAgent` (AutoMiner), `FishingAgent`, Javascript Sandbox Macros.
- **Anotação**: `@Order(300)`.
- **Justificativa**: Estes são os últimos a opinar. Eles olham o Mundo (já validado pelo Nível 0 e roteado pelo Nível 2) e emitem as `ActionIntents`.

### Nível 4: Camada de UI e Logs (Lowest Precedence)
- **Quem**: `WebConsoleService`, `DiscordWebhookService`.
- **Anotação**: `@Order(Ordered.LOWEST_PRECEDENCE)`.
- **Justificativa**: Renderizar a cor vermelha numa TextBox ou enviar um JSON para o WebSocket de um painel web em Nuvem é lento. Eles esperam todo o cérebro processar a física e depois pintam o resultado na tela de quem estiver observando de fora.

### Exemplo de Implementação Spring:
```java
@Component
public class WorldUpdaterService {
    @EventListener
    @Order(0) // Nível 0
    public void onBlockChanged(BlockChangedEvent event) {
        worldManager.setBlock(event.x(), event.y(), event.z(), event.newBlockId());
    }
}

@Component
public class MiningAgent implements Agent {
    @EventListener
    @Order(300) // Nível 3
    public void onBlockChanged(BlockChangedEvent event) {
        if (this.currentMiningTarget.equals(event.x(), event.y(), event.z())) {
            if (event.newBlockId() == 0) { // Virou Ar
                this.stopMining();
                this.calculateNextVein();
            }
        }
    }
}
```


---

## 11. Apêndice E: O Contra-Fluxo (Action Intents)

Se os `Eventos` são as "Entradas" (Inputs) sensoriais que o Bot recebe do Mundo, os `Intents` (Intenções) são as "Saídas" (Outputs). No bot original, as saídas eram chamadas diretas ao Socket TCP (`Bot.SendPacket`). Na arquitetura Hexagonal, as IAs produzem Intenções, e a Infraestrutura decide se elas viram pacotes reais ou não.

### 11.1 A Interface Base de Intenção
```java
package com.advancedbot.app.intents;

import com.advancedbot.domain.ports.MinecraftPacket;

/**
 * Representa uma ação que uma Inteligência Artificial DESEJA tomar.
 * Intenções podem ser canceladas, combinadas ou postergadas pelo IntentProcessor.
 */
public interface ActionIntent {
    /**
     * Opcional: Define a prioridade desta intenção sobre outras emitidas no mesmo Tick.
     */
    default int getPriority() { return 50; }
    
    /**
     * Instrui o Adaptador de Rede sobre qual byte enviar.
     */
    MinecraftPacket toPacket();
}
```

### 11.2 Intenções de Combate e Movimento
```java
package com.advancedbot.app.intents.combat;

import com.advancedbot.app.intents.ActionIntent;
import com.advancedbot.domain.ports.MinecraftPacket;

public record AttackEntityIntent(
    int targetEntityId
) implements ActionIntent {
    @Override
    public MinecraftPacket toPacket() {
        return new Packet02UseEntity(targetEntityId, UseEntityAction.ATTACK);
    }
}

public record MoveIntent(
    double x,
    double y,
    double z,
    boolean onGround
) implements ActionIntent {
    @Override
    public int getPriority() { return 90; } // Movimento é crítico para não tomar ban

    @Override
    public MinecraftPacket toPacket() {
        return new Packet04PlayerPosition(x, y, z, onGround);
    }
}

public record LookIntent(
    float yaw,
    float pitch,
    boolean onGround
) implements ActionIntent {
    @Override
    public MinecraftPacket toPacket() {
        return new Packet05PlayerLook(yaw, pitch, onGround);
    }
}
```

### 11.3 Intenções de Interação com Mundo (Mineração)
```java
package com.advancedbot.app.intents.world;

import com.advancedbot.app.intents.ActionIntent;
import com.advancedbot.domain.ports.MinecraftPacket;

public record StartDiggingIntent(
    int x,
    int y,
    int z,
    byte face
) implements ActionIntent {
    @Override
    public MinecraftPacket toPacket() {
        return new Packet07PlayerDigging(Status.STARTED_DIGGING, x, y, z, face);
    }
}

public record FinishDiggingIntent(
    int x,
    int y,
    int z,
    byte face
) implements ActionIntent {
    @Override
    public MinecraftPacket toPacket() {
        return new Packet07PlayerDigging(Status.FINISHED_DIGGING, x, y, z, face);
    }
}

public record PlaceBlockIntent(
    int x,
    int y,
    int z,
    byte face,
    short heldItemId,
    byte cursorX,
    byte cursorY,
    byte cursorZ
) implements ActionIntent {
    @Override
    public MinecraftPacket toPacket() {
        return new Packet08PlayerBlockPlacement(x, y, z, face, heldItemId, cursorX, cursorY, cursorZ);
    }
}
```

### 11.4 Intenções de Inventário e UI
```java
package com.advancedbot.app.intents.inventory;

import com.advancedbot.app.intents.ActionIntent;
import com.advancedbot.domain.ports.MinecraftPacket;

public record InventoryClickIntent(
    byte windowId,
    short slot,
    byte button,
    short actionNumber,
    byte mode,
    short clickedItemId
) implements ActionIntent {
    @Override
    public int getPriority() { return 100; } // Cliques de inventário param todo o resto

    @Override
    public MinecraftPacket toPacket() {
        return new Packet0EClickWindow(windowId, slot, button, actionNumber, mode, clickedItemId);
    }
}

public record ChangeHeldItemIntent(
    short slotIndex // 0 a 8
) implements ActionIntent {
    @Override
    public MinecraftPacket toPacket() {
        return new Packet09HeldItemChange(slotIndex);
    }
}
```

### 11.5 Envio de Chat
```java
package com.advancedbot.app.intents.chat;

import com.advancedbot.app.intents.ActionIntent;
import com.advancedbot.domain.ports.MinecraftPacket;

public record SendChatIntent(
    String message
) implements ActionIntent {
    @Override
    public int getPriority() { return 10; } // Bate-papo pode esperar.

    @Override
    public MinecraftPacket toPacket() {
        // Truncar para 256 caracteres para evitar kick instantâneo
        String safeMsg = message.length() > 256 ? message.substring(0, 256) : message;
        return new Packet01ChatMessage(safeMsg);
    }
}
```


---

## 12. O Motor de Intenções (Intent Processor)

O último elo da cadeia reativa do bot Java é a conversão das vontades abstratas (Intents) em bytes na rede. Se deixássemos o Netty enviar pacotes exatamente no milissegundo em que as IAs pedem, o Minecraft nos baniria por excesso de pacotes (*Too Many Packets*).

### 12.1 O Buffer de Ações (Action Queue)
O `IntentProcessorService` possui uma `PriorityBlockingQueue`.
1. Durante o tick atual (ex: ms 0 a ms 49), os agentes de Combate, Mineração e Pathfinding publicam `ActionIntent`s.
2. Essas intenções caem na fila e são ordenadas pelo método `getPriority()`. Cliques no inventário têm precedência sobre movimento.

### 12.2 A Janela de Flush (Tick 50ms)
No exato momento em que o relógio interno do Java bate 50ms, o orquestrador chama `flushIntents()`.
```java
package com.advancedbot.app.orchestrator;

import com.advancedbot.app.intents.ActionIntent;
import com.advancedbot.domain.ports.ProtocolSenderPort;
import java.util.PriorityQueue;

public class IntentProcessorService {
    
    private final PriorityQueue<ActionIntent> intentQueue = new PriorityQueue<>(
        (a, b) -> Integer.compare(b.getPriority(), a.getPriority())
    );
    private final ProtocolSenderPort networkPort;
    
    public IntentProcessorService(ProtocolSenderPort networkPort) {
        this.networkPort = networkPort;
    }
    
    public void submitIntent(ActionIntent intent) {
        intentQueue.add(intent);
    }
    
    public void flushAllToNetwork() {
        // Limita a 10 pacotes físicos por tick para prevenir Banimentos.
        int flushLimit = 10;
        
        while (!intentQueue.isEmpty() && flushLimit > 0) {
            ActionIntent intent = intentQueue.poll();
            
            // Validações tardias (ex: bot morreu no meio do tick?)
            if (SessionContext.isDead()) {
                intentQueue.clear();
                return;
            }
            
            // Envia para o Netty (A Porta)
            networkPort.sendPacket(intent.toPacket());
            flushLimit--;
        }
        
        // Se sobrou na fila, descarta intenções de baixa prioridade 
        // ou carrega para o próximo tick.
        intentQueue.clear(); 
    }
}
```

### 12.3 Prevenção de Ghost Packets (Dessincronização Espacial)
No legado em C#, se o Killaura pedisse para bater, mas o `AStar` tivesse andado 5 blocos, as vezes o ataque era processado *antes* do movimento ser finalizado pelo servidor.
Com o funil de `ActionIntents`, garantimos via código que pacotes do tipo `Packet04PlayerPosition` (prioridade 90) sempre sejam enviados ao socket ANTES de pacotes `Packet02UseEntity` (prioridade 50). O servidor, que é sequencial, interpretará o ataque após atualizar a hitbox do bot.

---

## 13. Conclusão sobre Mensageria e Eventos

A transição arquitetural catalogada neste **Mapa de Eventos** encerra de uma vez por todas o principal pesadelo técnico do `AdvancedBot` 2.4.5: O Deadlock (congelamento) do cliente C# frente ao alto fluxo de rede de grandes servidores (Network I/O Bottleneck).

Ao tratar o mundo externo como uma fonte puramente passiva de *Inputs* (Eventos LMAX Disruptor zero-allocation) e obrigar os módulos de IA a conversarem através de *Outputs* abstratos supervisionados (Action Intents), a nova infraestrutura garante que:
1. Um pacote malformado enviado pelo servidor não quebra o loop principal de decisão da IA.
2. Uma Macro JavaScript escrita incorretamente por um usuário não trava o recebimento de novos pacotes de rede (pois rodam em Threads isoladas que não interagem com o Netty `EventLoopGroup`).
3. O bot age como um humano natural, limitando os bursts de pacotes por segundo (PPS) e respeitando as transações rigorosas de inventário (`actionNumber`), tornando os pacotes anti-hack do servidor completamente inúteis.


---

## 14. Apêndice F: Dicionário de Ticks de Resfriamento (Cooldowns)

Para complementar o motor de Intenções (`IntentProcessorService`), é imperativo registrar os atrasos exatos (Cooldowns) que o bot original usava para simular a física de um jogador humano. Sem esses delays, as Intenções encheriam a fila e o servidor baniria a conta por automação desumana.

### 14.1 Tabela de Cooldowns Obrigatórios
Cada "Tick" equivale a 50 milissegundos (20 TPS). Os agentes devem dormir por este tempo após emitirem a Intenção correspondente.

| Ação Emitida (Intent) | Cooldown (Ticks) | Tempo Real (ms) | Risco se Ignorado (Anti-Cheat) |
|---|---|---|---|
| `AttackEntityIntent` | 10 a 14 Ticks | 500ms - 700ms | Ban por *KillAura / FastHit* |
| `StartDiggingIntent` | Depende do Bloco | Variável | Ban por *FastBreak / Nuker* |
| `InventoryClickIntent` | 2 a 4 Ticks | 100ms - 200ms | Ban por *InventoryTweaks / FastClick* |
| `ChangeHeldItemIntent` | 1 Tick | 50ms | *Rubberbanding* de slot pelo servidor |
| `SendChatIntent` | 40 a 60 Ticks | 2000ms - 3000ms | Kick imediato por *Spam* |
| `ConsumeItemIntent` (Comer) | 32 Ticks estritos | 1600ms | Ban por *FastEat* |

### 14.2 Implementação do Delay na IA
No Java, em vez de `Thread.Sleep`, o Agente retém estado interno:
```java
public class CombatAgent implements Agent {
    private int hitCooldown = 0;
    
    @Override
    public List<ActionIntent> onTick(SessionContext ctx) {
        if (hitCooldown > 0) {
            hitCooldown--; // Decrementa a cada 50ms
            return List.of();
        }
        
        // Verifica alvos...
        if (target != null) {
            hitCooldown = 12; // Aguarda 600ms para bater de novo
            return List.of(new AttackEntityIntent(target.getId()));
        }
    }
}
```

> **Considerações Finais sobre Mensageria**: A transição para Eventos LMAX Disruptor e Intenções em Fila (Queue) mata completamente o *Spaghetti Code* do antigo C#. O bot passará de reações caóticas e imprevisíveis para um relógio suíço determinístico de 50ms, impenetrável aos Anti-Cheats modernos.

