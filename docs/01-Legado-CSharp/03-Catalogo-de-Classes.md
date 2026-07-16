# 03 — Catálogo de Classes

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-03                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Catálogo Técnico                                 |
| **Escopo**         | Todas as classes, interfaces e enums do sistema  |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-02, LEG-04, LEG-08                 |

---

## Critério de Criticidade

| Nível | Definição |
|-------|-----------|
| **CRÍTICO** | Classe central. Bloqueante para a migração. Implementação incorreta compromete todo o sistema. |
| **ALTO** | Classe importante. Afeta múltiplos fluxos. Necessita implementação cuidadosa. |
| **MÉDIO** | Classe de suporte. Afeta fluxo específico. Implementação moderadamente complexa. |
| **BAIXO** | Classe auxiliar. Fácil de migrar. Baixo risco. |

---

## Módulo: AdvancedBot.Client

### MinecraftClient

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `MinecraftClient.cs` |
| **Tipo** | Classe concreta |
| **Implementa** | `IDisposable` |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Classe central de todo o sistema. Representa uma instância de bot conectado. Gerencia o ciclo de vida completo da conexão, o estado do jogo e o tick de atualização.

**Dependências:**
- `PacketStream`, `PacketQueue`
- `ProtocolHandler` (e suas implementações)
- `Entity`, `World`, `Inventory`
- `CommandManagerNew`
- `PathGuide`
- `PlayerManager`
- `SessionUtils`
- `Proxy`
- `Program.pluginManager`

**Campos Estáticos Relevantes:**
- `AutoReconnect` — controla a reconexão automática global.
- `Knockback` — habilita/desabilita knockback simulado.
- `MaximumChatLines` — limite de linhas no histórico de chat.
- `ReconnectType reconnectType` — tipo de reconexão.

**Campos de Instância Relevantes:**
- `instanceID` — identificador numérico do bot.
- `IP`, `Port` — endereço do servidor.
- `Username`, `Email`, `Password` — credenciais.
- `beingTicked` — flag indicando que a conexão está ativa.
- `canLoop` — flag do loop principal de leitura.
- `kickTicks` — contador de ticks até reconexão após kick.
- `connStatus` — estado da fase de handshake (0=login, 1=play).
- `keepAliveTicks` — contador de keep-alive.
- `LoggedIn` — indica autenticação AuthMe concluída.
- `PlayerVIP` — status VIP do jogador no servidor.

---

### Entity

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `Entity.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Motor de física do jogador. Implementa colisão AABB, movimento, gravidade, interação com fluidos e escadas, LookAt e RayCast.

**Dependências:** `World`, `AABB`, `Vec3d`, `Movement`

**Métodos Críticos:**
- `Tick()` — atualiza física a cada tick (20/s).
- `Move(xa, ya, za)` — aplica movimento com colisão.
- `MoveRelative(xa, za, speed)` — calcula vetor de movimento relativo à rotação.
- `LookTo(x, y, z)` — calcula Yaw/Pitch para mirar em ponto.
- `RayCastBlocks(radius)` — detecta bloco na linha de visão.

---

### Inventory

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `Inventory.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Gerencia os slots do inventário do jogador e de janelas abertas (baús, lojas). Implementa click com geração de número de transação.

**Dependências:** `ItemStack`, `PacketClickWindow`, `PacketConfirmTransaction`, `MinecraftClient`

---

### Proxy

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `Proxy.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Implementa conexão TCP via proxies HTTP, SOCKS4 e SOCKS5 em modo síncrono e assíncrono completo.

**Dependências:** `ProxyType`, `TcpClient`, `Socket`

---

### PacketStream

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `PacketStream.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Stream de rede assíncrona que lê pacotes Minecraft, suporta encriptação AES-CFB8 e compressão zlib, e dispara eventos ao receber pacotes completos.

**Dependências:** `AesStream`, `Ionic.Zlib`, `ReadBuffer`

---

### PacketQueue

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `PacketQueue.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Fila thread-safe de pacotes pendentes de envio. Serializa os pacotes e os envia no `Flush()`.

**Dependências:** `IPacket`, `PacketStream`, `WriteBuffer`, `ProtocolHandler`

---

### SessionUtils

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `SessionUtils.cs` |
| **Tipo** | Classe estática |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Comunicação com a API de autenticação Mojang (Yggdrasil). Executa login, join session e refresh de token.

**Dependências:** `HttpConnection`, `Newtonsoft.Json`, `LoginResponse`, `LoginCache`

---

### World

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Map` |
| **Arquivo** | `World.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Armazena todos os chunks recebidos. Expõe métodos para consultar blocos, dados de blocos, colisão, RayCast e fluxo de fluidos.

**Dependências:** `Chunk`, `ChunkSection`, `BlockUtils`, `AABB`, `Vec3d`, `HitResult`

---

### ChatParser

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `ChatParser.cs` |
| **Tipo** | Classe estática |
| **Criticidade** | **MÉDIO** |

**Responsabilidade:** Converte o JSON do formato de chat Minecraft para texto plano com códigos de cor `§`.

**Dependências:** `Newtonsoft.Json`

---

### AutoMiner

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `AutoMiner.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **MÉDIO** |

