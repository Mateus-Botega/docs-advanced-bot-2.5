# Especificação do Design System

## Objetivo

Este documento define a especificação arquitetural do Design System oficial para a nova interface do sistema AdvancedBot.
Ele estabelece os padrões visuais, o catálogo de componentes (atômicos, moleculares e organizacionais), os estados visuais,
as regras de feedback e os contratos de composição que orientarão o desenvolvimento do frontend em React.

---

## Escopo

A especificação se aplica a toda a interface gráfica do sistema. Este documento não contém código de implementação,
tags HTML, classes utilitárias de CSS ou componentes React. Trata-se de uma especificação funcional e arquitetural
destinada a garantir consistência visual, reutilização total e desacoplamento de regras de negócio.

---

## Sumário

1. [Princípios do Design System](#1-princípios-do-design-system)
2. [Componentes Atômicos](#2-componentes-atômicos)
3. [Componentes Moleculares](#3-componentes-moleculares)
4. [Componentes Organizacionais](#4-componentes-organizacionais)
5. [Estados Visuais](#5-estados-visuais)
6. [Feedback Visual](#6-feedback-visual)
7. [Regras Gerais](#7-regras-gerais)

---

## 1. Princípios do Design System

### Consistência

Todas as páginas, ferramentas e modais da aplicação devem utilizar rigorosamente os mesmos tokens visuais,
padrões de espaçamento, tipografia e comportamentos interativos. Elementos com a mesma função técnica devem ter
a mesma aparência e o mesmo modo de operação em qualquer parte do sistema.

### Reutilização

Nenhum elemento visual deve ser construído de forma ad-hoc ou duplicado. Todo componente deve ser desenhado para
ser universalmente genérico e composável, recebendo suas variações exclusivamente por parâmetros de propriedade.

### Acessibilidade

Todos os componentes interativos devem oferecer contraste visual adequado, suporte integral à navegação via teclado
(uso de teclas Tab, Enter, Espaço e Setas), rótulos explicativos claros e indicadores visuais de foco visíveis.

### Escalabilidade

O sistema de componentes deve suportar o crescimento da aplicação sem a necessidade de reescrever estruturas existentes.
Novas funcionalidades devem ser construídas reutilizando os componentes atômicos e moleculares já estabelecidos.

### Performance

Os componentes visuais devem ser otimizados para evitar re-renderizações desnecessárias no navegador. Componentes de alta
densidade de dados (como tabelas e listas de bots) devem utilizar técnicas de virtualização e renderização sob demanda.

### Feedback Visual

Toda e qualquer ação disparada pelo operador ou alteração de estado vinda do backend deve gerar uma resposta visual
imediata e inequívoca na interface em menos de 100 milissegundos.

### Responsividade

A interface deve se adaptar graciosamente a diferentes resoluções de tela e dimensões de viewport, reorganizando
painéis laterais e grids sem perda de funcionalidade ou sobreposição de conteúdo.

---

## 2. Componentes Atômicos

### Button

- **Objetivo**: Permitir a execução de uma ação direta pelo usuário.
- **Responsabilidade**: Capturar o comando de clique e apresentar o estado da ação.
- **Estados**: Normal, Hovered, Focused, Active, Disabled, Loading.
- **Variantes**: Primary (ação principal), Secondary (ação alternativa), Danger (destrutiva), Ghost (discreta), Icon-only (apenas ícone).
- **Quando utilizar**: Para disparar formulários, abrir modais, conectar bots ou salvar preferências.
- **Quando NÃO utilizar**: Para links de navegação simples entre páginas (utilizar componente de navegação).
- **Regras de comportamento**: Deve exibir Spinner e bloquear novos cliques quando no estado Loading.

### Input

- **Objetivo**: Coleta de textos em linha única.
- **Responsabilidade**: Capturar entradas alfanuméricas e exibir mensagens de validação.
- **Estados**: Normal, Hovered, Focused, Disabled, Error, Success.
- **Variantes**: Standard, WithIcon, Clearable.
- **Quando utilizar**: Para nomes de usuário, senhas, endereços IP e nomes de arquivos.
- **Quando NÃO utilizar**: Para entradas numéricas puras (usar NumberInput) ou textos multilinha (usar Textarea).
- **Regras de comportamento**: Deve destacar a borda no estado Error e exibir o texto explicativo do erro logo abaixo do campo.

### NumberInput

- **Objetivo**: Entrada de valores numéricos com controle de precisão.
- **Responsabilidade**: Garantir a digitação de números válidos dentro de uma faixa permitida.
- **Estados**: Normal, Hovered, Focused, Disabled, Error.
- **Variantes**: Standard (com botões de incremento/decremento), Compact.
- **Quando utilizar**: Para delays de conexão, raios de escavação, limites de FPS e portas de servidor.
- **Quando NÃO utilizar**: Para inserção de textos genéricos ou credenciais.
- **Regras de comportamento**: Bloquear automaticamente valores fora da faixa mínima e máxima configuradas.

### Textarea

- **Objetivo**: Inserção de blocos de texto com múltiplas linhas.
- **Responsabilidade**: Capturar listas cruas de dados ou textos longos.
- **Estados**: Normal, Hovered, Focused, Disabled, Error.
- **Variantes**: Standard, FixedHeight, Resizable.
- **Quando utilizar**: Para colagem de listas de contas (user:pass), listas de proxies ou modelos de mensagens.
- **Quando NÃO utilizar**: Para entradas de linha única ou códigos estruturados (usar CodeEditor).
- **Regras de comportamento**: Exibir contador opcional de linhas e caracteres no canto inferior.

### Checkbox

- **Objetivo**: Seleção binária (verdadeiro/falso) ou seleção múltipla de itens.
- **Responsabilidade**: Alternar e indicar graficamente a marcação de uma opção.
- **Estados**: Unchecked, Checked, Indeterminate, Disabled, Focused.
- **Variantes**: Standard, CardCheckbox.
- **Quando utilizar**: Para opções de física, ignorar ping, seleção de linhas em tabelas.
- **Quando NÃO utilizar**: Para alternância de estados imediatos de sistema (usar Switch).
- **Regras de comportamento**: O estado Indeterminate deve ser utilizado apenas quando um grupo pai possui seleção parcial de filhos.

### Switch

- **Objetivo**: Alternância imediata de um estado operacional (liga/desliga).
- **Responsabilidade**: Disparar a alteração imediata de uma funcionalidade sem depender do envio de um formulário.
- **Estados**: Off, On, Disabled, Focused, Loading.
- **Variantes**: Standard, WithLabel.
- **Quando utilizar**: Para ativar/desativar KillAura, Fly, Auto-Reconnect ou modo escuro.
- **Quando NÃO utilizar**: Para escolhas dentro de um formulário que exige confirmação no botão "Salvar".
- **Regras de comportamento**: Transição de animação suave e indicação visual imediata do novo estado.

### Radio

- **Objetivo**: Seleção exclusiva de uma única opção dentro de um conjunto pré-definido.
- **Responsabilidade**: Apresentar opções mutuamente exclusivas.
- **Estados**: Unselected, Selected, Disabled, Focused.
- **Variantes**: Standard, RadioGroup.
- **Quando utilizar**: Para escolha do modo de validação de proxy (Ping vs. Login).
- **Quando NÃO utilizar**: Quando múltiplas seleções forem permitidas (usar Checkbox).
- **Regras de comportamento**: Apresentar sempre uma opção selecionada por padrão caso o campo seja obrigatório.

### Select

- **Objetivo**: Seleção de uma opção a partir de uma lista suspensa de alternativas.
- **Responsabilidade**: Economizar espaço em tela permitindo a escolha em um menu retrátil.
- **Estados**: Closed, Opened, Disabled, Error, Focused.
- **Variantes**: SingleSelect, MultiSelect, SearchableSelect.
- **Quando utilizar**: Para escolha da versão do Minecraft, país da proxy ou ação para inventário cheio.
- **Quando NÃO utilizar**: Quando houver menos de 3 opções (usar Radio).
- **Regras de comportamento**: Fechar o menu suspenso ao pressionar Escape ou clicar fora da área.

### SearchBox

- **Objetivo**: Campo dedicado para filtragem dinâmica de listas e tabelas.
- **Responsabilidade**: Capturar o termo de pesquisa e disparar filtragens com controle de limite de chamadas (debounce).
- **Estados**: Normal, Focused, Searching, HasContent.
- **Variantes**: Compact, FullWidth.
- **Quando utilizar**: Para pesquisar bots por nome na barra lateral ou filtrar proxies na tabela.
- **Quando NÃO utilizar**: Para submeter dados para o servidor (usar Input tradicional com Button).
- **Regras de comportamento**: Apresentar ícone de lupa e botão para limpar busca quando houver texto digitado.

### Badge

- **Objetivo**: Indicador visual compacto para contagens ou categorias de baixa prioridade.
- **Responsabilidade**: Destacar quantitativos de itens ou marcas categorizadas.
- **Estados**: Normal.
- **Variantes**: Default, Primary, Success, Warning, Danger, Neutral.
- **Quando utilizar**: Para mostrar a quantidade de bots ativos no menu ou contagem de proxies selecionadas.
- **Quando NÃO utilizar**: Para mensagens longas explicativas (usar Toast ou Tooltip).
- **Regras de comportamento**: Exibir valores truncados quando o número for muito elevado (ex: +999).

### Tag

- **Objetivo**: Rotular ou classificar um item com suporte a remoção rápida.
- **Responsabilidade**: Representar atributos associados a uma entidade.
- **Estados**: Normal, Hovered, Removable.
- **Variantes**: Solid, Outlined.
- **Quando utilizar**: Para exibir blocos prioritários selecionados no minerador ou tipos de proxies aplicados.
- **Quando NÃO utilizar**: Para indicadores de contagem pura (usar Badge).
- **Regras de comportamento**: Exibir botão de fechamento (X) caso seja configurada como removível.

### Chip

- **Objetivo**: Elemento interativo compacto para seleção rápida de filtros rápidos.
- **Responsabilidade**: Atuar como um botão de alternância de estado de filtro.
- **Estados**: Unselected, Selected, Disabled.
- **Variantes**: FilterChip, ChoiceChip.
- **Quando utilizar**: Para filtrar bots rapidamente por status (Todos, Conectados, Erro).
- **Quando NÃO utilizar**: Para ações de envio de formulário (usar Button).
- **Regras de comportamento**: Mudar a cor de fundo visivelmente ao ser selecionado.

### Tooltip

- **Objetivo**: Exibir texto explicativo contextual flutuante ao passar o ponteiro do mouse sobre um elemento.
- **Responsabilidade**: Fornecer ajuda e esclarecer ícones ou botões sem texto.
- **Estados**: Hidden, Visible.
- **Variantes**: Top, Bottom, Left, Right.
- **Quando utilizar**: Para explicar a função de botões apenas com ícones ou siglas de estatísticas.
- **Quando NÃO utilizar**: Para informações essenciais para a execução da tarefa (devem estar visíveis no layout).
- **Regras de comportamento**: Exibir após um breve atraso ao passar o mouse e esconder imediatamente ao afastar.

### Spinner

- **Objetivo**: Indicador visual de carregamento indeterminado em andamento.
- **Responsabilidade**: Informar visualmente que o sistema está processando uma solicitação.
- **Estados**: Spinning.
- **Variantes**: Small, Medium, Large, OverlaySpinner.
- **Quando utilizar**: Dentro de botões em estado de carregamento ou no centro de painéis aguardando dados.
- **Quando NÃO utilizar**: Quando for possível mensurar o percentual de conclusão (usar ProgressBar).
- **Regras de comportamento**: Animação de rotação contínua e suave.

### Divider

- **Objetivo**: Separação de seções ou grupos de elementos visuais.
- **Responsabilidade**: Organizar a hierarquia visual do layout sem poluição gráfica.
- **Estados**: Normal.
- **Variantes**: Horizontal, Vertical.
- **Quando utilizar**: Para dividir grupos de botões na Toolbar ou separar seções dentro de um formulário.
- **Quando NÃO utilizar**: Como elemento puramente decorativo sem função de agrupar/separar.
- **Regras de comportamento**: Manter espessura e opacidade discretas.

### Icon

- **Objetivo**: Representação gráfica simbólica de uma ação, conceito ou entidade.
- **Responsabilidade**: Facilitar a identificação visual rápida de ações e status.
- **Estados**: Normal, Active, Disabled.
- **Variantes**: Small, Medium, Large.
- **Quando utilizar**: Acompanhando rótulos em botões, menus e indicadores de status.
- **Quando NÃO utilizar**: De forma isolada sem Tooltip quando o significado for ambíguo.
- **Regras de comportamento**: Utilizar sempre o mesmo conjunto de ícones (icon set) padronizado em toda a aplicação.

### Avatar

- **Objetivo**: Representação gráfica do perfil visual de um bot ou usuário.
- **Responsabilidade**: Exibir a cabeça de skin ou as iniciais do bot.
- **Estados**: Normal, Offline, Online, Busy.
- **Variantes**: Circle, RoundedSquare.
- **Quando utilizar**: Na lista de bots, no header do visualizador 3D e no Drawer de inspeção.
- **Quando NÃO utilizar**: Em tabelas de dados de alta densidade onde o espaço de linha for crítico.
- **Regras de comportamento**: Exibir imagem padrão genérica caso a skin não possa ser carregada.

### Toast

- **Objetivo**: Mensagem de notificação flutuante de curta duração.
- **Responsabilidade**: Alertar o usuário sobre eventos assíncronos do sistema.
- **Estados**: Entering, Visible, Exiting.
- **Variantes**: Success, Error, Warning, Info.
- **Quando utilizar**: Para avisar sobre salvamento de configurações, desconexões bruscas ou erros de validação.
- **Quando NÃO utilizar**: Para confirmações que exigem escolha imediata do usuário (usar ModalConfirm).
- **Regras de comportamento**: Desaparecer automaticamente após alguns segundos ou permitir fechamento manual.

### ProgressBar

- **Objetivo**: Indicador visual do progresso percentual de uma tarefa determinável.
- **Responsabilidade**: Demonstrar o avanço de um processo longo.
- **Estados**: Normal, Completed, Error.
- **Variantes**: Standard, Striped.
- **Quando utilizar**: Em validações de lista de proxies e checagem de contas Mojang.
- **Quando NÃO utilizar**: Para operações instantâneas ou sem duração previsível (usar Spinner).
- **Regras de comportamento**: Exibir o texto com a porcentagem exata atualizada continuamente.

---

## 3. Componentes Moleculares

### Toolbar

- **Objetivo**: Agrupar botões de ação e ferramentas de controle em uma barra horizontal organizadora.
- **Estrutura**: Contêiner horizontal flexível contendo botões, separadores e seletores.
- **Composição**: Composta por `Button`, `Divider`, `Select` e `Icon`.
- **Responsabilidades**: Oferecer atalhos para as ações principais da tela atual.
- **Reutilização**: Utilizada no topo do `DataTable`, do editor de macros e do painel de controle de bots.

### SearchToolbar

- **Objetivo**: Unir busca rápida, filtros e ações em uma única estrutura coesa.
- **Estrutura**: Barra contendo caixa de pesquisa à esquerda e ações/filtros à direita.
- **Composição**: Composta por `SearchBox`, `FilterDropdown`, `Chip` e `Button`.
- **Responsabilidades**: Facilitar a consulta e filtragem de grandes volumes de dados.
- **Reutilização**: Cabeçalho de gerenciamento de tabelas de bots, proxies e logs.

### FilterBar

- **Objetivo**: Apresentar um painel horizontal com múltiplos critérios de filtragem ativos.
- **Estrutura**: Linha contendo chips de filtros aplicados e botão para limpar todos os filtros.
- **Composição**: Composta por `Chip`, `Badge` e `Button` (ghost).
- **Responsabilidades**: Exibir claramente quais restrições de busca estão afetando os dados exibidos.
- **Reutilização**: Posicionada logo abaixo das barras de busca em páginas de dados.

### Pagination

- **Objetivo**: Controle de navegação entre páginas de dados divididas.
- **Estrutura**: Barra inferior contendo seletor de itens por página, contagem total e botões de navegação.
- **Composição**: Composta por `Button`, `Select` e rótulos de texto.
- **Responsabilidades**: Permitir a navegação fluida em grandes conjuntos de dados sem sobrecarregar a tela.
- **Reutilização**: Rodapé padrão de todas as tabelas do sistema (`DataTable`, `ProxyTable`).

### Tabs

- **Objetivo**: Organizar o conteúdo de uma mesma página em painéis alternáveis por abas.
- **Estrutura**: Barra de cabeçalhos de abas na parte superior e área de exibição de conteúdo correspondente.
- **Composição**: Composta por botões de abas, `Badge` opcional e `Divider`.
- **Responsabilidades**: Reduzir a poluição visual dividindo formulários complexos em seções temáticas.
- **Reutilização**: Utilizada em diálogos de configuração do minerador, opções de proxy e ferramentas.

### Accordion

- **Objetivo**: Painel retrátil empilhado para ocultar ou expandir seções secundárias de conteúdo.
- **Estrutura**: Cabeçalho clicável com ícone indicador de estado e painel de conteúdo ocultável.
- **Composição**: Composta por título de seção, `Icon` e container flexível.
- **Responsabilidades**: Permitir que o operador expanda apenas os detalhes que deseja analisar.
- **Reutilização**: Utilizado na listagem de logs avançados e grupos de opções secundárias.

### Modal

- **Objetivo**: Caixa de diálogo sobreposta para tarefas focadas ou confirmações.
- **Estrutura**: Máscara de fundo (backdrop), cabeçalho com título e botão fechar, corpo de conteúdo e rodapé de ações.
- **Composição**: Composta por máscara, contêiner flutuante, `Button` e títulos.
- **Responsabilidades**: Isolar a atenção do operador para a tomada de uma decisão sem mudar de página.
- **Reutilização**: Estrutura base para `ModalConfirm`, inserção de contas e configurações de prefixo.

### Drawer

- **Objetivo**: Painel lateral deslizante sobreposto para inspeção detalhada de um item.
- **Estrutura**: Contêiner vertical posicionado na lateral direita com animação de deslizamento.
- **Composição**: Composta por cabeçalho com ação de fechar, corpo com rolagem e rodapé de atalhos.
- **Responsabilidades**: Exibir o status completo, inventário ou logs de um bot sem perder a visão da tabela.
- **Reutilização**: Utilizado para inspeção rápida de bots, proxies e detalhes de execuções.

### Card

- **Objetivo**: Contêiner visual delimitado para agrupar informações relacionadas sobre um tema.
- **Estrutura**: Painel com borda, sombra discreta e preenchimento interno padronizado.
- **Composição**: Composto por cabeçalho, corpo e rodapé opcional.
- **Responsabilidades**: Isolar blocos de conteúdo na tela.
- **Reutilização**: Estrutura base para métricas, cards de bots e gráficos de monitoramento.

### MetricCard

- **Objetivo**: Exibir um indicador numérico chave de desempenho com rótulo e variação visual.
- **Estrutura**: Card compacto contendo título do indicador, valor destacado, ícone explicativo e indicador de tendência.
- **Composição**: Composto por `Card`, `Icon`, `Badge` e texto destacado.
- **Responsabilidades**: Apresentar instantaneamente métricas como total de bots, consumo de banda e latência.
- **Reutilização**: Utilizado na parte superior do `Dashboard` e da página de `Monitoramento`.

### BotCard

- **Objetivo**: Apresentar as informações resumidas de um bot individual em formato de card.
- **Estrutura**: Card com avatar da skin, nome do bot, servidor conectado, latência e botão de menu de ações.
- **Composição**: Composto por `Card`, `Avatar`, `BotStatusBadge`, `Button` e `Icon`.
- **Responsabilidades**: Permitir a visualização rápida e disparo de comandos individuais quando no modo grid.
- **Reutilização**: Utilizado na visualização em Grid da página de `Bots`.

### StatusCard

- **Objetivo**: Apresentar um status descritivo detalhado com cor temática de fundo.
- **Estrutura**: Card de destaque contendo ícone lateral, mensagem descritiva e ação recomendada.
- **Composição**: Composto por `Card`, `Icon` e `Button`.
- **Responsabilidades**: Informar ao operador sobre desconexões críticas ou perda de comunicação com o backend.
- **Reutilização**: Exibido no topo de páginas quando ocorrem erros operacionais globais.

### LogConsole

- **Objetivo**: Renderizar fluxos contínuos de logs e mensagens de chat em estilo terminal.
- **Estrutura**: Contêiner de fundo escuro com rolagem vertical automática, linhas coloridas por severidade e barra de busca.
- **Composição**: Composto por `Card`, `SearchBox`, `Button` (limpeza/auto-scroll) e linhas de texto estilizadas.
- **Responsabilidades**: Exibir em tempo real as mensagens enviadas e recebidas pelos bots.
- **Reutilização**: Utilizado no painel inferior global e no overlay do `Viewer 3D`.

### DataTable

- **Objetivo**: Renderizador de tabelas de alta performance para exibição de dados complexos.
- **Estrutura**: Cabeçalho de colunas com ordenação, corpo de linhas com seleção e rodapé de paginação.
- **Composição**: Composta por `Toolbar`, `Checkbox` (seleção de linha), `Pagination` e linhas interativas.
- **Responsabilidades**: Apresentar conjuntos massivos de dados com virtualização de linhas.
- **Reutilização**: Tabela universal utilizada para exibição de listas de bots e dados genéricos.

### ProxyTable

- **Objetivo**: Especialização da `DataTable` para exibição e gerenciamento de servidores proxy.
- **Estrutura**: Estrutura base da `DataTable` com colunas especializadas para latência (ping), tipo e país.
- **Composição**: Composta por `DataTable`, `Badge` (latência), `Tag` (tipo) e `Button` (ações de teste).
- **Responsabilidades**: Apresentar as proxies cadastradas e seus respectivos status de validação em tempo real.
- **Reutilização**: Utilizada na página de `Proxy` e nos modais de seleção de rede.

### InventoryGrid

- **Objetivo**: Representação gráfica da grade de inventário do personagem Minecraft.
- **Estrutura**: Matriz de slots quadrados representando o inventário principal, armaduras e barra rápida.
- **Composição**: Composta por slots individuais contendo ícone do item, quantidade e barra de durabilidade.
- **Responsabilidades**: Exibir o estado exato dos itens carregados pelo bot selecionado.
- **Reutilização**: Utilizada no Drawer de inspeção de bots e no overlay do `Viewer 3D`.

### CodeEditor

- **Objetivo**: Editor de texto avançado para criação e modificação de scripts de automação.
- **Estrutura**: Área de edição de código com numeração de linhas, destaque de sintaxe e barra de ferramentas.
- **Composição**: Composta por `Toolbar`, área de edição e `LogConsole` secundário para erros de compilação.
- **Responsabilidades**: Fornecer um ambiente focado para o desenvolvimento de macros JavaScript.
- **Reutilização**: Componente único utilizado na página de `Macros`.

### SidebarBotList

- **Objetivo**: Lista vertical compacta de bots para seleção contínua no layout lateral.
- **Estrutura**: Painel vertical contendo campo de busca superior e lista virtualizada de itens de bots.
- **Composição**: Composta por `SearchBox`, `Avatar`, `BotStatusBadge` e linhas de lista selecionáveis.
- **Responsabilidades**: Permitir que o operador selecione e altere o bot em foco a qualquer momento.
- **Reutilização**: Posicionada na barra lateral retrátil do layout principal.

### TopBar

- **Objetivo**: Barra superior de controle e informações globais do sistema.
- **Estrutura**: Faixa horizontal contendo logo, status do servidor backend, métricas de tráfego e ações globais.
- **Composição**: Composta por `MetricCard` compacto, `BotStatusBadge`, `Button` e `Avatar`.
- **Responsabilidades**: Manter os indicadores de saúde do sistema sempre visíveis.
- **Reutilização**: Componente de layout fixo no topo de todas as páginas.

### NavigationMenu

- **Objetivo**: Menu de links para alternância entre as páginas da aplicação.
- **Estrutura**: Lista de itens de navegação com ícones e rótulos indicadores de estado ativo.
- **Composição**: Composta por links interativos contendo `Icon` e `Badge` indicador de contagem.
- **Responsabilidades**: Permitir a navegação rápida entre os módulos do sistema.
- **Reutilização**: Posicionado dentro da `Sidebar` principal.

### CommandPalette

- **Objetivo**: Menu de busca rápida de comandos acionado via atalho de teclado (ex: Ctrl+K).
- **Estrutura**: Modal centralizado contendo campo de busca imediato e lista de ações executáveis.
- **Composição**: Composta por `SearchBox`, lista de atalhos e rótulos de comandos.
- **Responsabilidades**: Permitir que operadores avançados executem ações sem usar o mouse.
- **Reutilização**: Modal global acessível a partir de qualquer ponto da aplicação.

---

## 4. Componentes Organizacionais

### DashboardLayout

- **Composição**: Composto por `TopBar`, `Sidebar`, `Workspace` contendo grid de `MetricCard` e `DataTable` de bots.
- **Responsabilidades**: Organizar a visão consolidada de métricas globais e status de operação em uma tela de alto nível.
- **Reutilização**: Utilizado exclusivamente na rota principal da página de Dashboard.

### PageLayout

- **Composição**: Composto por cabeçalho da página (título, descrição e `ToolbarActionGroup`) e área principal de conteúdo.
- **Responsabilidades**: Fornecer a estrutura padrão para exibição de qualquer página de conteúdo da aplicação.
- **Reutilização**: Layout base utilizado pelas páginas de `Bots`, `Proxy`, `Macros`, `Minerador`, `Spammer` e `Configurações`.

### FormLayout

- **Composição**: Composto por estrutura centralizada dividida em seções temáticas (`Tabs` ou seções separadas por `Divider`).
- **Responsabilidades**: Organizar a apresentação de formulários longos garantindo legibilidade e fácil preenchimento.
- **Reutilização**: Utilizado na construção de telas de configurações complexas e preferências do sistema.

### ConsoleLayout

- **Composição**: Composto por painel superior de controles de filtro (`SearchToolbar`) e corpo preenchido pelo `LogConsole`.
- **Responsabilidades**: Maximizar a área de visualização de logs para depuração detalhada de eventos e mensagens de chat.
- **Reutilização**: Utilizado na visão expandida de monitoramento de logs e chat.

### ViewerLayout

- **Composição**: Composto por canvas 3D em tela cheia com overlays flutuantes (`PlayerStatusOverlay`, `InventoryGrid`, `LogConsole`).
- **Responsabilidades**: Prover a organização das camadas gráficas do visualizador 3D sobre o canvas WebGL.
- **Reutilização**: Utilizado exclusivamente na janela de renderização 3D do mundo do jogo.

### SettingsLayout

- **Composição**: Composto por menu lateral interno de navegação entre categorias de configuração e painel central de formulário.
- **Responsabilidades**: Agrupar e organizar o grande volume de parâmetros editáveis do sistema.
- **Reutilização**: Utilizado na página de `Configurações` globais e preferências de automação.

### Workspace

- **Composição**: Região central da aplicação que envolve dinamicamente o `PageLayout` da rota ativa.
- **Responsabilidades**: Gerenciar o preenchimento de tela, barras de rolagem principais e transições entre rotas.
- **Reutilização**: Componente estrutural permanente do layout da aplicação.

### Sidebar

- **Composição**: Contêiner lateral fixo que abriga o logo do sistema, o `NavigationMenu` e o resumo de status da sessão.
- **Responsabilidades**: Garantir a estrutura de navegação primária sempre visível ou retrátil na esquerda.
- **Reutilização**: Componente estrutural permanente do layout da aplicação.

### TopBar

- **Composição**: Faixa horizontal superior que contém os indicadores globais de rede, métricas do backend e ações globais.
- **Responsabilidades**: Manter a barra de ferramentas superior fixa independente da navegação do Workspace.
- **Reutilização**: Componente estrutural permanente do layout da aplicação.

### Footer

- **Composição**: Faixa horizontal discreta na base contendo versão do sistema, status do servidor e atalhos de ajuda.
- **Responsabilidades**: Exibir dados utilitários de rodapé e status de copyright/versão sem poluição visual.
- **Reutilização**: Componente posicionado na base das páginas do sistema.

---

## 5. Estados Visuais

Todos os componentes do Design System devem possuir representação visual clara para os seguintes estados:

- **Loading**: O componente está aguardando o carregamento de dados. Deve exibir `Spinner` ou efeito de esqueleto visual (skeleton).
- **Disabled**: O componente está visível, porém bloqueado para interação do usuário devido a regras de permissão ou dependências não atendidas.
- **Focused**: O componente recebeu o foco do teclado. Deve apresentar um anel de destaque visual (focus ring) bem definido.
- **Hovered**: O ponteiro do mouse está sobre o componente. Deve sofrer uma leve alteração de cor ou elevação para indicar interatividade.
- **Selected**: O componente foi marcado ou escolhido dentro de um grupo de opções (ex: linha de tabela ou card selecionado).
- **Error**: O campo ou componente contém dados inválidos ou ocorreu uma falha na operação. Deve exibir borda vermelha e texto explicativo.
- **Warning**: O componente está em uma situação de atenção que não impede a operação (ex: latência alta ou limite de memória próximo).
- **Success**: A operação associada ao componente foi concluída com êxito (ex: proxy validado ou alteração salva).
- **Empty**: O componente de exibição de lista ou tabela não possui registros para apresentar. Deve exibir uma ilustração discreta e texto explicativo.
- **Processing**: O componente disparou uma ação assíncrona que está em execução no backend Java.
- **Offline**: Indicador de que a entidade (bot ou servidor) está desconectada ou sem comunicação de rede.
- **Online**: Indicador de que a entidade está conectada, autenticada e operando normalmente.

---

## 6. Feedback Visual

 A interface deve responder às interações e mudanças de estado segundo as seguintes regras de feedback:

### Carregamentos

- Carregamentos rápidos (< 300 ms) não devem exibir spinners para evitar oscilações visuais (flicker).
- Carregamentos moderados devem utilizar `Spinner` integrado dentro do botão ou componente afetado.
- Carregamentos de páginas inteiras devem utilizar estrutura de esqueleto visual (`Skeleton Screen`) mantendo o layout estável.

### Sucesso

- Operações bem-sucedidas de salvamento ou execução devem exibir uma notificação `Toast` verde no canto inferior direito por 3 segundos.
- Atualizações de dados em tabelas devem piscar suavemente a linha alterada em tom verde para destacar a mudança.

### Erro

- Erros de formulário devem destacar imediatamente o campo com estado `Error` e focar o primeiro campo inválido.
- Falhas de conexão com o backend devem exibir um `StatusCard` de erro na parte superior da página afetada.

### Confirmações

- Ações destrutivas (como remover todas as contas ou desconectar todos os bots) exigem confirmação explícita através do `ModalConfirm`.
- O botão de confirmação dentro do modal destrutivo deve utilizar obrigatoriamente a variante `Danger`.

### Operações Longas

- Processos que exigem execução demorada em lote (como validação de proxies ou checagem de contas) devem exibir uma `ProgressBar` destacada.
- Durante a operação longa, os controles que alteram os parâmetros do teste devem ser temporariamente colocados em estado `Disabled`.

### Atualizações em Tempo Real

- Alterações de status vindas do WebSocket devem atualizar suavemente os dados na tela sem provocar o reposicionamento repentino da rolagem.
- Novas linhas inseridas no `ConsoleLogViewer` devem acompanhar a rolagem automática a menos que o usuário tenha rolado para analisar o histórico.

---

## 7. Regras Gerais

Para garantir a integridade da arquitetura, o desenvolvimento do frontend React deverá obedecer às seguintes regras rígidas:

1. **Isolamento de Regras de Negócio**: Nenhum componente atômico, molecular ou organizacional do Design System deve conter código de regra de negócio, chamadas HTTP diretas ou lógicas de protocolo de rede.
2. **Reutilização Estrita**: Nenhuma página do sistema pode criar componentes visuais ad-hoc ou duplicar estruturas já existentes no Design System.
3. **Padronização de Tabelas**: Todas as exibições de dados em formato de grade do sistema devem utilizar exclusivamente a estrutura base do `DataTable`.
4. **Padronização de Formulários**: Todos os formulários da aplicação devem ser construídos utilizando estritamente os componentes atômicos de entrada (`Input`, `NumberInput`, `Select`, `Checkbox`, `Switch`).
5. **Padronização de Modais**: Toda caixa de diálogo da aplicação deve derivar da estrutura base do componente molecular `Modal`.
