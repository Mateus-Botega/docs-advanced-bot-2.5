# 01 — Visão Geral do Sistema

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-01                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Documento Técnico — Visão Geral                  |
| **Escopo**         | Sistema legado AdvancedBot C#                    |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-02, LEG-03, LEG-04, LEG-07         |

---

## Finalidade da Aplicação

O **AdvancedBot** é um cliente robô (*bot*) para servidores Minecraft, desenvolvido em C# com Windows Forms.

Sua função principal é simular um jogador humano conectado a um servidor Minecraft, executando automaticamente tarefas repetitivas como:

- Pesca automatizada.
- Combate com mobs.
- Movimentação e navegação no mapa.
- Gerenciamento de inventário.
- Interação com blocos e cofres.
- Automação de vendas e compras.
- Anti-AFK (impede expulsão por inatividade).
- Suporte a servidores com anti-cheat.

---

## Contexto

O AdvancedBot é projetado especificamente para servidores Minecraft do tipo **cracked** (sem autenticação Mojang obrigatória), embora suporte autenticação via conta Mojang real.

O sistema foi desenvolvido incrementalmente por um único desenvolvedor, sem documentação técnica formal.

A versão analisada é a **v2.4.5.2-f**, compilada como executável Windows (`.exe`) de 32 bits.

---

## Versões de Protocolo Suportadas

O bot suporta múltiplas versões do protocolo Minecraft:

| Versão | Handler |
|--------|---------|
| 1.5.2 | `Handler_v152` |
| 1.7 / 1.7.10 | `Handler_v17` |
| 1.8 | `Handler_v18` |
| 1.9 | `Handler_v19` |

---

## Principais Funcionalidades

### Conexão e Autenticação

- Suporte a modo online (conta Mojang / Yggdrasil) e offline (cracked).
- Autenticação RSA e encriptação AES do protocolo Minecraft.
- Reconexão automática configurável.
- Suporte completo a proxies HTTP, SOCKS4 e SOCKS5.

### Gerenciamento Multi-Bot

- Múltiplas instâncias de bots rodando simultaneamente.
- Cada instância possui sua própria `Thread` de conexão.
- Painel de controle centralizado via interface Windows Forms.

### Sistema de Comandos

- Comandos internos prefixados com `$` (exemplo: `$goto`, `$miner`, `$follow`).
- Comandos enviados pelo chat do servidor.
- Sistema extensível via `ICommand`.

### Macros de Automação

- `CommandPesca` / `CommandPescaV2`: Macro de pesca com máquina de estados complexa.
- `CommandMob`: Macro de combate com mobs.
- `CommandMobPlus`: Versão estendida do combate.
- `CommandMobTeleport`: Teleporte até mobs.
- `CommandMiner` / `AutoMiner`: Mineração automática.
- `CommandFollow`: Seguir jogadores.
- `CommandPortal`: Navegar por portais.
- `CommandKillAura`: Kill Aura (PvP automatizado).

### Física e Movimentação

- Motor de física completo simulando a física do Minecraft:
  - Gravidade.
  - Colisão com blocos.
  - Movimento em água e lava.
  - Escada e trepadeira.
  - Passo automático (step).
- Pathfinding com algoritmo A* (`PathFinder`).
- Fila de movimentos (`MoveQueue`).

### Mapa e Mundo

- Armazenamento de chunks recebidos do servidor.
- Acesso a blocos por coordenadas.
- RayCast para detectar blocos e entidades na linha de visão.
- Suporte a NBT para dados de tiles (placas, baús).

### Inventário

- Gerenciamento completo do inventário do jogador.
- Abertura e interação com janelas externas (baús, lojas).
- Click em slots com suporte a shift-click.
- Equipamento de itens no hotbar.

### Entidades

- Rastreamento de entidades no mundo (mobs, jogadores).
- Cálculo de distância e visibilidade.
- Suporte a `KillAura` por rastreamento de entidades.

### Sistema de Plugins

- Carregamento dinâmico de plugins (`.dll` e `.abp` encriptados).
- Interface `IPlugin` com callbacks:
  - `onClientConnect`
  - `onReceiveChat`
  - `onSendChat`
  - `Tick`
  - `Unload`
- Hot reload de plugins via `FileSystemWatcher`.

### Renderizador 3D

