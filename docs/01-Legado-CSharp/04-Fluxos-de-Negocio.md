# 04 — Fluxos de Negócio

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-04                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Documento Técnico — Fluxos                       |
| **Escopo**         | Todos os fluxos principais de negócio            |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-01, LEG-03, LEG-07                 |

---

## Fluxo 1 — Inicialização da Aplicação

### Entrada

Usuário executa `AdvancedBot.exe`.

### Processamento

1. `Program.Main()` é chamado (STA Thread).
2. `ThreadPool.SetMaxThreads(16, 16)` — limita o pool a 16 threads.
3. `ProcessorAffinity` é configurada para usar todos os núcleos menos um.
4. Handlers de exceção global são registrados (`AppDomain.UnhandledException`, `Application.ThreadException`).
5. `Application.EnableVisualStyles()` é chamado.
6. `Main` (formulário principal) é instanciado.
7. `PluginManager.Init()` — carrega todos os plugins da pasta `Plugins/`.
8. `LoadConf()` — lê `conf.dat` em formato NBT e aplica configurações.
9. `FrmMain.CheckKey()` — valida a chave de licença (HWID).
10. `Application.Run(FrmMain)` — inicia o loop de mensagens do Windows Forms.

### Saída

Janela principal do bot exibida e pronta para uso.

---

## Fluxo 2 — Conexão e Handshake (v1.7 / v1.8 / v1.9)

### Entrada

Usuário configura IP, porta, usuário/senha e clica em "Conectar".

### Processamento

```
MinecraftClient.StartClient()
    └── ConnectAndHandshakeAsync()
            ├── [Se conta Mojang] → ThreadPool → AuthenticateMojang()
            │       └── SessionUtils.Login(Email, Password, Proxy)
            │               └── POST https://authserver.mojang.com/authenticate
            │                       └── Retorna AccessToken, UUID, Username
            │
            ├── [Se SendPing] → ThreadPool → Ping()
            │       └── Envia PacketHandshake(state=1) e lê resposta JSON
            │
            └── TcpClient.BeginConnect() ou Proxy.BeginConnect()
                    └── ConnectCallback()
                            ├── PacketStream criado
                            ├── PacketQueue criado
                            ├── PacketHandshake(version, ip, port, nextState=2) enviado
                            ├── PacketLoginStart(username) enviado
                            ├── SendQueue.Flush()
                            ├── connStatus = 0 (fase LOGIN)
                            └── Stream.OnPacketAvailable += HandlePacket
```

### Handshake de Encriptação (servidores online-mode)

```
Servidor → EncryptionRequest (serverID, publicKey, verifyToken)
    └── HandlePacket(pkt.ID == 1)
            ├── CryptoUtils.DecodeRSAPublicKey(publicKey)
            ├── CryptoUtils.GenerateAESPrivateKey() → sharedSecret (16 bytes)
            ├── SessionUtils.CheckSession(UUID, AccessToken, serverHash, proxy)
            │       └── POST https://sessionserver.mojang.com/session/minecraft/join
            ├── RSA.Encrypt(sharedSecret) → encryptedSecret
            ├── RSA.Encrypt(verifyToken) → encryptedToken
            ├── SendQueue.AddToQueue(PacketEncryptionResponse)
            ├── SendQueue.Flush()
            └── Stream.InitEncryption(sharedSecret) → AES-CFB8 ativado

Servidor → LoginSuccess (UUID, Username)
    └── HandlePacket(pkt.ID == 2)
            └── connStatus = 1 (fase PLAY)
```

### Saída

Bot conectado ao servidor, aguardando pacote `JoinGame`.

---

## Fluxo 3 — Conexão Legada (v1.5.2)

### Entrada

Versão configurada como `v1.5.2`.

### Processamento

Thread dedicada `MC152 {Username}` é criada e executa `Connect_v15()`:

