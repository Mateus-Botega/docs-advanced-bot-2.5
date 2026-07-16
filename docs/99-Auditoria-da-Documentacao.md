# Auditoria da Documentação — AdvancedBot

**Objetivo:** Avaliar a completude, precisão e atualidade da documentação existente em relação ao código-fonte da versão 2.4.5, visando subsidiar a reescrita para Java. Este relatório mapeia discrepâncias e lacunas que ainda persistem mesmo após atualizações recentes.

## Cobertura por Módulo

### Rede e Protocolo (`04-Rede`, `05-Protocolo-Minecraft`)
- **Informações ausentes:** Detalhes de como o `PacketStream` lida com corrupção de descriptografia (lança exceção engolida silenciosamente).
- **Algoritmos parcialmente documentados:** A lógica de reconexão recursiva em `ConnectAndHandshakeAsync`.

### Entidades (`06-Entidades`)
- **Regras de negócio não documentadas:** Física completa de fluidos (água e lava) divergem da movimentação terrestre e não estão descritas em detalhes.
- **Pontos fracos:** Mutabilidade in-place do objeto `AABB` durante a movimentação, o que não é *thread-safe*.

### Inventário (`07-Inventario`)
- **Informações incorretas:** A documentação prescreve um comportamento seguro, mas omite que a variável `ClickedItem` é estática e compartilhada globalmente.
- **Eventos omitidos:** Faltam detalhes sobre como cliques engolem exceções silenciosamente (`catch(Exception) {}`).

### Comandos e Macros (`09-Comandos`, `10-Macros`)
- **Dependências omitidas:** As dependências do motor de script (Jint) não estão versionadas claramente.
- **Informações ausentes:** Scripts rodam sem `CancellationToken`, criando vazamento de threads (zumbis) se a conexão cair.

### Plugins (`11-Plugins`)
- **Informações desatualizadas:** A documentação fala sobre recarregamento de plugins (Hot-Reload), mas omite que isso gera *Assembly Load Leaks* (arquivos `.dll` antigos não são removidos da memória RAM).

### Domínio (`19-Dominio`)
- **Fluxos incompletos:** Faltam os fluxos assíncronos da lógica de autenticação e falhas na validação.

## Cobertura por Funcionalidade

### Autenticação (AuthMe / Mojang)
- **Informações ausentes:** Tokens em cache (`login_cache.bin`) são guardados em texto pleno (sem criptografia).

### Movimentação / Pathfinding
- **Algoritmos parcialmente documentados:** O "A*" está descrito genericamente, mas as heurísticas de penalização aplicadas pelo *AutoMiner* (ex: custo alto para quebrar blocos sob os pés) estão ocultas.

### Combate e Pesca
- **Documentação contraditória:** Em um lugar o range de ataque é 3.0, no código chega a 20.0 (`CommandMobTeleport`) e 3.5 (`CommandMob`).
- **Estados omitidos:** A pesca real do Solk apenas lança a vara infinitamente e confia no servidor para recolher; a documentação descreve uma máquina irreal onde o bot "ouve o som e puxa".

## Cobertura por Classe

### `MinecraftClient`
- **Informações ausentes:** O ciclo de destruição de recursos via `Dispose()` e encerramento das *streams* está sub-documentado.

### `Inventory`
- **Pontos fracos:** A variável `ClickedItem` estática inviabiliza a arquitetura para suportar múltiplos bots paralelos na mesma JVM/CLR.

### `FileLock` (Macros Solk)
- **Limitações não documentadas:** Utiliza o `%TEMP%` local do Windows. Se a arquitetura Java for em containers (ex: Docker) distribuídos, o sistema de lock de baús falhará silenciosamente.

## Cobertura por Fluxo

### Fluxo de Conexão
- **Eventos omitidos:** Limites de recursão no handshake (limite = 3) estão omitidos.

