# 07 — Diagrama Arquitetural

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-07                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Documento Técnico — Diagramas                    |
| **Escopo**         | Arquitetura do sistema legado C#                 |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-01, LEG-02, LEG-04                 |

---

## Diagrama 1 — Arquitetura Geral do Sistema

```mermaid
flowchart TD

    Exe["AdvancedBot.exe\n.NET Framework 4.6.2 / x86"]

    subgraph UI["Camada de Interface (Windows Forms)"]
        Main["Main.cs\nFormulário Principal"]
        Start["Start.cs\nConfiguração dos Bots"]
        MacroEd["MacroEditor.cs\nEditor de Scripts"]
        ViewForm["ViewForm.cs\nRenderizador 3D"]
        Forms["Outros Formulários\n(Spammer, AccountChecker, BanCheck...)"]
    end

    subgraph Orq["Orquestração Global"]
        Program["Program.cs\nPonto de Entrada / Config / Log"]
        PluginMgr["PluginManager\nCarregamento de Plugins"]
    end

    subgraph Core["Core do Cliente"]
        MC["MinecraftClient\nEstado do Bot / Tick / Reconexão"]
        CmdMgr["CommandManagerNew\nExecução de Macros"]
        PathGuide["PathGuide\nNavegação Automática"]
    end

    subgraph Rede["Camada de Rede"]
        PkgStream["PacketStream\nI/O Assíncrono + Encriptação"]
        PkgQueue["PacketQueue\nFila de Envio"]
        ProtoH["ProtocolHandler\nv1.5.2 / v1.7 / v1.8 / v1.9"]
        Proxy["Proxy\nHTTP / SOCKS4 / SOCKS5"]
    end

    subgraph Dominio["Domínio do Jogo"]
        Entity["Entity\nFísica do Jogador"]
        World["World\nChunks e Blocos"]
        Inventory["Inventory\nItens e Slots"]
        EntityMgr["EntityManager\nMobs e Jogadores"]
    end

    subgraph Macros["Macros e Comandos"]
        Pesca["CommandPesca/V2\nMáquina de Pesca"]
        Mob["CommandMob/Plus\nMáquina de Mob"]
        MacroUtils["MacroUtils\nUtilitários Async"]
        Outros["Outros Comandos\n(Goto, Follow, KillAura...)"]
    end

    subgraph Infra["Infraestrutura"]
        NBT["NBT Parser\nconf.dat / Itens / Chunks"]
        Crypto["CryptoUtils\nAES-CFB8 / RSA / SHA-1"]
        Session["SessionUtils\nAutenticação Mojang"]
        Script["ScriptEngine\nProprietário + Jint JS"]
    end

    Exe --> Program
    Program --> Main
    Program --> PluginMgr
    Main --> MC
    Main --> Start
    Main --> MacroEd
    Main --> ViewForm

    MC --> PkgStream
    MC --> PkgQueue
    MC --> ProtoH
    MC --> Entity
    MC --> World
    MC --> Inventory
    MC --> CmdMgr
    MC --> PathGuide
    MC --> Session
    MC --> Proxy

    PkgStream --> Crypto
    ProtoH --> World
    ProtoH --> Entity
    ProtoH --> Inventory
    ProtoH --> EntityMgr

    CmdMgr --> Pesca
    CmdMgr --> Mob
    CmdMgr --> Outros
    Pesca --> MacroUtils
    Mob --> MacroUtils

    MacroUtils --> World
    MacroUtils --> Inventory
    MacroUtils --> Entity

    Program --> NBT
    NBT --> Infra

    ViewForm --> World
    ViewForm --> Entity
```

---

## Diagrama 2 — Ciclo de Vida de Conexão

```mermaid
stateDiagram-v2

    [*] --> Desconectado

    Desconectado --> Autenticando : StartClient() + conta Mojang
    Desconectado --> ConectandoTCP : StartClient() + cracked

    Autenticando --> ConectandoTCP : Login Mojang OK
    Autenticando --> Desconectado : Erro de autenticação

    ConectandoTCP --> HandshakeLogin : TCP conectado
    ConectandoTCP --> Desconectado : Falha de conexão / proxy

    HandshakeLogin --> EncriptandoRSA : EncryptionRequest recebido
    HandshakeLogin --> JogoAtivo : LoginSuccess (offline-mode)
    HandshakeLogin --> Desconectado : Disconnect (kick login)

    EncriptandoRSA --> JogoAtivo : AES ativado + LoginSuccess
    EncriptandoRSA --> Desconectado : Erro de sessão Mojang

    JogoAtivo --> JogoAtivo : Tick 50ms (posição, comandos, física)
    JogoAtivo --> Reconectando : Kick / Perda de conexão

    Reconectando --> ConectandoTCP : AutoReconnect + 2s delay
    Reconectando --> Desconectado : AutoReconnect desabilitado

    Desconectado --> [*]
```

