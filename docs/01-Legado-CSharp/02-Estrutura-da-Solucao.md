# 02 — Estrutura da Solução

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-02                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Documento Técnico — Estrutura                    |
| **Escopo**         | Solução Visual Studio e organização de módulos   |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-01, LEG-03, LEG-07                 |

---

## Solução Visual Studio

| Campo | Valor |
|-------|-------|
| **Arquivo** | `AdvancedBot_Crack.sln` |
| **Versão do Visual Studio** | Visual Studio 2022 (17.14) |
| **Formato da Solução** | `12.00` |
| **Configurações** | `Debug\|Any CPU`, `Release\|Any CPU` |
| **GUID da Solução** | `{FD618240-61DB-4EDB-B9E7-048C644D2B41}` |

---

## Projeto Principal

| Campo | Valor |
|-------|-------|
| **Arquivo** | `AdvancedBot_Crack.csproj` |
| **Nome do Assembly** | `AdvancedBot` |
| **Tipo de Output** | `Exe` (executável Windows) |
| **SDK** | `Microsoft.NET.Sdk.WindowsDesktop` |
| **Framework** | `net462` (.NET Framework 4.6.2) |
| **Plataforma** | `x86` (32 bits) |
| **Windows Forms** | `True` |
| **Versão da Linguagem** | `C# 7.3` |
| **Unsafe Blocks** | `True` |
| **GUID do Projeto** | `{58258F2F-0BC7-4AEF-740E-C4E3EDCE4D36}` |
| **Ícone** | `app.ico` |

---

## Organização de Módulos

O projeto é um único projeto C# com módulos organizados como pastas físicas no sistema de arquivos.

Não existem assemblies separados para os módulos internos.

### Mapa de Módulos

```
AdvancedBot_Crack.csproj (raiz)
│
├── AdvancedBot/                    — Interface gráfica e orquestração principal
├── AdvancedBot.Client/             — Core do cliente Minecraft
├── AdvancedBot.Client.Bypassing/   — Bypasses específicos de servidores
├── AdvancedBot.Client.Commands/    — Sistema de comandos e macros
│   └── Solk/                       — Macros avançadas (Pesca, Mob, Teleport)
├── AdvancedBot.Client.Crypto/      — Criptografia AES para o protocolo
├── AdvancedBot.Client.Entitybase/  — Entidades do mundo Minecraft
├── AdvancedBot.Client.Handler/     — Handlers de protocolo por versão
├── AdvancedBot.Client.Map/         — Mapa, chunks e utilitários de blocos
├── AdvancedBot.Client.NBT/         — Parser e writer do formato NBT
├── AdvancedBot.Client.Packets/     — Definições de pacotes de rede
├── AdvancedBot.Client.PathFinding/ — Algoritmo de pathfinding A*
├── AdvancedBot.Crypto/             — Criptografia AES para plugins encriptados
├── AdvancedBot.Plugins/            — Sistema de plugins
├── AdvancedBot.Properties/         — Metadados do assembly
├── AdvancedBot.Protection/         — Verificação de HWID
├── AdvancedBot.ProxyChecker/       — Verificador de proxies
├── AdvancedBot.Script/             — Motor de scripts proprietário e JS
├── AdvancedBot.Viewer/             — Renderizador 3D OpenGL
└── AdvancedBot.Viewer.Gui/         — Controles GUI para o renderizador
```

---

## Detalhamento por Módulo

### AdvancedBot — Interface e Orquestração

**Namespace:** `AdvancedBot`

