# Mapa de Dependências Cruzadas e Injeção (Grafo de Arquitetura)

Este documento é vital para garantir que a equipe de engenharia não recrie o pesadelo arquitetural do C# (Código Espaguete) no novo ambiente Java. O `AdvancedBot 2.4.5` era conhecido por possuir **Dependências Circulares Mortais**, onde o *Core* do bot conhecia a Interface Gráfica, e a Interface Gráfica conhecia os Plugins de IA, tornando a manutenção de código um campo minado.

O objetivo primário deste mapa é catalogar o grafo de acoplamento do legado e definir a topologia estrita de **Injeção de Dependências (DI)** que regerá o Java sob o paradigma Hexagonal (Ports & Adapters) usando o framework **Spring Boot**.

---

## 1. O Anti-Pattern Legado: A Dependência Circular C#

Antes de projetar o futuro, precisamos entender exatamente por que a arquitetura antiga falhou. O `AdvancedBot` foi estruturado inicialmente como um projeto `Windows Forms Application` clássico (Monolítico).

### 1.1 O "God Object" (Objeto Deus) `MPPlayer`
A classe `MPPlayer` (Minecraft Protocol Player) detinha referências explícitas (Hardcoded) para praticamente tudo.
```csharp
// Exemplo real do C# Legado
public class MPPlayer {
    public Form1 GUI;           // O core conhece a UI!
    public World World;
    public AutoMiner Miner;     // O core conhece o Plugin de IA!
    public PacketStream Net;
    
    public void OnTick() {
        this.Net.SendPacket(...);
        this.GUI.UpdateLog("Tick"); // Acoplamento Fatal
        this.Miner.DoMining();
    }
}
```

### 1.2 A Colisão de Responsabilidades
Quando o usuário queria rodar o bot no modo "Console" (Sem interface gráfica) para hospedar numa VPS Linux via Mono, o programa quebrava na inicialização (`NullReferenceException`), pois o `MPPlayer` tentava instanciar o `Form1` e o Linux não possuía bibliotecas X11/WinForms disponíveis.

**As consequências da amarração cruzada no C#**:
- **Impossibilidade de Testes Unitários**: Não se podia testar a física de pulo (`MPPlayer.ApplyPhysics()`) sem instanciar uma tela WinForms.
- **Vazamento de Memória (Memory Leak)**: A UI guardava referências dos Bots. Quando um Bot desconectava, o `Form1` segurava a instância do `MPPlayer`, impedindo o Garbage Collector de limpá-lo. Após reconectar 50 vezes, a RAM enchia e o C# crashava com `OutOfMemoryException`.

---

## 2. A Solução Java: Arquitetura Hexagonal (Ports & Adapters)

A nova regra de ouro do `AdvancedBot` é: **Dependências só apontam para dentro**. O módulo de domínio é cego para o mundo exterior.

### 2.1 A Cebola Arquitetural
1. **Núcleo (Domain)**: `WorldManager`, `Inventory`, `Physics`. Nenhuma biblioteca externa permitida. Não conhece Spring, Netty, nem Vue.js. Usa apenas puro Java (`java.util`, `java.lang`).
2. **Camada de Aplicação (Ports)**: `SessionManager`, `IntentProcessor`, Interfaces de IAs (`Agent`). Define o que o bot *pode* fazer.
3. **Adaptadores Externos (Adapters)**: `NettyNetworkAdapter` (fala com a Internet), `SpringRestUIAdapter` (fala com o Painel de Controle), `GraalVMPluginAdapter` (fala com Javascript).

### 2.2 O Grafo de Inversão de Controle (IoC)
Se a Física (Núcleo) não conhece o Netty, como ela envia um pacote de movimento?
A Física *emite* um `ActionIntent`. O Adaptador Netty *escuta* esse Intent via `EventBus` e o converte para Bytes. O Núcleo não faz ideia de como seus desejos são atendidos na rede.

---

## 3. Matriz de Injeção de Dependência (Spring Boot)

Para abolir os famigerados `new Class()` espalhados pelo código e a aberração arquitetural do padrão Singleton (`MPPlayer.Instance`), adotamos o Contêiner IoC do Spring Framework.

### 3.1 Escopos de Beans
A vida de um Bot é volátil. Ele conecta, desconecta, reconecta. Portanto, o escopo das dependências é rigorosamente classificado:

| Componente | Escopo (Spring Scope) | Justificativa de Escopo |
|---|---|---|
| `BotOrchestrator` | `@Singleton` | Só existe um Orquestrador para gerenciar múltiplos Bots. Fica vivo desde que a JVM ligou até ser desligada. |
| `BotRegistry` | `@Singleton` | Mapa concorrente (Cache) que armazena todos os bots atualmente ativos em memória. |
| `ConfigManager` | `@Singleton` | Lê o `application.yml` apenas uma vez. |
| `NettyServerContext` | `@Singleton` | Compartilha os mesmos `NioEventLoopGroup` para todas as sessões para poupar CPU. |
| `BotSession` | `@Prototype` | Uma instância inteiramente nova a cada conta logada. Se 50 bots ligarem, existirão 50 Beans deste. |
| `WorldManager` | `@Prototype` (vinculado à Sessão) | Cada bot vê o mundo de forma ligeiramente diferente (especialmente a posição dos Chunks carregados ao seu redor). |
| `InventoryManager` | `@Prototype` (vinculado à Sessão) | Cada bot tem sua própria mochila. |
| `WalkingAgent` | `@Prototype` (vinculado à Sessão) | Cada bot tem sua própria instância de Pathfinding rodando paralela aos outros. |

