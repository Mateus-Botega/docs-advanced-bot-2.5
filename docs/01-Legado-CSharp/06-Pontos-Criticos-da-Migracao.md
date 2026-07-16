# 06 — Pontos Críticos da Migração

---

| Campo              | Valor                                            |
|--------------------|--------------------------------------------------|
| **Código**         | LEG-06                                           |
| **Versão**         | 1.0.0                                            |
| **Status**         | Ativo                                            |
| **Responsável**    | Equipe de Migração AdvancedBot                   |
| **Última Atualização** | 2026-07-14                                   |
| **Tipo**           | Documento Técnico — Riscos                       |
| **Escopo**         | Todos os pontos de risco e complexidade          |
| **Documento Pai**  | docs/01-Legado-CSharp/00-README.md               |
| **Documentos Relacionados** | LEG-01, LEG-03, LEG-05, LEG-08         |

---

## Critério de Classificação de Risco

| Nível | Definição |
|-------|-----------|
| **BLOQUEANTE** | A migração não pode avançar sem resolver este ponto. |
| **CRÍTICO** | Implementação incorreta causa falha funcional completa. |
| **ALTO** | Implementação incorreta causa instabilidade ou comportamento incorreto. |
| **MÉDIO** | Implementação incorreta causa degradação de funcionalidade específica. |
| **BAIXO** | Impacto limitado, controlável ou descartável. |

---

## PC-01 — Encriptação AES-CFB8 do Protocolo Minecraft

| Campo | Valor |
|-------|-------|
| **Risco** | BLOQUEANTE |
| **Arquivo C#** | `AesStream.cs`, `CryptoUtils.cs` |

**Descrição:**

O protocolo Minecraft utiliza AES no modo CFB8 (Cipher Feedback de 8 bits) com IV igual à chave compartilhada. Este modo específico não é o padrão AES-CBC ou AES-ECB comum.

No C#, a implementação usa `RijndaelManaged` ou `AesCryptoServiceProvider` com `Mode = CipherMode.CFB` e `FeedbackSize = 8`.

**Risco na Migração:**

O Java suporta `AES/CFB8/NoPadding` via `javax.crypto.Cipher`, mas o provider padrão (`SunJCE`) pode não suportar CFB8 em todas as plataformas. Pode ser necessário usar o provider `BouncyCastle`.

**Ação Necessária:**

- Testar `Cipher.getInstance("AES/CFB8/NoPadding")` com o provider padrão do Java.
- Se falhar, adicionar `org.bouncycastle:bcprov-jdk18on` como dependência.
- Validar com teste de handshake real antes de prosseguir.

---

## PC-02 — Server Hash SHA-1 Não-Standard do Minecraft

| Campo | Valor |
|-------|-------|
| **Risco** | BLOQUEANTE |
| **Arquivo C#** | `CryptoUtils.GetServerHash()` |

**Descrição:**

O Minecraft calcula o "server hash" para autenticação usando SHA-1, mas o resultado é interpretado como um número inteiro com sinal (signed big-endian), o que pode produzir hashes negativos como `-2a96492db...`.

A implementação C# usa:

```csharp
BigInteger serverHash = new BigInteger(hashBytes);
if (serverHash < 0)
    return "-" + (-serverHash).ToString("x");
return serverHash.ToString("x");
```

**Risco na Migração:**

O `java.math.BigInteger` tem comportamento diferente do `System.Numerics.BigInteger` para bytes com bit alto setado. O algoritmo precisa ser implementado exatamente da mesma forma para autenticação funcionar.

**Ação Necessária:**

- Implementar o server hash com `new BigInteger(1, hashBytes)` verificando se o primeiro byte tem bit alto.
- Validar com casos de teste conhecidos (hash positivo e negativo).

---

## PC-03 — Motor de Física do Jogador

| Campo | Valor |
|-------|-------|
| **Risco** | CRÍTICO |
| **Arquivo C#** | `Entity.cs`, `AABB.cs`, `World.cs` |

**Descrição:**

A física do jogador (`Entity.Tick()`) reproduz fielmente a física do Minecraft Java Edition. Inclui:

- Gravidade (`MotionY -= 0.08` a cada tick).
- Resistência do ar (`MotionX *= 0.91`, `MotionY *= 0.98`).
- Atrito do solo (variável por tipo de bloco: `0.6 * 0.91` para solo normal, `0.98 * 0.91` para gelo).
- Step automático (0.5 blocos).
- Física de água (fluxo, flotação).
- Física de lava.
- Física de escada/trepadeira.
- Colisão AABB tridimensional com múltiplos blocos.