**Responsabilidade:** Ponto de entrada da aplicação, janelas Windows Forms, gerenciamento global de configurações e estado.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `Program.cs` | Ponto de entrada `Main()`, configuração global, `SaveConf`/`LoadConf`, log de erros |
| `Main.cs` | Formulário principal, timer de tick, exibição de bots, chat |
| `Start.cs` | Formulário de configuração e inicialização de bots |
| `FrmLogin.cs` | Formulário de login com conta Mojang |
| `MacroEditor.cs` | Editor de scripts de macro |
| `Spammer.cs` | Formulário do spammer de mensagens |
| `AccountChecker.cs` | Verificador de contas Minecraft |
| `BanCheck.cs` | Verificador de ban em servidores |
| `ProxyCheckerForm.cs` | Interface do verificador de proxies |
| `ProxyForm.cs` | Gerenciamento da lista de proxies |
| `ProxyListView.cs` | ListView customizado para proxies |
| `ProxyList.cs` | Modelo da lista de proxies |
| `Statistics.cs` | Estatísticas de uso e performance |
| `TestServer.cs` | Utilitário de teste de servidores |
| `MinerOptions.cs` | Formulário de opções do minerador |
| `NickGenerator.cs` | Gerador de nicks aleatórios |
| `SrvResolver.cs` | Resolução de DNS SRV para servidores Minecraft |
| `About.cs` / `AboutNew.cs` | Janelas "Sobre" |
| `Changelog.cs` | Log de mudanças da versão |
| `UserListBox.cs` | ListBox customizado para usuários |
| `PercentageProgressBar.cs` | Barra de progresso percentual |
| `RtfBuilder.cs` | Construtor de texto RTF para o chat |
| `SetPrefixForm.cs` | Configuração de prefixo de comandos |
| `ErrorMarker.cs` | Marcador de erros no editor |
| `FuncAutocompleteItem.cs` | Item de autocomplete do editor |

---

### AdvancedBot.Client — Core do Cliente

**Namespace:** `AdvancedBot.Client`

**Responsabilidade:** Toda a lógica de comunicação com o servidor Minecraft, estado do jogo, física do jogador, inventário e entidades.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `MinecraftClient.cs` | Classe central do bot. Gerencia conexão, tick, física, estado |
| `Entity.cs` | Física do jogador (posição, movimento, colisão, salto) |
| `Inventory.cs` | Inventário do jogador e janelas abertas |
| `ItemStack.cs` | Representação de um stack de itens |
| `Item.cs` | Dados estáticos de um item |
| `Items.cs` | Catálogo estático de todos os IDs de itens |
| `Blocks.cs` | Catálogo estático de todos os IDs de blocos e suas propriedades |
| `Block.cs` | Dados de um bloco (dureza, tipo de ferramenta, etc.) |
| `AABB.cs` | Axis-Aligned Bounding Box para colisão |
| `Vec3d.cs` | Vetor 3D de precisão dupla |
| `Vec3i.cs` | Vetor 3D de inteiros |
| `Entity.cs` | Entidade jogador com física completa |
| `PacketStream.cs` | Stream de rede com suporte a encriptação e compressão |
| `PacketQueue.cs` | Fila de envio de pacotes com serialização |
| `ReadBuffer.cs` | Buffer de leitura de pacotes |
| `WriteBuffer.cs` | Buffer de escrita de pacotes |
| `MinecraftStream.cs` | Wrapper de stream para o protocolo Minecraft |
| `HttpConnection.cs` | Conexão HTTP para comunicação com a API Mojang |
| `HttpResponse.cs` | Modelo de resposta HTTP |
| `SessionUtils.cs` | Login Mojang (Yggdrasil), check de sessão |
| `LoginCache.cs` | Cache de tokens de login Mojang |
| `LoginResponse.cs` | Modelo da resposta de autenticação |
| `ChatParser.cs` | Parser de JSON do chat Minecraft (§ codes) |
| `CommandManagerNew.cs` | Gerenciador de execução de comandos internos |
| `AutoMiner.cs` | Lógica de mineração automática |
| `DiggingHelper.cs` | Cálculo de velocidade de quebra de blocos |
| `LookInterpolator.cs` | Interpolação suave de rotação |
| `Proxy.cs` | Suporte a proxies HTTP, SOCKS4, SOCKS5 (sync e async) |
| `ProxyType.cs` | Enum dos tipos de proxy |
| `PlayerManager.cs` | Gerenciador da lista de jogadores online |
| `PlayerNick.cs` | Modelo de nick de jogador |
| `PlayerTab.cs` | Modelo de entrada da tab-list |
| `MPPlayer.cs` | Entidade de outro jogador no mundo |
| `Movement.cs` | Enum de direções de movimento |
| `ReconnectType.cs` | Enum de tipos de reconexão |
| `ClientVersion.cs` | Enum de versões suportadas do protocolo |
| `InventoryType.cs` | Enum de tipos de inventário |
| `IPacket.cs` | Interface base de todos os pacotes |
| `UUID.cs` | Utilitário para geração e manipulação de UUIDs |
| `Utils.cs` | Utilitários gerais (XorShift, GetTickCount64, StripColorCodes, etc.) |