### 3.2 Como Evitar Vazamentos (Prototype vs Singleton)
O `BotRegistry` (Singleton) guarda os `BotSession` (Prototypes).
Quando um bot é deletado pelo usuário no painel WEB, o `BotRegistry` deve remover o `BotSession` de seu `ConcurrentHashMap`. Como o Spring não gerencia a destruição de Prototypes automaticamente de coleções Singleton, se esquecermos de chamar `.remove()`, a JVM reterá o `BotSession`, o Netty, o World e as IAs para sempre (Memory Leak gravíssimo).


---

## 4. O Cinto de Castidade do Maven (Multi-Module)

No C# Legado, o projeto inteiro era compilado como um único `AdvancedBot.exe`. Isso permitia que qualquer classe importasse (`using`) qualquer outra classe.
No Java, o Maven será usado como uma ferramenta de **Castidade Arquitetural**. Se o módulo de Rede tentar importar o módulo de UI, o Maven explodirá o Build com um `Circular Dependency Error` antes mesmo de compilar.

### 4.1 `advancedbot-domain` (O Módulo Isolado)
**Dependências Permitidas (POM)**:
- Nenhuma (Nenhum JAR externo, nem Spring, nem Netty). Ocasionalmente Lombok para acelerar criação de `Records` e Builders, e bibliotecas matemáticas puras.
**Contém**: Toda a regra de negócio do Minecraft (AStar, Killaura, Física, NBT).
**Responsabilidade**: Ser testável em frações de milissegundo pelo JUnit, sem mockar nada de infraestrutura.

### 4.2 `advancedbot-app-core` (A Orquestração)
**Dependências Permitidas (POM)**:
- `advancedbot-domain`
- `spring-boot-starter` (Contêiner IoC)
- `spring-context`
**Contém**: `TickOrchestrator`, `AgentDispatcher`, Filas de Intenções (`IntentQueue`), Registro de Sessões.
**Responsabilidade**: Unir o Domínio com a Infraestrutura (Portas). Aqui é onde a mágica do Loop de 50ms acontece. Este módulo injeta as interfaces do Domínio para fazê-lo ganhar vida.

### 4.3 `advancedbot-infra-network` (A Porta TCP)
**Dependências Permitidas (POM)**:
- `advancedbot-app-core` (Apenas para disparar Eventos)
- `netty-all` (Framework de IO)
- `gson` (Tratamento de Chat JSON do Minecraft)
**Responsabilidade**: O Parser de Netty. Converte ByteBuffer em `MinecraftPacket` e injeta no `EventBus`.

### 4.4 `advancedbot-ui-rest` (A Porta HTTP/WebFlux)
**Dependências Permitidas (POM)**:
- `advancedbot-app-core`
- `spring-boot-starter-webflux`
**Responsabilidade**: Expoe Endpoints (`/api/bot/start`) para o Painel de Controle (Dashboard). Consome os dados do `BotRegistry` e os joga no barramento SSE (Server-Sent Events) para a Web.

### 4.5 `advancedbot-infra-plugins` (A Porta de Scripts)
**Dependências Permitidas (POM)**:
- `advancedbot-app-core`
- `graalvm-js` (Motor JavaScript polyglot)
**Responsabilidade**: Compilar o código `.js` feito pelo usuário em tempo de execução e ligar os Eventos do Bot nas funções JS (`function onChat(msg)`).

### 4.6 A Regra do `Parent POM`
Haverá um `pom.xml` raiz (Parent) que define a versão de todos os sub-módulos e dependências externas para garantir que não haja conflitos de ClassPath (o "Dependency Hell" do Java).
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.0</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
        <dependency>
            <groupId>io.netty</groupId>
            <artifactId>netty-bom</artifactId>
            <version>4.1.100.Final</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

---

## 5. Anatomia de Injeção de uma IA (Bot Agent)

Como uma Inteligência Artificial do Bot (`Agent`) é injetada no sistema se o Bot pode nascer e morrer dezenas de vezes?
Usaremos Padrão de Fábrica (Factory Pattern) associado ao Contexto do Spring.

### 5.1 O Erro do Legado (O Factory Rígido)
```csharp
// C# Antigo - Fortemente Acoplado e não extensível
public class AgentFactory {
    public static IAgent Create(string type) {
        if(type == "AutoMiner") return new AutoMiner();
        if(type == "Killaura") return new Killaura();
        return null;
    }
}
```
Isso violava o Princípio Aberto/Fechado (Open/Closed Principle) do SOLID. Toda vez que alguém criava um bot novo, tinha que alterar essa classe.

