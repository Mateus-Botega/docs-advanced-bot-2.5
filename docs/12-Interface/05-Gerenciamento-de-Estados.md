# Arquitetura de Gerenciamento de Estados

## 1. Objetivo

Este documento define a especificação arquitetural conceitual para o gerenciamento de estados na nova interface
web do sistema AdvancedBot. Ele estabelece o modelo de dados em memória, a classificação dos escopos de estado, a topologia de
sincronização em tempo real, as regras de persistência e as estratégias de otimização para suportar alta concorrência.

A aplicação frontend atuará como um cliente de alta densidade operando sob os seguintes requisitos críticos:
- Suporte a mais de 300 conexões simultâneas de bots gerenciadas em tempo real.
- Processamento de alto volume de eventos e streaming de mensagens de chat/logs (centenas de eventos por segundo).
- Renderização visual tridimensional contínua (Viewer 3D a 60 FPS).
- Acompanhamento de operações assíncronas em lote de longa duração sem congelamento da interface.
- Garantia de latência de resposta visual inferior a 100 milissegundos para interações do operador.

Esta especificação é estritamente arquitetural e conceitual. Não contém códigos de implementação nem dependências de bibliotecas específicas.

---

## 2. Classificação dos Estados

Para garantir a previsibilidade, desacoplamento e isolamento de escopo, todos os dados manipulados no frontend dividem-se em 5 categorias formais:

### Estado Global
- **Definição**: Dados de infraestrutura e sessão compartilhados por toda a aplicação.
- **Escopo**: Acessível por qualquer componente ou página.
- **Exemplos**: Status de autenticação do operador, estado de conexão WebSocket com o backend Java e configurações visuais globais (tema).

### Estado de Domínio
- **Definição**: Dados que representam as entidades de negócio centrais do sistema.
- **Escopo**: Compartilhado entre páginas funcionais e componentes organizacionais.
- **Exemplos**: Coleção de bots ativos (`BotState`), pool de proxies (`ProxyState`), scripts de automação (`MacroState`) e configurações de mineração (`MinerState`).

### Estado Local
- **Definição**: Dados temporários e restritos ao ciclo de vida interno de um único componente visual.
- **Escopo**: Privado do componente. Não é exposto nem compartilhado com a árvore superior.
- **Exemplos**: Texto digitado em uma caixa de busca antes da confirmação, estado de abertura/fechamento de um menu suspenso ou controle de aba ativa em um painel.

### Estado Efêmero
- **Definição**: Dados transitórios de curta duração voltados ao feedback visual imediato.
- **Escopo**: Mantido apenas durante o tempo de exibição ou execução da animação associada.
- **Exemplos**: Mensagens de notificação `Toast`, spinners de carregamento e indicadores temporários de progresso em linha.

### Estado Persistido
- **Definição**: Dados que devem ser preservados entre recargas da página ou entre sessões de uso da aplicação web.
- **Escopo**: Armazenado em mecanismos de persistência local ou sincronizado no banco de dados do backend.
- **Exemplos**: Preferências de layout do operador, histórico de buscas recentes, ordenação padrão de tabelas e credenciais de acesso salvas.

---

## 3. Catálogo de Estados

### 3.1 BotState

- **Responsabilidades**: Armazenar o vetor central de bots gerenciados, controlando individualmente o status de conexão, latência de rede (ping), posição geográfica no mapa (coordenadas X, Y, Z), pontos de vida/fome, slots de inventário e atividade operacional corrente.
- **Origem dos Dados**: Stream WebSocket assíncrono emitido pelo backend Java (`WS /ws/bots`).
- **Consumidores**: `DashboardLayout`, `BotsPage`, `SidebarBotList`, `DataTable`, `BotCard`, `ViewerLayout` e `Drawer` de inspeção.
- **Frequência de Atualização**: Alta frequência para dados de posição/vida (sob demanda de evento) e frequência moderada para ping/status (1 a 5 segundos).
- **Retenção**: Mantido em memória durante todo o ciclo de vida da sessão ativa dos bots.
- **Eventos Relacionados**: `BOT_CONNECTED`, `BOT_DISCONNECTED`, `BOT_STATUS_CHANGED`, `BOT_HEALTH_UPDATED`, `BOT_INVENTORY_UPDATED`.

