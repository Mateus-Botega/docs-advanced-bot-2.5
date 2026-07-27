# Especificação Funcional dos Fluxos do Usuário

## 1. Introdução

Este documento especifica detalhadamente os fluxos de navegação e operação da interface do sistema AdvancedBot.
Ele descreve a experiência do operador do início ao fim, estabelecendo o comportamento funcional esperado para cada
interação, independentemente da tecnologia de implementação utilizada.

Todas as descrições fundamentam-se na análise funcional do sistema legado em C# (documento 01), na arquitetura do
frontend (documento 02) e no Design System (documento 03). Quaisquer aprimoramentos de experiência de usuário em relação
ao sistema legado estão explicitamente sinalizados como "Melhoria Proposta". Este documento não contém trechos de código.

---

## 2. Fluxos Principais

### 2.1 Fluxo: Inicializar Aplicação

- **Objetivo**: Carregar a interface do sistema, estabelecer a sessão com o backend Spring Boot e apresentar a página inicial.
- **Pré-condições**: Backend Java em execução e acessível na rede.
- **Passo a passo da interação**:
  1. O operador acessa a URL da aplicação web no navegador.
  2. A interface exibe a tela de carregamento inicial (splash/skeleton).
  3. O sistema valida o token de sessão e estabelece o canal WebSocket.
  4. O painel `DashboardLayout` é renderizado apresentando a `TopBar`, `Sidebar` e as métricas do sistema.
- **Componentes envolvidos**: `DashboardLayout`, `TopBar`, `Sidebar`, `Spinner`, `Toast`.
- **Estados da interface**: Loading (durante conexão), Online (após carregamento completo).
- **Validações**: Verificar acessibilidade da REST API e handshake do WebSocket.
- **Comunicação esperada com o backend**: GET /api/v1/system/status e WS /ws/events.
- **Atualizações em tempo real**: Início do recebimento da telemetria global.
- **Resultado esperado**: Interface carregada na página Dashboard com indicação de backend conectado.
- **Possíveis erros**: Backend inacessível ou falha de autenticação.
- **Recuperação de erro**: Exibir modal de erro com botão de reconexão manual.
- **Feedback visual esperado**: Transição suave do estado Loading para o Dashboard com notificação Toast "Sistema Conectado".
- **Observações de UX**: (Melhoria Proposta) Preservar a última rota navegada pelo operador no LocalStorage.

---

### 2.2 Fluxo: Conectar um Bot

- **Objetivo**: Conectar uma única conta de bot a um servidor Minecraft específico.
- **Pré-condições**: Credenciais da conta válidas e endereço do servidor definido.
- **Passo a passo da interação**:
  1. O operador navega até a página de `Bots` ou clica em "Adicionar Bot" na `Sidebar`.
  2. Abre o modal ou formulário de conexão.
  3. Preenche o nick/conta, endereço do servidor, versão do Minecraft e clica em "Conectar".
  4. O bot é inserido na lista de bots com status "Autenticando".
  5. Após o login, o status altera para "Online".
- **Componentes envolvidos**: `PageLayout`, `Input`, `Select`, `Button`, `BotStatusBadge`, `SidebarBotList`.
- **Estados da interface**: Normal -> Processing -> Online.
- **Validações**: Endereço IP/Host preenchido, conta não vazia, versão válida.
- **Comunicação esperada com o backend**: POST /api/v1/bots/connect.
- **Atualizações em tempo real**: Alteração de status do bot na `SidebarBotList` e chegada de logs no `ConsoleLogViewer`.
- **Resultado esperado**: Bot conectado com sucesso no servidor e visível na lista ativa.
- **Possíveis erros**: Credenciais inválidas, servidor indisponível, timeout.
- **Recuperação de erro**: O status do bot muda para "Error" e a causa do erro é gravada no console de logs.
- **Feedback visual esperado**: Badge do bot altera de amarelo (autenticando) para verde (online).
- **Observações de UX**: Permitir acionar a conexão ao pressionar a tecla Enter dentro do formulário.

---

### 2.3 Fluxo: Conectar Vários Bots

- **Objetivo**: Iniciar a conexão sequencial ou paralela de um lote de bots.
- **Pré-condições**: Lista de contas colada no campo de texto e parâmetros de delay configurados.
- **Passo a passo da interação**:
  1. O operador acessa a tela `Start` / Formulário de Conexão em Lote.
  2. Cola o texto contendo múltiplas contas (formato `user:pass` ou nicks gerados).
  3. Configura o delay entre conexões (ex: 1500 ms) e conexões simultâneas máximas.
  4. Clica no botão "Conectar Lote".
  5. A interface inicia a fila de conexão exibindo a barra de progresso.
- **Componentes envolvidos**: `Textarea`, `NumberInput`, `Button`, `ProgressBar`, `SidebarBotList`.
- **Estados da interface**: Processing (em progresso de lote).
- **Validações**: Verificar se há pelo menos uma linha de conta válida digitada.
- **Comunicação esperada com o backend**: POST /api/v1/bots/connect-batch.
- **Atualizações em tempo real**: Inserção progressiva de novos bots na `SidebarBotList` conforme cada conexão é estabelecida.
- **Resultado esperado**: Todos os bots do lote inseridos e tentando conexão de acordo com o delay configurado.
- **Possíveis erros**: Erros parciais de conexão devido a bloqueio de IP.
- **Recuperação de erro**: Os bots com falha ficam em status "Error", enquanto os demais continuam conectando.
- **Feedback visual esperado**: Preenchimento contínuo da `ProgressBar` e Toast "Lote de Bots Iniciado".
- **Observações de UX**: (Melhoria Proposta) Botão para pausar ou cancelar a fila de conexões pendentes a qualquer momento.