**Responsabilidade:** Lógica de mineração automática em área configurável.

---

### AABB

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `AABB.cs` |
| **Tipo** | Estrutura (struct-like class) |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Caixa de colisão alinhada com os eixos. Implementa todos os algoritmos de clip de colisão usados pela física do jogador.

**Métodos Críticos:** `ClipXCollide`, `ClipYCollide`, `ClipZCollide`, `MoveClone`, `Expand`, `Grow`, `Move`

---

### DiggingHelper

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `DiggingHelper.cs` |
| **Tipo** | Classe estática |
| **Criticidade** | **MÉDIO** |

**Responsabilidade:** Cálculo da velocidade de quebra de blocos considerando ferramenta, eficiência e tipo de bloco.

---

### LoginCache

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `LoginCache.cs` |
| **Tipo** | Classe estática |
| **Criticidade** | **MÉDIO** |

**Responsabilidade:** Cache de tokens de autenticação Mojang em arquivo JSON para evitar relogins.

---

### LookInterpolator

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client` |
| **Arquivo** | `LookInterpolator.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **BAIXO** |

**Responsabilidade:** Interpolação suave de rotação de câmera entre dois ângulos.

---

## Módulo: AdvancedBot.Client.Handler

### ProtocolHandler (abstrata)

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Handler` |
| **Arquivo** | `ProtocolHandler.cs` |
| **Tipo** | Classe abstrata |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Define o contrato de todos os handlers de protocolo. Provê factory method `Create(version)` que retorna a implementação correta.

**Métodos Abstratos:**
- `HandlePacket(ReadBuffer pkt)` — interpreta um pacote recebido.
- `WritePacket(ref int v18id, IPacket pkt, WriteBuffer buf)` — serializa um pacote para envio.

---

### Handler_v152

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Handler` |
| **Arquivo** | `Handler_v152.cs` |
| **Tipo** | Classe concreta |
| **Herda** | `ProtocolHandler` |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Handler síncrono do protocolo Minecraft 1.5.2. Usa `TcpClient` diretamente com loop de leitura em thread dedicada.

---

### Handler_v17 / Handler_v18 / Handler_v19

| Classe | Protocolo | Criticidade |
|--------|-----------|-------------|
| `Handler_v17` | 1.7 / 1.7.10 | **ALTO** |
| `Handler_v18` | 1.8 | **ALTO** |
| `Handler_v19` | 1.9 | **MÉDIO** |

**Responsabilidade:** Handlers assíncronos usando `PacketStream`. Interpretam todos os pacotes do servidor na fase PLAY e chamam os métodos correspondentes em `MinecraftClient`.

---

## Módulo: AdvancedBot.Client.PathFinding

### PathFinder

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.PathFinding` |
| **Arquivo** | `PathFinder.cs` |
| **Tipo** | Classe estática |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Algoritmo A* para encontrar caminho entre dois pontos no mundo Minecraft. Considera blocos passáveis, pulos, quedas e água.

---

### PathGuide

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.PathFinding` |
| **Arquivo** | `PathGuide.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Executa o caminho encontrado pelo `PathFinder`, enfileirando movimentos na `MoveQueue` da entidade a cada tick.

---

## Módulo: AdvancedBot.Client.Commands

### ICommand (Interface)

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Commands` |
| **Arquivo** | `ICommand.cs` |
| **Tipo** | Interface |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Contrato base de todos os comandos e macros. Define os métodos de ciclo de vida.

**Métodos:**
- `Start(MinecraftClient client, string[] args)` — inicializa o comando.
- `Tick()` — chamado a cada tick enquanto ativo.
- `Stop()` — para a execução.
- `Name` — nome do comando.
- `IsRunning` — indica se está em execução.

---

### CommandPesca / CommandPescaV2

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Commands.Solk` |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Macro de pesca com máquina de estados complexa. Controla: lançamento da vara, detecção de peixe (análise de posição da bobber), puxada, movimentação até baú, depósito de itens.

---

### CommandMob / CommandMobPlus

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Commands.Solk` |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Macro de grinding de mobs. Detecta mobs próximos, navega até eles, aplica ataques, gerencia inventário e reparo de equipamento.

---