1. Login Mojang (se email fornecido).
2. `Handler_v152.Ping()` (se ativado).
3. `TcpClient` conectado diretamente (ou via proxy).
4. `Handler_v152.ConnectAndHandshake()` — handshake síncrono do protocolo antigo.
5. Loop principal de leitura em thread dedicada.
6. `handler_v.ReadAndHandlePacket()` chamado repetidamente.
7. Timeout por `keepAliveTicks > 200` (10 loops de 10ms = 2 segundos sem resposta).

### Diferença

O protocolo 1.5.2 usa um formato de pacotes completamente diferente (não VarInt, não comprimido).

---

## Fluxo 4 — Tick de Jogo

### Entrada

Timer Windows Forms no `Main.cs` dispara a cada 50ms (~20 ticks/s).

### Processamento

Para cada instância de `MinecraftClient` ativa:

```
MinecraftClient.Tick()
    ├── [Plugins] pluginManager.plugins → value.Tick()
    ├── tickStatus++ (ciclo 0-40)
    ├── [Se beingTicked]
    │       ├── SendQueue.Flush() — envia pacotes pendentes
    │       ├── [Se timeout de handshake > 20s] → Disconnect()
    │       ├── [Se keepAliveTicks > 750] → Disconnect("timeout")
    │       ├── [Se CurrentPath != null] → CurrentPath.Tick()
    │       ├── CmdManager.Tick() — executa comando/macro ativo
    │       └── [Se MapAndPhysics]
    │               ├── Player.Tick() — física do jogador
    │               ├── [Sprint changed] → PacketEntityAction(sprint/stop)
    │               ├── [Chunk não existe sob o jogador] → cai lentamente
    │               ├── [Player em portal + OnGround] → MoveQueue.Clear()
    │               └── Envia PacketPosAndLook / PacketPlayerPos / PacketPlayerLook / PacketUpdate
    │
    └── [Se AutoReconnect + kickTicks > 40 + type_0]
            └── StartClient() — reconecta
```

### Saída

Estado do bot atualizado, pacotes de posição enviados ao servidor.

---

## Fluxo 5 — Recepção de Pacotes

### Entrada

Dados chegam na `NetworkStream` do socket TCP.

### Processamento

```
PacketStream (thread interna de leitura)
    ├── Lê VarInt (tamanho do pacote)
    ├── [Se compressão] → Lê VarInt (tamanho descomprimido) → zlib decompress
    ├── Lê VarInt (ID do pacote)
    ├── Cria ReadBuffer com dados
    └── Dispara evento OnPacketAvailable(readBuffer)
            └── MinecraftClient.HandlePacket(pkt)
                    ├── [connStatus == 0] → Handshake
                    │       ├── ID 0 → Disconnect (kick login)
                    │       ├── ID 1 → EncryptionRequest → iniciar AES
                    │       ├── ID 2 → LoginSuccess → connStatus = 1
                    │       └── ID 3 → SetCompression → threshold AES
                    └── [connStatus == 1] → Handler.HandlePacket(pkt)
                            └── Handler_v18.HandlePacket() por exemplo
                                    ├── 0x00 → KeepAlive → envia resposta
                                    ├── 0x01 → JoinGame → HandlePacketJoinGame()
                                    ├── 0x02 → Chat → HandlePacketChat()
                                    ├── 0x08 → PlayerPosAndLook → SetPosition + PacketTeleportConfirm
                                    ├── 0x21 → ChunkData → World.SetChunk()
                                    ├── 0x30 → WindowItems → Inventory.SetSlots()
                                    └── ... (dezenas de outros pacotes)
```

### Saída

Estado do cliente (`World`, `Entity`, `Inventory`) atualizado.

---

## Fluxo 6 — Chat e Autenticação AuthMe

### Entrada

Servidor envia pacote de chat.

### Processamento