---

### 2.4 Fluxo: Desconectar Bots

- **Objetivo**: Encerrar a sessão de um bot individual ou de um grupo de bots.
- **Pré-condições**: Pelo menos um bot conectado ou em processo de conexão.
- **Passo a passo da interação**:
  1. O operador seleciona um ou mais bots na `DataTable` ou `SidebarBotList`.
  2. Clica no botão "Desconectar" na `Toolbar`.
  3. O sistema solicita confirmação se a seleção for em lote via `ModalConfirm`.
  4. Após confirmação, a solicitação de encerramento é enviada ao backend.
- **Componentes envolvidos**: `DataTable`, `Toolbar`, `ModalConfirm`, `Button`, `BotStatusBadge`.
- **Estados da interface**: Selected -> Processing -> Offline.
- **Validações**: Verificar se os bots selecionados estão em estado ativo ou conectando.
- **Comunicação esperada com o backend**: POST /api/v1/bots/disconnect.
- **Atualizações em tempo real**: Remoção ou alteração do badge para "Offline" e registro de desconexão nos logs.
- **Resultado esperado**: Sessões TCP encerradas no backend e bots atualizados para desconectados.
- **Possíveis erros**: Falha ao comunicar comando de desconexão.
- **Recuperação de erro**: Tentar desconexão forçada via timeout local.
- **Feedback visual esperado**: Toast informativo "10 Bots Desconectados com Sucesso".
- **Observações de UX**: Permitir desconectar com atalho de teclado quando a linha estiver selecionada.

---

### 2.5 Fluxo: Reconectar Bots

- **Objetivo**: Forçar a reconexão imediata de bots que foram desconectados ou caíram.
- **Pré-condições**: Bots cadastrados na sessão no estado "Offline" ou "Error".
- **Passo a passo da interação**:
  1. O operador filtra os bots pelo status "Desconectados".
  2. Seleciona os bots desejados e clica em "Reconectar" na `Toolbar`.
  3. O backend reinicia o processo de handshake respeitando as configurações de delay.
- **Componentes envolvidos**: `FilterBar`, `DataTable`, `Toolbar`, `Button`, `BotStatusBadge`.
- **Estados da interface**: Offline -> Processing -> Online.
- **Validações**: Garantir que apenas bots offline sejam submetidos à reconexão.
- **Comunicação esperada com o backend**: POST /api/v1/bots/reconnect.
- **Atualizações em tempo real**: Status dos bots altera para "Autenticando" e subsequentemente "Online".
- **Resultado esperado**: Bots reconectados ao servidor sem necessidade de digitar credenciais novamente.
- **Possíveis erros**: Servidor offline ou conta banida.
- **Recuperação de erro**: Marcação do bot com o erro específico retornado.
- **Feedback visual esperado**: Transição de cor do badge e Toast "Reconexão Iniciada".
- **Observações de UX**: Opção para marcar "Auto-Reconectar" diretamente na barra de ferramentas.

---

### 2.6 Fluxo: Adicionar Contas

- **Objetivo**: Cadastrar novas credenciais de contas no sistema para uso futuro.
- **Pré-condições**: Estar na tela de formulário de conexão ou configurações de contas.
- **Passo a passo da interação**:
  1. O operador acessa o formulário de cadastro de contas.
  2. Digita o nome do usuário e senha (se aplicável).
  3. Clica em "Salvar Conta".
- **Componentes envolvidos**: `FormLayout`, `Input`, `Button`, `Toast`.
- **Estados da interface**: Normal -> Success.
- **Validações**: Validar sintaxe do nome do usuário (sem caracteres inválidos de Minecraft).
- **Comunicação esperada com o backend**: POST /api/v1/accounts.
- **Atualizações em tempo real**: Atualização do catálogo de contas disponíveis.
- **Resultado esperado**: Conta armazenada com sucesso no perfil do operador.
- **Possíveis erros**: Conta duplicada.
- **Recuperação de erro**: Destacar o campo de texto com mensagem "Conta já cadastrada".
- **Feedback visual esperado**: Borda verde temporária e Toast de confirmação.
- **Observações de UX**: Suporte a geração automática de nicks com prefixos na própria tela de inserção.

---

### 2.7 Fluxo: Remover Contas

- **Objetivo**: Excluir credenciais cadastradas da lista de contas salvas.
- **Pré-condições**: Contas existentes salvas no sistema.
- **Passo a passo da interação**:
  1. O operador seleciona as contas desejadas na lista.
  2. Clica no ícone de lixeira / botão "Remover".
  3. Confirma a exclusão no `ModalConfirm`.
- **Componentes envolvidos**: `DataTable`, `ModalConfirm`, `Button`.
- **Estados da interface**: Selected -> Processing -> Normal.
- **Validações**: Verificar se a conta não está em uso por um bot conectado no momento.
- **Comunicação esperada com o backend**: DELETE /api/v1/accounts.
- **Atualizações em tempo real**: Remoção instantânea da linha na tabela.
- **Resultado esperado**: Registro da conta removido permanentemente.
- **Possíveis erros**: Tentativa de deletar conta com bot ativo.
- **Recuperação de erro**: Exibir aviso "Desconecte o bot antes de remover a conta".
- **Feedback visual esperado**: Linha da tabela desaparece com animação de fade-out.
- **Observações de UX**: Permitir desfazer a ação (Undo) através da notificação Toast por 5 segundos.