**Risco na Migração:**

Qualquer desvio nos valores numéricos de física resultará em:
- Bot preso em blocos (falso-positivo de colisão).
- Bot flutuando (falso-negativo de colisão).
- Pathfinding incorreto (A* calcula baseado na física real).
- Detecção incorreta de `OnGround`.
- Kick por anti-cheat por posições inválidas.

**Ação Necessária:**

- Migrar `Entity.cs` e `AABB.cs` de forma bit-a-bit idêntica ao C#.
- Usar `double` em todos os cálculos (não `float`).
- Validar com coordenadas reais de servidores conhecidos.

---

## PC-04 — Protocolo Assíncrono com Callbacks

| Campo | Valor |
|-------|-------|
| **Risco** | CRÍTICO |
| **Arquivo C#** | `PacketStream.cs`, `MinecraftClient.cs` |

**Descrição:**

O sistema de leitura de pacotes usa o modelo de I/O assíncrono do .NET (`BeginConnect`, `BeginReceive`, callbacks, `Stream.OnPacketAvailable`). Múltiplos callbacks podem ser invocados de diferentes threads do `ThreadPool`.

**Problemas Identificados:**

- `MinecraftClient.HandlePacket()` é chamado de thread do pool, mas modifica `ChatMessages` (protegido por lock) e outras estruturas.
- `SendQueue.Flush()` é chamado tanto do timer principal (UI thread) quanto de callbacks de rede (pool threads) — potencial race condition.
- `beingTicked` é um `bool` não-volatile modificado de múltiplas threads sem sincronização.
- `kickTicks` é um `int` incrementado no tick principal mas resetado em callbacks de rede — race condition.

**Risco na Migração:**

O Java deve implementar o modelo de threading correto. O uso de `CompletableFuture` com `ExecutorService` dedicado para I/O e outro para lógica de negócio é o caminho correto.

**Ação Necessária:**

- Definir modelo de threading do Java explicitamente (IOThread + GameThread + TimerThread).
- Usar `AtomicBoolean` para `beingTicked`.
- Usar `AtomicInteger` para `kickTicks`.
- Usar `CopyOnWriteArrayList` ou lock explícito para `ChatMessages`.

---

## PC-05 — Thread.Abort() Obsoleto

| Campo | Valor |
|-------|-------|
| **Risco** | ALTO |
| **Arquivo C#** | `MinecraftClient.cs` |

**Descrição:**

O C# usa `Thread.Abort()` para encerrar a thread de conexão do protocolo v1.5.2. Esta abordagem é obsoleta e não existe no Java.

```csharp
updateThread.Join(1000);
updateThread.Abort(); // lança ThreadAbortException
```

**Risco na Migração:**

O Java não possui `Thread.stop()` ou equivalente seguro. O encerramento deve ser cooperativo usando flags de cancelamento.

**Ação Necessária:**

- Substituir `Thread.Abort()` por flag `volatile boolean running = false` verificada no loop da thread.
- Ou usar `Thread.interrupt()` com tratamento de `InterruptedException`.

---

## PC-06 — Proxy SOCKS com Async State Machine Manual

| Campo | Valor |
|-------|-------|
| **Risco** | ALTO |
| **Arquivo C#** | `Proxy.cs` |

**Descrição:**

A implementação assíncrona do proxy (`BeginConnect`) é uma máquina de estados manual implementada com `IAsyncResult` e callbacks encadeados. Cobre SOCKS4, SOCKS5 e HTTP CONNECT com DNS assíncrono.

**Risco na Migração:**

A implementação Java pode ser simplificada usando `java.net.Socket` com suporte a SOCKS nativo (`java.net.Proxy`) ou via biblioteca.

**Ação Necessária:**

- Avaliar uso do suporte nativo do Java a SOCKS (`InetSocketAddress + java.net.Proxy`).
- Para HTTP CONNECT, implementar handshake manual como no C#.
- Para SOCKS4/5 com DNS remoto, implementar manualmente.
- Validar com proxies reais antes de prosseguir.

---

## PC-07 — Multi-Protocolo (4 Versões Diferentes)

| Campo | Valor |
|-------|-------|
| **Risco** | ALTO |
| **Arquivos C#** | `Handler_v152.cs`, `Handler_v17.cs`, `Handler_v18.cs`, `Handler_v19.cs` |

**Descrição:**

O bot suporta 4 versões de protocolo radicalmente diferentes. O protocolo 1.5.2 é completamente diferente dos demais (formato de pacotes, encriptação, sem compressão, sem VarInt). As versões 1.7/1.8/1.9 são mais similares, mas diferem em IDs de pacotes e estruturas.

