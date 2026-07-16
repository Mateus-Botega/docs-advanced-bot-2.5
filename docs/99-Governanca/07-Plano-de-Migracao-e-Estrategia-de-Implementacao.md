# 07 - Plano de Migração e Estratégia de Implementação

## Objetivo

Este documento oficializa o plano de execução para a migração do AdvancedBot legado (C#) para a nova arquitetura baseada em Java 21 e Spring Boot. 

O intuito deste planejamento é transformar a auditoria técnica aprovada e as diretrizes de governança em uma sequência estruturada, incremental e controlada de implementação. Deste modo, asseguramos que toda dívida técnica identificada não seja transferida e que as Decisões Arquiteturais (DEC-01 até DEC-10) sejam rigorosamente atendidas.

## 1. Estratégia Geral de Migração

A transição arquitetural obedecerá às seguintes estratégias:

- **Abordagem Escolhida:** Reescrita incremental controlada. A reescrita será feita do zero (greenfield) para o núcleo e rede, aproveitando lógicas de domínio testadas (ex: A* Pathfinding).
- **Separação entre Legado e Novo:** A base de código C# será mantida intocada (apenas para leitura e referência de comportamento). O projeto Java será construído em um repositório ou diretório limpo. O ambiente legado continuará operando para extração de requisitos, enquanto a nova plataforma Spring Boot gerenciará instâncias isoladas (Single-Tenant, DEC-09).
- **Motivo da Não Conversão Automática:** Ferramentas de conversão automática (C# para Java) carregariam consigo o acoplamento de estado global severo, a mistura de interface gráfica com lógica de pacotes de rede e a dependência de chamadas não-estruturadas (`Thread.Sleep`), o que inviabilizaria a nova arquitetura baseada em Virtual Threads (DEC-03) e API isolada.
- **Critérios de Validação entre Versões:** Paridade de comportamento. O bot recém-criado em Java deve demonstrar que atende aos mesmos cenários em rede, comportamentos de macro e simulação de atraso *legit* sem gerar falsos positivos nos *Anti-Cheats*. 

## 2. Fases da Migração

A migração foi dividida em fases progressivas para garantir que a rede e a base de domínio funcionem antes de introduzirmos as automações (macros).

| Fase | Objetivo | Componentes envolvidos | Status |
|---|---|---|---|
| 1 | Fundação do projeto Java | Estrutura Maven/Spring Boot, YAML (DEC-05) | Não Iniciado |
| 2 | Motor do Bot isolado | Core Domain, Virtual Threads (DEC-03), CLI | Não Iniciado |
| 3 | Sistema de conexão Minecraft | Protocol Layer 1.8 (DEC-01), Handshake | Concluído (Milestone 4, ver [11-Estado-Atual-Migracao.md](11-Estado-Atual-Migracao.md)) |
| 4 | Gerenciamento de sessão | Autenticação, Injeção Proxy Socket (DEC-10) | Não Iniciado |
| 5 | Eventos e pacotes | Serialização/Deserialização, BouncyCastle (DEC-04) | Não Iniciado |
| 6 | Sistema de comandos | Core de Comandos, Chat, Movimentação Básica | Não Iniciado |
| 7 | Inventário | Sincronização de Itens, Slots e Janelas | Não Iniciado |
| 8 | Macros | Pesca, Mineração, Venda, Reparação | Não Iniciado |
| 9 | Persistência/configuração | YAML Config, Load/Save de estados | Não Iniciado |
| 10| Interface Web React | API REST (DEC-02), Dashboard Orquestrador | Não Iniciado |
| 11| Testes | Unitários, Integração e Paridade Funcional | Não Iniciado |

## 3. Ordem de Implementação Técnica

Para garantir que a fundação exista e suporte os módulos de alto nível, a sequência técnica deve ser restritamente observada:

1. **Criar projeto base Java (Spring Boot):** Define o orquestrador (DEC-08) e a infraestrutura básica do projeto.
2. **Criar arquitetura de módulos:** Estabelece o esqueleto e pacotes (`domain`, `network`, `macro`) separando a API de administração do motor.
3. **Implementar camada Minecraft Protocol (1.8):** A fundação técnica real do bot é conseguir entender a linguagem do servidor.
4. **Criar ciclo de vida do Bot (Login/Criptografia):** Essencial para obter o estado `Play` após Handshake, e valida a criptografia AES (DEC-04) com Proxies (DEC-10).
5. **Implementar eventos de processamento (Pacotes):** Permitirá reagir a desconexões, movimentações de outros jogadores e estado global (chuva, tempo).
6. **Implementar comandos:** Lógicas básicas (chat, andador, quebra de blocos).
7. **Migrar automações e sistema de Eventos:** Construção final de Pesca, Mineração, e Reparação com Virtual Threads (DEC-03).
8. **Desenvolvimento Web:** Finalizar a visão geral da plataforma em React (DEC-02).

**Motivo da Ordem:** A abordagem construída do nível mais baixo (protocolo de rede e conexão) ao mais alto (automações e UI) garante feedback contínuo. Não é possível validar uma macro de pesca se o bot não consegue se manter conectado, e não podemos validar a conexão sem a camada de pacotes implementada.

## 4. Estratégia de Conversão C# → Java

O legado será modernizado substituindo anti-patterns por recursos nativos modernos do ecossistema Java:

| Código C# | Nova implementação Java | Estratégia |
|---|---|---|
| `Thread` e `Thread.Sleep` | Virtual Thread / ScheduledExecutor | Reescrita via Java 21 (DEC-03) |
| `Task` async | `CompletableFuture` / EventLoop | Reescrita adaptada |
| Configurações JSON/NBT | Classes DTO / `application.yml` | Substituição (DEC-05) |
| Estados Estáticos/Globais | Serviços Gerenciados Spring (`@Service`) | Reescrita arquitetural (DEC-08) |
| Eventos Próprios | EventBus interno / `ApplicationEventPublisher` | Reescrita de padrões de projeto |
| Implementação Crypto Própria | Biblioteca `BouncyCastle` (AES-CFB8) | Substituição de Segurança (DEC-04) |
| Interface Gráfica / OpenGL | API REST JSON / Interface React (Browser) | Descarte / Substituição (DEC-02) |
| Sistema de Plugins C# | Carregador de Extensões SPI ou Customizado | Substituição Segura (DEC-06) |

## 5. Critérios de Sucesso

O projeto será considerado executado com sucesso se atingir os seguintes indicadores objetivos:

- Bot conecta com sucesso no servidor Minecraft alvo (suporte inicial focado na 1.8, mas também demonstrando compatibilidade de conexão em 1.5.2).
- Mantém sessão estável (ping/keep-alive respondendo perfeitamente).
- Executa comandos básicos e movimentação coerente de pacotes.
- Executa macros sem bloquear a thread principal (*Virtual Threads* responsivas).
- Reconecta-se automaticamente após queda ou reinício do servidor.
- Trabalha com múltiplos bots simultâneos no modelo isolado (Uma JVM por bot - DEC-09).
- Cada bot utiliza proxy SOCKS5/HTTP individual e isolado (DEC-10).
- Possui gerenciamento integrado via painel Web.

## 6. Plano de Testes

Para garantir estabilidade máxima e mitigação da dívida técnica, aplicaremos:

- **Testes Unitários:** Focados em componentes independentes de I/O, como algoritmos de *Pathfinding (A*)*, transformações matemáticas de inventário e conversões de ID de itens.
- **Testes de Integração:** Testes de codificação e decodificação de *Packets* sem depender da rede externa, provando que um bytebuffer de resposta equivale ao fluxo esperado do servidor.
- **Testes de Estabilidade:** Execução em *Long-Running Sessions* com múltiplos bots (10+) rodando para verificar *Memory Leaks* ou *Deadlocks* de Virtual Threads.
- **Testes de Conexão:** Testes do aperto de mão criptográfico (*Handshake* e login AES-CFB8).
- **Testes contra Servidores Reais:** Criação de um ambiente de *Staging* (servidor Bukkit/Spigot local com Anti-Cheats instalados) para garantir que bots executem atividades legitimamente.

## 7. Riscos Durante Implementação

Foram mapeados os principais riscos durante a reescrita técnica e as devidas mitigações:

| Risco | Impacto | Mitigação |
|---|---|---|
| **Diferenças de Protocolo Minecraft (1.5.2 vs 1.8)** | Alto | Desenvolver a arquitetura inicial abstraindo o *Protocol Layer*. Centralizar esforços inicialmente apenas na versão 1.8 (DEC-01) para validar o motor. |
| **Incompatibilidade ou Curva do Java 21** | Baixo | Uso rigoroso de documentação focada em padrões Spring Boot 3.x e Virtual Threads. |
| **Comportamento Anti-Cheat Rigoroso** | Alto | Inspeção minuciosa das *delays*, *head-snapping*, ritos de envios de pacotes `PlayerPositionAndLook` do C# original para imitar a naturalidade. |
| **Sincronização de Inventário** | Médio | Validação forte dos pacotes `Transaction` e `WindowItems`. Falha em sincronização deve ser tratada como prioridade crítica e abortar macros em andamento. |
| **Latência e Gargalos de I/O** | Médio | Delegação de operações bloqueantes (I/O) e rede para as *Virtual Threads*, impedindo que atrasos em Proxies derrubem o sistema inteiro. |
| **Múltiplos bots simultâneos em colisão de memória** | Alto | Isolamento rígido de estado garantido pelo modelo Operacional *Single-Tenant* (Uma JVM por Bot - DEC-09). |

## 8. Critério de Início da Implementação

Este documento estabelece que a fase de planejamento arquitetural está concluída. 

**Checklist:**

- [x] Auditoria aprovada
- [x] Arquitetura definida
- [x] Estratégia definida
- [ ] Fundação Java iniciada

O cumprimento dos três primeiros itens da checklist autoriza o início imediato dos trabalhos de engenharia na nova base (Fase 1: Fundação do projeto Java).