---

### 2.8 Fluxo: Importar Contas

- **Objetivo**: Carregar em massa credenciais a partir de um arquivo de texto (.txt).
- **Pré-condições**: Arquivo .txt formatado contendo credenciais por linha.
- **Passo a passo da interação**:
  1. O operador clica no botão "Importar Arquivo".
  2. Seleciona o arquivo local no diálogo do sistema operacional.
  3. A interface faz o parse do arquivo e exibe a contagem de contas válidas encontradas.
  4. O operador clica em "Confirmar Importação".
- **Componentes envolvidos**: `Button`, `Modal`, `LogConsole`, `ProgressBar`.
- **Estados da interface**: Loading -> Success.
- **Validações**: Filtrar linhas vazias ou formatos inválidos durante o parse.
- **Comunicação esperada com o backend**: POST /api/v1/accounts/import (Multipart/Form-Data).
- **Atualizações em tempo real**: Barra de progresso da leitura do arquivo.
- **Resultado esperado**: Contas válidas importadas e prontas para conexão.
- **Possíveis erros**: Arquivo corrompido ou formato incompatível.
- **Recuperação de erro**: Exibir modal informando o número de linhas ignoradas.
- **Feedback visual esperado**: Toast "150 Contas Importadas com Sucesso".
- **Observações de UX**: Permitir recurso de arrastar-e-soltar (Drag and Drop) do arquivo direto na área.

---

### 2.9 Fluxo: Configurar Servidor

- **Objetivo**: Definir o IP/Domínio, Porta e Versão do servidor Minecraft alvo.
- **Pré-condições**: Nenhuma.
- **Passo a passo da interação**:
  1. O operador acessa os campos de parâmetro de servidor na página `Start` ou `Configurações`.
  2. Digita o endereço (ex: `jogar.servidor.com:25565`).
  3. Seleciona a versão no `Select` (ex: `1.5.2` ou `1.8`).
  4. As alterações são salvas automaticamente ou ao clicar em "Aplicar".
- **Componentes envolvidos**: `Input`, `Select`, `Button`, `Card`.
- **Estados da interface**: Focused -> Success.
- **Validações**: Validar sintaxe de host/IP e porta numérica.
- **Comunicação esperada com o backend**: PUT /api/v1/config/server.
- **Atualizações em tempo real**: Atualização do cabeçalho de status da aplicação.
- **Resultado esperado**: Parâmetros de servidor gravados como padrão para novas conexões.
- **Possíveis erros**: Porta inválida ou endereço mal formatado.
- **Recuperação de erro**: Exibir borda vermelha e texto de erro no campo de entrada.
- **Feedback visual esperado**: Ícone de check verde ao lado do campo.
- **Observações de UX**: Resolução automática de DNS para exibir o IP final resolvido em Tooltip.

---

### 2.10 Fluxo: Configurar Login Automático

- **Objetivo**: Definir os comandos de autenticação automáticos enviados ao entrar em servidores piratas.
- **Pré-condições**: Nenhuma.
- **Passo a passo da interação**:
  1. O operador abre a ferramenta / modal de `Auto-Login`.
  2. Define a máscara do comando de login (ex: `/login @pass`).
  3. Define a máscara do comando de registro (ex: `/register @pass @pass`).
  4. Clica em "Salvar Comandos".
- **Componentes envolvidos**: `Modal`, `Input`, `Button`, `Toast`.
- **Estados da interface**: Normal -> Success.
- **Validações**: Garantir que as máscaras contenham os tokens de substituição apropriados (`@pass`).
- **Comunicação esperada com o backend**: PUT /api/v1/config/auto-login.
- **Atualizações em tempo real**: Nenhuma.
- **Resultado esperado**: Bots enviarão os comandos formatados automaticamente ao detectar a mensagem de login.
- **Possíveis erros**: Comandos vazios ou sem parâmetros.
- **Recuperação de erro**: Alerta visual solicitando a inclusão do token `@pass`.
- **Feedback visual esperado**: Toast de confirmação "Comandos de Auto-Login Atualizados".
- **Observações de UX**: Exibir prévia (preview) em tempo real do comando final que será enviado ao servidor.

---

### 2.11 Fluxo: Configurar Proxies

- **Objetivo**: Gerenciar a lista de proxies HTTP/Socks a serem utilizadas na rota de conexão dos bots.
- **Local de Execução**: Página de `Proxy`.
- **Passo a passo da interação**:
  1. O operador acessa a página `Proxy`.
  2. Clica em "Adicionar Proxies" para abrir a caixa de inserção em lote.
  3. Cola a lista de proxies (`IP:Porta` ou `IP:Porta:User:Pass`).
  4. Seleciona o tipo de proxy (Socks4, Socks5, HTTP) no `RadioGroup`.
  5. Clica em "Salvar Proxies".