```
HandlePacketChat(chat, pos)
    ├── [Plugins] → value.onReceiveChat(chat, pos, this)
    ├── [Detecta "Apenas VIPs..."] → PlayerVIP = false
    ├── [Detecta mensagem de login AuthMe] → LoggedIn = true; SendMessage("/vip")
    ├── [pos != 2] → PrintToChat(chat)
    ├── [Detecta prompt de /register] → envia CmdRegister (máx 2x)
    └── [Detecta prompt de /login] → envia CmdLogin (máx 2x)
```

### Saída

Mensagem exibida no chat da UI. Autenticação AuthMe executada automaticamente.

---

## Fluxo 7 — Execução de Comando Interno

### Entrada

Usuário digita `$comando args` no campo de chat da interface ou o comando é enviado pelo servidor.

### Processamento

```
MinecraftClient.SendMessage("$goto 100 64 200")
    └── msg[0] == '$'
            └── CmdManager.RunCommand(msg)
                    ├── Parse: command = "goto", args = ["100", "64", "200"]
                    ├── [Se plugin command] → pluginManager.DoCommand()
                    └── [Se comando interno] → Localiza ICommand por nome
                            └── command.Start(client, args)
                                    └── [A cada tick] → CmdManager.Tick() → command.Tick()
```

### Saída

Macro/comando em execução. `MinecraftClient.currentMacro` é definido.

---

## Fluxo 8 — Macro de Pesca (CommandPesca / CommandPescaV2)

### Entrada

Usuário executa `$pesca` ou equivalente.

### Processamento (Máquina de Estados)

```
Estado: IDLE
    └── Aguarda início

Estado: LANCANDO
    ├── Seleciona vara de pescar no hotbar
    ├── Envia PacketBlockPlace (usar item = lançar vara)
    └── → Estado: AGUARDANDO_PEIXE

Estado: AGUARDANDO_PEIXE
    ├── Monitora posição da entidade bobber no mundo
    ├── [Bobber parou de se mover / subiu] → → Estado: PUXANDO
    └── [Timeout] → → Estado: LANCANDO (relança)

Estado: PUXANDO
    ├── Envia PacketBlockPlace (usar item = puxar vara)
    ├── Aguarda item cair no inventário
    └── → Estado: VERIFICANDO_INVENTARIO

Estado: VERIFICANDO_INVENTARIO
    ├── [Inventário cheio] → → Estado: INDO_BAU
    └── [Espaço livre] → → Estado: LANCANDO

Estado: INDO_BAU
    ├── PathFinder encontra rota até o baú configurado
    ├── MacroUtils.abrirBau()
    ├── MacroUtils.moverItemDoInventarioParaOBau()
    ├── MacroUtils.fecharBau()
    └── → Estado: LANCANDO
```

### Saída

Pesca automática contínua com depósito periódico de itens.

---

## Fluxo 9 — Macro de Mob (CommandMob / CommandMobPlus)

### Entrada

Usuário executa `$mob` ou equivalente.

### Processamento (Máquina de Estados)

```
Estado: BUSCANDO_MOB
    ├── Itera sobre Client.Entities
    ├── Filtra por tipo de mob configurado e distância
    ├── [Mob encontrado] → → Estado: INDO_ATE_MOB
    └── [Nenhum] → aguarda

Estado: INDO_ATE_MOB
    ├── RequestPathTo(mob.X, mob.Y, mob.Z)
    ├── Aguarda CurrentPath executar
    └── → Estado: ATACANDO

Estado: ATACANDO
    ├── Entity.LookTo(mob.X, mob.Y, mob.Z)
    ├── Envia PacketUseEntity(mob.ID, ATTACK)
    ├── [Mob morreu] → → Estado: COLETANDO_DROPS
    └── [Mob fugiu] → → Estado: BUSCANDO_MOB

Estado: COLETANDO_DROPS
    ├── Aguarda items no inventário
    └── → Estado: VERIFICANDO_INVENTARIO

Estado: VERIFICANDO_INVENTARIO
    ├── [Inventário cheio] → → Estado: INDO_BAU (depósito)
    ├── [Ferramenta quebrada] → → Estado: REPARANDO
    └── [OK] → → Estado: BUSCANDO_MOB
```