**Risco na Migração:**

Implementar todos os 4 protocolos corretamente exige mapeamento detalhado de cada pacote para cada versão.

**Ação Necessária:**

- Definir quais versões serão suportadas na versão Java (decisão arquitetural).
- Considerar foco inicial no protocolo 1.8 (mais usado nos servidores alvo).
- Mapear todos os pacotes por versão antes de implementar.

---

## PC-08 — Sistema de Plugins com Criptografia AES

| Campo | Valor |
|-------|-------|
| **Risco** | MÉDIO |
| **Arquivo C#** | `PluginManager.cs`, `AesEncryption.cs` |

**Descrição:**

Plugins `.abp` são assemblies .NET encriptados com AES usando a chave hardcoded `4a32544e013f37c028cedadb2d5b7c683de3d023`. O Java precisará de um sistema de plugin equivalente.

**Risco na Migração:**

- O Java não pode carregar assemblies .NET diretamente.
- Plugins existentes precisarão ser reescritos para Java.
- A chave de criptografia hardcoded é um risco de segurança.

**Ação Necessária:**

- Definir o formato de plugins Java (JARs com interface `IPlugin` equivalente).
- Decidir sobre criptografia de plugins (usar novo algoritmo ou manter por retrocompatibilidade de plugins).
- Implementar `PluginClassLoader` para isolamento de plugins.

---

## PC-09 — Macros com async/await e Task

| Campo | Valor |
|-------|-------|
| **Risco** | ALTO |
| **Arquivo C#** | `MacroUtils.cs`, `CommandPesca.cs`, `CommandMob.cs` |

**Descrição:**

As macros usam extensivamente `async/await` com `Task.Delay()` para temporização. Isso cria um modelo de programação assíncrona cooperativo que é transparente no C#.

```csharp
public static async Task<bool> abrirBau(MinecraftClient client, Vec3d coordBau) {
    for (int i = 0; i <= 10; i++) {
        client.Player.LookToBlock(x, y, z, false);
        await Task.Delay(1000);
        RightclickBlock(client, coordBau);
        await Task.Delay(1000);
        if (client.OpenWindow != null) return true;
    }
    return false;
}
```

**Risco na Migração:**

O Java não tem `async/await` nativamente. As alternativas são:
1. `CompletableFuture.thenCompose()` — verboso mas correto.
2. Thread dedicada por macro com `Thread.sleep()` — mais simples.
3. Máquinas de estado explícitas acionadas por tick — mais complexo mas mais eficiente.

**Ação Necessária:**

- Decidir o modelo de implementação de macros em Java (decisão arquitetural obrigatória).
- Recomendar: máquinas de estado por tick, eliminando `sleep()` e mantendo thread única.

---

## PC-10 — RayCast em Múltiplos Blocos

| Campo | Valor |
|-------|-------|
| **Risco** | MÉDIO |
| **Arquivo C#** | `World.RayCast()`, `BlockUtils.cs` |

**Descrição:**

O RayCast percorre o mundo voxel usando o algoritmo de DDA (Digital Differential Analyzer) para detectar o primeiro bloco sólido na linha de visão. É usado por:

- `Entity.RayCastBlocks()` — detecção de bloco na mira.
- `Entity.CanSeeEntity()` — verificação de linha de visão entre entidades.
- `MacroUtils` — abertura de baús e placas.

**Ação Necessária:**

- Migrar o algoritmo de RayCast integralmente.
- Validar com coordenadas e ângulos de teste conhecidos.

---

## PC-11 — Configuração em Formato NBT Binário

| Campo | Valor |
|-------|-------|
| **Risco** | MÉDIO |
| **Arquivo C#** | `NbtIO.cs`, `CompoundTag.cs`, `Program.cs` |

**Descrição:**

Toda a configuração é persistida em formato NBT binário (`conf.dat`), o mesmo formato usado internamente no protocolo Minecraft para itens e chunks.

**Risco na Migração:**

- Compatibilidade do arquivo de configuração entre C# e Java.
- O Java pode optar por usar outro formato (JSON, YAML, Properties) para configuração.
- Os dados NBT de itens e chunks precisam ser parseados corretamente independentemente.

**Ação Necessária:**

- Decidir se `conf.dat` será mantido ou migrado para novo formato (JSON/YAML).
- Manter o parser NBT para uso no protocolo (dados de itens e chunks usam NBT).

---

## PC-12 — Interface Gráfica Windows Forms