### 5.2 A Resolução Java: Autowired Lists (Descoberta Automática)
O Spring fará o Auto-Discovery de todas as classes que assinam a interface `AgentFactory`.

```java
package com.advancedbot.core.agents;

import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;
import java.util.function.Function;
import java.util.stream.Collectors;

@Service
public class AgentOrchestrator {
    
    // O Spring pega todas as implementações de AgentFactory do ClassPath 
    // e injeta nesta lista magicamente.
    private final Map<String, AgentFactory> availableFactories;

    public AgentOrchestrator(List<AgentFactory> factories) {
        this.availableFactories = factories.stream()
            .collect(Collectors.toMap(AgentFactory::getAgentName, Function.identity()));
    }

    /**
     * O usuário clica na Web: "Ligar Killaura". 
     * O WebFlux chama este método passando "Killaura".
     */
    public Agent instantiateAgentForSession(String agentName, BotSession session) {
        AgentFactory factory = availableFactories.get(agentName);
        if (factory == null) {
            throw new IllegalArgumentException("Agent Plugin não instalado: " + agentName);
        }
        return factory.createInstance(session);
    }
}
```
**O Resultado**: Se você quiser criar um novo Agente "AutoPesca", basta criar uma classe `AutoPescaFactory`, colocar a anotação `@Component`, e o Spring a plugará no sistema sem que você precise tocar em nenhuma outra classe. Total Desacoplamento.


---

## 6. O Grafo de Dependências do Motor de Física (Domain Layer)

O coração do `AdvancedBot` é o cálculo matemático do mundo. No legado, o `World` dependia do `Player`, o `Player` dependia do `World`, e a IA dependia de ambos. Isso impedia que a gente simulasse um "Mundo Falso" na memória para testes.

No Java, implementamos um grafo Direcional Acíclico (DAG).

### 6.1 `WorldManager` (A Raiz Absoluta)
Não depende de ninguém (exceto de VOs como `ChunkVO`).
```java
package com.advancedbot.domain.world;

public class WorldManager {
    // 0 injeções. Ele é puro.
    public WorldManager() { }
}
```

### 6.2 `EntityTracker` (O Visualizador)
Depende do `WorldManager`. O rastreador precisa saber se uma entidade que spawnou está dentro de uma parede, por exemplo.
```java
package com.advancedbot.domain.entity;

public class EntityTracker {
    private final WorldManager world;
    
    // Injeção via Construtor. Impossível criar um Tracker sem o World.
    public EntityTracker(WorldManager world) {
        this.world = world;
    }
}
```

### 6.3 `SessionContext` (O Agregador de Leitura)
Quando o `TickOrchestrator` vai acordar uma IA (Agent), ele não passa os "Managers" diretamente. Ele cria um `Record` imutável com as dependências apenas-leitura.
```java
package com.advancedbot.domain;

// Um Record Java (Imutável). A IA não pode fazer `session.world = null`.
public record SessionContext(
    String botUsername,
    WorldManager world,
    EntityTracker entities,
    InventoryManager inventory,
    BotState currentState
) {}
```

### 6.4 `AStarPathfinder` (O Calculador Periférico)
O algoritmo de Pathfinding precisa conhecer o mundo, mas o mundo não precisa conhecer o Pathfinding.
```java
package com.advancedbot.domain.pathfinding;

public class AStarPathfinder {
    private final WorldManager world;

    // Injeção de dependência pura.
    public AStarPathfinder(WorldManager world) {
        this.world = world;
    }
    
    public CompletableFuture<Path> calculate(Point3D start, Point3D end) { ... }
}
```

### 6.5 A Inversão de Controle do Output (Como o Domínio Fala?)
Se o Domínio (`WorldManager`, `AStar`) não conhece o Netty (Módulo de Infraestrutura), o que acontece quando a IA decide que o bot precisa dar um pulo?

**O Anti-Pattern Legado**:
```csharp
// Dentro do AStar.cs (Regra de Negócio)
MPPlayer.Net.SendPacket(new PacketPlayerPosition(...)); // Crime arquitetural
```

**A Arquitetura Nova (Ports & Adapters)**:
O Agente apenas *retorna* uma lista de Intenções (`ActionIntent`). Ele não executa nada. Quem executa é o `IntentProcessorService` (que mora na borda da Aplicação e conhece o Netty).

```java
// O Agente (Domínio puro)
public class WalkingAgent implements Agent {
    @Override
    public List<ActionIntent> onTick(SessionContext ctx) {
        if (precisaPular) {
            // Retorna um DTO. Não envia pacote. Não conhece socket.
            return List.of(new MoveIntent(x, y + 1.2, z, false)); 
        }
    }
}
```

---

## 7. O Ciclo de Vida do Grafo (Bootstrapping)

Quando o administrador roda o comando `java -jar bot.jar`, a ordem exata de inicialização do grafo de dependências pelo Spring Boot é milimetricamente calculada para evitar "BeanCurrentlyInCreationException".