### 3.2 SessionState

- **Responsabilidades**: Armazenar a identidade do operador autenticado, o token de autorização JWT, o estado de saúde do canal WebSocket de comunicação e as permissões de acesso da sessão.
- **Origem dos Dados**: REST API de login (`POST /api/v1/auth/login`) e eventos de controle do WebSocket (`WS /ws/control`).
- **Consumidores**: Toda a aplicação (verificadores de rota protegida, `TopBar`, interceptores de transporte REST/WS).
- **Frequência de Atualização**: Baixa frequência (na autenticação, renovação de token ou perda de sinal de rede).
- **Retenção**: Persistido no armazenamento local do navegador durante a validade do token.
- **Eventos Relacionados**: `SESSION_AUTHENTICATED`, `SESSION_EXPIRED`, `WEBSOCKET_CONNECTED`, `WEBSOCKET_DISCONNECTED`.

### 3.3 DashboardState

- **Responsabilidades**: Manter os dados agregados e consolidados do sistema, incluindo estatísticas globais de tráfego de rede (KB/s de entrada e saída), pacotes por segundo, totalização de bots por estado e histórico de métricas de hardware do backend.
- **Origem dos Dados**: Telemetria periódica via Server-Sent Events (SSE) ou polling REST (`GET /api/v1/dashboard/summary`).
- **Consumidores**: `DashboardLayout`, `MetricCard`, gráficos de monitoramento da `TopBar`.
- **Frequência de Atualização**: Intervalos fixos de 1 segundo.
- **Retenção**: Buffer circular em memória com os últimos 60 pontos de amostragem para renderização de gráficos de tendência.
- **Eventos Relacionados**: `METRICS_TICK`, `SYSTEM_ALERT_RAISED`.

### 3.4 LogState

- **Responsabilidades**: Armazenar os registros de mensagens de chat do Minecraft e logs de eventos técnicos emitidos pelo sistema ou por scripts de macros.
- **Origem dos Dados**: Stream WebSocket de alta velocidade (`WS /ws/logs`).
- **Consumidores**: `ConsoleLogViewer`, overlays de chat do `Viewer 3D` e leitores de log de inspeção.
- **Frequência de Atualização**: Altíssima frequência (múltiplas linhas por segundo sob alta carga).
- **Retenção**: Buffer circular rigorosamente limitado em memória (máximo de 2.000 linhas configuráveis). Linhas antigas que ultrapassam o limite são automaticamente descartadas da memória para evitar vazamento de RAM no navegador.
- **Eventos Relacionados**: `LOG_LINE_RECEIVED`, `CHAT_MESSAGE_RECEIVED`, `LOG_BUFFER_CLEARED`.

### 3.5 ProxyState

- **Responsabilidades**: Armazenar a coleção de proxies cadastradas (IP, Porta, Tipo, País), os estados de validação de latência em tempo real e o progresso de tarefas do checador de proxies.
- **Origem dos Dados**: REST API (`GET /api/v1/proxies`) e respostas do validador (`POST /api/v1/proxies/check`).
- **Consumidores**: `ProxyPage`, `ProxyTable`, modal `ProxyChecker` e seletores de conexão de rede.
- **Frequência de Atualização**: Sob demanda do operador ou continuamente durante a execução do `ProxyChecker`.
- **Retenção**: Persistido no backend e mantido em memória durante a navegação.
- **Eventos Relacionados**: `PROXIES_LOADED`, `PROXY_CHECK_STARTED`, `PROXY_LATENCY_UPDATED`, `PROXY_CHECK_COMPLETED`.