- **Componentes envolvidos**: `ProxyTable`, `Textarea`, `Radio`, `Button`, `Toolbar`.
- **Estados da interface**: Normal -> Processing -> Success.
- **Validações**: Verificar sintaxe de IP e portas válidas (1-65535).
- **Comunicação esperada com o backend**: POST /api/v1/proxies.
- **Atualizações em tempo real**: Atualização imediata da `ProxyTable`.
- **Resultado esperado**: Lista de proxies armazenada e pronta para uso e testes.
- **Possíveis erros**: Proxies duplicadas ou sintaxe incorreta.
- **Recuperação de erro**: O sistema destaca as linhas com falha de parse no editor.
- **Feedback visual esperado**: Atualização dos quantitativos nos cards de resumo de proxies.
- **Observações de UX**: (Melhoria Proposta) Identificação e rotulagem automática do país de origem do IP via GeoIP.

---

### 2.12 Fluxo: Validar Proxies

- **Objetivo**: Testar a latência e o status operacional das proxies cadastradas.
- **Pré-condições**: Proxies cadastradas no sistema.
- **Passo a passo da interação**:
  1. O operador acessa a página `Proxy` ou abre a ferramenta `ProxyChecker`.
  2. Seleciona o modo de teste: "Ping" (handshake) ou "Login" (teste completo).
  3. Clica em "Iniciar Validação".
  4. A interface exibe o avanço da verificação na `ProgressBar` e atualiza as latências em tempo real na `ProxyTable`.
  5. Ao final, clica em "Remover Inválidas".
- **Componentes envolvidos**: `ProxyTable`, `Button`, `ProgressBar`, `Radio`, `Badge`.
- **Estados da interface**: Processing (durante os testes).
- **Validações**: Bloquear edição da lista durante a execução do teste.
- **Comunicação esperada com o backend**: POST /api/v1/proxies/check.
- **Atualizações em tempo real**: Atualização da cor da latência (verde/amarela/vermelha) de cada linha conforme resposta.
- **Resultado esperado**: Proxies testadas com latências atualizadas e inválidas filtradas.
- **Possíveis erros**: Timeout generalizado devido a falha de internet local.
- **Recuperação de erro**: Botão para cancelar o teste e preservar os dados obtidos até o momento.
- **Feedback visual esperado**: Animação na `ProgressBar` e badges de latência coloridos.
- **Observações de UX**: Opção para auto-remover proxies com latência acima de X ms ao final do teste.

---

### 2.13 Fluxo: Executar Macro

- **Objetivo**: Iniciar a execução de um script de automação em JavaScript para bots selecionados.
- **Pré-condições**: Script de macro compilado sem erros e bots conectados.
- **Passo a passo da interação**:
  1. O operador acessa a página `Macros`.
  2. Seleciona o script desejado na lista.
  3. Escolha os bots alvo na `SidebarBotList` ou seleciona "Todos".
  4. Clica no botão "Executar Macro".
- **Componentes envolvidos**: `CodeEditor`, `SidebarBotList`, `Button`, `LogConsole`, `Badge`.
- **Estados da interface**: Normal -> Processing -> Running.
- **Validações**: Verificar se o código não possui erros de sintaxe antes do envio.
- **Comunicação esperada com o backend**: POST /api/v1/macros/execute.
- **Atualizações em tempo real**: Início do fluxo de logs do script no `LogConsole` e indicação de macro ativa nos bots.
- **Resultado esperado**: Script iniciado e rodando no motor de macros Java do backend.
- **Possíveis erros**: Erro de execução em tempo de execução (runtime exception no JS).
- **Recuperação de erro**: Exibir linha e erro no `CodeEditor` e alterar status da macro para "Error".
- **Feedback visual esperado**: Ícone de "Play" verde no badge do bot e Toast "Macro Iniciada".
- **Observações de UX**: Exibição da variável de saída ou retorno de funções em painel lateral do editor.

---

### 2.14 Fluxo: Parar Macro

- **Objetivo**: Interromper a execução de um script de macro ativo nos bots.
- **Pré-condições**: Pelo menos um bot executando uma macro.
- **Passo a passo da interação**:
  1. O operador clica no botão "Parar Macro" na `Toolbar` da página `Macros` ou no painel do bot.
  2. O backend interrompe a thread de execução do script.
- **Componentes envolvidos**: `Button`, `Toolbar`, `Badge`, `LogConsole`.
- **Estados da interface**: Running -> Normal.
- **Validações**: Confirmar interrupção de macros em lote se afetar múltiplos bots.
- **Comunicação esperada com o backend**: POST /api/v1/macros/stop.
- **Atualizações em tempo real**: Log de "Macro Interrompida pelo Operador" e remoção do badge de macro ativa.
- **Resultado esperado**: Script parado imediatamente e bot retornando ao estado ocioso.
- **Possíveis erros**: Macro presa em loop infinito no backend.
- **Recuperação de erro**: O backend força a interrupção da thread virtual do bot.
- **Feedback visual esperado**: Remoção do indicador de execução e Toast "Macro Parada".
- **Observações de UX**: Atalho rápido na `TopBar` para parar todas as macros ativas do sistema.

---

### 2.15 Fluxo: Abrir Viewer 3D

- **Objetivo**: Inicializar a renderização tridimensional do mundo do jogo a partir da visão de um bot selecionado.
- **Pré-condições**: Bot selecionado conectado no estado "Online".
- **Passo a passo da interação**:
  1. O operador clica duas vezes na linha do bot ou seleciona "Visualizar 3D" no menu de contexto.
  2. A página `Viewer` é carregada ou um canvas dedicado é expandido.
  3. O backend envia os chunks de blocos ao redor do bot via WebSocket.
  4. O canvas WebGL renderiza o ambiente 3D, mundo, entidades e o overlay de telemetria.