### 7.1 Fase 1: Beans Universais (Nível de Aplicação)
1. **ConfigManager**: Lê os Yaml/JSONs. Depende do FileSystem.
2. **NettyServerContext**: Inicia as Threads do NioEventLoop. Não depende de ninguem do bot.
3. **AgentOrchestrator**: Vasculha o ClassPath achando todas as IAs.
4. **BotRegistry**: Inicia vazio. É o catálogo de Sessões.
5. **WebFlux Controllers**: Inicia o painel HTTP na porta 8080. Eles dependem do `BotRegistry` para mostrar quantos bots estão online.

### 7.2 Fase 2: Instanciação de um Novo Bot (Sessão)
O usuário clica no painel WEB: "Conectar Bot 'Notch'". Ocorre o Scoping Prototype:

1. O Controller WEB chama `BotFactory.create("Notch")`.
2. A Factory instancia (usando o Spring) os Prototypes da Sessão:
   - Instancia `WorldManager`.
   - Instancia `EntityTracker` (injetando `WorldManager`).
   - Instancia `InventoryManager`.
   - Instancia `EventBus` (Um barramento exclusivo para este bot).
3. A Factory une essas instâncias em um objeto `BotSession`.
4. A Factory guarda o `BotSession` no `BotRegistry` global.
5. A Factory chama `session.connect()`.

### 7.3 Fase 3: Conexão e Destruição (Lifecycle Binding)
Quando o `connect()` é chamado, o adaptador Netty é instanciado.
```java
// NettyAdapter.java (Módulo de Infraestrutura)
public void connect(String ip, int port, EventBus sessionEventBus) {
    Bootstrap b = new Bootstrap();
    b.group(globalNettyContext.getEventLoopGroup())
     .channel(NioSocketChannel.class)
     .handler(new ChannelInitializer<SocketChannel>() {
         @Override
         public void initChannel(SocketChannel ch) {
             // O Pipeline recebe o EventBus DAQUELA sessão específica.
             ch.pipeline().addLast(new MinecraftPacketDecoder(sessionEventBus));
         }
     });
}
```
**A Desconstrução Segura**: Quando o Socket fecha (Timeout ou Kick), o Netty joga um `SessionDisconnectedEvent` no Barramento. 
O `BotSession` ouve isso, para o seu próprio loop (`ScheduledExecutor.shutdown()`), avisa o `BotRegistry` para deletá-lo do cache central, e suas instâncias ficam órfãs para o ZGC limpar. Sem vazamentos.


---

## 8. Apêndice A: Mapa Visual Textual do Grafo (PlantUML Style)

Para facilitar a leitura da arquitetura, abaixo está a representação em árvore das dependências. A seta `-->` significa "Conhece e Chama", a seta `<..` significa "Emite Eventos para".
Note que NENHUMA seta aponta da direita (Infra) para a esquerda (Domínio), apenas Interfaces (Ports) são injetadas.

```text
[Módulo: advancedbot-domain] (Nível 0 - Core)
  |-- WorldManager
  |-- InventoryManager
  |-- PhysicsEngine --> WorldManager
  |-- AStarPathfinder --> WorldManager
  |-- EntityTracker --> WorldManager
  |-- ActionIntent (DTO)
  |-- Agent (Interface)
  
[Módulo: advancedbot-app-core] (Nível 1 - Orquestração)
  |-- TickOrchestrator --> Agent (Chama onTick)
  |-- TickOrchestrator --> IntentProcessor
  |-- IntentProcessor --> NetworkPort (Interface)
  |-- SessionContext (Agregador)
  |-- EventBus (Recebe e Despacha)
  |-- AgentOrchestrator --> [Todas as Factories de Agents]
  
[Módulo: advancedbot-infra-network] (Nível 2 - Adaptador de Rede)
  |-- NettyAdapter (Implementa NetworkPort)
  |-- MinecraftPacketDecoder ..> EventBus (Publica Eventos)
  |-- MinecraftPacketEncoder <-- IntentProcessor
  
[Módulo: advancedbot-ui-rest] (Nível 2 - Adaptador de Painel)
  |-- BotController --> BotRegistry
  |-- BotController --> AgentOrchestrator (Para ligar IAs)
  |-- WebConsoleStream ..> EventBus (Ouve mensagens de Chat)
  
[Módulo: advancedbot-infra-plugins] (Nível 2 - Adaptador JS)
  |-- GraalVMContext --> EventBus (Injeta hooks JS no barramento)
  |-- JSAgent (Implementa Agent) --> IntentQueue
```

---

## 9. Apêndice B: A Camada de Anti-Corrupção (ACL)

O `AdvancedBot 2.4.5` C# possuía rotinas matemáticas mágicas cujas origens foram perdidas (fórmulas de colisão de Bounding Boxes, predição de queda de flechas). Ao portar isso para Java, há o risco de o novo código "poluir" o domínio limpo com variáveis espaguete.

### 9.1 Isolando Códigos Legados (O Padrão Façade)
Se a equipe não conseguir refatorar a classe `Physics.cs` imediatamente, ela deve ser portada "as-is" para o Java e encapsulada num **Adapter**.

**O Código C# Horrível (Que vamos copiar para Java temporariamente)**:
```java
// Legado encapsulado. Não usamos nomes Java-friendly aqui.
class LegacyPhysicsCalculator {
    public static double[] calc(double x, double y, double z, boolean og) {
        // ... 500 linhas de matemática if/else ilegíveis ...
        return new double[]{newX, newY, newZ};
    }
}
```