### 3.6 MacroState

- **Responsabilidades**: Manter a coleção de scripts JavaScript armazenados, o código em edição no `CodeEditor`, o status da compilação sintática e o mapeamento de quais bots estão executando macros ativas.
- **Origem dos Dados**: REST API (`GET /api/v1/macros`) e eventos do motor de macros Java.
- **Consumidores**: `MacrosPage`, `CodeEditor`, `SidebarBotList` e badges de status de automação.
- **Frequência de Atualização**: Sob demanda durante a edição e eventos de início/parada de scripts.
- **Retenção**: Código-fonte persistido no backend; estado de execução mantido em memória durante a sessão.
- **Eventos Relacionados**: `MACRO_CREATED`, `MACRO_COMPILED`, `MACRO_STARTED`, `MACRO_STOPPED`, `MACRO_RUNTIME_ERROR`.

### 3.7 MinerState

- **Responsabilidades**: Armazenar a fila de prioridades de blocos alvo de escavação, o raio de atuação configurado, as flags de troca automática de ferramentas e as regras ativas para inventário cheio.
- **Origem dos Dados**: REST API de configuração do minerador (`GET /api/v1/config/miner`).
- **Consumidores**: `MinerPage`, formulários de opção do minerador e Drawer de inspeção de bots.
- **Frequência de Atualização**: Baixa frequência (atualizado apenas quando o operador altera as configurações).
- **Retenção**: Persistido no banco de dados do backend e replicado na memória do cliente web.
- **Eventos Relacionados**: `MINER_CONFIG_UPDATED`, `MINER_PRIORITY_CHANGED`.

### 3.8 ViewerState

- **Responsabilidades**: Armazenar o cache de chunks tridimensionais recebidos do servidor, as coordenadas da câmera, os modos visuais ativos (Câmera Livre vs. Jogador), a lista de entidades 3D no raio de visão e as configurações de renderização WebGL (distância de visão, VBOs, limites de FPS).
- **Origem dos Dados**: Stream WebSocket dedicado de chunks e eventos (`WS /ws/viewer/{botId}`).
- **Consumidores**: `ViewerPage`, canvas WebGL, `PlayerStatusOverlay` e `InventoryGrid`.
- **Frequência de Atualização**: 60 atualizações por segundo (sincronizado com o loop de renderização do navegador via `requestAnimationFrame`).
- **Retenção**: Dados de chunks mantidos em memória apenas enquanto a página do visualizador estiver aberta. Totalmente purgado ao fechar o canvas para liberar memória de vídeo (VRAM).
- **Eventos Relacionados**: `CHUNK_DATA_RECEIVED`, `BLOCK_CHANGED`, `ENTITY_MOVED`, `CAMERA_MODE_TOGGLED`.

### 3.9 NotificationState

- **Responsabilidades**: Gerenciar a fila de mensagens de notificação flutuantes (`Toast`), controlando tempo de exibição, severidade (Sucesso, Alerta, Erro) e ações de fechamento.
- **Origem dos Dados**: Disparado por qualquer manipulador de erro, resposta REST ou evento assíncrono de sistema.
- **Consumidores**: Componente organizacional `ToastContainer` no topo da camada visual.
- **Frequência de Atualização**: Transitório / Sob demanda.
- **Retenção**: Temporária (removido da memória imediatamente após a expiração do tempo de exibição de 3 a 5 segundos).
- **Eventos Relacionados**: `TOAST_EMITTED`, `TOAST_DISMISSED`.

---

## 4. Máquina de Estados dos Bots

O ciclo de vida de cada bot individual cadastrado na sessão é estritamente regido pela seguinte Máquina de Estados Finitos:

```
     ┌──────────────┐
     │ Desconectado │
     └──────┬───────┘
            │ (Inicia Conexão)
            ▼
     ┌──────────────┐
     │  Conectando  │◄───────────────────┐
     └──────┬───────┘                    │
            │ (Handshake TCP OK)         │
            ▼                            │
     ┌──────────────┐                    │
     │ Autenticando │                    │ (Tentativa de Reconexão)
     └──────┬───────┘                    │
            │ (Login Aceito)             │
            ▼                            │
     ┌──────────────┐                    │
     │    Online    ├────────────────────┤
     └──────┬───────┴────────────────────┤
            │ (Inicia Script)            │
            ▼                            │
┌──────────────────────┐                 │
│   Executando Macro   │                 │
└──────────┬───────────┘                 │
           │ (Erro / Comando Stop)       │
           ├─────────────────────────────┘
           │
           │ (Falha Crítica / Kick)
           ▼
     ┌──────────────┐
     │     Erro     │
     └──────────────┘
```

### Detalhamento dos Estados da Máquina

#### 1. Desconectado
- **Significado**: O bot está cadastrado na aplicação, porém não possui qualquer socket ou thread ativa no backend Java.
- **Ações Permitidas**: Iniciar Conexão, Remover da Sessão, Editar Credenciais, Associar Proxy.
- **Ações Bloqueadas**: Abrir Viewer 3D, Executar Macro, Enviar Chat, Minerar.
- **Eventos de Transição**: `CMD_CONNECT` -> Transiciona para **Conectando**.

#### 2. Conectando
- **Significado**: O backend Java está realizando o handshake TCP com o servidor Minecraft e negociando a conexão de rede.
- **Ações Permitidas**: Cancelar Conexão.
- **Ações Bloqueadas**: Iniciar nova conexão, Executar Macro, Enviar Chat, Abrir Viewer 3D.
- **Eventos de Transição**: `TCP_CONNECTED` -> Transiciona para **Autenticando** | `SOCKET_TIMEOUT` / `REFUSED` -> Transiciona para **Erro**.

#### 3. Autenticando
- **Significado**: Conexão TCP estabelecida; o backend está enviando pacotes de login, resolvendo autenticação Mojang ou disparando os comandos de auto-login (`/login`).
- **Ações Permitidas**: Cancelar Conexão.
- **Ações Bloqueadas**: Executar Macro, Abrir Viewer 3D.
- **Eventos de Transição**: `LOGIN_SUCCESS` -> Transiciona para **Online** | `BAD_CREDENTIALS` / `KICKED` -> Transiciona para **Erro**.

#### 4. Online
- **Significado**: O bot está totalmente autenticado, spawnado no mundo do jogo e pronto para receber comandos operacionais.
- **Ações Permitidas**: Desconectar, Executar Macro, Iniciar Minerador, Abrir Viewer 3D, Enviar Chat, Inspecionar Inventário.
- **Ações Bloqueadas**: Editar Credenciais da conta ativa, Iniciar Conexão (já conectado).
- **Eventos de Transição**: `START_MACRO` -> Transiciona para **Executando Macro** | `CMD_DISCONNECT` -> Transiciona para **Desconectando** | `KICKED_BY_SERVER` -> Transiciona para **Erro**.

#### 5. Executando Macro
- **Significado**: O bot está online e possui uma thread ativa executando um script de automação em JavaScript.
- **Ações Permitidas**: Parar Macro, Desconectar, Abrir Viewer 3D, Monitorar Logs do Script.
- **Ações Bloqueadas**: Iniciar outra Macro simultânea (sem parar a atual), Editar Parâmetros do Minerador sem pausar.
- **Eventos de Transição**: `MACRO_FINISHED` / `MACRO_STOPPED` -> Retorna para **Online** | `RUNTIME_ERROR` / `KICKED` -> Transiciona para **Erro**.

#### 6. Desconectando
- **Significado**: Transição temporária indicando que o comando de encerramento de sessão foi enviado e o socket está sendo fechado de forma limpa.
- **Ações Permitidas**: Nenhuma (aguardando confirmação de encerramento).
- **Ações Bloqueadas**: Todas as ações operacionais.
- **Eventos de Transição**: `SOCKET_CLOSED` -> Retorna para **Desconectado**.