---

## Diagrama 3 — Fluxo de Pacotes de Entrada

```mermaid
flowchart TD

    Socket["Socket TCP\n(NetworkStream)"]
    PkgStream["PacketStream\nLeitura Assíncrona"]
    Decrypt["AES-CFB8\nDesencriptação"]
    Decomp["zlib Decompress\n(se threshold ativo)"]
    ReadBuf["ReadBuffer\n(VarInt ID + dados)"]

    LoginHandler["MinecraftClient.HandlePacket()\nFase LOGIN (connStatus=0)"]
    PlayHandler["ProtocolHandler.HandlePacket()\nFase PLAY (connStatus=1)"]

    subgraph LoginPkts["Pacotes de Login"]
        P0L["ID 0x00 — Disconnect"]
        P1L["ID 0x01 — EncryptionRequest"]
        P2L["ID 0x02 — LoginSuccess"]
        P3L["ID 0x03 — SetCompression"]
    end

    subgraph PlayPkts["Pacotes de Jogo (exemplos)"]
        KA["0x00 KeepAlive"]
        JG["0x01 JoinGame"]
        CH["0x02 Chat"]
        PL["0x08 PlayerPosAndLook"]
        CD["0x21 ChunkData"]
        WI["0x30 WindowItems"]
        SH["0x38 PlayerListItem"]
        KK["0x40 Disconnect"]
    end

    Socket --> PkgStream
    PkgStream --> Decrypt
    Decrypt --> Decomp
    Decomp --> ReadBuf
    ReadBuf --> LoginHandler
    ReadBuf --> PlayHandler

    LoginHandler --> P0L
    LoginHandler --> P1L
    LoginHandler --> P2L
    LoginHandler --> P3L

    PlayHandler --> KA
    PlayHandler --> JG
    PlayHandler --> CH
    PlayHandler --> PL
    PlayHandler --> CD
    PlayHandler --> WI
    PlayHandler --> SH
    PlayHandler --> KK
```

---

## Diagrama 4 — Módulos e Dependências

```mermaid
flowchart LR

    subgraph Legado["Sistema Legado C# — Dependências entre Módulos"]
        UI["AdvancedBot\n(Windows Forms / UI)"]
        Client["AdvancedBot.Client\n(Core)"]
        Handler["Client.Handler\n(Protocolo)"]
        Packets["Client.Packets\n(Serialização)"]
        Map["Client.Map\n(Mundo)"]
        NBT["Client.NBT\n(Dados)"]
        Crypto["Client.Crypto\n(Encriptação)"]
        Entity["Client.Entitybase\n(Entidades)"]
        Path["Client.PathFinding\n(Navegação)"]
        Cmds["Client.Commands\n(Macros)"]
        Bypass["Client.Bypassing\n(Servidor-Específico)"]
        Plugins["AdvancedBot.Plugins"]
        Script["AdvancedBot.Script"]
        Protection["AdvancedBot.Protection"]
        ProxyChk["AdvancedBot.ProxyChecker"]
        CryptoExt["AdvancedBot.Crypto\n(Plugins)"]
        Viewer["AdvancedBot.Viewer\n(3D)"]
        ViewerGui["Viewer.Gui\n(OpenGL GUI)"]
    end

    UI --> Client
    UI --> Plugins
    UI --> Viewer
    UI --> Script

    Client --> Handler
    Client --> Packets
    Client --> Map
    Client --> NBT
    Client --> Crypto
    Client --> Entity
    Client --> Path

    Handler --> Map
    Handler --> NBT

    Cmds --> Client
    Cmds --> Map
    Cmds --> Packets

    Plugins --> Client
    Plugins --> CryptoExt

    Viewer --> Map
    Viewer --> Client
    Viewer --> ViewerGui

    Bypass --> Client
    Script --> Client

    Protection --> UI
    ProxyChk --> Client
```

---

## Diagrama 5 — Máquina de Estados da Macro de Pesca

```mermaid
stateDiagram-v2

    [*] --> Iniciando

    Iniciando --> SelecionandoVara : start()

    SelecionandoVara --> Lancando : Vara selecionada no hotbar
    SelecionandoVara --> Encerrado : Vara não encontrada

    Lancando --> AguardandoPeixe : PacketBlockPlace enviado (lançar)

    AguardandoPeixe --> Puxando : Bobber subiu / parou
    AguardandoPeixe --> Lancando : Timeout de lançamento

    Puxando --> VerificandoInventario : PacketBlockPlace enviado (puxar)

    VerificandoInventario --> Lancando : Espaço disponível
    VerificandoInventario --> IndoAoBau : Inventário cheio

    IndoAoBau --> AbrindoBau : PathFinder chegou ao baú
    AbrindoBau --> DepositandoItens : Baú aberto
    DepositandoItens --> FeichandoBau : Itens depositados
    FeichandoBau --> Lancando : Retornou ao ponto de pesca

    Lancando --> Encerrado : stop() chamado
    Encerrado --> [*]
```