---

### AdvancedBot.Client.Handler — Handlers de Protocolo

**Namespace:** `AdvancedBot.Client.Handler`

**Responsabilidade:** Leitura e interpretação dos pacotes recebidos do servidor, por versão do protocolo.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `ProtocolHandler.cs` | Classe abstrata base para todos os handlers |
| `Handler_v152.cs` | Handler do protocolo Minecraft 1.5.2 (legado, síncrono) |
| `Handler_v17.cs` | Handler do protocolo Minecraft 1.7 / 1.7.10 |
| `Handler_v18.cs` | Handler do protocolo Minecraft 1.8 |
| `Handler_v19.cs` | Handler do protocolo Minecraft 1.9 |
| `PacketStreamLegacy.cs` | Stream legada para o protocolo 1.5.2 |

---

### AdvancedBot.Client.Packets — Definições de Pacotes

**Namespace:** `AdvancedBot.Client.Packets`

**Responsabilidade:** Serialização dos pacotes enviados pelo cliente ao servidor.

| Arquivo | Pacote | Descrição |
|---------|--------|-----------|
| `PacketHandshake.cs` | Handshake | Início da conexão |
| `PacketLoginStart.cs` | Login Start | Envio do username |
| `PacketEncryptionResponse.cs` | Encryption Response | Resposta de encriptação RSA |
| `PacketClientSettings.cs` | Client Settings | Configurações do cliente (view distance, idioma) |
| `PacketClientStatus.cs` | Client Status | Status (respawn) |
| `PacketPluginMessage.cs` | Plugin Message | Mensagem de plugin (MC\|Brand) |
| `PacketChatMessage.cs` | Chat Message | Envio de mensagem de chat |
| `PacketPlayerPos.cs` | Player Position | Posição do jogador |
| `PacketPlayerLook.cs` | Player Look | Rotação do jogador |
| `PacketPosAndLook.cs` | Position and Look | Posição e rotação combinadas |
| `PacketUpdate.cs` | Player (ground) | Atualização de `OnGround` |
| `PacketPlayerDigging.cs` | Player Digging | Início/fim de quebra de bloco |
| `PacketBlockPlace.cs` | Block Placement | Colocação de bloco ou uso de item |
| `PacketSwingArm.cs` | Animation (Swing) | Animação de swing do braço |
| `PacketHeldItemChange.cs` | Held Item Change | Mudança de slot ativo no hotbar |
| `PacketEntityAction.cs` | Entity Action | Agachar, correr, etc. |
| `PacketUseEntity.cs` | Use Entity | Interação com entidade (atacar, usar) |
| `PacketClickWindow.cs` | Click Window | Click em slot de inventário |
| `PacketConfirmTransaction.cs` | Confirm Transaction | Confirmação de transação de inventário |
| `PacketCloseWindow.cs` | Close Window | Fechamento de janela |
| `PacketCreativeInvAction.cs` | Creative Inventory Action | Ação de inventário criativo |
| `PacketKeepAlive.cs` | Keep Alive | Resposta de keep-alive |
| `PacketTeleportConfirm.cs` | Teleport Confirm | Confirmação de teleporte (1.9+) |
| `PacketUseItem.cs` | Use Item | Uso de item na mão (1.9+) |
| `UnsafeDirectPacket.cs` | Raw Packet | Envio de pacote raw (bytes diretos) |
| `DiggingStatus.cs` | — | Enum dos estados de quebra de bloco |

---

### AdvancedBot.Client.Commands — Comandos e Macros

**Namespace:** `AdvancedBot.Client.Commands`

**Responsabilidade:** Implementação de todos os comandos internos do bot.