#### 7. Erro
- **Significado**: O bot foi desconectado involuntariamente devido a erro de credencial, banimento, timeout de socket ou rejeição do servidor.
- **Ações Permitidas**: Reconectar, Remover da Sessão, Visualizar Log do Erro, Alterar Proxy.
- **Ações Bloqueadas**: Executar Macro, Abrir Viewer 3D.
- **Eventos de Transição**: `CMD_RECONNECT` -> Transiciona para **Conectando** | `CMD_REMOVE` -> Removido da memória.

---

## 5. Atualizações em Tempo Real

Para suportar o alto volume de dados sem degradar a performance do navegador, os canais de transporte de dados são distribuídos segundo a seguinte arquitetura:

| Domínio de Estado | Meio de Transporte | Frequência Esperada | Estratégia para Alta Concorrência (300+ Bots) |
|---|---|---|---|
| Status dos Bots (`BotState`) | WebSocket Stream | Eventual (sob alteração) | **Event Batching**: O backend agrupa alterações de status em janelas de 100ms e envia pacotes vetorizados para evitar disparos individuais. |
| Console de Logs (`LogState`) | WebSocket Stream | Altíssima (10 a 100 msgs/s) | **Circular Buffer Local**: Retenção máxima de 2.000 linhas no cliente web com descarte automático de memória e renderização por janela virtualizada. |
| Telemetria do Dashboard (`DashboardState`) | Server-Sent Events (SSE) | Intervalos fixos de 1s | **Metrics Sampling**: O backend envia estatísticas pré-agregadas de CPU/Rede, desonerando o cliente web de calcular médias. |
| Renderização 3D (`ViewerState`) | WebSocket Binário Dedicado | 60 FPS (Chunks e Blocos) | **Binary Protocol Parsing**: Transmissão de blocos em formato binário comprimido com atualização direta em buffers de memória da GPU (VBOs). |
| Inventário e Status (`BotState.Inv`) | WebSocket Stream | Sob evento de alteração | **Partial Mutation**: Envio exclusivo das posições dos slots alterados (ex: slot 36 modificado), evitando re-renderizar a grade inteira. |
| Operações de Longa Duração (`ProxyChecker`) | Server-Sent Events (SSE) | Conforme conclusão do item | **Progress Streaming**: Transmissão do percentual e atualização isolada apenas da linha de proxy testada na tabela. |

---

## 6. Estratégia de Performance

A preservação da fluidez da interface web sob carga extrema é garantida através das seguintes estratégias arquiteturais de gerenciamento de memória e renderização:

### Virtualização de Listas e Tabelas (DOM Virtualization)
- Componentes como `DataTable`, `SidebarBotList` e `ConsoleLogViewer` renderizam no DOM do navegador exclusivamente os elementos visíveis na janela de rolagem do operador (mais uma margem de segurança de 5 itens acima e abaixo).
- Mesmo que existam 1.000 bots cadastrados ou 2.000 linhas de log, o navegador manipula no máximo 30 a 40 elementos físicos na árvore visual.

### Throttling (Limitação de Frequência de Renderização)
- Atualizações de altíssima frequência (como coordenadas de posição ou dados de telemetria) passam por um limitador de taxa (Throttling) configurado para no máximo 60 atualizações por segundo.
- Atualizações que ocorrem em intervalos menores que 16,6 milissegundos são agrupadas e aplicadas no próximo ciclo de renderização (`requestAnimationFrame`).

### Debounce (Atraso Controlado de Entrada)
- Campos de busca dinâmica e filtros em tempo real (`SearchBox`) aplicam um atraso controlado de 250 milissegundos após a última digitação do operador antes de disparar a re-filtragem do estado local.