- **Componentes envolvidos**: `ViewerLayout`, `PlayerStatusOverlay`, `InventoryGrid`, `LogConsole`, `Button`.
- **Estados da interface**: Loading -> Active (Canvas WebGL ativo).
- **Validações**: Verificar suporte a WebGL no navegador do operador.
- **Comunicação esperada com o backend**: WS /ws/viewer/{botId}.
- **Atualizações em tempo real**: Renderização a 60 FPS da cena 3D, movimento do jogador e alteração do mapa de blocos.
- **Resultado esperado**: Mundo 3D exibido com precisão e controle de câmera interativo.
- **Possíveis erros**: Falha ao inicializar contexto WebGL ou estouro de memória de vídeo.
- **Recuperação de erro**: Exibir aviso "Modo 3D indisponível. Alternando para visualização 2D/Telemetria".
- **Feedback visual esperado**: Transição fluida do esqueleto para a renderização do mundo.
- **Observações de UX**: Painel retrátil de controles de atalho exibido na primeira abertura.

---

### 2.16 Fluxo: Alternar FreeCam

- **Objetivo**: Alternar entre o controle/visão centrada no jogador e a câmera livre voadora no ambiente 3D.
- **Pré-condições**: Visualizador 3D aberto e ativo.
- **Passo a passo da interação**:
  1. O operador pressiona a tecla de atalho `F6` ou clica no botão "FreeCam" na barra de ferramentas do Viewer.
  2. O foco da câmera se desacopla do boneco do bot.
  3. O operador utiliza as teclas W, A, S, D, Espaço e Shift para voar livremente pelo mapa.
  4. Pressiona `F6` novamente para acoplar a visão de volta ao bot.
- **Componentes envolvidos**: `ViewerLayout`, `PlayerStatusOverlay`, `Button`, `Switch`.
- **Estados da interface**: PlayerCam -> FreeCam.
- **Validações**: Nenhuma.
- **Comunicação esperada com o backend**: Nenhuma (a física da FreeCam é calculada localmente no cliente WebGL).
- **Atualizações em tempo real**: Atualização instantânea das coordenadas da câmera no overlay de depuração.
- **Resultado esperado**: Navegação de câmera livre através de blocos e entidades.
- **Possíveis erros**: Perda de foco do teclado.
- **Recuperação de erro**: Re-focar o canvas WebGL ao clicar com o mouse sobre ele.
- **Feedback visual esperado**: Alteração do indicador no overlay para "Modo: Câmera Livre (FreeCam)".
- **Observações de UX**: Exibir linha guia visual ou marcador indicando onde o corpo do bot permaneceu parado.

---

### 2.17 Fluxo: Alterar Configurações do Minerador

- **Objetivo**: Definir os parâmetros funcionais da automação de mineração (blocos alvo, raio e inventário cheio).
- **Local de Execução**: Página `Minerador` ou modal de opções do minerador.
- **Passo a passo da interação**:
  1. O operador acessa a página `Minerador`.
  2. Adiciona ou reordena a lista de blocos prioritários (ex: Diamante > Ouro > Ferro).
  3. Configura o raio de escavação no `NumberInput` (ex: 5 blocos).
  4. Marca a opção "Trocar ferramenta automaticamente".
  5. Define a ação ao encher inventário para "Executar comandos" e digita `/home` na caixa de texto.
  6. Clica em "Salvar Parâmetros".
- **Componentes envolvidos**: `PageLayout`, `NumberInput`, `Checkbox`, `Select`, `Textarea`, `Button`, `Tabs`.
- **Estados da interface**: Normal -> Success.
- **Validações**: Raio dentro do limite permitido (1 a 16 blocos).
- **Comunicação esperada com o backend**: PUT /api/v1/config/miner.
- **Atualizações em tempo real**: Envio dos novos parâmetros para a thread de IA do minerador.
- **Resultado esperado**: Bot passa a minerar seguindo rigorosamente as novas regras salvas.
- **Possíveis erros**: Lista de blocos alvo vazia com minerador ativo.
- **Recuperação de erro**: Alerta visual solicitando a inclusão de pelo menos um bloco alvo.
- **Feedback visual esperado**: Toast de confirmação e marcação de sucesso na aba.
- **Observações de UX**: Recurso de arrastar-e-soltar (Drag and Drop) para reordenar a prioridade visual dos blocos.

---

### 2.18 Fluxo: Enviar Mensagens

- **Objetivo**: Enviar uma mensagem manual de chat através de um bot selecionado ou de todos os bots.
- **Pré-condições**: Bot conectado e campo de digitação disponível no console.
- **Passo a passo da interação**:
  1. O operador digita a mensagem no campo de chat do `ConsoleLogViewer` ou da tela `Main`.
  2. Seleciona se a mensagem será enviada pelo bot focado ou por todos os bots conectados.
  3. Pressiona Enter ou clica no botão "Enviar".
