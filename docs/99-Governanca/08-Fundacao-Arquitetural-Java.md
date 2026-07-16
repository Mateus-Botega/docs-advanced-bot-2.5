# 08 - Fundação Arquitetural Java

## 1. Visão Geral da Nova Plataforma Java

O objetivo da nova implementação em Java é substituir o sistema legado em C# por uma arquitetura moderna, escalável e de fácil manutenção. O foco é resolver a dívida técnica acumulada, melhorar a performance no gerenciamento de múltiplas instâncias simultâneas e garantir a estabilidade a longo prazo.

**Princípios Arquiteturais:**
*   **Alta Coesão e Baixo Acoplamento:** Cada módulo ou pacote deve ter uma responsabilidade única e bem definida.
*   **Isolamento do Domínio:** A lógica de negócio e as regras do bot (Core) não devem depender de frameworks externos ou de detalhes de infraestrutura (IO, Banco de Dados, UI).
*   **Design Orientado a Eventos (Event-Driven):** O sistema reagirá a pacotes da rede e mudanças de estado de forma assíncrona e reativa.
*   **Imutabilidade por Padrão:** Utilização de estruturas de dados imutáveis (`Records`) sempre que possível para prevenir efeitos colaterais e *race conditions*.

**Separação entre Módulos e Responsabilidades:**
O sistema é dividido estritamente entre Core (regras de negócio puras, entidades) e Infraestrutura (comunicação, persistência, frameworks). Isso garante que mudanças em protocolos de rede ou métodos de persistência não afetem o comportamento do bot.

**Diferenças Principais em Relação ao Legado C#:**
*   Eliminação completa do acoplamento com interface gráfica (Windows Forms).
*   Substituição de lógicas imperativas e aninhadas por Máquinas de Estado bem definidas.
*   Gerenciamento eficiente de concorrência utilizando Virtual Threads em vez de criação manual de dezenas de `Threads` ou `BackgroundWorkers` bloqueantes.
*   Uso de Injeção de Dependência nativa/framework em vez de instâncias globais ou classes utilitárias estáticas.

---

## 2. Estrutura Inicial do Projeto Maven

O projeto adotará uma estrutura multi-module no Maven (ou organização de pacotes estrita em monorepo) orientada a domínio e infraestrutura.

**Estrutura de Pacotes Esperada:**

```text
com.advancedbot
 ├── core              # Entidades, regras de negócio puras, interfaces de domínio
 ├── network           # Conexão TCP, pipelines de I/O, gerenciamento de sessão
 ├── protocol          # Definição e serialização/desserialização de Packets
 ├── bot               # Bot Engine, ciclo de vida, máquina de estados principal
 ├── pathfinding       # Algoritmos de navegação e locomoção pelo mundo
 ├── inventory         # Gerenciamento de itens, menus e baús virtuais
 ├── automation        # Macros e tarefas automatizadas (farm, pesca, etc.)
 ├── proxy             # Gerenciamento, validação e roteamento de proxies
 ├── scheduler         # Agendamento de tarefas e timers
 ├── persistence       # Acesso a banco de dados, arquivos locais, repositórios
 └── api               # Controladores e APIs de exposição para front-end/dashboard
```

---

## 3. Definição das Camadas Arquiteturais

A plataforma será dividida logicamente nas seguintes camadas principais:

### **Core**
*   **Entidades:** Representação pura de elementos do jogo (`Player`, `Location`, `Block`, `Item`).
*   **Estados:** Enumerações e modelos de dados que representam o estado atual do bot.
*   **Regras Puras:** Funções matemáticas e lógicas de validação (ex: cálculo de distância, bounding boxes) sem dependências externas.

### **Network**
*   **Conexão TCP:** Gerenciamento do socket raw.
*   **Gerenciamento de Sessão:** Controle de conexão ativa e fluxos de I/O de bytes.
*   **Reconexão:** Lógica de *backoff* e restabelecimento de conexões perdidas em nível de soquete.

### **Protocol**
*   **Packets:** Classes (preferencialmente `Records`) que mapeiam pacotes de entrada e saída.
*   **Serialização:** Conversores de/para arrays de bytes (`ByteBuf` / fluxos de dados).
*   **Compatibilidade (1.5.2 e 1.8):** Registro e versionamento de identificadores de pacotes dependendo da versão do protocolo negociada no *handshake*.

