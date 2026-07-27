# Arquitetura do Frontend React

## Objetivo

Este documento especifica a arquitetura técnica da nova interface gráfica do sistema AdvancedBot.
Projetada para a web utilizando React e Tailwind CSS, a aplicação atuará como um cliente de alta performance,
capaz de gerenciar e monitorar simultaneamente centenas de instâncias de bots conectadas ao backend Java 21 + Spring Boot.

---

## Escopo

A especificação abrange a estrutura global de layout, o catálogo de páginas, o inventário de componentes reutilizáveis,
a topologia de gerenciamento de estados globais, o modelo de comunicação entre componentes, o tratamento de atualizações
em tempo real via streaming de dados e os princípios orientadores de UX. Este documento é estritamente arquitetural e não contém trechos de código.

---

## Sumário

1. [Objetivos da Arquitetura](#objetivos-da-arquitetura)
2. [Layout Global](#layout-global)
3. [Organização das Páginas](#organização-das-páginas)
4. [Organização dos Componentes](#organização-dos-componentes)
5. [Organização dos Estados Globais](#organização-dos-estados-globais)
6. [Comunicação entre Componentes](#comunicação-entre-componentes)
7. [Atualizações em Tempo Real](#atualizações-em-tempo-real)
8. [Princípios de UX](#princípios-de-ux)

---

## Objetivos da Arquitetura

### Princípios

- **Desempenho Extremo**: Garantir renderização contínua a 60 FPS no navegador, mesmo sob alto volume de eventos e logs por segundo.
- **Desacoplamento Completo**: Isolar totalmente a camada de apresentação gráfica da camada de comunicação WebSocket e REST.
- **Design Declarativo**: Construir a interface a partir de componentes funcionais puros driven por estado previsível.
- **Layout Adaptável**: Oferecer suporte nativo a layouts dinâmicos, modos escuros/claros e diferentes dimensões de tela.

### Escalabilidade

- **Suporte a Centenas de Bots**: Utilizar virtualização de listas no DOM (DOM Virtualization) para exibir milhares de elementos sem degradação de memória.
- **Buffering e Throttling de Eventos**: Agrupar atualizações de estado de alta frequência em lotes (batching) antes de disparar re-renderizações.
- **Divisão de Código (Code Splitting)**: Carregar módulos de páginas e ferramentas pesadas (como o visualizador 3D) sob demanda.

### Organização

- **Arquitetura Modular por Recursos (Feature-Based)**: Agrupar componentes, hooks e tipos por domínio funcional.
- **Nomenclatura Padronizada**: Utilizar regras rígidas e consistentes para nomes de arquivos, estados e componentes.

### Reutilização

- **Sistema de Design (Design System) Baseado em Tokens**: Padronizar cores, tipografia, espaçamentos e elevações através das utilidades do Tailwind CSS.
- **Componentes Atômicos Generativos**: Construir botões, tabelas e modais sem dependência direta de regras de negócio.

### Separação de Responsabilidades

- **Camada de Apresentação (View)**: Responsável apenas pela renderização visual e captura de interações do usuário.
- **Camada de Estado (Store)**: Responsável por armazenar, sincronizar e transformar os dados da aplicação.
- **Camada de Transporte (Data Stream Layer)**: Responsável pela conexão com o backend via REST e WebSockets.

---

## Layout Global

A estrutura visual da aplicação é composta pelos seguintes contêineres principais de layout:

### Sidebar (Barra Lateral de Navegação)

- Contêiner vertical fixo localizado à esquerda da tela.
- Contém a navegação primária entre as páginas da aplicação, seletores de workspace e um sumário visual compacto da contagem de bots ativos.

### TopBar (Barra Superior de Status)

- Contêiner horizontal fixo localizado no topo da aplicação.
- Apresenta o status global de conexão com o servidor Spring Boot, métricas consolidadas (CPU, Memória, I/O), seletor de tema e atalhos globais de ação.

### Workspace (Área de Trabalho Principal)

- Área central flexível onde o conteúdo da página selecionada é renderizado.
- Gerencia a exibição das visões de negócios (Dashboard, Lista de Bots, Proxies, Editor de Macros, etc.).

### Drawer (Painel Deslizante Lateral)

- Contêiner sobreposto que desliza a partir da borda direita da tela.
- Utilizado para inspeção detalhada de um bot específico, exibição rápida de inventários ou logs sem sair da página atual.

### Modal (Caixas de Diálogo Sobrepostas)

- Camada de sobreposição centralizada com máscara de fundo (backdrop).
- Utilizada para confirmações críticas, formulários de criação/importação de contas e configurações de prefixo.

### Toast (Notificações Flutuantes)

- Contêiner de avisos temporários posicionado no canto inferior direito.
- Exibe alertas de sucesso, avisos de desconexão de rede e erros de validação.

### Painel Lateral de Bots (`SidebarBotList`)

- Sub-barra retrátil dedicada à listagem contínua dos bots cadastrados na sessão.
- Permite filtro rápido por nome e seleção múltipla para aplicação de ações em lote.

### Console / Painel de Logs

- Painel inferior expansível ou acoplável.
- Renderiza o fluxo de logs do sistema e mensagens do chat em tempo real, com opção de maximização e limpeza de histórico.

### Área Principal

- Sub-região do Workspace onde tabelas de dados, formulários avançados ou o canvas 3D do WebGL são apresentados.

---

## Organização das Páginas

### 1. Página de Dashboard

- **Objetivo**: Fornecer uma visão panorâmica e consolidada da operação de todos os bots em execução.
- **Responsabilidades**: Agregar métricas globais de rede, status de saúde dos bots, alertas recentes e gráficos de tráfego.
- **Informações Apresentadas**: Total de bots conectados/desconectados, taxa de transferência de dados (KB/s), latência média de resposta, consumo de recursos do servidor e log de eventos recentes.
- **Ações Permitidas**: Iniciar todos os bots, pausar reconexões automáticas, disparar reconexão geral e navegar rapidamente para problemas detectados.

### 2. Página de Bots (`Bots`)

- **Objetivo**: Gerenciar individualmente ou em lote todas as instâncias de bots do sistema.
- **Responsabilidades**: Apresentar a listagem detalhada de bots, permitir filtros avançados por servidor ou status e disparar ações individuais.
- **Informações Apresentadas**: Tabela ou grid de cards contendo nome, servidor conectado, latência (ping), posição no mapa, status da saúde e estado da IA (minerando, parado, etc.).
- **Ações Permitidas**: Conectar/Desconectar bot selecionado, abrir inspeção no Drawer, abrir visualizador 3D, remover da sessão e alterar configurações de auto-login.

### 3. Página de Proxies (`Proxy`)

- **Objetivo**: Administrar o pool de servidores proxy utilizados na conexão dos bots.
- **Responsabilidades**: Gerenciar a lista de proxies, controlar a verificação de latência e filtrar proxies operacionais.
- **Informações Apresentadas**: Tabela de proxies (IP, Porta, Tipo: Socks4/Socks5/HTTP, País, Latência em ms, Status de validação).
- **Ações Permitidas**: Importar proxies em lote via texto, disparar validador de latência, remover proxies com alta latência, exportar proxies filtradas e associar proxies a bots específicos.

### 4. Página de Macros (`Macros`)

- **Objetivo**: Desenvolver, testar e atribuir scripts de automação em JavaScript para os bots.
- **Responsabilidades**: Fornecer ambiente para edição de código, exibição de erros sintáticos e controle de execução.
- **Informações Apresentadas**: Lista de scripts salvos, editor de código integrado, console de execução do script e tabela de parâmetros de entrada.
- **Ações Permitidas**: Criar/Editar/Excluir scripts, compilar código, atribuir script a um grupo de bots e iniciar/parar a execução de um macro.

### 5. Página do Minerador (`Minerador`)

- **Objetivo**: Configurar os parâmetros de automação de escavação e coleta de blocos.
- **Responsabilidades**: Definir prioridades de blocos alvo, ações para inventário cheio e raio de atuação.
- **Informações Apresentadas**: Lista priorizada de IDs/Nomes de blocos, raio de busca configurado, estado de seleção automática de ferramentas e regra ativa para inventário cheio (descartar, guardar, rodar comandos).
- **Ações Permitidas**: Reordenar prioridades de blocos via arrastar-e-soltar, alterar raio de escavação, ativar/desativar troca automática de ferramentas e definir lista de comandos customizados.

### 6. Página do Visualizador 3D (`Viewer`)

- **Objetivo**: Renderizar o ambiente tridimensional interativo do mundo Minecraft a partir da visão de um bot selecionado.
- **Responsabilidades**: Gerenciar o canvas WebGL, processar chunks recebidos e capturar entradas de controle de câmera/jogador.
- **Informações Apresentadas**: Viewport 3D do mundo, overlay com coordenadas (X, Y, Z), FPS, bloco apontado, entidades visíveis, inventário flutuante e console de chat integrado.
- **Ações Permitidas**: Alternar entre Câmera Livre (Freecam) e Controle do Jogador, abrir inventário do bot, enviar mensagens no chat do jogo, alternar Killaura/Fly e ajustar raio de renderização visual.

### 7. Página de Spammer (`Spammer`)

- **Objetivo**: Automatizar o envio sequencial ou em lote de mensagens de texto no chat dos servidores.
- **Responsabilidades**: Gerenciar modelos de mensagem, processar tokens dinâmicos e aplicar controle de velocidade.
- **Informações Apresentadas**: Caixa de texto com modelos de mensagens, assistente de autocompletar tokens, contador de caracteres por linha e parâmetros de delay.
- **Ações Permitidas**: Iniciar/Parar envio de mensagens, definir delay entre envios, alternar aplicação de delay por linha ou por lote de bots e salvar modelos de spam.

### 8. Página de Ferramentas (`Ferramentas`)

- **Objetivo**: Agrupar utilitários secundários de validação e checagem de contas.
- **Responsabilidades**: Executar testes em massa de credenciais Mojang e checagem de banimentos em servidores target.
- **Informações Apresentadas**: Formulários de entrada em lote, barras de progresso percentual, tabelas de resultado com status detalhado.
- **Ações Permitidas**: Iniciar/Pausar testes de autenticação Mojang, checar banimentos por IP/Handshake e exportar relatórios de contas válidas/banidas.

### 9. Página de Monitoramento (`Monitoramento`)

- **Objetivo**: Acompanhar o desempenho de infraestrutura da aplicação e consumo de recursos.
- **Responsabilidades**: Renderizar gráficos de dados em tempo real e tabelas de diagnóstico de sistema.
- **Informações Apresentadas**: Gráficos de tráfego de entrada e saída de dados (Bps/Kbps), pacotes enviados/recebidos por segundo, contagem de threads worker/IOCP e consumo de CPU/Memória.
- **Ações Permitidas**: Alterar janela de tempo dos gráficos, disparar limpeza de memória (Garbage Collection) e exportar métricas de performance.

### 10. Página de Configurações (`Configurações`)

- **Objetivo**: Definir preferências globais do sistema e templates padrão de operação.
- **Responsabilidades**: Gerenciar opções de auto-reconexão, templates de login/registro e prefixos de nicks.
- **Informações Apresentadas**: Formulários de parâmetros gerais, ordem de reconexão (lote ou individual), comandos globais de auto-login e histórico de versões.
- **Ações Permitidas**: Salvar configurações globais, resetar padrões e alternar modos de operação de reconexão.

---

## Organização dos Componentes

### 1. `BotCard`
- **Responsabilidade**: Exibir o resumo do status visual de um bot em formato de cartão compacto.
- **Onde será utilizado**: Páginas `Dashboard` e `Bots` (quando no modo de visualização em Grid).
- **Reutilização**: Altamente reutilizável; aceita dados de qualquer objeto de bot e dispara callbacks de ação genéricos.

### 2. `BotStatusBadge`
- **Responsabilidade**: Renderizar uma etiqueta colorida indicando visualmente o estado de conexão do bot (Conectado, Desconectado, Autenticando, Banido).
- **Onde será utilizado**: `BotCard`, `SidebarBotList`, tabelas de bots e cabeçalhos de inspeção.
- **Reutilização**: Componente atômico universal de indicação de status.

### 3. `SidebarBotList`
- **Responsabilidade**: Apresentar uma lista vertical filtrável com virtualização de linhas para navegação contínua entre centenas de bots.
- **Onde será utilizado**: Painel lateral fixo da aplicação e barra de seleção de bots em páginas de ferramentas.
- **Reutilização**: Componente estrutural reutilizado como seletor de alvo em múltiplos cenários.

### 4. `DataTable`
- **Responsabilidade**: Tabela de alta performance com suporte a ordenação por colunas, paginação, seleção múltipla de linhas e virtualização de conteúdo.
- **Onde será utilizado**: Páginas de `Bots`, `Proxy`, `AccountChecker` e `BanCheck`.
- **Reutilização**: Componente base genérico para exibição de qualquer conjunto de dados estruturados em colunas.

### 5. `ProxyTable`
- **Responsabilidade**: Especialização do `DataTable` com renderização customizada para latência (tags verde/amarela/vermelha) e colunas específicas de proxy.
- **Onde será utilizado**: Página de `Proxy` e modal de gerenciamento de rede.
- **Reutilização**: Utilizado em visões onde o gerenciamento detalhado de proxies é requerido.

### 6. `ConsoleLogViewer`
- **Responsabilidade**: Painel de exibição de mensagens de chat e logs de sistema em rolagem contínua, com suporte a cores estilizadas e busca.
- **Onde será utilizado**: Painel de console global do layout e página de `Bots`.
- **Reutilização**: Componente reutilizável para qualquer exibição de fluxo contínuo de logs de texto.

### 7. `InventoryGrid`
- **Responsabilidade**: Renderizar graficamente a grade do inventário do bot (slots, quantidades de itens e durabilidade de ferramentas).
- **Onde será utilizado**: Overlay do `Viewer 3D` e Drawer de inspeção individual de bots.
- **Reutilização**: Componente de apresentação do estado de inventário do Minecraft.

### 8. `PlayerStatusOverlay`
- **Responsabilidade**: Apresentar os dados de telemetria da câmera e posição do jogador (coordenadas X, Y, Z, FPS, direção e vida) sobrepostos na tela.
- **Onde será utilizado**: Página do `Viewer 3D`.
- **Reutilização**: Exclusivo para visualizações interativas em 3D.

### 9. `ModalConfirm`
- **Responsabilidade**: Exibir caixas de diálogo modais para confirmação de ações de alto impacto (ex: remover todas as contas, desconectar lote).
- **Onde será utilizado**: Globalmente em todas as páginas da aplicação.
- **Reutilização**: Componente atômico de confirmação de segurança.

### 10. `ToolbarActionGroup`
- **Responsabilidade**: Agrupar botões de ação frequente (Adicionar, Remover, Iniciar, Parar, Exportar) com estilo visual padronizado.
- **Onde será utilizado**: Topo de páginas de gerenciamento e cabeçalhos de tabelas.
- **Reutilização**: Estrutura atômica de agrupamento de botões de ação.

### 11. `SearchBox`
- **Responsabilidade**: Campo de entrada de texto com suporte a busca instantânea com debounce para filtragem de listas de alta densidade.
- **Onde será utilizado**: `SidebarBotList`, `DataTable` e seletores de ferramentas.
- **Reutilização**: Componente atômico universal de pesquisa.

### 12. `FilterDropdown`
- **Responsabilidade**: Menu suspenso para seleção de filtros múltiplos por categorias (ex: filtrar por versão, tipo de proxy ou servidor).
- **Onde será utilizado**: Cabeçalhos de tabelas de `Bots` e `Proxies`.
- **Reutilização**: Componente atômico reutilizável de filtragem de dados.

### 13. `NumberInput`
- **Responsabilidade**: Controle de entrada numérica com botões incrementais e decrementais de precisão e validação de faixa mínima/máxima.
- **Onde será utilizado**: Páginas `Start`, `Spammer`, `Minerador` e diálogos de configuração.
- **Reutilização**: Componente atômico universal de entrada numérica.

---

## Organização dos Estados Globais

### 1. `BotsState`
- **Quem produz**: Camada de transporte WebSocket (eventos de login, disconnect, alteração de vida ou movimento vindos do backend Java).
- **Quem consome**: Páginas `Dashboard`, `Bots`, `Viewer 3D`, componentes `SidebarBotList` e `BotCard`.
- **Ciclo de Vida**: Mantido durante toda a sessão ativa da aplicação web. Atualizado continuamente via streaming.

### 2. `LogsState`
- **Quem produz**: Mensagens de chat recebidas do servidor Minecraft e logs do sistema emitidos pelo backend.
- **Quem consome**: Componentes `ConsoleLogViewer`, overlays de chat no `Viewer 3D` e painel de inspeção.
- **Ciclo de Vida**: Histórico mantido em memória até o limite pré-configurado de linhas (buffer circular).

### 3. `NetworkState`
- **Quem produz**: Telemetria periódica do cliente REST/WebSocket referente ao consumo de tráfego (bps) e pacotes por segundo.
- **Quem consome**: Página de `Monitoramento` e indicadores da `TopBar`.
- **Ciclo de Vida**: Atualizado a cada intervalo de amostragem de dados de desempenho (ex: a cada 1 segundo).

### 4. `ConfigState`
- **Quem produz**: Formulários das páginas de `Configurações`, `FrmLogin`, `SetPrefixForm` ou carregamento de arquivo de estado.
- **Quem consome**: Motores de conexão de bots, geradores de nicks e tratadores de auto-reconexão.
- **Ciclo de Vida**: Persistido no armazenamento local do navegador (LocalStorage) ou sincronizado no banco de dados via REST API.

### 5. `ProxyState`
- **Quem produz**: Respostas das APIs de proxy e resultados em tempo real do `ProxyChecker`.
- **Quem consome**: Página de `Proxy`, formulários de conexão inicial e seletores de proxy.
- **Ciclo de Vida**: Carregado na inicialização e modificado durante verificações ou importações em lote.

### 6. `MacrosState`
- **Quem produz**: Editor de código da página `Macros` e atualizações de estado do interpretador de scripts Java.
- **Quem consome**: Página `Macros` e painéis de execução de scripts de bots.
- **Ciclo de Vida**: Persistido via API backend e mantido em memória durante a edição e execução.

### 7. `ViewerState`
- **Quem produz**: Interações do usuário no canvas WebGL (ajustes de visão, modo freecam ativo) e atualizações de chunks da API.
- **Quem consome**: Página `Viewer 3D` e overlays do renderizador visual.
- **Ciclo de Vida**: Instanciado ao abrir o visualizador 3D de um bot e destruído ao fechar o canvas para liberar memória de vídeo.

### 8. `SessionState`
- **Quem produz**: Serviço de autenticação e gerenciador de conexões com o backend Spring Boot.
- **Quem consome**: Toda a aplicação (define autorização e status de conexão com o servidor de controle).
- **Ciclo de Vida**: Ativo enquanto o operador estiver autenticado na aplicação web.

### 9. `NotificationState`
- **Quem produz**: Exceções de rede, confirmações de ação e alertas de sistema.
- **Quem consome**: Componente global `Toast`.
- **Ciclo de Vida**: Efêmero; as notificações são automaticamente removidas após um tempo de exibição.

---

## Comunicação entre Componentes

A transferência de dados entre as telas e componentes da interface React é organizada segundo as seguintes diretrizes arquiteturais:

### Fluxo Unidirecional de Dados (Top-Down)

- O estado reside em contêineres superiores (Stores Globais ou Contextos de Recurso).
- Os componentes visuais da camada de apresentação recebem dados exclusivamente via propriedades (Props) de leitura.
- Alterações visuais ou disparos de ação ocorrem invocando funções de callback repassadas aos componentes filhos.

### Camada de Abstração via Hooks Customizados

- A comunicação entre a interface visual e os dados é mediada por Hooks Customizados de domínio (ex: `useBotList`, `useProxyManager`, `useRealtimeLogs`).
- Os componentes visuais não conhecem os detalhes do protocolo de transporte (REST/WebSocket), apenas consomem os retornos expostos pelos Hooks.

### Barramento de Eventos de Desempenho (Event Bus) para Streams de Alta Frequência

- Para fluxos de dados extremamente rápidos, como pacotes do visualizador 3D ou logs massivos, a comunicação utiliza um padrão de assinatura de eventos (Pub/Sub) local.
- O componente `ConsoleLogViewer` subscreve-se diretamente ao stream de logs, evitando acionar a re-renderização da árvore inteira de componentes da aplicação.

---

## Atualizações em Tempo Real

A tabela a seguir especifica os setores da interface que requerem atualização automática e o comportamento esperado:

| Setor da Interface | Fonte dos Dados | Frequência | Comportamento Esperado |
|---|---|---|---|
| Lista de Bots (`SidebarBotList` / `Bots`) | Stream WebSocket | Contínua (baseada em eventos) | Atualizar badges de status e latência em tempo real sem reordenar bruscamente a lista durante a navegação. |
| Console de Logs e Chat (`ConsoleLogViewer`) | Stream WebSocket | Alta frequência (múltiplos eventos/s) | Inserção contínua de linhas no final do console com rolagem automática (Auto-scroll), caso o usuário não tenha rolado manualmente para cima. |
| Visualizador 3D (`Viewer`) | Stream WebGL / Chunks WebSocket | 30 a 60 FPS | Atualização contínua de frames da cena 3D, posição da câmera, entidades visíveis e blocos destruídos/colocados. |
| Métricas de Desempenho (`Monitoramento` / `TopBar`) | Polling HTTP ou SSE | Intervalos fixos (1s) | Atualização suave de pontos nos gráficos de tráfego de rede e barras de uso de CPU/Memória. |
| Status de Validação de Proxies (`ProxyTable`) | Stream de progresso do backend | Conforme conclusão do teste | Atualização individual da linha correspondente à proxy testada com nova latência e alteração da barra de progresso. |
| Inventário do Bot (`InventoryGrid`) | Evento de alteração de inventário | Sob alteração | Atualização imediata dos itens no slot correspondente assim que o bot coletar, mover ou descartar um item. |

---

## Princípios de UX

A nova interface React deve ser desenvolvida em estrita conformidade com os seguintes princípios de Experiência do Usuário (UX):

- **Feedback Imediato**: Toda ação disparada pelo operador (como clicar em conectar, excluir proxy ou executar macro) deve fornecer resposta visual instantânea (estados de carregamento, spinners ou alterações de cor) em menos de 100 ms, mesmo que o processamento no backend demore mais tempo.
- **Carregamento Progressivo (Skeleton Screens)**: Ao abrir páginas com alto volume de informações (como o Dashboard ou a Lista de Bots), a interface deve apresentar estruturas esqueleto (skeletons) antes da chegada dos dados completos, evitando saltos de layout.
- **Ações Agrupadas e em Lote**: Ações repetitivas (como reconectar, aplicar proxies ou alterar rotinas) devem ser disponibilizadas em barras de ferramentas superiores para execução coletiva sobre os itens selecionados.
- **Minimização de Clics**: Operações comuns da rotina de gerenciamento devem ser acessíveis com no máximo 2 cliques a partir da página principal.
- **Consistência Visual Sistemática**: Todos os estados de foco, hover, seleção e alerta devem obedecer rigorosamente às variáveis do Design System em Tailwind CSS, garantindo uniformidade entre todas as páginas.
- **Navegação Intuitiva e Atalhos de Teclado**: O operador deve ser capaz de navegar entre bot instâncias, abrir a visualização 3D ou alternar logs utilizando combinações simples de atalhos de teclado (shortcuts).