| Campo | Valor |
|-------|-------|
| **Risco** | ALTO |
| **Arquivos C#** | `Main.cs`, `Start.cs`, e demais formulários |

**Descrição:**

Toda a interface do usuário é Windows Forms, dependência exclusiva do Windows. O Java não tem equivalente direto do Windows Forms.

**Risco na Migração:**

- A interface completa precisa ser redesenhada do zero.
- O timer do Windows Forms é o motor de tick do sistema — precisa de substituto.

**Ação Necessária:**

- Decidir a estratégia da interface Java (headless, JavaFX, Swing, CLI, web).
- Substituir o timer do Windows Forms por `ScheduledExecutorService.scheduleAtFixedRate(50ms)`.

---

## PC-13 — Blocos Unsafe (Código Não Gerenciado)

| Campo | Valor |
|-------|-------|
| **Risco** | MÉDIO |
| **Configuração** | `AllowUnsafeBlocks = true` no csproj |

**Descrição:**

O projeto habilita blocos `unsafe` que permitem manipulação direta de ponteiros em C#. São usados provavelmente em:
- `GL.cs` (bindings OpenGL) — descartado na migração.
- Possivelmente em `AesStream.cs` para otimização de cópia de buffers.

**Ação Necessária:**

- Identificar todos os usos de `unsafe` no código.
- Substituir por equivalentes seguros em Java (`ByteBuffer`, `System.arraycopy()`).

---

## PC-14 — Ausência de Documentação Técnica Original

| Campo | Valor |
|-------|-------|
| **Risco** | ALTO |

**Descrição:**

O projeto C# foi desenvolvido sem documentação técnica. Não existem:
- Comentários XML de documentação.
- Arquivos README técnicos.
- Especificação de protocolo própria.
- Testes automatizados.

**Risco na Migração:**

Comportamentos não documentados podem ser perdidos na migração. Alguns comportamentos "corretos" podem ser acidentalmente alterados sem percepção.

**Ação Necessária:**

- Este documento de legado (LEG-04 Fluxos) deve ser revisado contra o código-fonte.
- Comportamentos críticos devem ser validados por testes em servidores reais durante a migração.

---

## PC-15 — Acoplamento Global via Program.FrmMain

| Campo | Valor |
|-------|-------|
| **Risco** | ALTO |
| **Arquivo C#** | `Program.cs`, `MinecraftClient.cs` |

**Descrição:**

Múltiplas classes acessam `Program.FrmMain` e `Program.pluginManager` diretamente como singleton global. Isso cria acoplamento extremo entre a lógica de negócio e a interface gráfica.

```csharp
// Em MinecraftClient.cs, dentro de um callback de rede:
ConProxy = Program.FrmMain.Proxies.NextProxy();
```

**Risco na Migração:**

- O Java precisa eliminar este acoplamento.
- A lógica de negócio deve ser independente da UI.

**Ação Necessária:**

- Implementar injeção de dependência no Java.
- Usar interfaces para desacoplar MinecraftClient do PluginManager e da UI.
- O Spring Boot com DI pode resolver naturalmente este problema.

---

## Resumo dos Pontos Críticos

| Código | Descrição | Risco | Decisão Necessária |
|--------|-----------|-------|--------------------|
| PC-01 | AES-CFB8 | BLOQUEANTE | Validar provider Java |
| PC-02 | Server Hash SHA-1 | BLOQUEANTE | Implementar algoritmo exato |
| PC-03 | Motor de Física | CRÍTICO | Migração bit-a-bit |
| PC-04 | Threading Assíncrono | CRÍTICO | Definir modelo threading Java |
| PC-05 | Thread.Abort() | ALTO | Flags de cancelamento |
| PC-06 | Proxy SOCKS Async | ALTO | Avaliar biblioteca Java |
| PC-07 | Multi-Protocolo | ALTO | Definir versões suportadas |
| PC-08 | Plugins AES | MÉDIO | Novo sistema de plugins Java |
| PC-09 | async/await em Macros | ALTO | Definir modelo de macros Java |
| PC-10 | RayCast | MÉDIO | Migração integral do algoritmo |
| PC-11 | Configuração NBT | MÉDIO | Decidir formato de configuração Java |
| PC-12 | Windows Forms | ALTO | Definir estratégia de GUI Java |
| PC-13 | Código Unsafe | MÉDIO | Identificar e substituir |
| PC-14 | Sem Documentação | ALTO | Validação por testes reais |
| PC-15 | Acoplamento Global | ALTO | Injeção de dependência |