---

## Diagrama 6 — Threads do Sistema

```mermaid
flowchart TD

    subgraph MainThread["Thread Principal (STA / UI Thread)"]
        AppRun["Application.Run()"]
        Timer["Timer 50ms"]
        Tick["MinecraftClient.Tick()"]
    end

    subgraph NetworkThreads["Threads de Rede (ThreadPool)"]
        Async["ConnectAndHandshakeAsync()"]
        Auth["AuthenticateMojang()"]
        Ping["Ping()"]
        ReadCB["PacketStream — Callbacks de Leitura"]
    end

    subgraph DedicatedThreads["Threads Dedicadas"]
        MC152["Thread MC 1.5.2\nConnect_v15() loop"]
        PathFind["Task FindPath\n(ThreadPool)"]
        ChunkBld["AsyncChunkBuilder\n(ThreadPool)"]
    end

    subgraph PluginThread["Thread de Filesystem"]
        FSW["FileSystemWatcher\nHot Reload de Plugins"]
    end

    Timer --> Tick
    Tick --> Async
    Async --> Auth
    Async --> Ping
    Async --> ReadCB

    Tick --> MC152

    Tick --> PathFind
    Tick --> ChunkBld

    FSW --> PluginMgr["PluginManager.LoadPlugin()"]
```

---

## Diagrama 7 — Sequência de Autenticação Mojang

```mermaid
sequenceDiagram

    Bot->>MojangAuth: POST /authenticate (email, password, clientToken)
    MojangAuth-->>Bot: accessToken, selectedProfile.id, selectedProfile.name

    Bot->>Servidor: TCP Connect
    Bot->>Servidor: PacketHandshake (version, host, port, nextState=2)
    Bot->>Servidor: PacketLoginStart (username)

    Servidor-->>Bot: EncryptionRequest (serverID, publicKey, verifyToken)

    Bot->>Bot: GenerateAESKey (sharedSecret 16 bytes)
    Bot->>Bot: ComputeServerHash (SHA1 de serverID + sharedSecret + publicKey)

    Bot->>MojangSession: POST /session/minecraft/join (accessToken, uuid, serverHash)
    MojangSession-->>Bot: 204 No Content (OK)

    Bot->>Bot: RSA.Encrypt(sharedSecret, publicKey)
    Bot->>Bot: RSA.Encrypt(verifyToken, publicKey)
    Bot->>Servidor: PacketEncryptionResponse (encryptedSecret, encryptedToken)

    Bot->>Bot: AES-CFB8 ativado (sharedSecret como IV e Key)

    Servidor-->>Bot: LoginSuccess (UUID, username)
    Servidor-->>Bot: JoinGame (playerID, dimension, gamemode)
```

---

## Diagrama 8 — Relação entre Entidades do Domínio

```mermaid
classDiagram

    class MinecraftClient {
        +string IP
        +ushort Port
        +string Username
        +Entity Player
        +World TheWorld
        +Inventory Inventory
        +Inventory OpenWindow
        +PacketQueue SendQueue
        +StartClient()
        +Tick()
        +SendMessage()
    }

    class Entity {
        +double PosX, PosY, PosZ
        +float Yaw, Pitch
        +bool OnGround
        +AABB AABB
        +Queue MoveQueue
        +Tick()
        +Move()
        +LookTo()
        +RayCastBlocks()
    }

    class World {
        +Dictionary Chunks
        +GetBlock()
        +SetChunk()
        +RayCast()
        +GetCollisionBoxes()
    }

    class Inventory {
        +ItemStack[] Slots
        +short NumSlots
        +byte WindowID
        +Click()
    }

    class ItemStack {
        +short ID
        +byte Metadata
        +byte Count
        +CompoundTag NBT
    }

    class Chunk {
        +int X, Z
        +ChunkSection[] Sections
        +GetBlock()
        +SetBlock()
    }

    MinecraftClient "1" --> "1" Entity : Player
    MinecraftClient "1" --> "1" World : TheWorld
    MinecraftClient "1" --> "1" Inventory : Inventory
    MinecraftClient "1" --> "0..1" Inventory : OpenWindow
    Entity --> World : usa
    World "1" --> "*" Chunk : chunks
    Inventory "1" --> "*" ItemStack : Slots
    Chunk --> ChunkSection : Sections
```