### **Bot Engine**
*   **Ciclo Principal:** *Tick loop* virtual ou gerenciador orientado a eventos para atualização do mundo.
*   **Estados:** O cérebro do bot; sabe se o bot está conectando, no lobby, minerando, etc.
*   **Comandos:** Receptores de instruções externas (ex: mover para X, usar item).
*   **Eventos:** Barramento interno (`EventBus`) para notificar o sistema sobre *HealthUpdate*, *ChatReceived*, etc.

### **Automation**
*   **Macros:** Tarefas agendadas e condicionais (ex: Auto-Reconnect, Auto-Soup).
*   **Mineração, Pesca, Venda, Reparo, Combate:** Módulos isolados de Inteligência Artificial e scripts comportamentais que reagem aos eventos do *Bot Engine*.

### **Infrastructure**
*   **Logs:** Geração, formatação e rotação de arquivos de log do sistema e instâncias.
*   **Configuração:** Carregamento dinâmico de `application.yml` e arquivos JSON/YAML.
*   **Proxy:** Serviços para atribuição de proxies aos sockets TCP.
*   **Arquivos & Persistência:** Implementação dos repositórios, cache local e persistência em banco de dados ou flat-files.

---

## 4. Padrões Técnicos Java

Os seguintes padrões oficiais devem ser respeitados rigorosamente em todo o código Java:

*   **Linguagem:** Java 21 (versão oficial definida para a plataforma Java).
*   **Framework Principal:** Spring Boot (aplicável para a camada de gerenciamento, API e orquestração de beans, evitando acoplar os bots diretamente aos ciclos web).
*   **Injeção de Dependência:** Obrigatório. Sem singletons estáticos, usar construtores injetados.
*   **Configuração:** Centralizada via `application.yml` (Spring) ou arquivos YAML mapeados.
*   **Logs:** SLF4J integrado com Logback. Proibido uso de `System.out.println`.
*   **Build System:** Maven.
*   **Boilerplate:** Uso do Lombok está aprovado para `@Getter`, `@Setter`, `@Builder`, etc., mantendo o código limpo.
*   **Imutabilidade:** Uso de `Records` onde fizer sentido (ex: DTOs de Packets e Eventos estáticos).
*   **Concorrência:** Utilização de Virtual Threads (se executando em JVM compatível/Java 21) ou *Thread Pools* assíncronas dedicadas onde houver ganho real de I/O não bloqueante.

---

## 5. Modelo de Execução do Bot

Cada instância do bot deve seguir um ciclo de vida rigorosamente controlado:

1.  **Inicialização:** Construção do objeto bot, injeção de dependências, carregamento de conta e configuração de proxy associado.
2.  **Conexão:** Estabelecimento do Socket TCP via proxy, finalização do *handshake*.
3.  **Login:** Negociação de encriptação (se aplicável), envio de credenciais de login, recepção de pacote `Login Success`.
4.  **Carregamento do Mundo:** Recepção de pacotes `ChunkData`, inicialização de entidades e posição (`PlayerPositionAndLook`).
5.  **Execução de Tarefas:** Transição para o estado *Idle/Active*. O sistema de eventos reage aos *ticks* do servidor e ativa módulos de *Automation*.
6.  **Tratamento de Falhas:** Em caso de `Disconnect` ou `Timeout`, o bot avalia a causa e programa uma rotina de *retry* isolada.
7.  **Desligamento Seguro:** Fechamento de sockets (*Graceful Shutdown*), gravação de estatísticas, liberação de memória e encerramento de threads.

---

## 6. Sistema de Estados

Para substituir o modelo legado em C# (altamente acoplado e procedural), a nova arquitetura adotará uma **Máquina de Estados (State Machine)** formal para cada Bot.

*   **State Machine:** Define em qual estado global o bot se encontra (Ex: `DISCONNECTED`, `CONNECTING`, `LOGGING_IN`, `IN_GAME`, `RECONNECT_WAIT`).
*   **Eventos:** Entradas que causam reações na máquina de estado (Ex: `PacketDisconnectReceived`, `WorldLoaded`).
*   **Transições:** Mudanças formais e válidas de estado. O bot não pode ir de `DISCONNECTED` direto para `IN_GAME`.
*   **Timeout e Retry:** Estados como `CONNECTING` devem ter timers associados. Ao expirar, a máquina aciona a transição de falha.
*   **Recuperação Automática:** Em estados de falha, o bot tentará reinicializar o fluxo após um *backoff* configurável (ex: Auto-Reconnect).

---

## 7. Arquitetura de Rede

A infraestrutura de comunicação será desenhada para suportar alta densidade de conexões simultâneas de bots.