**O Wrapper de Anti-Corrupção**:
```java
package com.advancedbot.domain.physics;

import org.springframework.stereotype.Component;

/**
 * ACL (Anti-Corruption Layer). Protege o nosso código limpo
 * da matemática suja do antigo C#.
 */
@Component
public class PhysicsEngine {
    
    public Point3D applyGravityAndFriction(Point3D currentPos, boolean isOnGround) {
        // Escondemos o legado aqui dentro. 
        // O resto do Bot chama applyGravityAndFriction e recebe um Point3D limpo.
        double[] raw = LegacyPhysicsCalculator.calc(
            currentPos.x(), currentPos.y(), currentPos.z(), isOnGround
        );
        return new Point3D(raw[0], raw[1], raw[2]);
    }
}
```
**Efeito Arquitetural**: Quando tivermos tempo para reescrever a matemática, mudaremos apenas a `PhysicsEngine`. Nenhuma IA ou Módulo de Rede perceberá a mudança, pois a interface pública `Point3D` manteve-se constante.


---

## 10. Apêndice C: Padrão Pub/Sub (O EventBus Interno)

A ferramenta principal que permite a quebra das dependências cruzadas é o `EventBus` (Barramento de Eventos). O C# usava C# `Events` (Delegates: `public event EventHandler OnChat;`). O problema dos C# Events é que eles geram *Strong References*. Se o Killaura se inscrevia no `OnTick` do `MPPlayer` e você desligava o Killaura sem dar um `-= OnTick`, o Killaura ficava vivo na memória RAM para sempre (Memory Leak clássico de C#).

### 10.1 O EventBus do Java (Weak References e Desacoplamento)
O novo Bot utilizará uma implementação de EventBus (ex: Guava EventBus ou LMAX Disruptor) baseada no Spring ApplicationContext.

Nenhum módulo chamará métodos diretamente em outros módulos, eles apenas **Gritam no vazio** (Publish) ou **Ouvem o vazio** (Subscribe).

**Exemplo de Comunicação Desacoplada**:
1. O Netty (Infra) recebe um pacote de Chat e publica um `ChatMessageReceivedEvent`. Ele não sabe se há alguém ouvindo.
2. O `WebConsoleStream` (UI Rest) escuta esse evento e manda via Websocket pro navegador.
3. A `AutoReplyMacro` (Plugins) escuta esse evento e checa se a mensagem contém "Qual é a capital do Brasil?".
4. O Killaura não escuta esse evento.

### 10.2 A Hierarquia de Prioridades (`@Order`)
Como as IAs funcionam como ouvintes independentes, precisamos ditar quem reage primeiro. O Spring permite isso via anotação `@Order`.

```java
// O Radar Anti-Hack (Avisa de ameaças)
@Component
public class DangerRadarAgent implements Agent {
    
    // Ouve primeiro que todo mundo
    @EventListener
    @Order(1)
    public void onPlayerSpawn(PlayerSpawnedEvent event) {
        if(isEnemy(event)) triggerAlarm();
    }
}

// O Killaura (Bate)
@Component
public class KillauraAgent implements Agent {
    
    // Ouve só depois do radar
    @EventListener
    @Order(10)
    public void onPlayerSpawn(PlayerSpawnedEvent event) {
        // ... ataca ...
    }
}
```

---

## 11. Apêndice D: Gerenciamento de Configurações (Config Injection)

No `AdvancedBot 2.4.5` C#, as configurações eram um arquivo `.ini` ou um XML que era lido aleatoriamente no meio das funções (ex: `if (File.ReadAllText("config.txt").Contains("killaura=true"))`). Isso causava IO bloqueante de disco e travava o bot na hora que ele ia bater no mob.

### 11.1 A Abordagem Spring `@ConfigurationProperties`
No Java, o disco só é lido UMA VEZ na inicialização do módulo (Fase 1 do Bootstrapping). O Spring converte o arquivo `application.yml` em Beans fortemente tipados que são injetados nas classes.

**O Arquivo application.yml**:
```yaml
bot:
  network:
    timeout-ms: 30000
    compression-threshold: 256
  ai:
    killaura:
      max-reach: 3.8
      cps-min: 8
      cps-max: 12
    pathfinding:
      max-iterations: 5000
```

**A Injeção de Dependência da Config**:
O `KillauraAgent` não lê arquivos. Ele apenas recebe o objeto `KillauraProperties` pronto.
```java
package com.advancedbot.core.config;

import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "bot.ai.killaura")
public class KillauraProperties {
    private double maxReach;
    private int cpsMin;
    private int cpsMax;
    // Getters e Setters gerados pelo Lombok
}
```