| Arquivo | Comando | Descrição |
|---------|---------|-----------|
| `ICommand.cs` | Interface | Contrato base para todos os comandos |
| `CommandResult.cs` | — | Resultado de execução de um comando |
| `CommandAntiAFK.cs` | `$antiafk` | Prevenção de kick por AFK |
| `CommandBreakBlock.cs` | `$breakblock` | Quebrar bloco por coordenada |
| `CommandClearChat.cs` | `$clearchat` | Limpar o chat |
| `CommandClickBlock.cs` | `$clickblock` | Clicar em bloco específico |
| `CommandDropAll.cs` | `$dropall` | Dropar todos os itens |
| `CommandFollow.cs` | `$follow` | Seguir um jogador |
| `CommandGive.cs` | `$give` | Dar item a si mesmo (modo criativo) |
| `CommandGoto.cs` | `$goto` | Mover para coordenada |
| `CommandHelp.cs` | `$help` | Listar comandos disponíveis |
| `CommandHerbalism.cs` | `$herbalism` | Automação de coleta de plantas |
| `CommandHotbarClick.cs` | `$hotbarclick` | Clicar no hotbar |
| `CommandInvCaptcha.cs` | `$invcaptcha` | Resolver captcha de inventário |
| `CommandInvClick.cs` | `$invclick` | Clicar em slot de inventário |
| `CommandKillAura.cs` | `$killaura` | Kill Aura (ataque automático de entidades) |
| `CommandMiner.cs` | `$miner` | Iniciar mineração automática |
| `CommandMove.cs` | `$move` | Mover em direção específica |
| `CommandPlaceBlock.cs` | `$placeblock` | Colocar bloco em coordenada |
| `CommandPlayerList.cs` | `$playerlist` | Listar jogadores online |
| `CommandPortal.cs` | `$portal` | Navegar por portal do Nether/End |
| `CommandProxy.cs` | `$proxy` | Exibir proxy atual |
| `CommandReco.cs` | `$reco` | Forçar reconexão |
| `CommandRetard.cs` | `$retard` | Movimento aleatório (humanização) |
| `CommandScript.cs` | `$script` | Executar script |
| `CommandSneak.cs` | `$sneak` | Agachar/Desagachar |
| `CommandTwerk.cs` | `$twerk` | Animação de twerk (agachar repetidamente) |
| `CommandUseBow.cs` | `$usebow` | Usar arco automaticamente |
| `CommandUseEntity.cs` | `$useentity` | Usar entidade específica |

#### Submódulo Solk (Macros Avançadas)

**Namespace:** `AdvancedBot.Client.Commands.Solk`

| Arquivo | Responsabilidade |
|---------|-----------------|
| `CommandPesca.cs` | Macro de pesca V1 com máquina de estados |
| `CommandPescaV2.cs` | Macro de pesca V2 com máquina de estados estendida |
| `CommandMob.cs` | Macro de combate com mobs (grinding) |
| `CommandMobPlus.cs` | Versão estendida do CommandMob com mais funcionalidades |
| `CommandMobTeleport.cs` | Teleporte até mobs para grinding |
| `MacroUtils.cs` | Utilitários compartilhados entre macros (findItem, openChest, etc.) |
| `ConfigMob.cs` | Configurações da macro de mob |
| `FileLock.cs` | Controle de lock de arquivo para persistência de estado |

---

### AdvancedBot.Client.Map — Mapa e Mundo

**Namespace:** `AdvancedBot.Client.Map`

**Responsabilidade:** Armazenamento e consulta do estado do mundo Minecraft.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `World.cs` | Gerenciamento de chunks, consulta de blocos, RayCast, pathfinding helpers |
| `Chunk.cs` | Estrutura de dados de um chunk (16x256x16) |
| `ChunkSection.cs` | Seção de chunk (16x16x16) |
| `BlockUtils.cs` | Utilitários de blocos (fluxo de fluidos, colisão, etc.) |
| `HitResult.cs` | Resultado de um RayCast (bloco atingido e face) |
| `SignTile.cs` | Dados de uma placa (tile entity NBT) |

---

### AdvancedBot.Client.NBT — Formato NBT

**Namespace:** `AdvancedBot.Client.NBT`

**Responsabilidade:** Parsing e serialização do formato NBT (Named Binary Tag), usado pelo Minecraft para dados de itens, chunks e configuração.