- **Componentes envolvidos**: `ConsoleLogViewer`, `Input`, `Button`, `Select`.
- **Estados da interface**: Normal -> Processing -> Normal.
- **Validações**: Limitar o texto a no máximo 100 caracteres (`MaxLength = 100`).
- **Comunicação esperada com o backend**: POST /api/v1/bots/chat.
- **Atualizações em tempo real**: A mensagem enviada aparece imediatamente no histórico do `ConsoleLogViewer`.
- **Resultado esperado**: Pacote de chat enviado ao servidor Minecraft e mensagem visível para outros jogadores.
- **Possíveis erros**: Tentativa de enviar mensagem com o bot desconectado ou caracteres proibidos.
- **Recuperação de erro**: Notificação no console "Falha ao enviar: Bot desconectado".
- **Feedback visual esperado**: Limpeza automática do campo de texto após o envio bem-sucedido.
- **Observações de UX**: Histórico de mensagens enviadas navegável através das teclas de seta Para Cima / Para Baixo.

---

### 2.19 Fluxo: Utilizar Spammer

- **Objetivo**: Configurar e disparar o envio automatizado e repetitivo de mensagens de chat com variáveis dinâmicas.
- **Local de Execução**: Página `Spammer`.
- **Passo a passo da interação**:
  1. O operador acessa a página `Spammer`.
  2. Digita o modelo de mensagem utilizando a sintaxe de tokens (ex: `Olá {rand,09,4}!`).
  3. Utiliza o assistente de autocompletar ao digitar barras ou chaves.
  4. Configura o intervalo de envio em milissegundos (ex: 5000 ms).
  5. Clica em "Iniciar Spammer".
- **Componentes envolvidos**: `PageLayout`, `Textarea`, `NumberInput`, `Button`, `Switch`, `Badge`.
- **Estados da interface**: Normal -> Running.
- **Validações**: Garantir delay mínimo configurado para evitar kick imediato por spam no servidor.
- **Comunicação esperada com o backend**: POST /api/v1/spammer/start.
- **Atualizações em tempo real**: Exibição do contador de mensagens enviadas e visualização dos envios no `ConsoleLogViewer`.
- **Resultado esperado**: Spammer rodando em segundo plano e disparando frases dinamizadas nos intervalos definidos.
- **Possíveis erros**: Sintaxe de token dinâmico malformatada.
- **Recuperação de erro**: Destaque visual da linha de texto que contém o erro de token.
- **Feedback visual esperado**: Indicador de status "Spammer Rodando" em destaque no painel.
- **Observações de UX**: Lista suspensa de sugestões de tokens exibida automaticamente ao digitar o caractere `{`.

---

### 2.20 Fluxo: Consultar Dashboard

- **Objetivo**: Visualizar o estado geral da aplicação, resumos de desempenho e atalhos operacionais.
- **Pré-condições**: Nenhuma.
- **Passo a passo da interação**:
  1. O operador clica no item "Dashboard" no `NavigationMenu` da `Sidebar`.
  2. A interface renderiza o `DashboardLayout` com os cards de métricas e tabelas de resumo.
  3. O operador analisa gráficos de tráfego, contagem de bots e alertas.
- **Componentes envolvidos**: `DashboardLayout`, `MetricCard`, `DataTable`, `StatusCard`, `Button`.
- **Estados da interface**: Normal (com atualização contínua).
- **Validações**: Nenhuma.
- **Comunicação esperada com o backend**: GET /api/v1/dashboard/summary e streaming via WebSocket.
- **Atualizações em tempo real**: Movimentação suave das linhas dos gráficos e atualização dos números nos `MetricCard`.
- **Resultado esperado**: Painel completo e atualizado sobre a saúde de toda a operação.
- **Possíveis erros**: Desconexão parcial de dados de telemetria.
- **Recuperação de erro**: Exibição do `StatusCard` de aviso solicitando reconexão da telemetria.
- **Feedback visual esperado**: Animação de entrada dos gráficos e numerais.
- **Observações de UX**: Layout de cards arranjável que permite reordenar os gráficos conforme a preferência do operador.

---

### 2.21 Fluxo: Monitorar Logs

- **Objetivo**: Filtrar, buscar e analisar o fluxo contínuo de logs de sistema e mensagens de chat de todos os bots.
- **Passo a passo da interação**:
  1. O operador expande o `LogConsole` inferior ou acessa a visão expandida de logs.
  2. Digita um termo de busca no `SearchBox` do console (ex: `desconectou` ou `[Minerador]`).
  3. Aplica filtros por severidade (Info, Warning, Error, Chat).
  4. Analisa as mensagens filtradas na tela.
- **Componentes envolvidos**: `LogConsole`, `SearchToolbar`, `Chip`, `Button`.
- **Estados da interface**: Normal (com filtragem ativa).
- **Validações**: Nenhuma.
- **Comunicação esperada com o backend**: Stream WebSocket de logs.
- **Atualizações em tempo real**: O console recebe novas linhas e aplica o filtro de busca instantaneamente.
- **Resultado esperado**: Visualização focada apenas nas linhas de log correspondentes ao filtro aplicado.
- **Possíveis erros**: Estouro de memória por retenção de histórico excessivo de logs.
- **Recuperação de erro**: O console aplica o limite circular de retenção (ex: máximo de 2000 linhas) automaticamente.
- **Feedback visual esperado**: Destacar o termo buscado em cor de fundo amarela nas linhas do log.
- **Observações de UX**: Botão para pausar temporariamente a rolagem automática (Auto-scroll) para facilitar a leitura.

---

### 2.22 Fluxo: Exportar Informações