**O Consumo no Domínio**:
```java
package com.advancedbot.core.agents;

public class KillauraAgent implements Agent {
    private final KillauraProperties config;
    
    // O Spring injeta a configuração via Construtor.
    public KillauraAgent(KillauraProperties config) {
        this.config = config;
    }
    
    public void onTick() {
        if (distanceToEnemy <= config.getMaxReach()) {
            attack();
        }
    }
}
```
**Impacto Arquitetural**: Alterar configurações agora é *Type-Safe*. Se o YAML estiver mal formatado, o Bot nem liga (Fail-Fast). E o IO de disco (lento) não atrapalha a latência da rede de 50ms, abolindo as famigeradas "congeladas" do legado C#.


---

## 12. Apêndice E: O Benefício Imediato - Testabilidade (Mocking)

O maior pecado da dependência circular no `AdvancedBot 2.4.5` (C#) era que o desenvolvedor precisava logar num servidor pirata toda vez que quisesse testar se o Killaura estava calculando a distância correta. Isso demorava 20 segundos por teste.

Com a nova árvore de dependências apontando para o Domínio e usando Injeção de Dependências, o JUnit 5 pode instanciar o núcleo do bot em *1 milissegundo*.

### 12.1 O Mock do Ambiente Físico (Sem Rede)
Como o `WorldManager` não possui injetores de rede nem de UI, podemos populá-lo manualmente num teste unitário.

```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.assertTrue;

class KillauraAgentTest {

    @Test
    void naoDeveAtacarAlemDoAlcanceMaximo() {
        // 1. Arrange (Preparo do Grafo Desacoplado)
        WorldManager fakeWorld = new WorldManager();
        EntityTracker fakeTracker = new EntityTracker(fakeWorld);
        
        // Spawnamos o bot no 0,0,0
        fakeTracker.addEntity(new Entity(0, 0, 0, 0));
        
        // Spawnamos o inimigo no 10,0,0 (Distância = 10 blocos)
        fakeTracker.addEntity(new Entity(1, 10, 0, 0));
        
        // Criamos as configs injetáveis (Max Reach = 3.8)
        KillauraProperties config = new KillauraProperties();
        config.setMaxReach(3.8);
        
        // Injetamos a config no Agente puro
        KillauraAgent killaura = new KillauraAgent(config);
        
        SessionContext ctx = new SessionContext("FakeBot", fakeWorld, fakeTracker, null, BotState.PLAYING);
        
        // 2. Act (A IA tenta agir)
        List<ActionIntent> intencoes = killaura.onTick(ctx);
        
        // 3. Assert (Nenhum pacote de ataque deve ser gerado, pois 10 > 3.8)
        assertTrue(intencoes.isEmpty(), "O Killaura tentou atacar fora do alcance!");
    }
}
```
**O Valor Disto**: Se um programador quebrar a matemática da classe `KillauraAgent` no futuro, o CI/CD (GitHub Actions) rodará este teste e impedirá o `Merge Request`. No legado, o erro só seria descoberto quando um usuário fosse banido do servidor.

---

## 13. Apêndice F: Padrão Decorator (Monitoramento Não-Intrusivo)

Outro erro do código antigo era encher o núcleo da aplicação com `Console.WriteLine()` e `Stopwatch` para descobrir por que o bot estava lento. Isso misturava código de telemetria com código de negócio.

O Spring Boot nos permite usar **Aspect-Oriented Programming (AOP)** ou o Padrão **Decorator** graças à Injeção de Dependências via Interfaces.

### 13.1 O Problema do Logging Bloqueante
Se você colocar `Logger.info()` dentro do método `calculate` do `AStarPathfinder`, que roda milhares de vezes por segundo, a formatação da String matará a performance.

### 13.2 A Solução: Interceptadores via Proxies do Spring
Podemos criar um Aspecto que "envolve" nossos Beans injetados e monitora quanto tempo eles demoram, sem que eles saibam disso.

```java
package com.advancedbot.core.telemetry;

import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.springframework.stereotype.Component;

@Aspect
@Component
public class AgentPerformanceMonitor {

    // "Intercepta qualquer método 'onTick' de qualquer classe que implemente 'Agent'"
    @Around("execution(* com.advancedbot.core.agents.Agent.onTick(..))")
    public Object monitorTickDuration(ProceedingJoinPoint joinPoint) throws Throwable {
        
        long start = System.nanoTime();
        
        // O Spring manda a IA rodar
        Object result = joinPoint.proceed(); 
        
        long durationMs = (System.nanoTime() - start) / 1_000_000;
        
        // Se a IA (ex: AStar ou Killaura) segurar o Tick Loop por mais de 5ms, dispara alarme.
        if (durationMs > 5) {
            String agentName = joinPoint.getTarget().getClass().getSimpleName();
            Logger.warn("ALERTA DE GARGALO: {} demorou {} ms no onTick!", agentName, durationMs);
        }
        
        return result;
    }
}
```
Este código será vital para a depuração de gargalos quando portarmos a Física do C# para o Java.

---

## 14. Apêndice G: Tabela de Interfaces de Serviço (The Ports)

O padrão *Ports & Adapters* exige que a comunicação do Núcleo com o Mundo Externo seja mediada por Interfaces (Ports). Abaixo estão os contratos Java que devem ser criados no módulo `advancedbot-app-core` e implementados por Adaptadores da Infraestrutura.

### 14.1 A Porta de Rede (Network Outbound)
O domínio jamais toca no Netty. Ele exige que alguém implemente a porta de comunicação.
```java
package com.advancedbot.core.ports;

/**
 * Contrato de envio de pacotes. Implementado pelo NettyAdapter.
 */
public interface NetworkPort {
    /**
     * Envia um pacote de alto nível e limpa seu buffer de retaguarda.
     */
    void sendPacket(MinecraftPacket packet);
    
    /**
     * Fecha a conexão de forma abrupta.
     */
    void disconnect(String reason);
}
```

### 14.2 A Porta de Persistência (Data Outbound)
Se o bot minerou 500 diamantes e precisa salvar isso no banco de dados, a IA chama esta interface.
```java
package com.advancedbot.core.ports;

/**
 * Contrato para persistir as estatísticas de sessão. 
 * Implementado por um PostgresAdapter ou MongoAdapter no futuro.
 */
public interface SessionMetricsPort {
    void saveBlocksMined(String botId, int amount);
    void saveDeaths(String botId, int amount);
}
```

### 14.3 A Porta de Scripts Externos (JS / Python)
Para que o Java possa invocar funções escritas pelos usuários na Web.
```java
package com.advancedbot.core.ports;

/**
 * Contrato implementado pelo GraalVMPluginAdapter.
 */
public interface ScriptingPort {
    /**
     * Executa um script JS em um contexto sandbox seguro.
     */
    void executeScript(String rawJavascript);
    
    /**
     * Dispara um evento para o motor JS chamar as funções onChat().
     */
    void dispatchToScript(String eventName, Object dataPayload);
}
```

### 14.4 Tabela de Implementadores Esperados

Para clarificar a divisão de tarefas entre a equipe, segue o mapeamento de quem implementará qual Porta:

| Interface (Port) | Módulo que Define (Core) | Módulo que Implementa (Adapter) | Classe Implementadora Sugerida |
|---|---|---|---|
| `NetworkPort` | `advancedbot-app-core` | `advancedbot-infra-network` | `NettyClientAdapter` |
| `ScriptingPort` | `advancedbot-app-core` | `advancedbot-infra-plugins` | `GraalVMEngineAdapter` |
| `WebConsolePort`| `advancedbot-app-core` | `advancedbot-ui-rest` | `SseConsoleAdapter` (Server-Sent Events) |
| `MetricsPort` | `advancedbot-app-core` | `advancedbot-infra-database` | `PostgresMetricsAdapter` |
| `ConfigPort` | `advancedbot-app-core` | `advancedbot-app-core` (Self) | `YamlConfigurationAdapter` |

Ao injetar essas interfaces nos contrutores do `TickOrchestrator`, o Spring Boot fará o "casamento" automático entre quem pede a interface e o Adaptador anotado com `@Component` que a implementa, sem que os dois JARs precisem saber da existência um do outro.

---

## 15. Apêndice H: O Paradigma de Tratamento de Erros no Grafo

Se a camada de `NetworkPort` perder a conexão (cabo puxado), como ela avisa as 15 IAs que dependem do Socket sem causar um `NullReferenceException` em cadeia como acontecia no C#?

1. **Jamais Retorne Null**: Se a rede cair, o `NettyClientAdapter` não retorna nulo.
2. **Propagação de Eventos**: O Adaptador publica `SessionDisconnectedEvent` no `EventBus`.
3. **Escuta Limpa**: O `BotSession` ouve o evento e muda seu estado para `OFFLINE`.
4. **Isolamento de IA**: Na próxima volta do relógio de 50ms, o `TickOrchestrator` percebe o estado `OFFLINE` e nem sequer chama o `onTick()` das IAs. As IAs nunca precisarão escrever `if (socket == null) return;` em seus códigos.

---

## 16. Apêndice I: Glossário de Injeção de Dependências (Dicionário Spring)

Para os desenvolvedores de C# legados que estão migrando para Java e podem se sentir perdidos no jargão do Spring Framework, este glossário traduz os conceitos do IoC (Inversion of Control) presentes no grafo mapeado.

| Termo / Anotação | Significado no Paradigma do Bot | Equivalente em C# Legado |
|---|---|---|
| `Injeção de Dependência` | A classe não faz `new ClasseA()`. Ela pede a `ClasseA` no construtor e alguém magicamente entrega. | Não existia. O C# usava instâncias rígidas (`new A()`) ou Singletons globais. |
| `@Component` | Diz ao Spring: "Por favor, crie uma instância desta classe e guarde na memória para injetar em quem pedir". | `public static ClassName Instance` |
| `@Service` | Apenas um sinônimo semântico de `@Component` usado para classes que têm lógica de negócios pesada (Ex: `TickOrchestrator`). | N/A |
| `@Configuration` | Uma classe especial que ensina o Spring a fabricar Beans complexos (Ex: Configurar o `NioEventLoopGroup` do Netty). | Métodos estáticos de setup no `Program.cs`. |
| `@Bean` | Fica dentro de `@Configuration`. É o método que devolve o objeto que será injetado nos construtores de outras classes. | N/A |
| `@Autowired` | Usado antigamente para injetar campos. **Proibido no Bot**. Use injeção por construtor (`public MinhaClasse(Dependencia d)`). | N/A |
| `@Qualifier` | O Bot pode ter várias implementações de `Agent`. O qualifier diz "Eu quero o agente cujo nome é 'Killaura'". | `switch(name) { case "Killaura": }` |
| `ApplicationContext` | A caixa mágica do Spring que guarda todas as instâncias do programa. É o próprio "Grafo Vivo". | O próprio `AdvancedBot.exe` em execução. |

### 16.1 O Perigo das Dependências Escondidas (ServiceLocator)

Um anti-padrão que pode emergir se a equipe não entender o IoC é tentar buscar Beans manualmente no meio das IAs:
```java
// RUIM: Padrão Service Locator (Acoplamento Escondido)
public void onTick() {
    WorldManager world = SpringContext.getBean(WorldManager.class); // ERRO!
}
```
Isso **destrói a testabilidade**. A IA descrita acima não expõe que precisa do `WorldManager` em seu construtor. Para um teste unitário funcionar, o desenvolvedor seria obrigado a inicializar todo o `SpringContext` (que demora segundos). 

A regra é absoluta: **Se uma IA precisa de um dado ou serviço, ela deve pedi-lo no Construtor ou exigi-lo no parâmetro do método (`SessionContext`)**. 

---

## 17. Apêndice J: Prevenção de Component Scan Sujo (Otimização de Boot)

O `AdvancedBot 2.4.5` ligava quase instantaneamente porque o C# não usava reflexão maciça na inicialização. Quando migramos para o Spring Boot, o recurso de `@ComponentScan` (que varre todas as pastas do projeto atrás de anotações `@Component`) pode retardar severamente o início da aplicação (Boot Time).

### 17.1 Restringindo a Árvore de Varredura
Para evitar que o Spring varra bibliotecas de terceiros (como o pacote inteiro do Netty ou do Gson), o Ponto de Entrada da aplicação (`main`) deve definir rigidamente o pacote base.

```java
package com.advancedbot.app;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;

// Define exatamentes quais caixas o Spring deve olhar, poupando CPU
@SpringBootApplication(scanBasePackages = {
    "com.advancedbot.core",
    "com.advancedbot.infra",
    "com.advancedbot.ui"
})
public class BotApplication {
    public static void main(String[] args) {
        SpringApplication.run(BotApplication.class, args);
    }
}
```

### 17.2 Beans Lazy (Injeção Tardia)
Para Agentes (IAs) muito complexos que talvez o usuário nunca ative na sessão (Ex: `AutoBuildAgent`), podemos anotá-los com `@Lazy`.
Isso diz ao grafo de dependências: "Não instancie isso agora. Instancie apenas no exato momento que alguém tentar injetá-lo". Isso poupa RAM considerável em ambientes VPS de 1GB.

### 17.3 O Impacto do JIT Compiler
Vale notar que, no Java, as dependências são interligadas rapidamente, mas o código atinge performance máxima apenas após o Just-In-Time (JIT) Compiler analisar os *hot paths*. Portanto, o bot pode apresentar leve instabilidade (Lag Spikes) nos primeiros 10 segundos de vida enquanto a JVM otimiza as injeções em tempo real.
Para mitigar isso, em produção, o sistema poderá fazer o pré-aquecimento (Warmup) do Domínio chamando métodos fictícios do `WorldManager` antes de abrir a porta TCP pública, garantindo que o C2 Compiler já tenha compilado o código de máquina nativo.
Esse warmup será essencial para competir com bots nativos C++ em latência de combate.
As métricas de JIT e GC poderão ser visualizadas no módulo `advancedbot-ui-rest` via Spring Actuator.

---

## 18. Conclusão do Mapa de Dependências

O **AdvancedBot 2.4.5** em C# foi vítima de seu próprio crescimento descontrolado (Big Ball of Mud). Classes com 5.000 linhas, referências circulares entre a Tela e a Rede, e travamentos baseados em concorrência de leitura tornaram a adição de novas funcionalidades um processo traumático.

Ao adotar o Java 21, o Spring Boot e a Arquitetura Hexagonal descrita neste documento:

1. **Módulos Independentes**: O desenvolvedor da API WebFlux não precisa entender de pacotes de Minecraft (Netty), e o criador do AStar não precisa entender de JSONs da UI.
2. **Mocking Imediato**: A quebra das dependências permitirá que o núcleo do Bot seja instanciado em milissegundos durante testes automatizados, simulando milhares de mundos falsos para validar o Killaura.
3. **Fim dos Memory Leaks**: Ao abolir os `EventHandlers` circulares do C# e substituí-los pelo `EventBus` reativo com `WeakReferences`, o Java garante que quando um Bot for deletado, suas IAs, Mapas e Sockets serão varridos perfeitamente pelo ZGC.

Com o Grafo de Dependências desenhado, a equipe tem sinal verde para criar a estrutura dos módulos no Maven/Gradle. O próximo passo lógico é definir a documentação de Testabilidade e Cobertura (`08-Cobertura-Funcional.md`), que se apoiará fortemente na arquitetura de injeção apresentada aqui.
