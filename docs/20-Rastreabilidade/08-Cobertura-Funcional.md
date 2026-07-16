# Plano de Cobertura Funcional e Bateria de Testes (QA/Testes)

O `AdvancedBot 2.4.5` (C#) sofria de uma falha crônica de engenharia de software: **Zero Testes Automatizados**. Cada alteração no código exigia que o desenvolvedor compilasse o bot, abrisse uma conta no Minecraft, entrasse num servidor de testes, andasse com o bot e orasse para não ser banido pelo Anti-Cheat.

Este documento de Rastreabilidade e Cobertura Funcional decreta o fim dessa era empírica. No projeto Java, adotamos a metodologia de **Test-Driven Development (TDD) Adaptado para Redes** e a integração contínua (CI). Nenhuma linha de código irá para produção sem que robôs matemáticos garantam seu funcionamento.

---

## 1. O Paradoxo de Testar um Bot "Headless"

Testar um cliente de jogo sem interface gráfica (Headless) é um dos maiores desafios de arquitetura. O bot vive em um ambiente altamente dependente do estado do servidor e sofre com a latência de rede.

### 1.1 O Falso Positivo de Rede
No C#, testes esporádicos tentavam se conectar a um servidor online. Se o servidor estivesse desligado, o teste falhava, mesmo que o código do Bot estivesse perfeito (Flaky Tests).
**A Nova Regra Java**: Nenhum Teste Unitário ou de Integração tem permissão de abrir um Socket TCP para a internet. Toda a rede será simulada via Loopback interno (Netty Embedded Channel) ou Testcontainers.

### 1.2 O Paradigma de Tempo Real (Tick Mocks)
A lógica do Minecraft ocorre em *Ticks* (50ms). Se um teste JUnit espera que o bot ande de X=0 para X=10, ele não pode esperar 500ms reais (Isso tornaria a suíte de testes muito lenta).
**A Solução**: O `TickOrchestrator` do Java aceitará a injeção de um `MockClock` (Relógio Falso). Os testes unitários avançarão o tempo artificialmente chamando `orchestrator.forceTick()` milhares de vezes em um único milissegundo.

---

## 2. A Pirâmide de Testes do AdvancedBot

A arquitetura hexagonal nos permite focar 80% do esforço na base da pirâmide (Testes de Domínio rápidos) e apenas 20% no topo (Simulação de Rede).

### 2.1 Camada Nível 1: Testes de Domínio (Puro Java)
Testam a regra de negócio do Minecraft isolada do mundo. Se a Microsoft mudar o peso da gravidade de `0.08` para `0.09`, estes são os testes que quebrarão.

**Alvos Principais:**
1. Motor de Física (`PhysicsEngine.java`).
2. Gerenciador de Inventário (`InventoryManager.java`).
3. Algoritmo de Caminho (`AStarPathfinder.java`).
4. Máquinas de Estado de Agentes (`KillauraAgent.java`).

**Padrão de Qualidade Exigido (Cobertura de Linhas: 95%+)**:
Esses testes não têm desculpa para falharem ou para não existirem. Eles não usam Spring Boot, não carregam configuração, rodam de forma puramente funcional em frações de milissegundo.

### 2.2 Camada Nível 2: Testes de Integração de Eventos
Garante que se a Infraestrutura (Netty) gerar o pacote `0x02` em byte bruto, o EventBus disparará o evento `ChatMessageEvent` e as Macros cadastradas vão recebê-lo.

**Alvos Principais:**
1. Codificadores e Decodificadores do Netty.
2. Desserializadores de Chunk GZIP.
3. Tratamento de Exceções Desconectantes.

**Padrão de Qualidade Exigido (Cobertura de Linhas: 80%)**:
Requer a inicialização parcial do *Spring ApplicationContext* para injetar os Beans do EventBus.

### 2.3 Camada Nível 3: Testes de Integração End-to-End (E2E Fake Server)
A cereja do bolo. Usaremos o `Testcontainers` (ou um servidor Mock nativo) para bootar um Servidor Minecraft 1.8 *falso* localmente, rodar o Bot completo (como se fosse o usuário), logar no servidor, enviar "oi" no chat, e desconectar. Tudo dentro da pipeline do GitHub Actions.

---

## 3. Matriz Exaustiva: Testes Unitários de Domínio

Abaixo detalhamos o que *exatamente* os testes JUnit do módulo `advancedbot-domain` precisarão assertar. Se o desenvolvedor tentar pular essas asserções, o PR (Pull Request) será bloqueado.

### 3.1 `PhysicsEngineTests` (As Leis de Newton Quadradas)

| Nome do Teste | Cenário Preparado (Arrange) | Ação (Act) | Asserção Exigida (Assert) |
|---|---|---|---|
| `botDeveCairAteBaterNoChao` | Bot flutuando no ar em Y=10. Nenhum bloco abaixo além de Y=4. | Loopar `physics.onTick()` 20 vezes. | A posição final Y do bot deve ser rigorosamente exata à posição gerada pela queda gravitacional Vanilla. Ao encostar no bloco em Y=4, `isOnGround` vira `true`. |
| `botDevePararAoBaterNaParede` | Bot em X=0, Z=0. Parede (Pedra) em X=1. | Bot tenta aplicar velocidade X=+2. | A hitbox do bot deve colidir com o bloco e ele deve parar em X=0.7 (borda do bloco). |
| `puloDeveAplicarVelocidadeExata` | Bot no chão. | `physics.jump()`. 1 tick avança. | O vetor Y deve ser `0.42` redondo (ou `0.52` com poção de pulo nível 1). Sem esse valor exato, o Anti-Cheat dá banimento no ato. |
| `aguaAplicaArrasteCorreto` | Bot dentro de um bloco de Água. | Loopar física simulando queda. | A gravidade na água não é 0.08, e sim uma porcentagem menor. O bot deve afundar lentamente, validando a constante de arraste líquido. |
| `knockbackAoLevarDano` | Bot parado. Recebe vetor de Explosão. | Tick da física. | A fricção do chão deve puxar o bot para trás, mas os vetores horizontais são reduzidos a cada tick conforme a constante `0.91` do Vanilla. |

### 3.2 `InventoryManagerTests` (O Quebra-Cabeças de Slots)

A UI do inventário do Minecraft é insana. O pacote `0x2F Set Slot` usa o Slot `45` para a mão off-hand, mas o pacote `0x09 Held Item` diz que a espada está no slot `0`. Essa conversão matou o C# várias vezes.

| Nome do Teste | Cenário Preparado (Arrange) | Ação (Act) | Asserção Exigida (Assert) |
|---|---|---|---|
| `conversaoDeSlotProtocoloParaUI` | Item chega no Slot ID 36. | `inventory.getItem(36)`. | O sistema deve saber que o Slot 36 do protocolo equivale ao primeiro Slot da Hotbar (Baixo do inventário). |
| `cliqueEmBau` | Baú falso criado com 27 slots abertos. | `inventory.click(SlotID 0)`. | A lógica de transação deve prever o ID da janela. O bot pega o item do baú e o prende no "Cursor" (Mouse falso). |
| `shiftClickEmpilhaItens` | Baú com 30 pedras. Inventário do bot com 40 pedras. | Bot faz `Shift-Click` nas pedras do baú. | O bot não "puxa" 30 pedras. Ele soma. 40+30 = 70. Mas o limite do pack é 64. O inventário falso do bot deve calcular: 64 num slot, e os 6 restantes em um slot vazio. |


### 3.3 `AStarPathfinderTests` (A Fuga do Labirinto)

O Killaura ou o AutoMiner dependem 100% de o AStar calcular a rota de volta antes de bater no alvo. Como a AStar C# paralisava a thread principal e as vezes entrava em Loop Infinito, os testes Java devem garantir a limitação de *Depth* (Profundidade).

| Nome do Teste | Cenário Preparado (Arrange) | Ação (Act) | Asserção Exigida (Assert) |
|---|---|---|---|
| `caminhoLinearPerfeito` | Terreno plano de grama (10 blocos X). | `astar.calc(0,0,0 ate 10,0,0)`. | Retorna Array contendo 10 pontos sequenciais (X=1, X=2, X=3...). A execução deve durar menos de 2ms. |
| `pularMuralhaBasica` | Parede de 1 bloco de altura no meio do caminho plano. | Tenta cruzar a parede. | O Pathfinder entende que o bot pode pular 1 bloco, inserindo Y=1 e em seguida Y=0. |
| `desistirSeInalcancavel` | Alvo dentro de uma caixa de bedrock fechada (Zero aberturas). | Tenta calcular caminho. | O algoritmo deve iterar até o limite do raio definido e **desistir** (`PathStatus.UNREACHABLE`), sem causar stack overflow ou consumir 100% da CPU. |
| `calcularEmThreadSeparada` | Mundo gigante falso. | Executar via `CompletableFuture`. | O teste falhará se a thread do chamador bloquear a thread de testes do JUnit por mais de 5ms, validando o isolamento de concorrência. |

---

## 4. Matriz de Testes de Integração de Infraestrutura (Eventos)

A camada de Netty lida com manipulação perigosa de bytes *Off-Heap*. Os testes aqui garantem que o Bot não sofra corrupção de memória.

### 4.1 `MinecraftPacketDecoderTests` (Fuzzing / Buffer Leak)

Em vez de usar a internet, nós injetamos matrizes de bytes (`byte[]`) corrompidas no Pipeline de teste do Netty (`EmbeddedChannel`) e analisamos a reação.

| Nome do Teste | Cenário (Bytes Falsos) | Asserção (Assert) | Prevenção C# Legado |
|---|---|---|---|
| `decodePacoteDescomprimidoNormal` | Array contendo Chat Message "Oi". | O `EmbeddedChannel` deve soltar um objeto `MinecraftPacket(ID 0x02, Data "Oi")` no final da fila. | O bot deve ler o VarInt de tamanho sem falhar. |
| `leituraIncompletaAguardarBytes` | Recebemos metade de um pacote Chunk e a rede do teste "cai" de propósito. | O Decodificador **NÃO** deve lançar Exception. Ele deve pausar o ponteiro (Reader Index) e aguardar o Netty chamar a função novamente no futuro. | O C# antigo tentava ler além do Stream e quebrava com `EndOfStreamException`. |
| `nbtZipBombPrevention` | Injeta bytes contendo o pacote `0x2F SetSlot` com 500 NBTs `Compound` recursivas no item. | Lança `DecoderException` e fecha o canal. | Sem isso, o servidor malicioso explodiria a memória RAM do Bot usando o loop recursivo do NBT Parser. |
| `zlibInflateLimites` | Envia pacote simulando Threshold de compressão estourado (Dizendo que o pacote final tem 5 Gigabytes). | Lança `DecoderException(Tamanho Irreal)`. | Impede estourar o Heap do Java (`OutOfMemoryError`). |

### 4.2 Testes do `EventBus` (Pub/Sub)
Simulamos o disparo de eventos de alta carga.

```java
@Test
void garantePrioridadeDeExecucaoOrder() {
    // 1. Instancia Agente VIP (Order 1) e Agente Lerdo (Order 100)
    // 2. Dispara `HealthUpdateEvent`
    // 3. O Teste coleta a ordem em que as funções onTick() foram chamadas via um AtomicLong falso.
    // 4. Assert que o Agente VIP agiu 1 nanossegundo antes do Lerdo.
}
```

---

## 5. Bateria de Testes End-to-End (Mock Server Anti-Cheat)

Esta é a inovação que eleva o `AdvancedBot` da era amadora para a era de software corporativo. 
No GitHub Actions, antes de compilar o `.jar` de produção, a Pipeline vai levantar um servidor local usando um Mock focado em reproduzir os "Checkers" dos piores Anti-Cheats (NoCheatPlus, GrimAC).

### 5.1 O Motor de "Check" Falso

A bateria cria um Socket TCP em `localhost:25565`. O bot (com as configurações reais de usuário) é ligado para conectar nele.

#### Teste 5.1.1: O Teste de "Fast Break" (Tempo de Mineração)
1. **Setup**: O servidor falso avisa o bot (pacote) para começar a minerar uma Obsidiana (Dura 10 segundos Vanilla).
2. **Bot Ação**: O `AutoMinerAgent` é ligado, ele olha pra obsidiana e envia o pacote `0x07 Block Digging (START)`.
3. **Validação do Mock Server**:
   - Se o bot mandar o pacote `0x07 Block Digging (FINISH)` em 3 segundos. O Mock Server rejeita a transação e marca o teste como `FALHOU (O bot foi banido por FastBreak)`.
   - Se o bot mandar em 10.05 segundos, o teste **Passa**. O algoritmo de Timer do Killaura provou que é Legit.

#### Teste 5.1.2: O Teste do "Reach" (Braço Longo)
1. **Setup**: O Mock Server envia uma Vaca andando para trás, ficando a 5 blocos de distância do bot.
2. **Bot Ação**: O `KillauraAgent` está ligado com configuração máxima de 3.8.
3. **Validação do Mock Server**:
   - O Mock Server escuta os pacotes do Bot. Se receber `0x02 Use Entity (ATTACK)` apontado para a vaca que está a 5 blocos, o teste falha: o Bot ignorou as leis da física e quebrou o Anti-Cheat.

#### Teste 5.1.3: O Teste de Inicialização "Forge Spoofing"
1. **Validação do Mock Server**:
   - Assim que o bot conecta e passa para a Fase `PLAY`, o Mock Server pausa a simulação.
   - Ele analisa o Histórico de Pacotes na ordem exata.
   - Se o pacote `0x15 Client Settings` não for o primeiro. (FALHA).
   - Se o pacote `0x17 Plugin Message (MC|Brand)` estiver faltando. (FALHA).

---

## 6. Matriz de Testes do Motor de Plugins (GraalVM)

O `AdvancedBot 2.4.5` suportava scripts rudimentares usando bibliotecas C# defasadas (ex: Jint ou compilador em tempo real `CodeDom`). Muitas vezes um script de usuário com loop infinito travava o bot inteiro (Travamento da Thread Principal).
A migração para Java 21 introduz o `GraalVM Polyglot Engine`, que executa o Javascript em um Sandbox estrito. A cobertura de testes deve garantir que os usuários não possam destruir o Bot de propósito.

### 6.1 `ScriptSandboxTests` (Proteção Contra Código Malicioso)

Abaixo estão os testes focados em Segurança da Informação (InfoSec) do Sandbox.

| Nome do Teste | Cenário Preparado (Script JS) | Ação (Act) | Asserção Exigida (Assert) |
|---|---|---|---|
| `scriptComLoopInfinitoMorre` | Script: `while(true) {}` | `scriptPort.execute(script)`. | O bot deve matar o contexto Javascript em exatos 500ms usando `Context.close(true)` e emitir um erro no log. A Thread do bot deve continuar intacta. |
| `scriptAcessandoSystemExit` | Script: `java.lang.System.exit(0)` | Tenta rodar a função. | O GraalVM deve lançar uma Exceção de Segurança. O Sandbox não permite `HostAccess` indiscriminado (O bot não pode ser desligado por um script de terceiro). |
| `scriptAcessandoDisco` | Script: `new java.io.File("/etc/passwd").read()` | Tenta rodar a função. | O GraalVM deve proibir acesso IO (FileSystem). |
| `alocacaoDeMemoriaEstourada`| Script cria arrays gigantes `[]` num loop for. | Tenta rodar. | O GraalVM lança erro de Resource Limit após exceder 50 Megabytes alocados pelo script. O bot principal sobrevive. |

### 6.2 Testes Funcionais da API Javascript

O usuário tem acesso a um objeto injetado no Javascript chamado `bot`. Os testes garantem que o mapeamento `Java -> JS -> Java` está funcionando.

```java
@Test
void garanteQueOJsPodeMandarMensagemNoChat() {
    // 1. Injeta um Mock da NetworkPort no Motor JS.
    ScriptingPort motor = new GraalVMEngineAdapter(mockNetworkPort);
    
    // 2. Roda o código do usuário: bot.chat("Ola Mundo");
    motor.executeScript("bot.chat('Ola Mundo');");
    
    // 3. Verifica se a porta de rede recebeu a requisição corretamente
    verify(mockNetworkPort).sendPacket(argThat(packet -> 
        packet.getId() == 0x01 && 
        packet.getData().contains("Ola Mundo")
    ));
}
```

---

## 7. Rastreabilidade Funcional Legada (O Que Não Pode Faltar)

A tabela a seguir mapeia as funcionalidades centrais do C# e como serão invalidadas (testadas) no ecossistema Java. Nenhuma feature antiga será deixada para trás.

| Funcionalidade Legada (C#) | Como será Coberta no Java | Classe Alvo de Teste (JUnit) |
|---|---|---|
| `AutoSoup.cs` | Teste garantindo que ao ficar com Vida < 5, o Bot envia `HeldItemChange`, `UseItem` e volta pro `HeldItemChange` em 3 milissegundos. | `AutoSoupAgentTest.java` |
| `AutoFish.cs` | Teste injetando o pacote `SoundEffect` (Pescando). O bot deve reagir em menos de 100ms enviando `BlockPlacement` no ar (clique direito). | `AutoFishAgentTest.java` |
| `AutoArmor.cs` | Teste colocando armadura de Diamante no baú e armadura de Ferro no chão. O bot deve preferir vestir a de Diamante calculando o multiplicador do Item NBT. | `AutoArmorAgentTest.java` |
| `ChestStealer.cs` | Teste checando a latência introduzida de propósito. O bot *não pode* sugar os itens no mesmo milissegundo, deve respeitar a config `delayMs` para evitar kick. | `ChestStealerAgentTest.java` |
| `AutoDisconnect.cs` | Teste simulando jogador VIP (Inimigo) entrando a 10 blocos de distância. O bot deve chamar `networkPort.disconnect()` antes do Killaura tentar atacá-lo. | `DangerRadarAgentTest.java` |


---

## 8. Apêndice A: Configuração do Testcontainers (Docker)

Para o ambiente de Integração E2E (Seção 5), não podemos exigir que o desenvolvedor baixe e inicie um servidor Minecraft manualmente na sua máquina antes de rodar os testes. Isso fere o princípio do *One-Click Build*.
A solução é usar a biblioteca `Testcontainers` com a imagem oficial do `itzg/minecraft-server`.

### 8.1 Exemplo de Base de Teste JUnit 5 (Servidor Efêmero)
```java
package com.advancedbot.e2e;

import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.testcontainers.containers.GenericContainer;
import org.testcontainers.utility.DockerImageName;

public abstract class AbstractE2ETest {
    
    // Inicia um container Docker efêmero rodando um servidor Minecraft 1.8.8
    static final GenericContainer<?> minecraftServer = new GenericContainer<>(DockerImageName.parse("itzg/minecraft-server:latest"))
        .withExposedPorts(25565)
        .withEnv("VERSION", "1.8.8")
        .withEnv("EULA", "TRUE")
        .withEnv("ONLINE_MODE", "FALSE") // Permite bots piratas
        .withEnv("TYPE", "PAPER"); // PaperMC para simular anti-cheats
        
    @BeforeAll
    static void startServer() {
        minecraftServer.start();
        // O Testcontainers cuida de mapear a porta 25565 para uma porta aleatória do Host
        System.setProperty("test.server.port", minecraftServer.getMappedPort(25565).toString());
        System.setProperty("test.server.ip", minecraftServer.getHost());
    }

    @AfterAll
    static void stopServer() {
        minecraftServer.stop(); // Mata o container no final
    }
}
```
**Impacto na Arquitetura**: Todo teste que herdar `AbstractE2ETest` terá acesso a um servidor Minecraft real vazio para fazer o que quiser, 100% isolado de outros testes. Quando a suíte termina, o Docker destrói a imagem e libera a porta.

---

## 9. Apêndice B: Estratégia de CI/CD (GitHub Actions)

Como o projeto agora é feito em Java (Maven), a integração contínua (CI) é mil vezes mais simples que no C# legado (que dependia do MSBuild de janelas).
Toda vez que alguém fizer um *Push* para a branch `main`, a nuvem executará as três camadas da pirâmide.

### 9.1 O Pipeline de Cobertura de Código (Jacoco)
Se a equipe aceitar um Pull Request que não tenha testes, a dívida técnica começará a crescer novamente. A ferramenta **JaCoCo (Java Code Coverage)** será inserida no `pom.xml`.
Se a cobertura geral do `advancedbot-domain` cair para menos de 95%, o GitHub Actions rejeitará o PR.

### 9.2 O Arquivo `ci.yml` (Workflow)
```yaml
name: AdvancedBot Build and Test Pipeline

on:
  push:
    branches: [ "main", "develop" ]
  pull_request:
    branches: [ "main" ]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up JDK 21
      uses: actions/setup-java@v3
      with:
        java-version: '21'
        distribution: 'temurin'
        cache: maven
        
    - name: Compile and Run Unit Tests (Domain & Core)
      run: mvn test -pl advancedbot-domain,advancedbot-app-core
      
    - name: Run E2E Integration Tests (Testcontainers)
      run: mvn verify -pl advancedbot-e2e
      # Note: O GitHub Actions (ubuntu-latest) já tem o Docker pré-instalado.
      
    - name: Check Code Coverage (Jacoco)
      run: mvn jacoco:check
      
    - name: Build Fat JAR (Final Executable)
      run: mvn package -DskipTests
      
    - name: Upload Artifact
      uses: actions/upload-artifact@v3
      with:
        name: advancedbot-java-2.5.0
        path: advancedbot-app-core/target/advancedbot-core-*.jar
```

### 9.3 O Papel do Teste de E2E no CD (Continuous Delivery)
Somente se TODAS as camadas de testes passarem, o Bot será empacotado (`package`) num Fat JAR. 
Isso garante que os usuários do bot (que pagarão mensalidades) nunca recebam uma versão "quebrada" que dá crash ao andar ou que é instantaneamente banida por Anti-Cheats (como acontecia frequentemente nas versões menores do C#).


---

## 10. Apêndice C: Tratamento de Flaky Tests (Testes Instáveis)

Um "Flaky Test" é um teste que passa na máquina do desenvolvedor, mas falha aleatoriamente no servidor do GitHub Actions. Em ambientes assíncronos (como o Netty e o EventBus), isso é o pesadelo da equipe de QA.

### 10.1 A Armadilha do `Thread.sleep()`
No legado C#, quando queriam testar se o Killaura matava o Zumbi, o código de teste fazia:
```csharp
bot.SpawnZombie();
Thread.Sleep(2000); // Espera 2 segundos pro bot agir
Assert.IsTrue(zombie.IsDead);
```
Se a CPU do servidor de CI estivesse sobrecarregada, o Killaura demorava 2.1 segundos para processar. O teste falhava (Falso Negativo).

### 10.2 A Solução Java: `Awaitility`
O projeto Java deve PROIBIR estritamente o uso de `Thread.sleep()` nos testes. Em vez disso, usaremos a biblioteca `Awaitility`.

```java
import static org.awaitility.Awaitility.await;
import java.time.Duration;

@Test
void killauraDeveMatarZumbi() {
    bot.spawnEntity(zombie);
    
    // Fica checando a condição a cada 50ms, mas desiste se passar de 3 segundos.
    // Se o Zumbi morrer no milissegundo 100, o teste termina imediatamente (Fast-Pass).
    await()
        .atMost(Duration.ofSeconds(3))
        .pollInterval(Duration.ofMillis(50))
        .until(() -> zombie.isDead());
}
```

---

## 11. Apêndice D: Matriz de Testes de Macros Complexas

Enquanto o AutoSoup e o AutoArmor são fáceis de testar (dependem apenas do Inventário), outras macros exigem interação com NBTs, Mapeamento de Chunks e RayCasting complexo. Abaixo a matriz de testes para esses casos avançados.

### 11.1 `AutoBuildAgentTests` (Construção Automática)

O AutoBuild lê um arquivo `Schematic` e recria a construção colocando blocos no mundo real.

| Nome do Teste | Cenário Preparado | Ação | Asserção |
|---|---|---|---|
| `botNaoTentaConstruirSeBlocoFaltar` | Schematic pede 10 pedras. O baú do bot tem 5. | Liga o AutoBuild. | Emite evento `NotEnoughMaterialsEvent`. Não envia pacote de BlockPlacement. |
| `pularBlocoQueJaExiste` | Schematic pede Pedra em X=1. O Chunk já tem Pedra em X=1. | Liga o AutoBuild. | O bot pula o bloco e vai para o próximo. Valida a leitura rápida do `ChunkManager`. |
| `rotacaoDeEscadas` | Schematic tem escada virada para o Norte. | Bot coloca o bloco. | O Vector 3D da visão do Bot (`PlayerLook`) tem que girar forçadamente para o Sul *antes* de colocar o bloco, ou a escada ficará torta. |

### 11.2 `ParkourAgentTests` (Pulo Preciso)

| Nome do Teste | Cenário Preparado | Ação | Asserção |
|---|---|---|---|
| `puloGapDoisBlocos` | Chão em X=0, Buraco em X=1, Chão em X=2. | Liga o Agente de Movimento. | Bot chega na borda de X=0, emite Pulo. O motor de física falso deve registrar a parábola exata pousando em X=2. |
| `baterNaCabeca` | Teto baixo (Y=2). Chão em Y=0. | Bot pula. | A colisão deve detectar o teto, interromper o pulo no meio e empurrar o bot pra baixo no mesmo tick. Validação pura de Bounding Boxes. |


---

## 12. Apêndice E: Testes de Stress e Carga (Load Testing)

O AdvancedBot C# legado frequentemente travava se fosse colocado em um servidor com "Mob Spawners" absurdos (ex: 50.000 vacas no mesmo chunk). O parseamento do pacote `0x0F Spawn Mob` congestionava o pipeline e causava "Read Timeout". 

O Java precisa provar que consegue lidar com alta densidade de dados. Usaremos testes de stress com JMH (Java Microbenchmark Harness) e Gatling/JMeter integrados ao Pipeline.

### 12.1 `EntitySpamBenchmark` (JMH)
Este teste não roda no JUnit comum. Ele é executado separadamente para medir o `Throughput` (Operações por segundo).

```java
import org.openjdk.jmh.annotations.*;

@State(Scope.Thread)
public class EntitySpamBenchmark {

    private EntityTracker tracker;
    private MinecraftPacket spawnPacket;

    @Setup
    public void prepare() {
        tracker = new EntityTracker(new WorldManager());
        // Prepara um pacote falso de Spawn
        spawnPacket = createFakeSpawnPacket(100); 
    }

    @Benchmark
    @BenchmarkMode(Mode.Throughput)
    @OutputTimeUnit(TimeUnit.SECONDS)
    public void testSpawnRate() {
        // Tenta processar o máximo possível
        tracker.handleSpawn(spawnPacket);
    }
}
```
**Meta de Aceite**: O `EntityTracker` Java, rodando num Ryzen 5 (ou CPU de GitHub Actions), deve ser capaz de processar no mínimo **150.000 pacotes de spawn por segundo**. Se o Throughput cair abaixo disso (por exemplo, devido ao uso de Collections não otimizadas como `ArrayList.remove()`), a build falha.

### 12.2 Teste de Concorrência Extrema (100 Bots Simultâneos)
Usando o `Testcontainers`, o `Mock Server` ligará e o teste instruirá o `BotRegistry` a disparar 100 instâncias simultâneas do bot para conectar na mesma porta local.

**Objetivos da Validação**:
1. **Thread Pool**: Verificar se o `Netty NioEventLoopGroup` (configurado com 4 threads) suporta a I/O não bloqueante de 100 bots sem estourar o limite de 50ms (Tick Rate).
2. **Garbage Collection (ZGC)**: Garantir que o uso de RAM não escale de forma linear desproporcional. Se 1 bot consome 10MB, 100 bots devem consumir algo perto de 1GB (Compartilhando classes comuns). Se o consumo for de 5GB, existe um problema de instanciamento pesado (Factory Leak).
3. **Desconexão em Massa (Thundering Herd)**: O servidor falso kickará os 100 bots no mesmo milissegundo. O teste garante que o bot não causará `ConcurrentModificationException` ao limpar o `BotRegistry`.

---

## 14. Apêndice F: Matriz de Testes de Regressão de Anti-Cheat

Um dos maiores diferenciais do `AdvancedBot` sempre foi sua capacidade de operar em servidores altamente vigiados (Raizlandia, MushMC) sem ser detectado. A refatoração para Java introduz o risco de regressions (falhas que já haviam sido consertadas voltarem a acontecer).
Para evitar isso, a pipeline exigirá testes E2E focados exclusivamente em Anti-Cheats modernos.

### 14.1 Simulação de Burlar o GrimAC (Movement Prediction)
O `GrimAC` não analisa heurística de velocidade, ele re-simula no servidor o pacote `0x04 PlayerPosition` enviado pelo bot para ver se é matematicamente possível.

| Nome do Teste de Regressão | Cenário (Ambiente E2E com GrimAC) | Ação do Bot | Asserção (Pass / Fail) |
|---|---|---|---|
| `knockbackDeveAfetarPosicao` | Bot leva uma flechada em pleno ar. | A classe de Física do bot atualiza seu vetor X,Z baseada na flecha. | O bot **DEVE** enviar um pacote de movimento recuando. Se o bot tentar andar pra frente ignorando a flecha, o GrimAC flaggará "Velocity/AntiKnockback". (Assert Fail se flaggado). |
| `puloSprintandoGeraMaisArraste` | Bot corre e pula. | Física simula a inércia aérea (Sprint Jump). | O vetor X,Z aéreo tem uma resistência ao ar diferente do pulo parado. A curva gerada pelo Java deve bater perfeitamente com a curva Vanilla. |
| `passosNaTeiaDeAranha` | Bot entra em uma Cobweb. | Agente de movimento tenta cruzar. | A velocidade do bot deve ser dividida e multiplicada pela constante Vanilla para Cobweb. O bot não pode cruzar em menos de 5 segundos. |

### 14.2 Simulação de Burlar o NoCheatPlus (NCP) / Matrix (Heuristics)
Esses Anti-Cheats analisam o padrão de rotação da cabeça (Aimbot) e a consistência de cliques (AutoClicker).

| Nome do Teste de Regressão | Cenário (Ambiente E2E com NCP) | Ação do Bot | Asserção (Pass / Fail) |
|---|---|---|---|
| `aimbotRotacaoSuave` | Zumbi spawna nas costas do bot. | O `KillauraAgent` vira a câmera para o zumbi. | O teste intercepta os pacotes `0x05 PlayerLook` e verifica o Delta de rotação. Se o Yaw mudar 180 graus em 1 único tick (Snap), o teste FALHA. A rotação deve ser dividida em no mínimo 3 ticks (Interpolada). |
| `cpsRandomization` | Killaura bate no Zumbi. | `KillauraAgent` ataca. | O Mock Server mede o tempo entre cada pacote de ataque. Se a diferença for *exatamente* a mesma por 10 hits seguidos (ex: exatamente 100ms), o teste FALHA (Robotic Click detected). O bot deve introduzir um Jitter de +/- 20ms aleatório. |
| `reachRaycastPelaParede`| Inimigo está a 2 blocos, mas atrás de um muro fino. | `KillauraAgent` tenta atacar. | O teste provê o mapa completo. A classe de `RayTrace` do Java deve perceber a parede e proibir o pacote de ataque. Se atacar, é WallHack (FALHA). |

---

## 15. Apêndice G: Testes de Fuzzing na Desserialização

Fuzzing é a técnica de injetar lixo aleatório num software para ver se ele crasha. No legado C#, se o servidor enviasse um ID de pacote desconhecido, a Thread travava para sempre.
No Java, implementaremos o `JQF (Java QuickCheck Fuzzing)` integrado ao JUnit para disparar raios contra nosso Decoder Netty.

### 15.1 Código Exemplo do Teste de Fuzzer
```java
import edu.berkeley.cs.jqf.fuzz.JQF;
import edu.berkeley.cs.jqf.fuzz.Fuzz;
import org.junit.runner.RunWith;

@RunWith(JQF.class)
public class NetworkDecoderFuzzTest {

    // O gerador injeta byte arrays completamente aleatórios (lixo)
    @Fuzz
    public void testDecoderNaoPodeCrasharJVM(byte[] lixoDeRede) {
        EmbeddedChannel channel = new EmbeddedChannel(new MinecraftPacketDecoder());
        
        try {
            channel.writeInbound(Unpooled.wrappedBuffer(lixoDeRede));
        } catch (DecoderException e) {
            // Sucesso! O Netty capturou o lixo e lançou uma Exception controlada.
            // O teste só falha se houver NullPointerException ou OutOfMemoryError.
        } finally {
            channel.close();
        }
    }
}
```
**O Valor Disto**: Se um administrador de servidor descobrir que nosso Bot quebra ao receber um NBT modificado contendo Strings com caracteres de escape ilegais, ele poderia criar uma "Armadilha de Bot". O Fuzzer garante que o Bot é um tanque de guerra à prova de travamentos.

---

## 16. Apêndice H: Cobertura de Mutação (Mutation Testing com PIT)

Testes unitários normais podem mentir. Se um desenvolvedor escrever um teste com `Assert.assertTrue(true)`, a cobertura de código (JaCoCo) dirá que a linha foi testada, mas a lógica de negócios continua desprotegida. Para combater isso na reconstrução do Bot em Java, utilizaremos **Mutation Testing** através do `PIT (Pitest)`.

### 16.1 O que é Mutation Testing?
Durante a fase de build no Maven, o PIT injetará "mutantes" (erros propositais) no código compilado do `advancedbot-domain`.
- Ele troca `>` por `<`.
- Ele troca `+` por `-`.
- Ele remove chamadas de método (`void`).

Depois de injetar o erro, o PIT roda a suíte de testes JUnit. **Se os testes PASSAM, o mutante sobreviveu.** Isso significa que seus testes são fracos. **Se os testes FALHAM, o mutante foi morto.** Isso significa que a suíte de testes é robusta.

### 16.2 Matriz de Mutantes Críticos (Sobrevivência Proibida)

| Classe Mutada (Alvo) | Mutante Injetado pelo PIT | O Teste Esperado para Matá-lo | Risco Se Sobreviver |
|---|---|---|---|
| `PhysicsEngine.java` | Trocou `Math.max(0, drag)` por `Math.min(0, drag)` | `aguaAplicaArrasteCorreto` | O bot voaria infinitamente dentro da água (Fly Hack detectado instantaneamente). |
| `AStarPathfinder.java` | Removeu condicional `if(isSolid(block))` | `pularMuralhaBasica` | O bot tentaria andar por dentro das paredes (Phase/NoClip Hack detectado). |
| `AutoSoupAgent.java` | Alterou `health < 5` para `health < 15` | `deveTomarSopaApenasEmRisco` | O bot tomaria todas as sopas assim que perdesse meia vida, esgotando o inventário em 2 segundos de PVP. |
| `MinecraftZlibEncoder.java`| Removeu `buf.release()` do Netty | `leituraIncompletaAguardarBytes` | Vazamento fatal de memória Off-Heap. O bot crasharia a VPS após 1 hora ligado. |

**Meta da Pipeline**: O GitHub Actions será configurado com `mutationThreshold = 85%`. Se mais de 15% dos mutantes sobreviverem nas classes de domínio, o Pull Request é rejeitado sumariamente.

---

## 17. Apêndice I: Testes de Comportamento (BDD com Cucumber)

Enquanto o JUnit testa a matemática e o Netty testa os bytes, precisamos testar a **Experiência do Usuário (UX)** e o comportamento de alto nível das IAs. Para isso, adotaremos o BDD (Behavior-Driven Development) escrito em Gherkin. Isso permite que até usuários não-programadores escrevam cenários de teste para o Bot.

### 17.1 Cenário BDD: Fugir de Inimigos Vip
Abaixo está um exemplo de arquivo `danger-radar.feature` que será lido e automatizado pelo framework Cucumber.

```gherkin
Feature: Fuga Automática de Ameaças (Danger Radar)

  Background:
    Given que o bot "Notch" está logado e no estado "PLAYING"
    And a configuração "auto-disconnect.enabled" está "true"
    And a configuração "auto-disconnect.radius" é "15" blocos

  Scenario: Inimigo VIP se aproxima voando
    Given que um jogador "HackerVip" spawna na coordenada X=100, Y=64, Z=100
    And o bot está na coordenada X=100, Y=64, Z=120 (Distância = 20)
    When o jogador "HackerVip" se move para a coordenada X=100, Y=64, Z=110 (Distância = 10)
    Then o bot deve emitir o evento "SessionDisconnectRequest" imediatamente
    And a razão da desconexão deve conter "Ameaça VIP detectada a 10 blocos"
    And o "NettyClientAdapter" deve fechar o socket TCP em menos de 50 milissegundos
```

### 17.2 Cenário BDD: Reconexão Automática (AutoReconnect)
O Bot legado costumava ficar preso num loop de tentar logar 1.000 vezes por segundo se o servidor o kickasse, resultando num banimento de IP permanente.

```gherkin
Feature: Reconexão com Exponential Backoff

  Scenario: Servidor rejeita a conexão múltiplas vezes
    Given que o servidor Mock está configurado para retornar pacote 0x00 (Disconnect) no Handshake
    And a configuração de AutoReconnect tem o delay inicial de "5" segundos
    When o bot inicia o processo de "Connect"
    Then o servidor recusa a conexão
    And o bot deve esperar "5" segundos antes da próxima tentativa
    When o servidor recusa a conexão pela segunda vez
    Then o bot deve esperar "10" segundos antes da terceira tentativa (Backoff Duplicado)
    And não deve estourar a pilha de concorrência (StackOverflow)
```
Estes arquivos Gherkin servirão não apenas como testes vivos, mas também como **Documentação Funcional** para os futuros mantenedores do projeto.

---

## 18. Apêndice J: Engenharia do Caos (Chaos Testing / Alta Latência)

O Minecraft é jogado pela internet, e a internet falha. O `AdvancedBot 2.4.5` (C#) frequentemente duplicava itens no baú se o servidor demorasse mais de 2 segundos para responder a um pacote `0x0E Click Window`. Isso ocorria porque a UI assumia que o item já havia sido movido antes de receber a confirmação `0x32 Confirm Transaction`.
O Java resolverá isso com estado transacional, mas isso precisa ser testado com **Chaos Engineering** utilizando o `ToxiProxy` ou filtros customizados do Netty.

### 18.1 Injeção de Latência (Lag Simulator)
Nos testes de integração End-to-End, podemos inserir um Handler no pipeline do Netty que segura o pacote aleatoriamente antes de mandá-lo para a frente.

```java
// Interceptador de Caos
public class ChaosLatencyHandler extends ChannelOutboundHandlerAdapter {
    @Override
    public void write(ChannelHandlerContext ctx, Object msg, ChannelPromise promise) {
        if (Math.random() < 0.1) { // 10% de chance de dar Lag Spike
            ctx.executor().schedule(() -> {
                ctx.write(msg, promise);
            }, 1500, TimeUnit.MILLISECONDS); // Atrasa em 1.5s
        } else {
            ctx.write(msg, promise);
        }
    }
}
```

### 18.2 Matriz de Testes sob Caos (Tolerância a Falhas)

| Cenário de Caos | Ação da IA | Comportamento Esperado (Resiliência) |
|---|---|---|
| Latência de 2 segundos. | Killaura tenta atacar alvo em movimento. | O Killaura não deve atacar baseado na posição atual visual (Lag), mas sim no vetor previsto ou pausar ataques se o Ping ultrapassar 500ms (Evitar Kick por Spam). |
| Pacotes `0x21 Chunk Data` fragmentados. | Bot tentando achar minérios. | O AStar deve aguardar pacientemente a liberação da *Semaphore* do ChunkManager sem gerar `TimeoutException`. |
| Perda de Pacotes (Packet Loss de 5%). | `ChestStealer` pega itens. | Se o servidor não mandar o `0x32 Confirm Transaction`, o inventário virtual do bot deve fazer *Rollback* do item, não duplicá-lo na memória. |

---

## 19. Apêndice K: Tracing de Performance e Profiling

A garantia de que o novo `AdvancedBot` é superior ao antigo C# será validada via Profiling Contínuo com o **Async-Profiler** (ferramenta padrão ouro para JVM).

### 19.1 Flame Graphs na Pipeline de CI
Uma vez por semana, uma GitHub Action agendada executará uma simulação pesada de 1 hora e anexará um *Flame Graph* HTML aos artefatos do build.
A equipe de engenharia verificará visualmente onde os ciclos de CPU estão sendo gastos.

**Metas de Profiling Absolutas**:
1. **Regra do JsonParser**: A conversão de pacotes JSON (`Gson`) de chat nunca deve ultrapassar 5% do total de amostras de CPU. O legado sofria com regex lentos.
2. **Regra do ArrayCopy**: O Netty `ByteBuf.readBytes` deve aparecer no topo do I/O, mas `System.arraycopy` nas classes de Domínio devem ser evitadas.
3. **Alocação Zero (Escape Analysis)**: O Java C2 Compiler deve provar que a alocação do `ActionIntent` dentro do loop da IA está sendo otimizada no Stack (e não no Heap). Se os relatórios do JFR (Java Flight Recorder) mostrarem alta Pressão de Alocação (Allocation Rate > 200MB/s) no pacote `com.advancedbot.domain`, a build falhará.

---

## 20. Apêndice L: Estratégias de Teste para o WebFlux (UI Rest)

O módulo `advancedbot-ui-rest` é a porta de entrada para os comandos do usuário (Dashboard React/Vue). Se a API estiver instável, o usuário perderá o controle do bot, mesmo que o núcleo Java esteja perfeito.

Para testar os Endpoints Reativos do Spring WebFlux, não usaremos o `MockMvc` bloqueante, mas sim o `WebTestClient`.

### 20.1 Testando o Fluxo de Ligar o Bot
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class BotControllerTest {

    @Autowired
    private WebTestClient webClient;
    
    @MockBean
    private BotOrchestrator orchestrator;

    @Test
    void deveRetornar200AoLigarBot() {
        // Arrange
        Mockito.doNothing().when(orchestrator).startBot(anyString());

        // Act & Assert
        webClient.post().uri("/api/bots/start")
                .contentType(MediaType.APPLICATION_JSON)
                .bodyValue("{\"username\": \"TesteBot\", \"type\": \"PREMIUM\"}")
                .exchange()
                .expectStatus().isOk()
                .expectBody()
                .jsonPath("$.status").isEqualTo("STARTING");
    }
}
```

### 20.2 Testando o Barramento SSE (Server-Sent Events)
O Console do bot na Web é atualizado em tempo real via SSE.
O teste deve garantir que o `Flux<ServerSentEvent>` propague as mensagens de chat recebidas no `EventBus` até a requisição HTTP.

| Nome do Teste (WebFlux) | Cenário Preparado | Ação | Asserção |
|---|---|---|---|
| `sseTransmiteChatEmTempoReal` | Cliente HTTP se inscreve no endpoint `/api/bots/console`. | `EventBus` recebe um `ChatEvent`. | O Fluxo SSE não pode fechar. Ele deve emitir um Data Packet HTTP em menos de 10ms contendo a string JSON exata da mensagem. |
| `rejeitaComandoSemAutorizacao`| Endpoint exige JWT. | Usuário envia POST sem Header de Auth. | HTTP 401 Unauthorized imediato, garantindo a segurança do painel C&C. |

---

## 21. Apêndice M: Reprodutibilidade de Bugs (Bug Bounties)

No C# Legado, quando um usuário reportava um bug ("O bot crashou quando tentei quebrar vidro no servidor X"), a equipe passava semanas tentando reproduzir o cenário no escuro.

Na nova era Java, implementaremos a **Gravação de Estados (State Record/Replay)** focada em QA.

### 21.1 O Formato `.abrec` (AdvancedBot Record)
Em produção, se o usuário ligar o modo de depuração, o Bot usará um Adaptador de Persistência que salva TODOS os `MinecraftPacket` inbound em um arquivo binário ultracomprimido (similar a uma fita cassete).

### 21.2 Replay no JUnit
Quando o usuário enviar esse arquivo para a equipe de desenvolvimento, o Teste Unitário carregará o arquivo `.abrec` e o injetará no `EmbeddedChannel` do Netty.
A execução do Bot na máquina do desenvolvedor será uma réplica bit-a-bit milimetricamente idêntica ao que ocorreu no PC do usuário, tornando a reprodução de bugs 100% determinística.

**Regra Arquitetural de Testes**: Todo Bugfix (Correção) submetido via Pull Request DEVE vir acompanhado de um novo arquivo de Replay que comprove a falha anterior, e um teste JUnit que valide a correção carregando esse Replay. Sem esse teste, o PR é negado.

---

## 22. Apêndice N: Simulação e Testes de Inventário (Slot Mapping & Desyncs)

Um dos maiores motivos de falhas nos bots (e bans por "Inventory Hacks") é o Desync do inventário. Se o bot achar que tem uma Maçã Dourada no slot 1, tentar comê-la, mas o servidor souber que o slot 1 está vazio, o anti-cheat flagga uma ação impossível.

Para evitar isso, a arquitetura de testes deve validar o mapeamento de ID de slots do `InventoryManager` contra uma base de conhecimento fixa.

### 22.1 A Complexidade dos IDs de Janela (Window ID)
No Minecraft, o inventário pessoal do jogador tem Window ID `0`. Se ele abre um Baú, o servidor envia `0x2D Open Window` e atribui o ID `1`. Todas as operações subsequentes de clique devem referenciar o ID `1`. O bot legado frequentemente clicava usando ID `0` enquanto o baú estava aberto.

| Teste de Estado de Janela | Cenário Injetado | Ação do Bot | Asserção |
|---|---|---|---|
| `bloqueioDeCliqueJanelaFechada` | Nenhuma janela extra aberta (Apenas ID 0). | Bot tenta transferir item do Baú. | O `InventoryManager` Java DEVE lançar `IllegalStateException` ao invés de enviar o pacote pro servidor. (Fail-Fast Client-side). |
| `fechamentoForcadoPeloServidor` | Bot abre baú (ID 1). Em seguida servidor envia `0x2E Close Window (ID 1)`. | Bot tenta clicar. | A ActionIntent gerada deve ser negada. A matriz interna deve limpar os 27 slots do baú fantasma da memória. |
| `dragAndDropTest` (Arrastar) | Bot simula clique esquerdo segurado (Drag) dividindo 64 terras por 4 slots. | `inventory.dragSpread(...)`. | Os pacotes `0x0E Click Window` gerados devem seguir o protocolo restrito de "Drag Begin (Mode 5)", "Drag Add Slot", "Drag End". Qualquer pacote enviado fora dessa ordem exata resulta em banimento. |

---

## 23. Apêndice O: Verificação de Ticks (Tick Consistency Testing)

O relógio central do bot (TickOrchestrator) precisa girar a 20 TPS (Ticks Per Second), ou seja, a cada 50ms. Se a CPU estiver sob carga pesada, os ticks podem atrasar.
Um teste automatizado rodará no CI/CD simulando concorrência alta para ver se o relógio compensa atrasos.

### 23.1 Teste de Catch-Up (Compensação de Lag)
Se o Thread do bot congelar por 150ms (devido a um pico de Garbage Collector - GC Pause), ele perdeu 3 Ticks de 50ms. O que o bot deve fazer?

No C# Legado, ele ignorava o passado.
No Java, ele deve aplicar o **Catch-Up**: Rodar os 3 Ticks perdidos instantaneamente (sem IO de rede) para atualizar a máquina de estado (ex: se ele estava caindo, ele tem que calcular os 3 ticks de queda física na memória e só enviar o pacote final de onde ele caiu pro servidor).

```java
@Test
void testaCompensacaoDeCatchUpTick() {
    MockClock clock = new MockClock();
    TickOrchestrator orchestrator = new TickOrchestrator(clock);
    
    // Inicia e roda 1 tick
    orchestrator.start();
    clock.advance(50, TimeUnit.MILLISECONDS);
    assertEquals(1, orchestrator.getCurrentTick());
    
    // Simula GC Pause de 200ms
    clock.advance(200, TimeUnit.MILLISECONDS);
    
    // O Orchestrator deve perceber que atrasou e rodar o loop interno 4 vezes rapidamente
    orchestrator.forceEvaluation();
    assertEquals(5, orchestrator.getCurrentTick()); // 1 + 4
}
```

---

## 24. Apêndice P: Monitoramento de Alocações (Escape Analysis Test)

O `AdvancedBot 2.4.5` (C#) frequentemente acionava o Garbage Collector (GC) porque criava novos objetos `Vector3` para cada bloco que o Killaura olhava. Em Java, a alocação de objetos no Heap para cálculos temporários deve ser erradicada usando a técnica de Escape Analysis (Alocação na Stack).

### 24.1 O Teste `AllocationRateTest` (JFR)
Para impedir que um desenvolvedor suba código sujo que aloque memória sem necessidade, a pipeline terá um teste rodando o Java Flight Recorder (JFR).

| Nome do Teste | Ação do Bot | Asserção do JFR | Motivo da Regra |
|---|---|---|---|
| `alocacaoNeutraNoLoopPrincipal` | Bot parado. Recebe 10.000 pacotes de KeepAlive e devolve. | A Allocation Rate medida pelo JFR DEVE ser estritamente zero (`0 bytes/sec`). | Se o bot precisar dar `new KeepAliveEvent()` no Heap, com 10.000 bots ativos, a RAM encherá de lixo em segundos, causando picos de lentidão no GC. |
| `reusoDePathNodes` | AStar calcula rota de 500 blocos. | A quantidade de instâncias de `PathNode` geradas não pode ultrapassar o tamanho do Pool pré-alocado. | Uso obrigatório do padrão `Object Pool` (Recycler) em vez do comando `new`. |

---

## 25. Apêndice Q: O Custo Computacional no CI/CD (GitHub Actions Billing)

Como o projeto adotou uma pirâmide de testes onde o topo (E2E) levanta um servidor Minecraft em Docker via `Testcontainers`, o tempo da Pipeline do GitHub Actions subirá significativamente. Cada minuto de CI custa dinheiro ou minutos da cota gratuita.

### 25.1 Estratégia de Paralelismo (Forks)
O `maven-surefire-plugin` e o `maven-failsafe-plugin` serão configurados para testar o domínio de forma massivamente paralela.

```xml
<!-- No pom.xml do advancedbot-domain -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-surefire-plugin</artifactId>
    <version>3.1.2</version>
    <configuration>
        <!-- Roda testes em threads paralelas (1 thread por core da CPU virtual) -->
        <parallel>classes</parallel>
        <threadCount>1C</threadCount>
        <perCoreThreadCount>true</perCoreThreadCount>
    </configuration>
</plugin>
```

### 25.2 Otimização do Testcontainers
Levantar o Servidor Vanilla a cada teste da camada E2E demoraria 15 segundos por teste. Se houverem 40 testes E2E, a pipeline levaria 10 minutos apenas bootando Minecrafts falsos.
Para evitar isso, usaremos o **Container Reuse** (Padrão Singleton de Testcontainers).

Um único contêiner do `itzg/minecraft-server` será iniciado no início da suite E2E e **compartilhado** entre todas as classes de teste. No final de cada teste, rodamos o comando `/kill @e` e `/clear` via RCON para limpar o estado do mapa sem precisar reiniciar a JVM do servidor.

---

## 26. Apêndice R: Testes de Segurança e Criptografia (Security Tests)

Como o Bot lida com credenciais originais (E-mail e Senha) para autenticar na Mojang (Yggdrasil), o módulo de segurança requer testes específicos. Se o código Java salvar a senha em Plain-Text no disco, isso viola a confiança do usuário.

### 26.1 `CredentialStorageTests`
```java
@Test
void senhasNaoDevemSerSalvasEmTextoPuro() {
    // 1. O usuário digita a senha no WebFlux
    configManager.saveAccount("user@mail.com", "senhaSuperSecreta");
    
    // 2. O Teste lê o arquivo físico do disco
    String fileContent = Files.readString(Path.of("accounts.json"));
    
    // 3. A senha NÃO pode estar lá em texto puro
    assertFalse(fileContent.contains("senhaSuperSecreta"));
    assertTrue(fileContent.contains("AES")); // Deve estar cifrada com AES
}
```

---

## 27. Apêndice S: A Diferença entre Mocks e Fakes no Domínio

Muitas equipes abusam do framework `Mockito` para tudo. O problema de usar `@Mock` para objetos de domínio (como `WorldManager`) é que os testes se tornam acoplados à implementação, não ao comportamento. Se você usar `when(world.getBlock(x,y,z)).thenReturn(AIR)`, o teste passa, mas não garante que a física daquele bloco estaria correta na vida real.

**A Regra do Bot**: Para Infraestrutura (Rede, BD), usamos **Mocks** (Mockito). Para o Domínio (Física, Inventário, Pathfinder), usamos **Fakes**.
Um "Fake" é uma implementação real que roda apenas na memória (ex: `FakeWorldManager` que de fato guarda arrays tridimensionais de blocos, mas não escuta a rede).
Isso garante que os testes do Killaura e do AStar sejam testes de estado real, reduzindo Falsos Positivos drásticamente.

---

## 28. Apêndice T: Simulando a API da Mojang (Yggdrasil Mock)

Nos testes de integração End-to-End, o servidor falso rodará em `ONLINE_MODE=FALSE` para agilizar os testes. Porém, pelo menos um teste da suíte precisa validar o fluxo completo de Autenticação Premium (criptografia RSA e troca de Hashes MD5 com a sessão da Mojang).
Para isso, não faremos requisições HTTP para os servidores reais da Mojang (pois eles dariam Rate Limit no IP do GitHub Actions). Usaremos a biblioteca `WireMock` para subir um servidor web falso na porta 8080 que retorna as mesmas respostas JSON que o `authserver.mojang.com` retornaria.
Dessa forma, provamos que o algoritmo de Login Premium do Bot funciona, sem nunca tocar na internet.

### 28.1 Fallback para Offline Mode
Caso a API da Mojang de fato caia no mundo real (HTTP 503), o teste deve validar se o bot exibe o log de aviso e faz fallback gracioso, ou se a thread crasha tentando ler JSON vazio.
A telemetria deve ser notificada dessa falha para os desenvolvedores.

---

## 29. Conclusão da Estratégia de Qualidade

A ausência total de Testes Automatizados transformou o `AdvancedBot 2.4.5` (C#) em um projeto zumbi: ninguém tinha coragem de refatorar a classe `MPPlayer` porque ninguém sabia se aquilo quebraria a detecção de Anti-Cheat em servidores obscuros. O desenvolvimento virou um jogo de adivinhação.

O **Plano de Cobertura Funcional** descrito neste documento implementa uma barreira arquitetural impenetrável:
1. **Velocidade de Feedback**: Testes unitários (Mocks de Física) rodam em 1ms, permitindo refatorações drásticas sem medo.
2. **Mocking de Anti-Cheat**: Nenhuma IA será mesclada na `main` sem antes provar que respeita os limites de PPS (Packets Per Second) estipulados nos cenários End-to-End.
3. **Tratamento de Exceções Reativas**: Ao testar anomalias de rede injetando bytes corrompidos, blindamos o bot contra pacotes malformados de servidores maliciosos (NBT Bombs).

Com este artefato finalizado, os desenvolvedores de Backend têm o *Blueprint* exato de "O Quê" e "Como" testar o núcleo Java antes de avançarem para a criação das rotas do Painel Web. O próximo documento lógico é o mapeamento dos Riscos de Migração (`09-Riscos-da-Migracao.md`), detalhando onde esse processo de TDD e conversão corre o maior risco de falhar.