- **Objetivo**: Salvar arquivos locais contendo listas de proxies filtradas, credenciais válidas ou históricos de logs.
- **Passo a passo da interação**:
  1. O operador acessa a página contendo os dados a serem exportados (`Proxy` ou `AccountChecker`).
  2. Aplica os filtros desejados (ex: apenas proxies Socks5 válidas).
  3. Clica no botão "Exportar" e escolhe o formato (.txt ou .json).
  4. O navegador dispara o download do arquivo gerado.
- **Componentes envolvidos**: `Toolbar`, `Button`, `Select`, `Toast`.
- **Estados da interface**: Normal -> Processing -> Success.
- **Validações**: Verificar se há pelo menos um registro válido a ser exportado.
- **Comunicação esperada com o backend**: GET /api/v1/export/{type}.
- **Atualizações em tempo real**: Nenhuma.
- **Resultado esperado**: Arquivo baixado na máquina do operador contendo os dados organizados.
- **Possíveis erros**: Nenhum dado disponível para exportação.
- **Recuperação de erro**: Alerta visual "Não há registros correspondentes aos filtros para exportar".
- **Feedback visual esperado**: Toast "Arquivo Exportado com Sucesso".
- **Observações de UX**: Formatação clara e amigável dos dados exportados sem linhas corrompidas.

---

### 2.23 Fluxo: Alterar Configurações Gerais

- **Objetivo**: Modificar parâmetros globais de funcionamento do cliente web e preferências da sessão.
- **Local de Execução**: Página `Configurações`.
- **Passo a passo da interação**:
  1. O operador acessa a página `Configurações`.
  2. Altera preferências visuais (Tema Escuro/Claro, limite de linhas do console, notificações sonoras).
  3. Configura parâmetros de rede padrão (timeout de socket, limites de chunk visual).
  4. Clica em "Salvar Configurações Globais".
- **Componentes envolvidos**: `SettingsLayout`, `FormLayout`, `Switch`, `NumberInput`, `Select`, `Button`.
- **Estados da interface**: Normal -> Success.
- **Validações**: Faixas numéricas válidas para timeouts e limites.
- **Comunicação esperada com o backend**: PUT /api/v1/config/global.
- **Atualizações em tempo real**: Aplicação imediata dos novos temas e preferências na interface.
- **Resultado esperado**: Parâmetros globais atualizados e persistidos no LocalStorage e backend.
- **Possíveis erros**: Falha ao salvar no armazenamento local.
- **Recuperação de erro**: Exibir notificação de erro e manter as configurações atuais.
- **Feedback visual esperado**: Notificação Toast de confirmação e alteração do tema visual se alterado.
- **Observações de UX**: Botão para "Restaurar Padrões de Fábrica" em todas as seções de configuração.

---

## 3. Fluxos Alternativos

A interface deve responder com resiliência e clareza visual aos seguintes cenários de exceção:

### 3.1 Backend Indisponível
- **Comportamento**: A interface exibe um `StatusCard` de erro vermelho no topo do layout informando "Comunicação com o Backend Perdida".
- **Ação de Recuperação**: O sistema dispara tentativas de reconexão automática a cada 5 segundos e exibe um botão de "Reconectar Agora". Os botões de ação ficam desabilitados.

### 3.2 Perda de Conexão de Rede (Client Offline)
- **Comportamento**: A barra `TopBar` exibe o badge "Sem Internet" em cor vermelha.
- **Ação de Recuperação**: A interface entra em modo de leitura (Read-Only) e restaura o estado operacional assim que a conexão de rede local for reestabelecida.

### 3.3 Timeout de Requisição
- **Comportamento**: Se uma requisição REST demorar mais de 10 segundos sem resposta, o estado `Loading` do componente afetado é interrompido.
- **Ação de Recuperação**: Notificação Toast de aviso "A operação demorou muito para responder. Tente novamente".

### 3.4 Proxy Inválida ou Inacessível
- **Comportamento**: Ao tentar conectar um bot através de uma proxy que caiu, a proxy é destacada em vermelho na `ProxyTable`.
- **Ação de Recuperação**: O bot tenta automaticamente a próxima proxy disponível no pool ou marca a conta com erro de proxy.

### 3.5 Conta Inválida / Senha Incorreta
- **Comportamento**: O backend retorna a mensagem de rejeição de autenticação (Bad Credentials / Invalid Session).
- **Ação de Recuperação**: O bot é movido para o status "Error" com o detalhe "Credenciais Inválidas", e a conta é destacada no validador.

### 3.6 Login Recusado pelo Servidor Minecraft
- **Comportamento**: Servidor recusa o bot (ex: "Você já está conectado neste servidor").
- **Ação de Recuperação**: O sistema registra a mensagem de kick exata no console e interrompe a tentativa de auto-reconexão imediata para evitar loops.

### 3.7 Servidor Cheio (Server Full)
- **Comportamento**: O servidor Minecraft rejeita a conexão informando que atingiu o limite de jogadores.
- **Ação de Recuperação**: O bot entra em fila de reconexão agendada (Backoff exponencial) e exibe o badge "Aguardando Vaga".

### 3.8 Bot Banido do Servidor
- **Comportamento**: O pacote de desconexão indica que a conta foi banida (ex: "Você foi banido permanentemente").
- **Ação de Recuperação**: O bot é alterado para o status "Banido" em cor vermelha marcante e removido das filas de auto-reconexão.

### 3.9 Macro com Erro em Tempo de Execução
- **Comportamento**: O script JavaScript lança uma exceção não tratada durante a execução.
- **Ação de Recuperação**: A macro é interrompida imediatamente para aquele bot, e o erro com a linha exata é destacado no `CodeEditor`.