### MacroUtils

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Commands.Solk` |
| **Arquivo** | `MacroUtils.cs` |
| **Tipo** | Classe estática |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Biblioteca de utilitários async compartilhados pelas macros. Inclui: encontrar itens, abrir/fechar baús, mover itens, vender, comprar, selecionar melhor ferramenta.

---

## Módulo: AdvancedBot.Client.Crypto

### AesStream

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Crypto` |
| **Arquivo** | `AesStream.cs` |
| **Tipo** | Classe concreta |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Implementa stream AES-CFB8 bidirecional para encriptação do protocolo Minecraft após o handshake.

---

### CryptoUtils

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.Crypto` |
| **Arquivo** | `CryptoUtils.cs` |
| **Tipo** | Classe estática |
| **Criticidade** | **CRÍTICO** |

**Responsabilidade:** Geração de chave AES aleatória, decode de chave pública RSA (SubjectPublicKeyInfo), cálculo do server hash SHA-1 (algoritmo não-standard do Minecraft).

---

## Módulo: AdvancedBot.Client.NBT

### Tag (abstrata)

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Client.NBT` |
| **Tipo** | Classe abstrata |
| **Criticidade** | **ALTO** |

**Responsabilidade:** Hierarquia completa de tags NBT do Minecraft. Usada para configuração (`conf.dat`) e dados de itens/tiles do protocolo.

---

## Módulo: AdvancedBot.Plugins

### IPlugin (Interface)

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Plugins` |
| **Tipo** | Interface |
| **Criticidade** | **MÉDIO** |

**Responsabilidade:** Contrato que todos os plugins externos devem implementar.

**Métodos:**
- `onClientConnect(MinecraftClient client)`
- `onReceiveChat(string chat, byte pos, MinecraftClient client)`
- `onSendChat(string msg, MinecraftClient client)`
- `Tick()`
- `Unload()`

---

### PluginManager

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Plugins` |
| **Tipo** | Classe concreta |
| **Criticidade** | **MÉDIO** |

**Responsabilidade:** Carrega plugins de `.dll` (texto plano) e `.abp` (encriptado AES). Usa `FileSystemWatcher` para hot reload. Gerencia comandos registrados pelos plugins.

---

## Módulo: AdvancedBot.Viewer

### ViewForm

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Viewer` |
| **Tipo** | Classe concreta (Windows Form) |
| **Criticidade** | **BAIXO** |

**Responsabilidade:** Formulário Windows Forms com contexto OpenGL (WGL). Implementa o loop de renderização 3D do mundo, controle de câmera, GUI em overlay.

---

### ChunkRenderer

| Campo | Valor |
|-------|-------|
| **Namespace** | `AdvancedBot.Viewer` |
| **Tipo** | Classe estática |
| **Criticidade** | **BAIXO** |

**Responsabilidade:** Renderização de chunks usando VBOs ou immediate mode OpenGL. Suporta transparência e Z-sorting.

---

## Enums do Sistema

| Enum | Namespace | Valores Principais |
|------|-----------|-------------------|
| `ClientVersion` | `AdvancedBot.Client` | `v1_5_2`, `v1_7`, `v1_7_10`, `v1_8`, `v1_9`, `Unknown` |
| `ProxyType` | `AdvancedBot.Client` | `HTTP`, `Socks4`, `Socks5` |
| `ReconnectType` | `AdvancedBot.Client` | `type_0`, `type_1` |
| `InventoryType` | `AdvancedBot.Client` | Tipos de janela de inventário |
| `Movement` | `AdvancedBot.Client` | `None`, `Forward`, `Back`, `Left`, `Right`, `Jump` |
| `DiggingStatus` | `AdvancedBot.Client.Packets` | `StartedDigging`, `CancelledDigging`, `FinishedDigging` |
| `TokenType` | `AdvancedBot.Script` | Tokens do script parser |

---

## Interfaces do Sistema

| Interface | Namespace | Implementações |
|-----------|-----------|----------------|
| `IPacket` | `AdvancedBot.Client` | Todos os `Packet*.cs` |
| `ICommand` | `AdvancedBot.Client.Commands` | Todos os `Command*.cs` |
| `IEntity` | `AdvancedBot.Client.Entitybase` | `EntityMob`, `MPPlayer` |
| `IPlugin` | `AdvancedBot.Plugins` | Plugins externos |
| `IGuiControl` | `AdvancedBot.Viewer.Gui` | `GuiCheckBox`, `GuiTrackBar`, `GuiOptions` |

---

## Resumo Quantitativo

| Categoria | Quantidade |
|-----------|------------|
| Classes concretas | ~95 |
| Classes estáticas | ~12 |
| Classes abstratas | 2 |
| Interfaces | 5 |
| Enums | 7 |
| **Total** | **~121** |
