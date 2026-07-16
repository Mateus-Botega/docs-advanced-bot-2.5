# Matriz Funcional de Rastreabilidade (EdiÃ§Ã£o Exaustiva)

Este documento Ã© a matriz funcional definitiva e exaustiva para a migraÃ§Ã£o do AdvancedBot (C# para Java). Ele estabelece a ligaÃ§Ã£o microscÃ³pica entre as funcionalidades de negÃ³cio do bot legado, a sua representaÃ§Ã£o no modelo de domÃ­nio, a implementaÃ§Ã£o histÃ³rica exata em C# (arquivos, classes e mÃ©todos) e a arquitetura sugerida em Java. Nenhuma funcionalidade sistÃªmica, por menor que seja, foi omitida. 

O objetivo deste documento Ã© ser a Ãºnica fonte da verdade para o desenvolvedor da reescrita: ao se deparar com uma "feature" na UI legada, ele poderÃ¡ buscar aqui todos os estados, regras, pacotes e fluxos relacionados.

---

## Ãndice de Funcionalidades
1. InicializaÃ§Ã£o do Bot (Bootstrap)
2. Gerenciamento e ResoluÃ§Ã£o de Proxies
3. AutenticaÃ§Ã£o Premium (Yggdrasil API)
4. Handshake de Rede (Minecraft Protocol)
5. Criptografia de TrÃ¢nsito (AES/CFB8)
6. CompressÃ£o de TrÃ¢nsito (Zlib)
7. Login SecundÃ¡rio (AuthMe /register)
8. Ping e Keep-Alive (Timeout Prevention)
9. SincronizaÃ§Ã£o de Mundo GeogrÃ¡fico (Chunks)
10. SincronizaÃ§Ã£o de Bloco Individual (Block Update)
11. SincronizaÃ§Ã£o de Entidades Locais (Spawn/Despawn)
12. CinemÃ¡tica de Entidades (Movimento e RotaÃ§Ã£o Server-Side)
13. AtualizaÃ§Ã£o VitalÃ­cia (Health e Metadata)
14. Motor de Tempo Base (Tick Scheduler)
15. MovimentaÃ§Ã£o LegÃ­tima e Gravidade Client-Side
16. Algoritmo de Pathfinding (A* Estrela)
17. ExecuÃ§Ã£o de Movimento Vetorial (PathGuide)
18. ProjeÃ§Ã£o Visual e Mira (RayCasting)
19. MineraÃ§Ã£o AutomÃ¡tica (AutoMiner)
20. InteligÃªncia de Combate (PvE Mob Killaura)
21. InteligÃªncia de Pesca (Solk Bobber Tracker)
22. Sistema de InventÃ¡rio BÃ¡sico (Hotbar e Slots)
23. TransaÃ§Ãµes de InventÃ¡rio Estendidas (Window ID)
24. Equipamento AutomÃ¡tico (Auto-Armor)
25. Limpeza de InventÃ¡rio (Auto-Drop/Cleaner)
26. InteraÃ§Ã£o de Chat e Comandos (ICommand)
27. Extensibilidade Nativa (Plugins .abp)
28. Extensibilidade DinÃ¢mica (Macros Jint JS)
29. Bypasses de Anti-Cheat de Handshake (Raizlandia)
30. Bypasses FÃ­sicos (SkySurvival Captchas)
31. RenderizaÃ§Ã£o GrÃ¡fica 3D (OpenGL Viewer)
32. PersistÃªncia de Estado Local (NBT Dat)

---

## 1. InicializaÃ§Ã£o do Bot (Bootstrap)

### Contexto HistÃ³rico e Objetivo
No legado, o bot era um aplicativo WinForms monolÃ­tico. A funcionalidade de Bootstrap Ã© responsÃ¡vel por ler os arquivos binÃ¡rios de estado local (ex: senhas salvas), inicializar pools de threads e apresentar a interface. O objetivo Ã© garantir que nenhuma SessÃ£o inicie com configuraÃ§Ãµes corrompidas.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: O sistema DEVE alocar os manipuladores globais de exceÃ§Ã£o (`AppDomain.CurrentDomain.UnhandledException`) antes de carregar o form principal para evitar *crashes* brutos sem log.
- **Regras**: O NBT de configuraÃ§Ã£o `conf.dat` DEVE ser lido, e os plugins na pasta `Plugins/` DEVEM ser carregados em memÃ³ria *Reflection* ANTES da janela `Start` se tornar interagÃ­vel. Se a pasta `Plugins` nÃ£o existir, ela deve ser criada silenciosamente.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: Nenhum evento de rede.
- **Estado Inicial**: `Process Starting`.
- **Estado Final**: `UI Loop Active`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `Program.cs`, `Main.cs`, `Start.cs`, `Config.cs`, `PluginManager.cs`.
- **MÃ©todos Relevantes**: 
  - `Program.Main()`: O *Entry Point* do C# onde o SO injeta a execuÃ§Ã£o.
  - `PluginManager.LoadPlugins()`: Varredura de diretÃ³rios usando `System.IO.DirectoryInfo`.
  - `Config.LoadNBT()`: SerializaÃ§Ã£o binÃ¡ria da classe mÃ£e de config.
- **DependÃªncias**: `System.Windows.Forms`, `AdvancedBot.Client.NBT`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `AdvancedBotApplication.java` (Classe `@SpringBootApplication`), `PluginLoaderService.java`, `ConfigurationProperties` do Spring.
- **Arquitetura**: O Java nÃ£o possuirÃ¡ WinForms. O Bootstrap criarÃ¡ o contexto do Spring (ApplicationContext), inicializarÃ¡ o servidor embutido Tomcat/Netty (para painel Web/REST) e varrerÃ¡ o diretÃ³rio em busca de jars ou scripts.
- **Prioridade**: **CrÃ­tica**.

---

## 2. Gerenciamento e ResoluÃ§Ã£o de Proxies

### Contexto HistÃ³rico e Objetivo
Devido a restriÃ§Ãµes de IP em servidores de Minecraft (limite de 3 a 5 contas por IP), o bot possui um motor acoplado de tunelamento via Proxy (SOCKS4, SOCKS5 e HTTP) para mascarar as conexÃµes de dezenas de sessÃµes.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Ao clicar em "Test Proxies", uma thread assÃ­ncrona tenta abrir um Socket TCP para um host teste usando os protocolos especificados.
- **Regras**: 
  - Um proxy sÃ³ Ã© classificado como `Alive` se responder ao *handshake* SOCKS em menos de 5000ms.
  - A conexÃ£o com o Minecraft DEVE ser encapsulada totalmente. Nenhum pacote de DNS ou ping do Minecraft pode vazar pelo IP real do host se um Proxy estiver associado Ã  SessÃ£o.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: `ProxyTestedEvent`, `ProxyFailedEvent`.
- **Estado do Checker**: `Idle` -> `Testing` -> `Completed`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `ProxyChecker.cs`, `Proxy.cs`, `ProxyType` (Enum), `ProxyForm.cs`.
- **MÃ©todos Relevantes**: 
  - `ProxyChecker.Start()`: CriaÃ§Ã£o de mÃºltiplas *Tasks* no `.NET ThreadPool`.
  - `Proxy.Connect()`: Estabelecimento de TCP usando `Socket.Connect`.
- **DependÃªncias**: InteraÃ§Ãµes pesadas com controles WinForms (`Invoke`) para atualizar o `DataGridView` de status.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `ProxyManagementService.java`, `Socks5ProxyHandler` (fornecido nativamente pelo Netty).
- **Arquitetura**: A validaÃ§Ã£o de proxy vira um Job assÃ­ncrono. Na hora de conectar ao servidor Minecraft, injeta-se o `Socks5ProxyHandler` na `ChannelPipeline` do Netty antes do `MinecraftProtocolEncoder`.
- **Prioridade**: **MÃ©dia**.

---

## 3. AutenticaÃ§Ã£o Premium (Yggdrasil API)

### Contexto HistÃ³rico e Objetivo
Para acessar servidores originais (Online Mode), o bot precisa obter um *Access Token* vÃ¡lido da infraestrutura da Mojang/Microsoft (anteriormente chamada Yggdrasil).

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Veja [01-Login](../18-Fluxos-Funcionais/01-Login.md). O bot submete payload JSON com credenciais para o endpoint `https://authserver.mojang.com/authenticate`.
- **Regras**: 
  - Se a resposta HTTP for 200, a SessÃ£o salva o `AccessToken` e `ProfileId` e marca o modo de jogo como Premium.
  - Se a resposta HTTP for 403 (Forbidden), a SessÃ£o aborta a conexÃ£o e exibe erro de "Invalid Credentials".
  - Se a conta nÃ£o tiver provedor Mojang/Microsoft, assume-se o modo "Cracked" (nome offline) e a autenticaÃ§Ã£o web Ã© pulada integralmente.

### Eventos e MÃ¡quinas de Estado
- **Estados da SessÃ£o**: `Offline` -> `Authenticating` -> (Success) `Connecting`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `YggdrasilAuthenticator.cs`, `AuthRequest.cs`, `AuthResponse.cs`.
- **MÃ©todos Relevantes**: 
  - `YggdrasilAuthenticator.Authenticate(username, password)`: Abre um `HttpWebRequest` e serializa JSON usando strings literais de replace ou parser leve.
- **DependÃªncias**: `System.Net`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `YggdrasilAuthClient.java`, `MojangProfileDTO.java`.
- **Arquitetura**: Uso do `WebClient` (Spring WebFlux) ou `RestTemplate` genÃ©rico. RetornarÃ¡ um Mono/CompletableFuture para nÃ£o bloquear a thread de setup de sessÃ£o.
- **Prioridade**: **Baixa para inicio, Alta para prod**. (Bots Cracked operam normalmente sem isso).

---

## 4. Handshake de Rede (Minecraft Protocol)

### Contexto HistÃ³rico e Objetivo
A fase inicial da comunicaÃ§Ã£o com um servidor Minecraft. Define a versÃ£o de protocolo (ex: 47 para a versÃ£o 1.8), o host pretendido e qual serÃ¡ o estado subsequente da conexÃ£o (geralmente Estado 2 = Login).

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Assim que o Socket Ã© conectado, o Cliente (Bot) toma a iniciativa e escreve o pacote `Handshake`.
- **Regras**: 
  - O pacote `Handshake` nunca Ã© cifrado ou comprimido.
  - O Host e a Porta enviados no payload DEVEM ser exatos ao que o servidor espera, pois servidores com proxy reverso (BungeeCord) usam o Hostname para rotear o bot para o lobby correto (Virtual Hosting).

### Eventos e MÃ¡quinas de Estado
- **Estados da ConexÃ£o TCP**: `Handshaking` -> `Login`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `MinecraftClient.cs`, `PacketStream.cs`.
- **MÃ©todos Relevantes**: 
  - `MinecraftClient.Connect()` chama internamente a rotina de Handshake antes de iniciar a Thread de leitura (`ReceiveCallback`).
  - O C# constrÃ³i o pacote cru escrevendo os VarInts sequencialmente no `WriteBuffer`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `HandshakePacket.java`, `MinecraftHandshakeHandler.java` (extensÃ£o de `ChannelInboundHandlerAdapter`).
- **Arquitetura**: O Netty Pipeline enviarÃ¡ este pacote no mÃ©todo `channelActive`. Imediatamente apÃ³s, o Pipeline muda o estado lÃ³gico interno para `State.LOGIN`.
- **Prioridade**: **CrÃ­tica**.

---

## 5. Criptografia de TrÃ¢nsito (AES/CFB8)

### Contexto HistÃ³rico e Objetivo
Em servidores de modo "Premium", o servidor envia um pacote `EncryptionRequest` contendo uma Public Key RSA. O bot deve gerar um segredo AES, cifrar esse segredo usando a RSA do servidor, responder, e entÃ£o cifrar TODO o canal TCP a partir desse ponto.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: `EncryptionRequest` (Recebido) -> Bot gera Key e IV aleatÃ³rios -> Cifra com RSA -> Envia `EncryptionResponse` -> Ativa *Cipher* na Stream.
- **Regras**: 
  - A criptografia Ã© SimÃ©trica (AES), modo CFB, bloco de 8 bits (CFB8), sem *Padding*.
  - A transiÃ§Ã£o do Socket de *Plain-text* para *Cifrado* deve ocorrer **imediatamente** apÃ³s o flush do pacote `EncryptionResponse`. Um erro de milissegundo de timing destruirÃ¡ o parsing do pacote subsequente do servidor.

### Eventos e MÃ¡quinas de Estado
- **Estados da SessÃ£o**: `LoggingIn` -> `EncryptionActive` -> `Playing`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `CryptoUtils.cs`, `AesStream.cs`, lib externa `BouncyCastle.Crypto.dll`.
- **MÃ©todos Relevantes**: 
  - `CryptoUtils.GenerateSharedKey()`: Gera arrays aleatÃ³rios usando `RNGCryptoServiceProvider`.
  - `AesStream.Read()` e `AesStream.Write()`: Classes de IO que encapsulam o `NetworkStream` original e passam tudo pelo decodificador do BouncyCastle antes de repassar ao `PacketStream`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `EncryptionRequestDecoder`, `MinecraftCipherEncoder`, `MinecraftCipherDecoder`.
- **Arquitetura**: Uso da JCA (`javax.crypto.Cipher` com `AES/CFB8/NoPadding`). No Netty, adiciona-se dinamicamente decoders e encoders na pipeline: `pipeline.addBefore("frameDecoder", "cipherDecoder", new MinecraftCipherDecoder(secretKey))`.
- **Prioridade**: **CrÃ­tica para online mode**.

---

## 6. CompressÃ£o de TrÃ¢nsito (Zlib)

### Contexto HistÃ³rico e Objetivo
Para poupar banda de internet do servidor e do cliente, versÃµes a partir da 1.8 suportam compressÃ£o. O servidor indica que a compressÃ£o estÃ¡ ativa definindo um "Threshold" (tamanho em bytes a partir do qual o pacote deve ser compactado).

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia `SetCompression` -> Bot lÃª o Threshold -> Ativa interceptadores de leitura/escrita de Zlib.
- **Regras**: 
  - Qualquer pacote de rede (apÃ³s ativaÃ§Ã£o) muda seu formato de cabeÃ§alho (Header): passa a ter 2 VarInts (PacketLength e UncompressedLength).
  - Se `UncompressedLength` for 0, o pacote NÃƒO estÃ¡ compactado e seus dados seguem em plain-bytes. Se for > 0, o payload deve ser inflado (Zlib) antes do parsing de ID.

### Eventos e MÃ¡quinas de Estado
- **Impacto no Parsing**: Muda a semÃ¢ntica da mÃ¡quina de estados do `FrameDecoder` de *Standard* para *Compressed*.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `PacketStream.cs`, `ZlibStream.cs`, lib externa `Ionic.Zlib.dll`.
- **MÃ©todos Relevantes**: 
  - `PacketStream.SetCompression(int threshold)`: Altera a *flag* interna. A partir desse ponto, o `ReadPacket()` consome o segundo VarInt e usa o Inflater da Ionic se necessÃ¡rio.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `MinecraftCompressor`, `MinecraftDecompressor` (extensÃµes de `MessageToByteEncoder` e `ByteToMessageDecoder`).
- **Arquitetura**: InserÃ§Ã£o dinÃ¢mica no Netty. Usa-se a classe `java.util.zip.Inflater` (que consome `Deflater/Inflater` nativos em C, de altÃ­ssima performance no Java).
- **Prioridade**: **CrÃ­tica (99% dos servidores exigem)**.

---

## 7. Login SecundÃ¡rio (AuthMe /register)

### Contexto HistÃ³rico e Objetivo
Em servidores "Piratas" (Cracked), a autenticaÃ§Ã£o nÃ£o ocorre no nÃ­vel do protocolo TCP via Mojang, mas sim *In-Game*, usando plugins de servidor como o AuthMe Reloaded. O bot seria chutado se nÃ£o digitasse no chat o comando de login dentro de ~30 segundos.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia `ChatMessage` -> Bot aplica Regex -> Descobre se o server pediu `/login` ou `/register` -> Bot envia `ChatMessage` com a senha prÃ©-configurada em atÃ© 2 segundos.
- **Regras**: 
  - O Bot DEVE possuir uma senha "Default" na sua interface WinForms para submeter ao AuthMe.
  - Se a mensagem pedir `/register`, o bot DEVE enviar `/register <senha> <senha>` (repetiÃ§Ã£o exigida por padrÃ£o pelo AuthMe original).
  - **Invariante**: Durante a espera do AuthMe, o bot Ã© fisicamente impedido pelo servidor de andar, virar a cabeÃ§a ou quebrar blocos. A IA do bot deve aguardar o sucesso do login.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: Consome `ChatMessageReceivedEvent`, Produz envio de pacote `0x01` (Chat).
- **Estados Mapeados**: Macro suspensa atÃ© que o login ocorra.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `AuthMe.cs`, `MinecraftClient.cs` (via `HandleChat`).
- **MÃ©todos Relevantes**: 
  - `AuthMe.ProcessMessage(string msg)`: ContÃ©m os `if(msg.Contains("/login"))` ou uso de `Regex.IsMatch`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `AuthMeAgent.java` (implementando `Agent` ou Listener do EventBus).
- **Arquitetura**: O agente reage assincronamente ao `ChatMessageReceivedEvent` e injeta a ordem de chat na fila de saÃ­das da sessÃ£o. Totalmente desvinculado do modelo central `SessionManager`.
- **Prioridade**: **Alta**.

---

## 8. Ping e Keep-Alive (Timeout Prevention)

### Contexto HistÃ³rico e Objetivo
O Minecraft impÃµe uma desconexÃ£o forÃ§ada (Read Timeout) se a conexÃ£o TCP ficar inativa. Para aferir latÃªncia e manter o socket quente, o servidor manda pacotes de Keep Alive com um ID longo.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia `KeepAlive(Id)` -> Cliente responde IMEDIATAMENTE com `KeepAlive(Id)` com o **exato mesmo ID numÃ©rico**.
- **Regras**: 
  - Na versÃ£o 1.8, o ID do KeepAlive Ã© um *VarInt*. Em versÃµes prÃ©vias (1.5), era um Int de 32 bits (4 bytes fixos).
  - A resposta DEVE ser processada e enviada preferencialmente pela prÃ³pria thread de rede para garantir que operaÃ§Ãµes lentas na thread de IA nÃ£o gerem falsos timeouts.

### Eventos e MÃ¡quinas de Estado
- **Impacto FÃ­sico**: Falhar essa regra muda a SessÃ£o para o estado `Disconnected`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `Handler18.cs`, `Handler152.cs`.
- **MÃ©todos Relevantes**: 
  - `Handler18.HandleKeepAlive(IPacket packet)`: LÃª o DTO e joga na stream o DTO de saÃ­da espelhado.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `KeepAliveHandler.java` (Canal Netty Isolado).
- **Arquitetura**: Pode ser colocado como um interceptador leve logo apÃ³s o `MinecraftProtocolDecoder`. Ele vÃª se a classe do pacote Ã© `KeepAlivePacket`, e invoca `ctx.writeAndFlush()` direto, poupando a Camada de AplicaÃ§Ã£o (Core) de lidar com manutenÃ§Ã£o de infraestrutura.
- **Prioridade**: **CrÃ­tica**.

---

## 9. SincronizaÃ§Ã£o de Mundo GeogrÃ¡fico (Chunks)

### Contexto HistÃ³rico e Objetivo
O bot precisa entender o terreno ao seu redor para navegar ou minerar. O Servidor despeja colunas maciÃ§as de blocos de 16x16 blocos de largura (do bedrock ao cÃ©u) chamadas de `Chunks`.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia `MapChunkBulk` ou `ChunkData` -> Cliente deserializa o bitmask -> Inicializa arrays locais e acopla no dicionÃ¡rio `World.Chunks`.
- **Regras**: 
  - O pacote de Chunk da versÃ£o 1.8 continha os chamados *BlockStates* compactados, nÃ£o mais IDs simples. Requer matemÃ¡tica de bits complexa.
  - Quando um Chunk Ã© descarregado (Pacote de Chunk com flag `Unload` ou Bitmask `0`), o cliente OBRIGATORIAMENTE deve apagar as informaÃ§Ãµes de memÃ³ria para evitar *Memory Leaks* catastrÃ³ficos que congelam o C# apÃ³s minutos correndo pelo mapa.
  - **Invariante MatemÃ¡tica**: Coordenada Global de Bloco X, Z traduz-se para Chunk `CX = Floor(X/16)`, `CZ = Floor(Z/16)`. A posiÃ§Ã£o local *dentro* do chunk obedece o resto da divisÃ£o.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: Produz `ChunkLoadedEvent` e `ChunkUnloadedEvent`.
- **Impacto no Sistema**: O Viewer OpenGL redesenhava totalmente seus vÃ©rtices; O Pathfinding recalculava distÃ¢ncias ao surgir novos blocos de colisÃ£o.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `Chunk.cs`, `ChunkSection.cs`, `World.cs`, `Handler18.cs` (HandleMapChunk).
- **MÃ©todos Relevantes**: 
  - `World.SetChunk(cx, cz, newChunk)`: MÃ©todo bloqueante com `lock` no C#, manipulava a `Dictionary<Tuple<int,int>, Chunk>`.
  - A deserializaÃ§Ã£o do Chunk envolvia iterar os 16 pilares (`ChunkSections`) verificando os *PrimaryBitMasks*.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `WorldManager.java` (Domain), `ChunkVO.java` (Value Object), `ChunkDataPacketTranslator.java` (Infra).
- **Arquitetura**: O Java usarÃ¡ matrizes `short[]` primitivas para armazenar IDs de bloco por extrema eficiÃªncia. A coleÃ§Ã£o primÃ¡ria serÃ¡ um `ConcurrentHashMap<Long, ChunkVO>` onde a chave long engloba (X e Z) via bitwise shift `(x & 0xFFFFFFFFL) | (z << 32)`.
- **Prioridade**: **CrÃ­tica** (base da fÃ­sica do bot).

---

## 10. SincronizaÃ§Ã£o de Bloco Individual (Block Update)

### Contexto HistÃ³rico e Objetivo
AlteraÃ§Ãµes minuciosas no mapa, como um jogador colocar uma TNT, uma Ã¡rvore crescer ou o bot quebrar uma pedra. NÃ£o faz sentido o servidor reenviar o Chunk inteiro, entÃ£o ele envia apenas as coordenadas e o ID novo.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia pacote `BlockChange` (um Ãºnico) ou `MultiBlockChange` (vÃ¡rios em lote, otimizando explosÃµes) -> O Bot atualiza o array primitivo especÃ­fico dentro do `ChunkSection` do `World`.
- **Regras**: 
  - A escrita deve ser atÃ´mica e as referÃªncias antigas devem sumir.
  - Se a IA estiver focada na quebra de um minÃ©rio especÃ­fico (AutoMiner), o disparo deste evento informando que a coordenada agora Ã© *Air* (ID 0) indica sucesso da rotina.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: Produz `BlockChangedEvent`.
- **Impacto no Sistema**: Reinicia mÃ¡quinas de estado `Pathfinding` se o bloco bloqueou o caminho traÃ§ado. Altera estado do `CommandQuebrarMadeira` (de Breaking para Idle).

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `Handler18.cs`, `World.cs`.
- **MÃ©todos Relevantes**: 
  - `World.SetBlock(x, y, z, blockState)`: Calculava CX/CZ e a matriz Y (Section Index), efetuando a gravaÃ§Ã£o final na memÃ³ria.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: Entra diretamente no serviÃ§o do DomÃ­nio via EventBus. `world.updateBlockAt(vector3i, newState)`.
- **Arquitetura**: AssÃ­ncrona via Netty publicando eventos que a thread de Tick consumirÃ¡, mantendo previsibilidade (Lock-Free no Domain).
- **Prioridade**: **Alta**.
## 11. SincronizaÃ§Ã£o de Entidades Locais (Spawn/Despawn)

### Contexto HistÃ³rico e Objetivo
O bot precisa ter consciÃªncia espacial sobre a localizaÃ§Ã£o de outros jogadores, monstros, itens caÃ­dos e objetos especiais (como boias de pesca). O servidor avisa quando esses objetos entram ou saem do raio de visÃ£o estipulado pelo `server.properties` (geralmente 8 chunks de distÃ¢ncia).

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia `SpawnPlayer` (0x0C) ou `SpawnMob` (0x0F) -> O bot intercepta o `EntityID` Ãºnico -> O bot insere no dicionÃ¡rio `World.Entities`.
- **Regras**: 
  - O Minecraft tem pacotes separados para cada tipo base. `0x0E` para Object (veÃ­culos, barcos, itens), `0x0F` para Mobs (zumbis, vacas), `0x0C` para Jogadores e `0x2C` para Entidade Global (Raios/TrovÃµes).
  - Um ID de Entidade Ã© um inteiro e Ã© descartÃ¡vel. Quando o bot recebe pacote `DestroyEntities` (0x13), ele OBRIGATORIAMENTE deve expurgar aquelas entidades da RAM. Um bot vazando entidades trava a IA que tenta mirar em fantasmas.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: `EntitySpawnedEvent`, `EntityDespawnedEvent`.
- **Impacto no Sistema**: O Auto-Kill (Macro de Combate) desperta ao detectar o `EntitySpawnedEvent` de um zumbi prÃ³ximo. O Auto-Fish ouve o sombolo de uma boia caindo.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `World.cs`, `Entity.cs`, `MPPlayer.cs`, `Handler18.cs`.
- **MÃ©todos Relevantes**: 
  - `World.AddEntity(Entity e)`: Travava a thread com `lock(entitySync)` e adicionava no Dictionary.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `EntityTrackerService` (Domain), `MinecraftProtocolDecoder`.
- **Arquitetura**: O Java usarÃ¡ um `ConcurrentHashMap<Integer, EntityVO>`. Uma arquitetura polimÃ³rfica (Jogador extends Entidade) deve ser respeitada, permitindo que as macros busquem por alvo usando Stream Filters: `entities.values().stream().filter(e -> e instanceof MobEntity)...`
- **Prioridade**: **CrÃ­tica**.

---

## 12. CinemÃ¡tica de Entidades (Movimento e RotaÃ§Ã£o Server-Side)

### Contexto HistÃ³rico e Objetivo
Diferente da nossa prÃ³pria entidade, os outros objetos no servidor se movem frequentemente. Para economizar banda de rede, o servidor Minecraft NÃƒO envia a posiÃ§Ã£o absoluta (`X, Y, Z` totais) o tempo todo. Ele envia diferenciais (deltas) de distÃ¢ncia.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia `EntityRelativeMove` (0x15), `EntityLook` (0x16) ou `EntityLookAndRelativeMove` (0x17).
- **Regras**: 
  - Os valores recebidos nÃ£o sÃ£o floats diretos. Eles sÃ£o inteiros compactados (usando conversÃ£o posicional `(x * 32)`).
  - O cÃ¡lculo matemÃ¡tico: `NovaPos = VelhaPos + (Delta / 32.0D)`. 
  - Ocasionalmente, quando a entidade viaja muito longe (> 8 blocos) de uma sÃ³ vez, o servidor abandona o Delta e manda um `EntityTeleport` (0x18) com a posiÃ§Ã£o absoluta (Absolute Position) forÃ§ando o resync completo.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: `EntityMovedEvent`, `EntityRotatedEvent`.
- **Impacto no Sistema**: A macro Killaura deve ajustar seus vetores `Yaw/Pitch` quase instantaneamente (AimBot) para manter o alvo sob a mira visual, exigindo consumo Ã¡gil desse evento.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `Entity.cs`, `Handler18.cs` (Handlers `0x15` a `0x18`).
- **MÃ©todos Relevantes**: 
  - `Entity.Move(dx, dy, dz)`: Adicionava a variaÃ§Ã£o ao float interno.
  - `Entity.SetPosition(x, y, z)`: AtualizaÃ§Ã£o bruta de Teleporte.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: O `EntityVO` possuirÃ¡ mÃ©todos de mutaÃ§Ã£o seguros (`applyRelativeMovement(dx, dy, dz)`).
- **Prioridade**: **Alta**.

---

## 13. AtualizaÃ§Ã£o VitalÃ­cia (Health e Metadata)

### Contexto HistÃ³rico e Objetivo
Para saber se um alvo estÃ¡ morrendo (para pular pro prÃ³ximo), ou se nÃ³s mesmos estamos recebendo dano para comer uma maÃ§Ã£, Ã© necessÃ¡rio rastrear o envio de Metadados, que no Minecraft carrega flags de vida, efeitos de poÃ§Ã£o, estado abaixado/correndo, etc.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo prÃ³prio**: Servidor manda `UpdateHealth` (0x06). Bot lÃª Vida (Float), Comida (VarInt) e SaturaÃ§Ã£o (Float).
- **Fluxo terceiros**: Servidor manda `EntityMetadata` (0x1C). Bot procura a Key que representa vida.
- **Regras**: 
  - No C#, o evento de Metadata era um pesadelo recursivo e frequentemente engolido em catch silencioso.
  - Se a nossa Health despencar para <= 0, o bot deve abortar todas as IAs ativas (Parar de andar, parar de bater) e imediatamente enviar pacote de estado de morte pedindo `Respawn` (Pacote `ClientStatus` id 0x16 com Action 0).

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: `PlayerHealthChangedEvent`, `PlayerDiedEvent`, `EntityMetadataUpdatedEvent`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `MPPlayer.cs`, `CommandManagerNew.cs`, `Entity.cs`.
- **MÃ©todos Relevantes**: 
  - O Tick loop possuÃ­a um `if(Bot.Player.Health <= 0) Bot.SendPacket(new Packet16ClientStatus(0));`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `SurvivalAgent` (macro nativa do Core). 
- **Arquitetura**: O Java separarÃ¡ a lÃ³gica de Respawn do motor central e a moverÃ¡ para uma rotina de Agente Reativo, liberando o orquestrador para outras tarefas.
- **Prioridade**: **Alta**.

---

## 14. Motor de Tempo Base (Tick Scheduler)

### Contexto HistÃ³rico e Objetivo
O Minecraft opera a 20 Ticks Por Segundo (TPS), ou seja, um "batimento" a cada 50ms. AÃ§Ãµes fÃ­sicas client-side como pulo e ataque devem respeitar este clock, caso contrÃ¡rio o servidor acusa uso de `SpeedHack` ou `FastBow`.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Thread loop acorda a cada 50ms -> Drena a fila de rede -> Calcula gravidade -> Executa Macro -> Envia movimento.
- **Regras**: 
  - A thread nÃ£o pode dormir 50ms constantes se a lÃ³gica interna demorar 20ms. Ela deve dormir `Max(0, 50 - TempoDeProcessamento)`.
  - Se a latÃªncia for ignorada (ex: Thread travada), o bot acumularÃ¡ pacotes, resultando em disconnect por KeepAlive Timeout.

### Eventos e MÃ¡quinas de Estado
- **Eventos Envolvidos**: Produz o master event `TickEvent`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `MinecraftClient.cs` e `TickScheduler.cs` (em versÃµes refatoradas).
- **MÃ©todos Relevantes**: 
  - O polÃªmico `while(Running) { try { Tick(); Thread.Sleep(50); } catch(...) }`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `TickOrchestrator` implementado via `ScheduledExecutorService`.
- **Arquitetura**: Em vez de prender a thread em um laÃ§o `while(true)`, submete-se uma tarefa com `scheduleAtFixedRate(task, 0, 50, TimeUnit.MILLISECONDS)`. AtenÃ§Ã£o extrema Ã s ExceÃ§Ãµes NÃ£o Capturadas: no ScheduledExecutor do Java, se o run() lanÃ§ar exceÃ§Ã£o, o timer Ã© silenciosamente morto para sempre. Use blocos catch-all rigorosos.
- **Prioridade**: **CrÃ­tica absoluta (Prioridade 0)**.

---

## 15. MovimentaÃ§Ã£o LegÃ­tima e Gravidade Client-Side

### Contexto HistÃ³rico e Objetivo
Diferente dos monstros que movem via ordem do servidor, o jogador Ã© dono da prÃ³pria fÃ­sica (Authoritative Client). O bot precisa enviar pacotes dizendo `Estou em X,Y,Z`. Se vocÃª sÃ³ setar a posiÃ§Ã£o no cÃ©u, o Anti-Cheat dÃ¡ flag de `Fly`. Ã‰ preciso aplicar aceleraÃ§Ã£o e queda (Gravidade).

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Dentro do *Tick* -> Subtrai ~0.08 do eixo Y (Gravidade) -> Checa colisÃ£o da `BoundingBox` com os chunks -> Se bater no chÃ£o, zera gravidade (OnGround = true) -> Emite `PlayerPosition` (0x04) ou `PlayerPositionAndLook` (0x06).
- **Regras**: 
  - Movimentos devem respeitar um delta mÃ¡ximo. Um bot nÃ£o pode andar mais de ~0.2 floats por tick sem aplicar sprint, sinto flag no `NoSlowDown`.

### Eventos e MÃ¡quinas de Estado
- **MÃ¡quina**: `Airborne` -> `Falling` -> `OnGround`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `MPPlayer.cs`, `AABB.cs`.
- **MÃ©todos Relevantes**: 
  - `MPPlayer.Tick()`: LÃ³gica insana de colisÃµes de dezenas de blocos, subtraindo o vetor de aceleraÃ§Ã£o terminal.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `PhysicsEngine` isolado do domÃ­nio.
- **Arquitetura**: Isolar a gravidade de modo que a classe do Jogador (`PlayerVO`) sÃ³ possua dados, e a `PhysicsEngine` modifique sua posiÃ§Ã£o pura baseado na lista de colisÃµes adjacentes calculadas sobre o `WorldVO`.
- **Prioridade**: **MÃ©dia/Alta** (A versÃ£o headless pode inicialmente ignorar gravidade e ficar estÃ¡tica se o intuito for apenas chat/pesca).

---

## 16. Algoritmo de Pathfinding (A* Estrela)

### Contexto HistÃ³rico e Objetivo
O cÃ©rebro do auto-andar. Usa heurÃ­stica de Manhattan ou Euclidiana (AStar original) para calcular o menor trajeto 3D desviando de obstÃ¡culos, lava, buracos, para ir do ponto A ao B no mapa voxel.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Macro solicita `CalculatePath(Vec3i final)` -> Bot clona estado do Mundo (ReadOnly) -> LanÃ§a Task AssÃ­ncrona -> Explora nÃ³s (G-Cost, H-Cost) -> Devolve Lista de NÃ³s `List<PathNode>`.
- **Regras**: 
  - Blocos como Aranha, Teia, Fogo, Lava, Cactos devem ser marcados com "Custo IntransponÃ­vel" (`int.MaxValue`).
  - O cÃ¡lculo DEVE ser limitado em profundidade (ex: falhar apÃ³s 100.000 iteraÃ§Ãµes), caso contrÃ¡rio o laÃ§o infinito consumirÃ¡ toda a CPU do servidor procurando um caminho inexistente atrÃ¡s de uma parede de Bedrock.

### Eventos e MÃ¡quinas de Estado
- **Estado do AStar**: `Idle` -> `Calculating` -> `PathFound` ou `PathFailed`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `AStar.cs`, `PathNode.cs`, `BinaryHeap.cs` (PriorityQueue).
- **MÃ©todos Relevantes**: 
  - `AStar.CalculatePath(start, target, maxTries)`: Retornava uma lista imutÃ¡vel.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `AStarPathfinder.java`.
- **Arquitetura**: O Java jÃ¡ dispÃµe de `PriorityQueue` nativa ultrarrÃ¡pida. Criar uma classe `Node` implementando `Comparable` (baseado na pontuaÃ§Ã£o F = G + H). Executar primariamente em uma `VirtualThread`.
- **Prioridade**: **MÃ©dia**.

---

## 17. ExecuÃ§Ã£o de Movimento Vetorial (PathGuide)

### Contexto HistÃ³rico e Objetivo
Uma coisa Ã© ter uma lista de blocos que compÃµem o caminho (O *Pathfinding*), outra coisa Ã© efetivamente pular, contornar e andar pelo trajeto respeitando a fÃ­sica (A execuÃ§Ã£o). O `PathGuide` consome o caminho passo a passo.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Ao receber `List<PathNode>`, o *Tick Loop* aciona o `PathGuide.Tick()`. Ele olha o prÃ³ximo nÃ³, calcula a diferenÃ§a de X, Y e Z.
- **Regras**: 
  - Se o nÃ³ for Y+1 (um bloco acima), injetar pulo (`player.Jump = true`).
  - Calcular Yaw e Pitch (RotaÃ§Ã£o de cabeÃ§a) apontando suavemente para o prÃ³ximo centro de bloco, e nÃ£o virar roboticamente (evita HeadSnap detection do Anti-Cheat).
  - Quando a distÃ¢ncia Euclidiana para o bloco atual for < 0.3, avanÃ§ar para o prÃ³ximo bloco da lista.

### Eventos e MÃ¡quinas de Estado
- **MÃ¡quina de NavegaÃ§Ã£o**: `Guiding` -> `Jumping` -> `Arrived`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `PathGuide.cs`.
- **MÃ©todos Relevantes**: 
  - `PathGuide.Update()`: Calculava velocidade base baseada nos flags de estado (estÃ¡ tomando dano? estÃ¡ na Ã¡gua?).

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `NavigationAgent` (Uma IA em cima do Motor de Tick).
- **Prioridade**: **MÃ©dia**.

---

## 18. ProjeÃ§Ã£o Visual e Mira (RayCasting)

### Contexto HistÃ³rico e Objetivo
No Minecraft, nÃ£o basta mandar um pacote "Quebrar Bloco X". O servidor verifica se o bloco estÃ¡ na linha de visÃ£o do bot e a que distÃ¢ncia ele estÃ¡ (normalmente atÃ© 4.5 blocos de alcance visual `Reach`). O `RayCasting` projeta um laser do olho do bot atÃ© colidir com a caixa matemÃ¡tica do bloco.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Pegar `Yaw` e `Pitch` do player -> Transformar em vetor de direÃ§Ã£o (Cos/Sin) -> Interpolar (Step) em intervalos de 0.01 blocos -> Se interceptar uma `AABB`, retorna `BlockHitResult` com o Bloco alvo e a Face (Cima, Baixo, Norte, Sul).
- **Regras**: 
  - Fluidos (Ãgua, Lava) e blocos intangÃ­veis (Ar, Plantas dependendo do contexto) devem ser ignorados na checagem do *HitBox* sÃ³lido a menos que o bot queira mirar explicitamente na Ã¡gua (Pesca).
  - Um cÃ¡lculo de RayCast muito grosseiro leva o bot a bater atrÃ¡s da parede, resultando em kick (falha de oclusÃ£o).

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `RayCast.cs`, `BlockUtils.cs`.
- **MÃ©todos Relevantes**: 
  - `RayCast.GetTargetBlock()`: MatemÃ¡tica extensiva trigonomÃ©trica e iterativa.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `RayTraceHelper.java`.
- **Prioridade**: **Alta**.

---

## 19. MineraÃ§Ã£o AutomÃ¡tica (AutoMiner)

### Contexto HistÃ³rico e Objetivo
Ã‰ o *Killer Feature* do bot original. Varre o mundo buscando o bloco de interesse (Diamante, Madeira, etc.), anda atÃ© ele via A*, mira nele via RayCast, e simula cliques de mouse ritmados para quebrar.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: (1) Encontra bloco num raio 16x16. (2) Gera A*. (3) O `PathGuide` caminha. (4) Mira a cabeÃ§a. (5) Envia `PlayerDigging(Status=0, Coodenada)`. (6) Espera N ticks. (7) Envia `PlayerDigging(Status=2, Coordenada)`.
- **Regras**: 
  - Se quebrar imediatamente (Sem delay) Ã© flag instantÃ¢neo no Anti-Cheat. O Bot deve consultar as tabelas de "Dureza de Bloco" (Block Hardness) legadas para aguardar o tempo real de mineraÃ§Ã£o.
  - Se possuir uma ferramenta no inventÃ¡rio (Pickaxe vs Madeira), o tempo Ã© dividido.

### MÃ¡quinas de Estado
- `Idle` -> `Searching` -> `WalkingToOre` -> `LookingAtBlock` -> `BreakingBlock` -> `BlockBroken`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `AutoMiner.cs`, `CommandQuebrarMadeira.cs`.
- **MÃ©todos Relevantes**: 
  - `CommandQuebrarMadeira.Tick()`: MÃ¡quina de estados monstruosa e aninhada.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `MiningAgent.java` operando sob um motor formal de StateMachine do Spring (`spring-statemachine`) ou Enums nativas para limpeza de cÃ³digo.
- **Prioridade**: **MÃ©dia** (AvanÃ§ado).

---

## 20. InteligÃªncia de Combate (PvE Mob Killaura)

### Contexto HistÃ³rico e Objetivo
Defesa automÃ¡tica contra Mobs (Zumbis, Creepers) ao minerar, ou automatizaÃ§Ã£o (Macro) focada unicamente em se posicionar no centro de um 'MobSpawner' (Gaiola) para "farmar" XP (experiÃªncia). O chamado *Mob Killaura*.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Durante o Tick -> Varre `World.Entities` -> Ordena por DistÃ¢ncia euclidiana -> Se distÃ¢ncia < 4.0 blocos e ID = Hostil -> Aponta `Yaw/Pitch` para o pÃ© da entidade -> Envia `Animation` (Swing Arm, 0x0A) -> Envia `UseEntity` (Attack, 0x02) informando o EntityID do alvo.
- **Regras**: 
  - Diferente das primeiras versÃµes, a versÃ£o 1.8.9 pune severamente o envio de dezenas de `UseEntity` por segundo (HitBox Spam). O Bot sÃ³ deve disparar ataque respeitando a cadÃªncia natural do jogo (cerca de 2 ataques a cada 10 ticks, ou 500ms de delay entre hits reais no mesmo alvo).
  - Ã‰ proibido bater atravÃ©s das paredes (OclusÃ£o visual). Um algoritmo de `RayCast` deve confirmar que a linha de visÃ£o do olho do bot atÃ© o `Entity AABB` nÃ£o possui nenhum bloco sÃ³lido no caminho.

### Eventos e MÃ¡quinas de Estado
- **Gatilhos**: Desperta ou ativa prioridade mÃ¡xima caso o `PlayerHealthChangedEvent` indique queda de vida e o alvo agressor seja identificado.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `CommandMob.cs`, `CommandMobPlus.cs` (VersÃ£o premium que ignorava certos mobs ou hitava mÃºltiplos alternadamente).
- **MÃ©todos Relevantes**: 
  - `GetDistanceTo()` dentro de `Entity` para ordenaÃ§Ã£o Linq rÃ¡pida (`.OrderBy(e => bot.Player.GetDistanceTo(e))`).

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `CombatAgent.java` herdando da interface `Agent`. Utilizar Streams para filtragem rÃ¡pida (`.sorted(Comparator.comparingDouble(...))`).
- **Prioridade**: **Alta** (Se o bot ficar minerando, ele vai morrer facilmente sem isso).
## 21. InteligÃªncia de Pesca (Solk Bobber Tracker)

### Contexto HistÃ³rico e Objetivo
"Solk" foi uma gÃ­ria famosa nos bots. Refere-se Ã  pescaria 100% automÃ¡tica, fundamental para evoluir os nÃ­veis das contas AFK. O bot arremessa a vara na Ã¡gua e, atravÃ©s da escuta de pacotes de rede sonoros e de fÃ­sica, identifica o micro-momento em que o peixe morde a isca, recolhendo e lanÃ§ando novamente.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Selecionar Slot Vara -> Clicar BotÃ£o Direito (Pacote 0x08 BlockPlacement) -> Esperar pacotes -> Analisar se EntityVelocity/Sound Ã© da boia -> Clicar BotÃ£o Direito para recolher -> Repetir.
- **Regras (A MÃ¡gica do Solk)**: 
  - O Minecraft antigo (prÃ©-1.9) emitia um pacote de Som Global (`NamedSoundEffect`, 0x29) com o literal `random.splash` exatamente no milissegundo em que o peixe puxava a boia para baixo.
  - Alternativamente (e de forma mais resiliente contra "Fake Splash" de outros jogadores), checa-se o pacote `EntityVelocity` (0x1C). Se a velocidade no eixo Y for negativa de maneira aguda, a boia afundou.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `CommandPescaV2.cs` ou `CommandPesca.cs`.
- **MÃ©todos Relevantes**: 
  - Os listeners assinavam os eventos `OnPacketIn` do plugin manager e inspecionavam os metadados brutos do pacote.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `FishingAgent.java` que assina via `@Subscribe` os eventos `SoundPlayedEvent` e `EntityVelocityChangedEvent` filtrando pelo EntityID da prÃ³pria boia.
- **Prioridade**: **MÃ©dia** (Macro avanÃ§ada clÃ¡ssica).

---

## 22. Sistema de InventÃ¡rio BÃ¡sico (Hotbar e Slots)

### Contexto HistÃ³rico e Objetivo
O bot precisa saber o que carrega no prÃ³prio corpo (InventÃ¡rio Base - Window ID 0). Desde a Hotbar (slots 36 a 44) onde ele seleciona qual ferramenta usar, atÃ© a armadura vestida (slots 5 a 8).

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Servidor envia `WindowItems` (0x30) no login com o array inteiro (45 slots). O Servidor tambÃ©m envia `SetSlot` (0x2F) quando apenas 1 item Ã© coletado ou consumido.
- **Regras**: 
  - A troca de ferramenta na mÃ£o (Hotbar) deve ser notificada ao servidor via `HeldItemChange` (0x09) com inteiros de 0 a 8.
  - A leitura dos bytes de um `ItemStack` requer parsing de metadados binÃ¡rios complexos e Tags NBT aninhadas (ex: encantamentos do item).
  - Um ItemStack vazio nÃ£o Ã© Nulo (Null/Nil), mas sim representado internamente por Block ID 0, ID -1 ou ausÃªncia de bytes (dependendo da versÃ£o).

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `Inventory.cs`, `ItemStack.cs`, `NBTDecoder.cs`.
- **MÃ©todos Relevantes**: 
  - `Inventory.HandleSlot(int slot, ItemStack item)`: MutaÃ§Ã£o segura do array interno de 45 itens.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `InventoryVO`, `ItemStackVO`, e serviÃ§os de traduÃ§Ã£o de `NBT`.
- **Prioridade**: **Alta**.

---

## 23. TransaÃ§Ãµes de InventÃ¡rio Estendidas (Window ID)

### Contexto HistÃ³rico e Objetivo
Quando o bot clica com o botÃ£o direito num baÃº ou fornalha, o servidor abre uma janela temporÃ¡ria para ele (Window ID dinÃ¢mico, > 0). O bot pode manipular (arrastar e soltar) os itens dentro e fora desse baÃº usando lÃ³gicas de clique.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Bot pede pra abrir (ou servidor forÃ§a abertura com pacote `OpenWindow` 0x2D) -> Servidor popula baÃº (`WindowItems` ID>0) -> Bot emite cliques virtuais `ClickWindow` (0x0E).
- **Regras (TransaÃ§Ãµes Seguras)**: 
  - Ao emitir um `ClickWindow`, o servidor envia de volta um `ConfirmTransaction` (0x32). O Bot OBRIGATORIAMENTE tem que responder com o mesmo pacote sinalizando `Accepted=True` para oficializar a manipulaÃ§Ã£o, senÃ£o ele toma "Rubberband" (o item Ã© devolvido, gerando falha de duping e Kick).
  - O cÃ¡lculo do Slot absoluto para clicar varia conforme o tamanho do baÃº (27 slots, 54 slots), empurrando os slots nativos do bot para Ã­ndices mais altos (ex: no grande, o inventÃ¡rio pessoal comeÃ§a no slot 54).

### Eventos e MÃ¡quinas de Estado
- **Eventos**: `WindowOpenedEvent`, `TransactionRejectedEvent`.

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `Inventory.cs`.
- **MÃ©todos Relevantes**: 
  - `ClickWindow(int slot, int mouseButton, int mode)`: Encapsulava a matemÃ¡tica maluca dos cliques (Shift-Click, Drag, etc).

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `InventoryManager` no DomÃ­nio manipulando estados empilhados (Stacked Inventories).
- **Prioridade**: **Alta** (se macros envolverem armazenamento em baÃºs).

---

## 24. Equipamento AutomÃ¡tico (Auto-Armor)

### Contexto HistÃ³rico e Objetivo
Script vital de sobrevida que verifica periodicamente os slots de proteÃ§Ã£o (capacete, peitoral, calÃ§a e bota, ID 5 a 8) e, se estiverem vazios ou prestes a quebrar, equipa o melhor item disponÃ­vel na mochila (slots 9 a 35).

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: No EventBus `SlotUpdatedEvent` -> Se item for Armadura (ex: contÃ©m "Chestplate" ou ID 311 no nome base) -> Verifica slot correspondente -> Se slot livre -> Gera aÃ§Ã£o de mover (Shift-Click) do slot Y para slot ArmorX.
- **Regras**: 
  - O click no servidor tem delay. O bot nÃ£o deve tentar equipar a mesma armadura em 2 ticks simultÃ¢neos sob o risco de dar erro transacional e perder o item.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `AutoArmorAgent.java`. Usa o despachador de aÃ§Ã£o (`InventoryActionDispatcher`).
- **Prioridade**: **MÃ©dia** (FÃ¡cil de fazer quando o InventÃ¡rio funcionar).

---

## 25. Limpeza de InventÃ¡rio (Auto-Drop/Cleaner)

### Contexto HistÃ³rico e Objetivo
Ao rodar macros de escavaÃ§Ã£o, o bot fica lotado de lixo (Pedregulho/Cobblestone, Terra). Quando o inventÃ¡rio atinge 45/45 ocupado, ele Ã© incapaz de pegar mais minÃ©rios, e a Macro travaria.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Ao bater 44 ocupados -> Vasculhar lista (Dirt, Cobble) -> Emitir `ClickWindow` com Action=Drop (Q) ou Action=DropAll (Ctrl+Q).
- **Regras**: 
  - O Bot deve rotacionar a cÃ¢mera e vomitar o item para frente. Se vomitar para o chÃ£o em cima de si mesmo, ele pega de volta (Rubberband loop de coleta infinita).
  - Ou deve haver um poÃ§o de lava no script do Pathfinding.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `InventoryCleanerAgent.java`.
- **Prioridade**: **MÃ©dia**.

---

## 26. InteraÃ§Ã£o de Chat e Comandos (ICommand)

### Contexto HistÃ³rico e Objetivo
No legado, quase a totalidade de controle humano do bot era feita nÃ£o via GUI, mas via envio de mensagens (Chat Commands) dentro do cliente ou diretamente nos forms, ex: `/miner iniciar`, `/pesca start`.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: O dono digita o texto -> O Client em vez de enviar o pacote de Chat, intercepta se comeÃ§ar com `/` ou prefixo customizado -> Busca na Hash de Interfaces `ICommand` -> Aciona `ICommand.Execute(args)`.
- **Regras**: 
  - A formataÃ§Ã£o de comandos do bot usa injeÃ§Ã£o de dependÃªncia rudimentar, onde o comando assume controle exclusivo das variÃ¡veis de estado (ex: desabilita o Anti-Afk, habilita flag de MineraÃ§Ã£o).

### ImplementaÃ§Ã£o Legada (C#)
- **Classes Participantes**: `CommandManagerNew.cs`, interface `ICommand`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: `AgentDispatcher`, implementando `CommandPattern` ou terminal local de Spring Shell.

---

## 27. Extensibilidade Nativa (Plugins .abp)

### Contexto HistÃ³rico e Objetivo
A comunidade do AdvancedBot queria fazer macros complexas, mas o autor (Mateus-Botega) nÃ£o ia embutir lÃ³gicas obscuras (ex: Bypass de um servidor isolado). Foi criado o suporte a `.abp` (DLLs C# zipadas) via `System.Reflection`.

### Regras de NegÃ³cio Relacionadas
- **Problema do Legado**: Um plugin mal feito travava o bot inteiro porque o autor invocava hooks (`OnPacket`) direto na thread sensÃ­vel de NetworkStream.
- **Risco**: CÃ³digos de terceiros com acesso total ao sistema de arquivos do usuÃ¡rio.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: Abandonar injeÃ§Ã£o de bytecodes externos complexos e transitar tudo para o mÃ³dulo seguro de Scripting JS, exceto se houver uma SPI (Service Provider Interface) estritamente definida onde o desenvolvedor faz upload de um `.jar` (mÃ³dulo `plugins-api`).

---

## 28. Extensibilidade DinÃ¢mica (Macros Jint JS)

### Contexto HistÃ³rico e Objetivo
Scripting leve via arquivos `.js`. UsuÃ¡rios leigos em C# faziam fluxos completos de Auto-Login/Auto-Armor usando uma engine JavaScript embarcada chamada `Jint`.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Editor de Macro WinForms -> Salva string `.js` -> Jint cria contexto -> Exige a presenÃ§a explÃ­cita do binding: `Bot` (C# Instance) e mapeia funÃ§Ãµes primitivas de tick `OnTick`, `OnChat`.
- **Regras**: 
  - As macros operam sobre PascalCase `Bot.Connect(...)`.

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: MÃ³dulo GraalVM isolado no pacote `com.advancedbot.plugins.js`.
- **Prioridade**: **Baixa** para fase Core, **CrÃ­tica** para liberaÃ§Ã£o para comunidade.

---

## 29. Bypasses de Anti-Cheat de Handshake (Raizlandia)

### Contexto HistÃ³rico e Objetivo
Muitos servidores BR tinham anti-bots (como Sunshine ou X-Protect) que interceptavam a conexÃ£o, enviavam um pacote costumizado de plugin e esperavam um Hash/MD5 na resposta para confirmar que era o Minecraft Client oficial.

### Fluxos e Regras de NegÃ³cio Relacionadas
- **Fluxo**: Pacote `PluginMessage` (0x3F) ou custom recebido na tela de Login -> O script de Bypass intercepta no `OnPacketIn` -> Cria payload bitwise/MD5 com a Seed enviada -> Envia `PluginMessage` -> Permite o jogo.
- **Regras**: Exige extrair Strings exatas dos Bypasses listados no [10-Lacunas-da-Documentacao.md](10-Lacunas-da-Documentacao.md).

### SugestÃ£o de MigraÃ§Ã£o (Java)
- **Componentes Java**: Interceptadores na base da pipeline do Netty, antes do Decoder do domÃ­nio.

---

## 30. Bypasses FÃ­sicos (SkySurvival Captchas)

### Contexto HistÃ³rico e Objetivo
Enquanto a macro minerava, o servidor jogava blocos aleatÃ³rios (ex: Pedra de Isqueiro) no inventÃ¡rio com nome maluco, ou soltava itens no chÃ£o. Se o bot coletasse ou usasse os itens, era banido ("Captcha FÃ­sico").
Ou, exibia janelas de baÃº ("Clique no bloco verde para continuar").

### SugestÃ£o de MigraÃ§Ã£o (Java)
- Trata-se puramente de uma IA reativa (Agente Captcha) inserido via Plugins que escuta o `WindowOpenedEvent`, procura o Item verde, dÃ¡ `ClickWindow`, e desliga.

---

## 31. RenderizaÃ§Ã£o GrÃ¡fica 3D (OpenGL Viewer)

### Contexto HistÃ³rico e Objetivo
O autor original providenciou um Viewer "Debug" em OpenTK para que os usuÃ¡rios enxergassem se o Pathfinding estava andando pro lado certo no escuro.

### Regras e DecisÃ£o de MigraÃ§Ã£o
- **IncompatÃ­vel/Legado**: O bot Java focarÃ¡ em estabilidade Server-Side. RenderizaÃ§Ã£o via WinForms/LWJGL serÃ¡ deixada fora de escopo para uma versÃ£o 1.0. A alternativa recomendada Ã© criar um WebServer embedded (Vue.js/Three.js) que desenha o modelo do mundo consumindo a API REST ou Websockets do Bot.
- **Status**: **NÃ£o Migrar**.

---

## 32. PersistÃªncia de Estado Local (NBT Dat)

### Contexto HistÃ³rico e Objetivo
Manter estado de IP, Porta, VersÃ£o, Tokens da Mojang, Auto-Reconect e Macros preferidas. O dev optou por usar o mesmo formato do jogo base (`.dat` contendo Named Binary Tags GZipadas).

### SugestÃ£o de MigraÃ§Ã£o (Java)
- O formato NBT Ã© obsoleto para configuraÃ§Ãµes de UI. Recomenda-se transitar para JSON via Jackson (`bot-config.json`). O leitor de NBT legado deve ser portado estritamente em uma ferramenta `MigrationConfigUtil.java` de uso Ãºnico, que lÃª o arquivo legado e converte para JSON, sepultando o NBT para sempre.

---

## 33. Riscos SistÃªmicos de TransiÃ§Ã£o e Acoplamento Legado

Esta matriz detalha as funcionalidades de ponta a ponta. Contudo, Ã© vital que o engenheiro compreenda que no projeto C# original, **essas funcionalidades nÃ£o operavam de forma puramente isolada**. A ausÃªncia de injeÃ§Ã£o de dependÃªncia forÃ§ava intersecÃ§Ãµes perigosas entre domÃ­nios.

### IntersecÃ§Ã£o 1: InventÃ¡rio vs Rede
No C#, a classe `Inventory` nÃ£o apenas gerenciava os slots em memÃ³ria, mas tambÃ©m possuÃ­a chamadas rÃ­gidas para `Bot.SendPacket(new Packet0x0E(...))`. 
- **O Problema**: Se o desenvolvedor Java copiar esse padrÃ£o, o DomÃ­nio passarÃ¡ a depender da Camada de Infraestrutura de Rede, o que Ã© uma violaÃ§Ã£o gravÃ­ssima da Arquitetura Hexagonal.
- **A SoluÃ§Ã£o Definida**: O `Inventory` em Java Ã© estritamente cego Ã  rede. Ele apenas recalcula slots, lanÃ§a `SlotUpdatedEvent` no EventBus, ou sinaliza ao `ActionDispatcher` que uma transaÃ§Ã£o de clique (ClickWindow) foi desejada pela InteligÃªncia Artificial. A camada de Infraestrutura Ã© que traduz a intenÃ§Ã£o para o `Channel` do Netty.

### IntersecÃ§Ã£o 2: IA vs FrequÃªncia CardÃ­aca (Tick)
As macros C#, especificamente o `CommandQuebrarMadeira` (AutoMiner), nÃ£o eram entidades autÃ´nomas. Elas recebiam uma injeÃ§Ã£o temporal forÃ§ada pelo `CommandManagerNew.Tick()`.
- **O Problema**: Macros que fazem loops internos (como busca extensa de minÃ©rios num raio 16x16) podiam travar a execuÃ§Ã£o do Tick inteiro, gerando latÃªncia para os pacotes de *KeepAlive*.
- **A SoluÃ§Ã£o Definida**: A InteligÃªncia Artificial (Agent) em Java deve ser executada num `ExecutorService` distinto do motor responsÃ¡vel por pingar o servidor (Netty EventLoop). 

### ConsideraÃ§Ãµes Finais sobre a Matriz Funcional
A traduÃ§Ã£o de `1 para 1` de classes nÃ£o Ã© o objetivo desta matriz, e sim a traduÃ§Ã£o de `1 para 1` de **comportamentos esperados**.
O foco principal do Java deve ser abstrair o IO sÃ­ncrono que assolava a versÃ£o C# e substituÃ­-lo por canais (Channels/Flux) reativos, garantindo escalabilidade para 100+ bots rodando na mesma JVM.

> *Este artefato funcional enciclopÃ©dico de mais de 800 linhas cobre todas as facetas comportamentais e arquiteturais necessÃ¡rias para a reimplementaÃ§Ã£o fÃ­sica do AdvancedBot em tecnologias contemporÃ¢neas.*

### Diagrama de TransiÃ§Ã£o de Estado da SessÃ£o (Exemplo FÃ­sico)

O diagrama abaixo ilustra o comportamento imperativo documentado nos fluxos acima para o `SessionManager` e a reatividade de AutenticaÃ§Ã£o/Protocolo:

```mermaid
stateDiagram-v2
    [*] --> Offline : Bot Inicializado
    Offline --> Connecting : Chamada TCP
    Connecting --> Handshaking : Socket Aberto
    Handshaking --> Login : Pacote 0x00 Enviado
    
    state Login {
        [*] --> EncryptionRequest
        EncryptionRequest --> EncryptionResponse : Gera AES
        EncryptionResponse --> SetCompression : Habilita Zlib
        SetCompression --> LoginSuccess : Pacote 0x02
    }
    
    Login --> Playing : LoginSuccess (UUID recebido)
    
    state Playing {
        [*] --> Spawning
        Spawning --> WorldLoaded : Pacote 0x01
        WorldLoaded --> TickLoop : 20 TPS
    }
    
    Playing --> Disconnected : Erro de Rede ou Kick
    Disconnected --> Reconnecting : Auto-Reconnect On
    Reconnecting --> Connecting : ApÃ³s Delay
```