### Fluxos de Macros Assíncronas
- **Informações duplicadas/contraditórias:** O fluxo de Pesca diverge entre o `18-Fluxos-Funcionais` e a implementação `Solk`.
- **Dependências cruzadas omitidas:** O evento `onReceiveChat` da thread de rede (IOCP) tenta mutar os estados da macro ao mesmo tempo em que a thread principal lê, gerando *Race Conditions*.

## Cobertura por Protocolo

### Pacotes Diretos
- **Regras de negócio não documentadas:** O `UnsafeDirectPacket` contorna a fila de pacotes tradicional e não está plenamente documentado de porquê existe (provável uso para bypass explícito).

## Cobertura por Algoritmo

### Bypass de Captcha (`SkySurvival`)
- **Algoritmos parcialmente documentados:** O uso de *Reflection* do .NET para mapear os itens através dos metadados não está explicitado.

### Corner Glitch
- **Documentação incorreta:** O algoritmo em `CommandMobTeleport` está documentado como algo ativo, mas no código a linha de ataque está literalmente comentada (função inacabada).

## Pontos Fracos

1. **Estado Global Mutável:** Uso irresponsável de estáticos (`Inventory.ClickedItem`) impedindo paralelismo seguro.
2. **Ciclo de Vida Zumbi (Threads):** Tarefas `async void` e Scripts JavaScript funcionam no formato "fire-and-forget", nunca sendo encerradas adequadamente numa desconexão.
3. **Engolimento de Exceções:** Inúmeros `catch(Exception)` vazios (Swallowing) na rede e no inventário, dificultando muito o monitoramento de erros de estado.
4. **Acoplamento Severo:** Callbacks de Plugins recebem acesso direto à instância inteira do `MinecraftClient`, podendo causar estragos irremediáveis nos pacotes.

## Contradições

1. **Range de Combate:** Regras de domínio documentam 3.0 blocos. Código usa 3.5, 4.0 e 20.0 blocos sem nenhum aviso.
2. **Taxa de TPS (Ticks por Segundo):** Regras de Invariantes dizem que o sistema jamais emite pacotes massivos repetidos no mesmo tick. No entanto, o `CommandMob` emite **850 pacotes** consecutivos num loop em milissegundos para hitar mobs no *Craftlandia*.
3. **Máquina de Pesca:** Descrita na documentação geral como completa (lança, espera o som/partícula, puxa). No código atual (Solk) ela não puxa a vara; confia 100% num comportamento exótico do servidor.

## Prioridade das Correções

### CRÍTICO
- **Race Conditions e Variáveis Estáticas (Inventário/EntityManager):** Deve-se reimplementar tudo orientando a escopo de Sessão (`BotSession`). Não migrar nenhuma estrutura estática do C# para o Java.
- **Memory Leaks de Plugins e Threads:** O Java não suporta descarregamento tradicional de classes (`ClassLoader` leaks). E as rotinas de script exigirão *Thread Interrupts* precisos.

### ALTO
- **Padronização do Range de Ataque e TPS:** Engenharia precisa decidir qual é o *range* correto e mitigar a função `hitarMob()` que dispara 850 pacotes de uma vez, pois isso causaria banimentos instantâneos em anti-cheats atualizados (GrimAC / Polar).
- **Fail-Fast em Falhas Silenciosas:** Os `try-catch` ignorados da versão legada não devem ser migrados. Toda falha de *Parser* de inventário ou rede deve derrubar a sessão para garantir a segurança da conta.

### MÉDIO
- **Implementação do Lock de Arquivos (`FileLock`):** Para o Java (pensando em escalabilidade Cloud), migrar a trava de `%TEMP%` do Windows para um *Distributed Lock* com Redis, permitindo que a farm de mobs rode distribuída.
- **Física Replicada (Água/Lava):** Complementar documentação dos vetores exatos e multiplicadores de atrito que estavam ocultos.

### BAIXO
- **Persistência de Sessão:** Migrar o texto simples (`login_cache.bin`) para mecanismos seguros no disco.
- **Comandos Inoperantes:** Limpar lixo do legado, como o *Corner Glitch* que jamais foi finalizado, não portando isso para a nova arquitetura de forma cega.