### Batching (Agrupamento de Ações de Estado)
- Quando múltiplos bots alteram de estado simultaneamente (ex: reconexão em lote de 50 bots), as atualizações são combinadas em uma única mutação em lote no estado local, disparando exatamente **uma** re-renderização na árvore de componentes.

### Isolamento de Componentes (Re-render Isolation)
- O estado de cada linha da tabela de bots é isolado. Alterações de latência (ping) em um bot específico provocam a atualização estrita daquela célula visual, sem re-renderizar as demais 299 linhas da tabela.

### Descarte Programado de Dados Antigos (Garbage Collection Strategy)
- Logs que ultrapassam o limite circular de memória e dados de chunks 3D de janelas fechadas são explicitamente nulos e liberados para coleta de lixo (Garbage Collector) do navegador, mantendo a pegada de memória RAM da aba estabilizada abaixo de 250 MB.

---

## 7. Persistência

A matriz de persistência de dados define o local oficial de armazenamento e ciclo de vida de cada informação do sistema:

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Estratégia de Persistência                       │
└────────────────────────────────────────────────────────────────────────┘
  │
  ├─► Apenas Memória (RAM do Navegador)
  │   ├── Stream de Logs e Chat (LogState)
  │   ├── Chunks e Entidades 3D (ViewerState)
  │   └── Notificações Efêmeras (NotificationState)
  │
  ├─► Navegador (LocalStorage / IndexedDB)
  │   ├── Preferências Visuais (Tema, Limite do Console, Sons)
  │   ├── Token de Autenticação JWT (SessionState)
  │   └── Histórico de Pesquisas e Filtros Aplicados
  │
  └─► Backend e Banco de Dados (Spring Boot / DB)
      ├── Credenciais de Contas e Listas de Proxies
      ├── Scripts de Macros (MacroState)
      ├── Configurações do Minerador e Parâmetros de Auto-Login
      └── Perfis de Servidores e Parâmetros Globais
```

---

## 8. Regras de Consistência

Para evitar estados corrompidos ou inconsistências visuais entre os componentes, a arquitetura obedece às seguintes regras fundamentais:

1. **Fonte Única da Verdade (Single Source of Truth)**: Cada dado possui uma única fonte oficial no frontend. Componentes visuais não mantêm cópias paralelas de estados globais ou de domínio.
2. **Imutabilidade e Mutações por Ações**: Componentes visuais não alteram o estado diretamente. Qualquer modificação deve ocorrer mediante o disparo de Ações explícitas (Actions) que aplicam transformações puras e previsíveis no estado.
3. **Confirmação Obrigatória para Transições Críticas**: Operações que resultem em perda de dados ou desconexão em massa exigem estado de confirmação intermediário (`ModalConfirm`) antes de propagar a ação para o estado global.
4. **Rastreabilidade de Operações Longas**: Tarefas em lote (como o `ProxyChecker`) possuem um objeto de estado dedicado que rastreia o ID da tarefa, total de itens, progresso atual e contadores de sucesso/erro.

---

## 9. Observabilidade

A arquitetura expõe estados visuais padronizados de observabilidade para garantir que o operador sempre compreenda o status interno do sistema:

- **State: Loading**: Indicador de que a aplicação está aguardando a resposta inicial de uma requisição REST ou carregamento de módulo.
- **State: Syncing**: Indicador de que o cliente web está sincronizando alterações locais com o backend Java.
- **State: Reconnecting**: Indicador de que a conexão WebSocket com o servidor caiu e a aplicação está tentando restabelecer o canal de streaming.
- **State: Offline**: Indicador de que a aplicação perdeu a comunicação com o backend e os dados exibidos podem estar desatualizados (read-only mode).
- **State: Processing**: Indicador de que uma tarefa assíncrona em lote está em execução e a barra de progresso está ativa.
- **State: Error**: Indicador de que ocorreu uma falha na transição de estado, acompanhado da mensagem explicativa da exceção no painel visual.