### 3.10 Viewer 3D Indisponível (Falha de WebGL/VRAM)
- **Comportamento**: O navegador do operador não consegue alocar o contexto WebGL para o renderizador 3D.
- **Ação de Recuperação**: O sistema exibe mensagem informativa e oferece a visão alternativa de telemetria baseada em texto e inventário 2D.

### 3.11 Falha ao Importar Arquivos (Formato Incompatível)
- **Comportamento**: O arquivo selecionado para importação contém caracteres binários ou codificação inválida (não UTF-8).
- **Ação de Recuperação**: O modal de importação interrompe o parse e exibe "Formato de arquivo incompatível. Utilize arquivos de texto .txt em UTF-8".

### 3.12 Operação Cancelada pelo Usuário
- **Comportamento**: O operador clica no botão "Cancelar" durante a execução de uma tarefa em lote (ex: validação de proxies).
- **Ação de Recuperação**: O backend interrompe os novos testes e a interface preserva e exibe todos os resultados parciais obtidos até o momento do cancelamento.

---

## 4. Fluxos em Lote

Para garantir uma navegação fluida ao gerenciar dezenas ou centenas de bots simultaneamente, a interface obedecerá às seguintes regras funcionais para operações em lote:

### Processamento e Filas

- Operações disparadas para múltiplos bots (como conectar 100 bots) não são enviadas como 100 requisições individuais.
- A interface envia um único comando de lote contendo o vetor de alvos para o backend.
- O backend processa o lote utilizando threads virtuais e envia eventos de progresso individuais.

### Exibição de Progresso

- Durante qualquer processamento em lote, uma barra `ProgressBar` fixa ou no topo do componente exibirá a porcentagem concluída e o item atual (ex: "Processando 45 de 100").
- A tabela de dados continua interativa e permite a visualização das linhas que já foram concluídas.

### Cancelamento

- Toda operação em lote em andamento deve apresentar um botão visível de "Cancelar Operação".
- Ao ser acionado, a interface envia o sinal de interrupção e altera o status da barra de progresso para "Cancelado pelo Operador", mantendo os dados parciais.

### Conclusão

- Ao finalizar o lote, a interface fecha a barra de progresso e exibe uma notificação `Toast` contendo o resumo da execução (ex: "Lote concluído: 85 sucessos, 15 falhas").

---

## 5. Fluxos em Tempo Real

As atualizações reativas enviadas pelo backend via streaming deverão se comportar conforme as regras descritas abaixo:

### Atualização de Status dos Bots
- **Comportamento**: Quando um bot altera de estado (ex: de Autenticando para Online), o badge na `SidebarBotList` e na `DataTable` é atualizado suavemente sem forçar a reconstrução de toda a tabela nem perder a linha selecionada pelo operador.

### Streaming de Logs e Chat
- **Comportamento**: Novas mensagens recebidas são inseridas na base do `ConsoleLogViewer`. Se o operador tiver rolado a tela para cima para analisar mensagens antigas, a rolagem automática é temporariamente pausada e um aviso "Novos logs abaixo" é exibido.

### Dashboard e Métricas
- **Comportamento**: Os gráficos de linha de tráfego de dados e os contadores numéricos dos `MetricCard` são atualizados em intervalos regulares (1 segundo) através de transições suaves de animação.

### Renderização do Viewer 3D
- **Comportamento**: Os pacotes de movimento de blocos e entidades recebidos são processados no loop de renderização do WebGL a 60 FPS, mantendo a cena sincronizada com a posição real do bot no servidor.

### Sincronização de Inventário
- **Comportamento**: Ao coletar ou gastar um item, a grade do `InventoryGrid` atualiza instantaneamente o slot afetado, apresentando uma breve animação de brilho no slot alterado.

### Notificações Globais
- **Comportamento**: Eventos críticos do sistema (como a desconexão de um lote inteiro de bots ou detecção de banimento) disparam notificações `Toast` imediatas e alertas sonoros configuráveis.

---

## 6. Diretrizes Gerais

Todos os fluxos de interface do sistema AdvancedBot devem atender estritamente aos seguintes princípios operacionais de experiência do usuário:

1. **Interface Nunca Bloqueante**: Nenhuma operação assíncrona ou de rede deve travar ou congelar a interface visual do navegador. A aplicação deve permanecer sempre responsiva aos comandos do operador.
2. **Feedback Imediato (SLA < 100ms)**: Toda interação física do operador (cliques, digitações, seleções) deve gerar uma resposta visual perceptível imediata, mesmo que o resultado final dependa do backend.
3. **Cancelamento de Operações Longas**: Qualquer tarefa que demande mais de 3 segundos para ser concluída deve oferecer um meio fácil e claro de cancelamento por parte do operador.
4. **Preservação de Contexto e Estado**: Ao alternar entre páginas da aplicação e retornar, a interface deve preservar os filtros aplicados, os termos buscados, as seleções de linhas e a posição de rolagem das tabelas.
5. **Estabilidade de Seleção**: A ordenação automática ou chegada de novos dados em tempo real não deve desselecionar um bot ou item que o operador esteja inspecionando no momento.
6. **Consistência Visual Integral**: Todos os fluxos funcionais descritos neste documento devem ser implementados utilizando exclusivamente os componentes atômicos, moleculares e organizacionais definidos no Design System (documento 03).