| Arquivo | Tag NBT |
|---------|---------|
| `Tag.cs` | Classe base abstrata de todas as tags |
| `CompoundTag.cs` | Tag composta (chave-valor) |
| `ListTag.cs` | Tag lista |
| `ByteTag.cs` | Tag byte |
| `ShortTag.cs` | Tag short |
| `IntTag.cs` | Tag int |
| `LongTag.cs` | Tag long |
| `FloatTag.cs` | Tag float |
| `DoubleTag.cs` | Tag double |
| `StringTag.cs` | Tag string |
| `ByteArrayTag.cs` | Tag array de bytes |
| `IntArrayTag.cs` | Tag array de inteiros |
| `EndTag.cs` | Tag de encerramento de Compound |
| `DataInput.cs` | Leitor binário para desserialização |
| `DataOutput.cs` | Escritor binário para serialização |
| `NbtIO.cs` | Leitura/escrita de arquivos NBT no disco |

---

### AdvancedBot.Client.Crypto — Criptografia de Protocolo

**Namespace:** `AdvancedBot.Client.Crypto`

**Responsabilidade:** Implementação da criptografia AES usada no protocolo Minecraft após o handshake.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `AesStream.cs` | Stream encriptada/desencriptada com AES-CFB8 |
| `CryptoUtils.cs` | Utilitários: geração de chave AES, decode de chave pública RSA, server hash |

---

### AdvancedBot.Client.Entitybase — Entidades

**Namespace:** `AdvancedBot.Client.Entitybase`

**Responsabilidade:** Representação das entidades do mundo (mobs, outros jogadores).

| Arquivo | Responsabilidade |
|---------|-----------------|
| `IEntity.cs` | Interface base de entidade |
| `EntityMob.cs` | Entidade mob (posição, tipo, propriedades) |
| `EntityProperty.cs` | Propriedades de um tipo de mob (altura, hitbox) |
| `EntityManager.cs` | Gerenciador de entidades do mundo |

---

### AdvancedBot.Client.PathFinding — Pathfinding

**Namespace:** `AdvancedBot.Client.PathFinding`

**Responsabilidade:** Algoritmo de navegação automática A*.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `PathFinder.cs` | Implementação do algoritmo A* |
| `Path.cs` | Sequência de pontos de um caminho |
| `PathPoint.cs` | Ponto individual do caminho com custo G, H, F |
| `PathGuide.cs` | Guia de execução do caminho (integração com Entity) |

---

### AdvancedBot.Client.Bypassing — Bypasses

**Namespace:** `AdvancedBot.Client.Bypassing`

**Responsabilidade:** Implementação de lógicas específicas de bypass para servidores particulares.

| Arquivo | Servidor | Responsabilidade |
|---------|----------|-----------------|
| `SkySurvival.cs` | SkySurvival | Bypass específico do servidor SkySurvival |
| `WorldCraftBP.cs` | WorldCraft | Bypass específico do servidor WorldCraft |

---

### AdvancedBot.Script — Motor de Scripts

**Namespace:** `AdvancedBot.Script`

**Responsabilidade:** Motor de scripts proprietário para automação configurável pelo usuário.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `ScriptParser.cs` | Parser da linguagem de script proprietária |
| `ScriptContext.cs` | Contexto de execução do script (variáveis, funções, comandos) |
| `JsScriptContext.cs` | Contexto de execução JavaScript via `Jint` |
| `Token.cs` | Token léxico do parser |
| `TokenType.cs` | Enum dos tipos de token |
| `Expression.cs` | Expressão do script |
| `Function.cs` | Definição de função no script |
| `Variable.cs` | Variável do script |
| `ParserException.cs` | Exceção de parsing |
| `ScriptException.cs` | Exceção de execução de script |

---

### AdvancedBot.Plugins — Sistema de Plugins

**Namespace:** `AdvancedBot.Plugins`

**Responsabilidade:** Carregamento, gerenciamento e execução de plugins externos.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `IPlugin.cs` | Interface que todo plugin deve implementar |
| `PluginManager.cs` | Carregamento de `.dll` e `.abp`, hot reload, execução de callbacks |
| `AdvancedBotAPI.cs` | API pública exposta aos plugins |