*   **Cliente Minecraft:** Abstraído através de uma interface de rede. Cada bot possui sua própria sessão e pipeline independentes.
*   **Socket TCP:** Conexão direta e persistente.
*   **Pipeline de Pacotes:** Padrão interceptor (Decoder -> Handler -> Encoder). Fluxo contínuo de recepção, tradução de bytes para Packet Objects e disparo de eventos para o *Bot Engine*.
*   **Threads:** Isolamento entre as threads de I/O (Network) e as threads de lógica (Bot Engine/Events). Uma thread de rede não pode ser bloqueada pelo *pathfinding*.
*   **Múltiplos Bots:** A plataforma gerenciará os bots como instâncias independentes (`Session`), sem estado global compartilhado.

---

## 8. Arquitetura de Proxy

Módulo crítico para contornar restrições de IP (Anti-Bot) e proteção de contas.

*   **Suporte:** Compatibilidade nativa com protocolos HTTP(S) e SOCKS(4/5).
*   **Configuração Individual:** Proxy definido por IP, Porta e opcionalmente Credenciais de Autenticação.
*   **Isolamento:** Cada bot recebe seu próprio canal de proxy isolado na criação do Socket.
*   **Validação e Rotação:** O sistema deve validar proxies (verificação de *reachability* e ping) e prever estrutura futura para rotação automática de IPs falhos.
*   **Compatibilidade com Servidores:** Necessário para servidores que implementam *Rate Limiting* (X contas por IP).

---

## 9. Compatibilidade Minecraft

A nova fundação deve ser desenhada para abstrair a versão do Minecraft, resolvendo o problema de *hardcoding* de pacotes presente no C#.

### Protocolo Minecraft 1.5.2
*   Protocolo numérico estático para pacotes.
*   Limitações próprias na movimentação (física de água, blocos).
*   Particularidades de chat e formatação de texto antigas.

### Protocolo Minecraft 1.8
*   Estrutura de pacotes baseada em VarInt (comprimento prefixado) e compressão de pacotes introduzida.
*   Diferenças drásticas em pacotes de Inventário, Movimento e Metadados de Entidades.

**Estratégia:**
A camada `protocol` utilizará **Adapters por Versão (Version Adapters)** e Registro de Pacotes Mapeados. O *Bot Engine* ouvirá eventos genéricos (ex: `ChatEvent`, `HealthEvent`), independentemente de como o pacote cru foi recebido na 1.5.2 ou 1.8.

---

## 10. Estratégia de Testes da Fundação

A fundação Java deve ser validada desde o início com testes rigorosos:

*   **Testes de Conexão:** Mocks de servidor TCP validando *handshake* e falhas.
*   **Testes de Packets (Serialização/Desserialização):** Garantir que bytes de entrada gerem o `Record` correto para ambas as versões do jogo.
*   **Testes de Estados:** Testes Unitários de Máquina de Estado validando transições válidas e rejeição de transições ilegais.
*   **Testes de Lógica Limpa (Pathfinding/Inventário):** Validação de regras matemáticas e indexação de baús sem dependência de rede.
*   **Testes de Proxy:** Mocks de proxy local para verificar tráfego passando corretamente pela ponte.
*   **Testes de Reconexão:** Simulação de timeouts e verificação de re-acionamento das conexões.

---

## 11. Decisões Arquiteturais (ADRs)

A base técnica Java segue formalmente as seguintes premissas aprovadas:

*   **ADR-001:** Java Greenfield ao invés de conversão automática. O C# servirá unicamente como consulta de lógicas antigas, não haverá tradução direta linha a linha.
*   **ADR-002:** Separação estrita entre Core / Infraestrutura, eliminando acoplamentos de lógica de bot com a camada de visualização/gerenciamento.
*   **ADR-003:** Suporte multi-versão Minecraft de forma abstraída (Protocol Adapters) visando primordialmente 1.5.2 e 1.8.
*   **ADR-004:** Arquitetura preparada *by-design* para suportar múltiplos bots rodando simultaneamente na mesma JVM, sem vazamento de estado global.

---

## 12. Critério de Aprovação da Fundação

Este documento é considerado aprovado e permite o início da criação do código base em Java quando:

- [ ] A arquitetura foi totalmente revisada e aprovada pelos engenheiros líderes.
- [ ] O modelo de pacotes Maven foi validado.
- [ ] O padrão tecnológico (Java 21, Spring, Maven) está acordado.
- [ ] As responsabilidades das camadas (`core`, `network`, `automation`, etc.) estão claras, sem sobreposições.
- [ ] Nenhuma classe Java foi criada até o fechamento deste acordo arquitetural.