- Visualizador 3D do mundo usando OpenGL via WGL/P/Invoke.
- Renderização de chunks, entidades e efeitos.
- Suporte a VBO para otimização de performance.
- Renderização de texto de placas.
- GUI personalizada dentro do contexto OpenGL.

### Script Engine

- Motor de scripts proprietário com dois motores:
  - Motor de linguagem própria (`ScriptParser`, `ScriptContext`).
  - Integração com motor JavaScript via biblioteca `Jint`.

### Utilitários Auxiliares

- `ProxyChecker`: Verificador de proxies em massa.
- `AccountChecker`: Verificador de contas Minecraft.
- `BanCheck`: Verificação de ban em servidores.
- `Spammer`: Envio automático de mensagens.
- `NickGenerator`: Geração de nicks aleatórios.
- `SrvResolver`: Resolução de endereços SRV do Minecraft.

---

## Arquitetura Atual

O sistema é um **monólito Windows Forms** organizado logicamente em módulos (pastas) dentro de um único projeto `.csproj`.

Não há separação física de assemblies (DLLs) para os módulos internos. Toda a aplicação compila em um único executável.

### Camadas Identificadas

| Camada | Responsabilidade |
|--------|-----------------|
| Interface Gráfica | Windows Forms (`Main.cs`, `Start.cs`, formulários) |
| Orquestração | `Program.cs` — ponto de entrada e gerenciamento global |
| Protocolo | `MinecraftClient.cs`, `PacketStream.cs`, Handlers |
| Domínio | `Entity.cs`, `World.cs`, `Inventory.cs`, `Blocks.cs` |
| Macros/Comandos | `ICommand`, `CommandPesca`, `CommandMob`, etc. |
| Infraestrutura | `Proxy.cs`, `SessionUtils.cs`, `LoginCache.cs` |
| Renderização | `ViewForm.cs`, `ChunkRenderer.cs`, `GL.cs` |
| Plugins | `PluginManager.cs`, `IPlugin.cs` |
| Script | `ScriptParser.cs`, `JsScriptContext.cs` |

---

## Pontos de Entrada

| Ponto de Entrada | Localização | Descrição |
|-----------------|-------------|-----------|
| `Main()` | `Program.cs` | Inicialização da aplicação |
| `Application.Run(FrmMain)` | `Program.cs` | Loop principal do Windows Forms |
| `MinecraftClient.StartClient()` | `MinecraftClient.cs` | Início da conexão com o servidor |
| `ConnectAndHandshakeAsync()` | `MinecraftClient.cs` | Handshake assíncrono (v1.7+) |
| `Connect_v15()` | `MinecraftClient.cs` | Conexão síncrona (v1.5.2) |

---

## Configuração

A configuração é persistida em formato **NBT binário** no arquivo `conf.dat`.

O acesso é feito via `Program.Config` (instância global de `CompoundTag`).

Configurações persistidas:

- `Knockback` — habilitação de knockback.
- `ViewerTextures` — uso de texturas no renderizador.
- `ViewerRenderSignText` — renderização de texto de placas.
- `AutoReconnect` — reconexão automática.
- `ViewerUseVBO` — uso de VBO na renderização.
- `ViewerFpsLimit` — limite de FPS do viewer.
- `ViewerZSorting` — ordenação Z para transparência.
- `ChatMaxLines` — número máximo de linhas de chat.

---

## Ciclo de Vida da Aplicação

1. `Program.Main()` configura o `ThreadPool` (máximo 16 threads).
2. `ProcessorAffinity` é configurada para usar todos os núcleos menos um.
3. `Main` (formulário principal) é criado e exibido.
4. `PluginManager.Init()` carrega todos os plugins da pasta `Plugins/`.
5. `LoadConf()` lê o arquivo `conf.dat`.
6. `FrmMain.CheckKey()` valida a chave de licença.
7. `Application.Run(FrmMain)` inicia o loop de mensagens do Windows.
8. O timer do formulário principal chama `MinecraftClient.Tick()` periodicamente.

---

## Observações Técnicas

- A versão utiliza `C# 7.3` com `unsafe blocks` habilitados.
- O target é `.NET Framework 4.6.2` (net462).
- A plataforma é exclusivamente `x86` (32 bits).
- Blocos `unsafe` são utilizados em contextos de performance crítica.
- O thread principal é STA (Single Thread Apartment), exigência do Windows Forms.
- `Thread.Abort()` é utilizado para encerramento de threads (prática obsoleta no .NET moderno).