---

### AdvancedBot.Protection — Proteção

**Namespace:** `AdvancedBot.Protection`

**Responsabilidade:** Verificação de licença por hardware.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `HWIDChecker.cs` | Verificação do HWID (Hardware ID) da máquina |

---

### AdvancedBot.ProxyChecker — Verificador de Proxies

**Namespace:** `AdvancedBot.ProxyChecker`

**Responsabilidade:** Verificação em massa da disponibilidade de proxies.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `ProxyCheckQueue.cs` | Fila de verificação de proxies |
| `ProxyInfo.cs` | Modelo de resultado de verificação de um proxy |
| `ProxyUtils.cs` | Utilitários de parsing e verificação de proxies |

---

### AdvancedBot.Crypto — Criptografia para Plugins

**Namespace:** `AdvancedBot.Crypto`

**Responsabilidade:** Criptografia AES usada para proteger plugins `.abp`.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `AesEncryption.cs` | Encriptação/desencriptação AES de arquivos de plugin |
| `ResponseQuery.cs` | Modelo de resposta de consulta |
| `WebClient.cs` | Cliente HTTP para validação de licença |

---

### AdvancedBot.Viewer — Renderizador 3D

**Namespace:** `AdvancedBot.Viewer`

**Responsabilidade:** Visualização 3D do mundo Minecraft usando OpenGL via P/Invoke.

| Arquivo | Responsabilidade |
|---------|-----------------|
| `ViewForm.cs` | Formulário Windows Forms com contexto OpenGL, loop de renderização |
| `ChunkRenderer.cs` | Renderização de chunks (blocos, transparências) |
| `WorldRenderer.cs` | Renderização do mundo completo |
| `AsyncChunkBuilder.cs` | Construção assíncrona de VBOs de chunks |
| `GL.cs` | Bindings OpenGL via P/Invoke (função a função) |
| `WGL.cs` | Bindings WGL (Windows OpenGL extensions) |
| `VBO.cs` | Vertex Buffer Object |
| `Tessellator.cs` | Tessellador de vértices para renderização |
| `TextureManager.cs` | Gerenciamento de texturas OpenGL |
| `Font.cs` | Renderização de texto via OpenGL |
| `Frustum.cs` | Frustum culling para otimização |

---

### AdvancedBot.Viewer.Gui — Controles GUI OpenGL

**Namespace:** `AdvancedBot.Viewer.Gui`

**Responsabilidade:** Controles GUI renderizados dentro do contexto OpenGL (sobreposto ao mundo 3D).

| Arquivo | Responsabilidade |
|---------|-----------------|
| `IGuiControl.cs` | Interface base dos controles GUI OpenGL |
| `GuiUtils.cs` | Utilitários de renderização de GUI |
| `GuiOptions.cs` | Painel de opções da câmera e visualização |
| `GuiCheckBox.cs` | Checkbox renderizado via OpenGL |
| `GuiTrackBar.cs` | TrackBar renderizado via OpenGL |

---

## Dependências Externas (DLLs)

| Biblioteca | Arquivo | Origem |
|------------|---------|--------|
| `Newtonsoft.Json` | `Newtonsoft.Json.dll` | NuGet |
| `Jint` | `Jint.dll` | NuGet / Compilado localmente |
| `Transitions` | `Transitions.dll` | Biblioteca de animações WinForms |
| `System.Core` | Framework | .NET Framework 4.6.2 |
| `System.Windows.Forms.DataVisualization` | Framework | Gráficos de dados |
| `System.Xml` | Framework | Parsing XML |
| `System.Management` | Framework | WMI (usado pelo HWID checker) |

---

## Arquivos de Configuração e Dados

| Arquivo | Localização | Formato | Descrição |
|---------|-------------|---------|-----------|
| `conf.dat` | Raiz do executável | NBT binário | Configurações persistidas da aplicação |
| `errlogs/*.txt` | `errlogs/` | Texto | Logs de erro e crash |
| `Plugins/*.dll` | `Plugins/` | Assembly .NET | Plugins em formato aberto |
| `Plugins/*.abp` | `Plugins/` | AES-encriptado | Plugins em formato encriptado |