### Saída

Grinding automático de mobs com gerenciamento de inventário e equipamento.

---

## Fluxo 10 — Pathfinding

### Entrada

`MinecraftClient.RequestPathTo(x, y, z)` chamado.

### Processamento

```
Task.Factory.StartNew(FindPath)
    └── PathGuide.Create(Player, x, y, z)
            └── PathFinder.FindPath(world, start, end)
                    ├── Open list / Closed list (A*)
                    ├── Calcula custos G (custo real) e H (heurística Manhattan)
                    ├── Considera: blocos sólidos, pulos, quedas, água, portas
                    └── Retorna Path (lista de PathPoint)
```

```
A cada Tick() → CurrentPath.Tick()
    ├── Calcula distância ao próximo ponto
    ├── [Próximo ao ponto] → avança para o próximo
    ├── Enfileira movimentos em Entity.MoveQueue
    │       └── Movement.Forward + Movement.Jump (se necessário)
    └── [Chegou ao destino] → CurrentPath = null
```

### Saída

Bot navega autonomamente até o destino.

---

## Fluxo 11 — Sistema de Plugins

### Entrada

Arquivo `.dll` ou `.abp` presente na pasta `Plugins/`.

### Processamento

```
PluginManager.Init()
    ├── [*.dll] → Assembly.Load(bytes) → Reflection → Instancia IPlugin
    ├── [*.abp] → AesEncryption.DecryptFileToByteArray() → Assembly.Load() → IPlugin
    └── FileSystemWatcher monitorando Plugins/ (*.dll) → hot reload
```

```
A cada evento do MinecraftClient:
    ├── onClientConnect(client) — ao iniciar conexão
    ├── onReceiveChat(chat, pos, client) — ao receber chat
    ├── onSendChat(msg, client) — ao enviar mensagem
    └── Tick() — a cada tick do timer principal
```

### Saída

Plugins executados em todos os eventos relevantes do bot.

---

## Fluxo 12 — Reconexão Automática

### Entrada

Bot recebe kick ou perde conexão.

### Processamento

```
HandlePacketDisconnect(reason)
    ├── [Razão = "muitas contas com IP"] → Proxy = ProxyList.NextProxy()
    ├── PrintToChat("Kick: " + reason)
    ├── beingTicked = false
    └── kickTicks = 0

Tick() a cada tick:
    ├── [AutoReconnect && kickTicks != -1 && kickTicks++ > 40 && type_0]
    │       ├── kickTicks = -1
    │       └── StartClient() ← reconecta após ~2 segundos
    └── [type_1] → reconexão controlada externamente
```

### Saída

Bot reconectado automaticamente após 2 segundos (40 ticks × 50ms).

---

## Fluxo 13 — Verificador de Proxies

### Entrada

Usuário carrega lista de proxies e clica em "Verificar".

### Processamento

```
ProxyCheckQueue.Start()
    ├── Cria N threads paralelas (configurável)
    ├── Cada thread retira proxy da fila
    ├── Proxy.CreateSocks4/5/Http() → tentativa de conexão ao servidor de teste
    └── ProxyInfo.Latency preenchido e resultado salvo
```

### Saída

Lista de proxies funcionais com latência medida.

---

## Integração entre Componentes

```
Program (Global State)
    ├── FrmMain (UI Thread)
    │       └── Timer 50ms → MinecraftClient.Tick() para cada instância
    │
    ├── PluginManager (Global)
    │       └── plugins: Dictionary<Assembly, IPlugin>
    │
    └── Config: CompoundTag (conf.dat)

MinecraftClient (por instância de bot)
    ├── PacketStream ← rede TCP
    ├── PacketQueue → rede TCP
    ├── ProtocolHandler → interpreta pacotes recebidos
    ├── Entity → física do jogador
    ├── World → estado do mapa
    ├── Inventory → inventário
    ├── CommandManagerNew → executa macros
    └── PathGuide → navegação
```
