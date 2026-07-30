# Matriz Frontend × Backend

> Documento de trabalho para toda implementação futura de EPIC-FRONT. Complementa [06-Plano-Construcao-Frontend.md](06-Plano-Construcao-Frontend.md) (decisões travadas). Cada linha de feature abaixo é a referência única de endpoints/eventos/DTOs/hooks/componentes daquela feature — não redefinir em outro documento.

Convenções: DTOs vêm de `shared/api/generated/` (Orval, gerado do OpenAPI do backend — nomes exatos só são conhecidos após rodar `npm run generate:api`; nomes de classe Java abaixo servem de referência para localizar o schema equivalente). Hooks gerados = 1 por endpoint REST (Orval). Hooks de domínio = escritos à mão, combinam gerado + WS + Zustand.

> Atualização EPIC-FRONT-01: `npm run generate:api` já foi executado contra o backend real (13 controllers, 57 paths, 71 arquivos gerados em `advancedbot-frontend/src/shared/api/generated/{endpoints,models}`). Os nomes de arquivo/função gerados (ex. `bot-controller.ts`, `trocarProxy`, `criarBotRequest.ts`) confirmam os DTOs referenciados abaixo — cada EPIC-FRONT de feature deve conferir o arquivo gerado correspondente antes de escrever o hook de domínio, e rodar o script de novo se o backend tiver mudado desde a última geração.

---

## Feature: Dashboard — **IMPLEMENTADA (EPIC-FRONT-02)**

**Tela**: Dashboard (rota raiz `/`, `features/dashboard/`)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/metricas`, `GET /api/v1/contas?limit=1` (só `total`), `GET /api/v1/servidores?limit=1` (só `total`), `GET /api/v1/proxies` (array sem paginação, total = tamanho da lista) |
| Eventos WS | `/ws/events`, tipo `"estado"` — invalida a query de métricas imediatamente quando qualquer bot muda de estado de execução/sessão. CPU/memória/tick não têm evento dedicado (mudam independente de estado de bot) — cobertos por polling (`refetchInterval: 5000ms`, decisão travada #16/#17) |
| DTOs (gerados) | `MetricasResponse`, `PageResponseContaResponse`, `PageResponseServidorResponse`, `ProxyResponse[]` — todos de `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useMetricas` (metricas-controller), `useListar2` (conta-controller, `GET /contas`), `useListar` (servidor-controller, `GET /servidores`), `useListar1` (proxy-controller, `GET /proxies`) |
| Hooks de domínio | `useDashboardMetrics` (wrap de `useMetricas` + invalidação via `wsBus` no evento `"estado"`), `useDashboardTotals` (wrap de `useListar2`/`useListar`/`useListar1`), `useMetricsHistory` (acumula amostras reais em memória para os gráficos de tendência, sem endpoint de série histórica no backend) |
| Componentes | `MetricCard` (12 instâncias), `StateBreakdownCard` (distribuição por estado, barra CSS), `MetricTrendChart` (sparkline SVG de CPU/TPS), `DashboardGapNotice` (registro explícito dos 3 GAPs abaixo) |
| Dependências | Nenhuma — paralelizável (Fase 1), confirmado nesta sessão |
| GAP confirmado | **Threads da JVM**: não exposto por `MetricasController`. **Proxy em uso**: `ProxyResponse` não tem campo de associação a bot, só total cadastrado. **Heap vs. memória não-heap**: backend expõe só heap (`Runtime.totalMemory/freeMemory/maxMemory`) — "Memória JVM" e "Heap" pedidos como métricas separadas colapsam num único card real, não dois. |

## Feature: Bots (list) — **IMPLEMENTADA (EPIC-FRONT-05)**

**Tela**: Bots (lista, rota `/bots`, `features/bots/`, lazy-loaded)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `POST /api/v1/bots`, `GET /api/v1/bots?offset=&limit=`, `GET /api/v1/bots/{id}`, `DELETE /api/v1/bots/{id}`, `POST /api/v1/bots/{id}/start\|stop\|pause\|resume\|connect\|disconnect\|reconnect`, `PUT /api/v1/bots/{id}/proxy`, `PUT /api/v1/bots/{id}/auto-reconnect` — confirmado, sem `PUT /api/v1/bots/{id}` para edição completa (ver limitação abaixo) |
| Eventos WS | `/ws/events` (global, tipo `"estado"` — invalida `getListar3QueryKey()` sem refetch manual) |
| DTOs (gerados) | `BotResponse{id,username,host,port,estadoExecucao,estadoSessao,autoReconnect,proxy:ProxyResponse,macrosAtivas,posicao:PosicaoResponse,vida:VidaResponse,msDesdeUltimoKeepAlive}` (`estadoExecucao`/`estadoSessao` são `string` cru — sem enum gerado, valores reais tipados manualmente em `features/bots/services/botState.ts` a partir de `EstadoExecucao.java`/`EstadoSessao.java`), `PageResponseBotResponse`, `CriarBotRequest{contaId,servidorId,username,email,password,host,port}`, `ProxyBotRequest{host,port,tipo}`, `AutoReconnectRequest{ativo}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useListar3`/`listar3` (`GET /bots`), `useCriar2`/`criar2` (`POST /bots`), `useDetalhar2`/`detalhar2` (`GET /bots/{id}`), `useRemover3`/`remover3` (`DELETE /bots/{id}`), `useIniciar`/`iniciar`, `useParar`/`parar`, `usePausar`/`pausar`, `useRetomar`/`retomar`, `useConectar`/`conectar`, `useDesconectar`/`desconectar`, `useReconectar`/`reconectar`, `useTrocarProxy`/`trocarProxy`, `useDefinirAutoReconnect`/`definirAutoReconnect` — bot-controller (nomes reais confirmados no arquivo gerado, não os nomes hipotéticos `useGetApiV1Bots*` do rascunho original desta matriz) |
| Hooks de domínio | `useBotList` (wrap `useListar3`, `offset=0&limit=500`, paginação client-side + assinatura WS `"estado"` via `invalidateQueries`), `useBotTableState` (busca+página, estado de UI local), `useBotMutations` (`useCreateBot`/`useDeleteBot`, sem `useUpdateBot` — GAP abaixo), `useBotActions` (uma mutation por ação, todas via hooks gerados) + `useBatchBotAction` (`Promise.allSettled` sobre as funções cruas geradas — GAP de endpoint de lote) |
| Componentes | `DataTable` (reuso), `BotStatusBadge` (novo, dois badges independentes execução/sessão), `BotTable` (especialização de `DataTable`, ações contextuais por estado + checkbox de seleção para lote), `BotFormModal` (reusa `Tabs` para alternar conta/servidor existente vs. inline), `BotProxyModal` (reusa validação de Proxy promovida para `shared`), `ConfirmDialog`/`SearchBox`/`Select` (reuso direto) |
| Dependências | Contas/Servidores (`useContaList`/`useServidorList` reutilizados diretamente no `BotFormModal` para as opções de conta/servidor existente) |
| **Limitação real do backend** | Sem `PUT /api/v1/bots/{id}` para edição completa. Diferente de Proxy/Conta/Servidor, a composição client-side remover+criar (mesmo padrão de `useUpdateProxy`/`useUpdateConta`) **não é uma opção segura aqui**: `BotResponse` nunca devolve a senha usada na criação (`CriarBotRequest.password` é write-only) — recriar o bot exigiria pedir a senha de novo ao operador de qualquer forma, sem ganho real sobre simplesmente excluir e criar de novo manualmente. A Feature Bots oferece só criar e excluir; "editar" não foi implementada. Ações em lote continuam sendo N chamadas paralelas ao endpoint individual (sem `connect-batch` nativo, GAP §8 do doc 06). |

## Feature: Bots/details/console — **IMPLEMENTADA (EPIC-FRONT-06)**

**Tela**: Bot Details — Console/Logs (rota `/bots/:id/console`)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/bots/{id}/logs`, `DELETE /api/v1/bots/{id}/logs`, `POST /api/v1/bots/{id}/chat` (`acao-controller`), `POST /api/v1/bots/{id}/commands` (`comando-controller`) |
| Eventos WS | `/ws/bots/{id}/events`, tipo `"log"` (alta frequência, fora do cache TanStack Query) e `"comando"` (payload `ComandoExecutadoPayload{linha,resultado}`, também emitido por `MacroController` ao ativar/desativar) |
| DTOs (gerados) | `string[]` (logs, sem paginação — `LogController.listar` devolve o buffer inteiro), `ChatRequest{mensagem}`, `ComandoRequest{linha}`, `ComandoResponse{resultado}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useListar5`/`listar5` (`GET /logs`, log-controller), `useLimpar` (`DELETE /logs`, log-controller), `useEnviarMensagemDeChat` (`POST /chat`, acao-controller), `useExecutar` (`POST /commands`, comando-controller) |
| Hooks de domínio | `useRealtimeLogs` (busca inicial via `useListar5` + buffer local alimentado por `wsBus` no `"log"`, concatenados na leitura — sem sincronizar `query.data` para um `useState` a cada render), `useBotConsole` (chat + comando, histórico alimentado pela resposta direta de `useExecutar` e pelo evento WS `"comando"`) |
| Componentes | `ConsoleLogViewer` (novo, `features/bots/details/components/` — só 1 consumidor até agora, não promovido) |
| Dependências | Bots/list (precisa `{id}` selecionado) |
| **Achado/GAP confirmado (validado contra backend real)** | Duplicidade rara possível no histórico de comandos: `useExecutar` já popula o histórico via `onSuccess`, e o mesmo comando pode também chegar via evento WS `"comando"` — aceito como limitação conhecida, documentada no código (`useBotConsole.ts`), não uma tentativa de contorno. |

## Feature: Bots/details/acoes — **IMPLEMENTADA (EPIC-FRONT-06)**

**Tela**: Bot Details — Ações de movimento/interação (rota `/bots/:id/acoes`)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `POST /api/v1/bots/{id}/mover`, `/olhar`, `/mover-e-olhar`, `/agachar`, `/parar-de-agachar`, `/balancar-braco`, `/olhar-bloco`, `/olhar-entidade`, `/quebrar-bloco/iniciar\|cancelar\|finalizar`, `/colocar-bloco`, `/interagir-entidade`, `/usar-item` — todos em `acao-controller` |
| Eventos WS | Nenhum dedicado — GAP §8 do doc 06 (sem push de mudança de mundo/ação) |
| DTOs (gerados) | `MoverRequest{x,y,z,onGround}`, `OlharRequest{yaw,pitch,onGround}`, `MoverEOlharRequest`, `QuebraDeBlocoRequest{x,y,z,face}`, `BlocoAlvoRequest{x,y,z}`, `ColocarBlocoRequest{x,y,z,direction,item:ItemStackDto,cursorX,cursorY,cursorZ}`, `EntidadeIdRequest{entityId}`, `InteragirComEntidadeRequest{entityId,tipoAcao}`, `UsarItemRequest{item:ItemStackDto}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useMover`, `useOlhar`, `useMoverEOlhar`, `useAgachar`, `usePararDeAgachar`, `useBalancarBraco`, `useOlharParaBloco`, `useOlharParaEntidade`, `useIniciarQuebraDeBloco`, `useCancelarQuebraDeBloco`, `useFinalizarQuebraDeBloco`, `useColocarBloco`, `useInteragirComEntidade`, `useUsarItemNaMao` — todos `acao-controller` |
| Hooks de domínio | `useBotActionsPanel` (agrupa as 14 mutations; ações de movimento/câmera/postura invalidam `getEstadoQueryKey(id)` do `mundo-controller` ao terminar, para refletir posição atualizada) |
| Componentes | Painel de controle de ação (`AcoesPage`, novo, não reutilizado por outra feature) |
| Dependências | Bots/list |
| **GAP confirmado (validado contra backend real)** | `AcaoController`/`InventarioController`/`MundoController`/`MacroController` não têm bean validation (`@Valid`/`jakarta.validation`) nos request DTOs — igual ao já registrado para `CriarBotRequest`; validação de faixa/obrigatoriedade fica só no cliente. Todos os endpoints respondem `409 Conflito de estado — "Bot não está em uma sessão de jogo ativa (PLAY)"` quando o bot não está conectado a um servidor — confirmado manualmente, tratado como erro normal (`ErrorState`/toast), não simulado nem contornado. |

## Feature: Bots/details/inventario — **IMPLEMENTADA (EPIC-FRONT-06)**

**Tela**: Bot Details — Inventário/Equipamento/Janela (rota `/bots/:id/inventario`)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/bots/{id}/inventario`, `GET .../inventario/equipamento`, `GET .../inventario/janela`, `POST .../clicar-slot`, `/shift-clicar`, `/pegar-item`, `/largar-item`, `/mover-item`, `/trocar-slots`, `/limpar-cursor`, `/selecionar-hotbar`, `/confirmar-transacao`, `/fechar-janela` — todos em `inventario-controller` |
| Eventos WS | Nenhum dedicado — GAP §8, refetch REST no foco da janela/aba |
| DTOs (gerados) | `InventarioResponse{slotAtivo,slots:ItemStackDto[]}`, `EquipamentoResponse{capacete,peitoral,calca,botas,itemNaMao:ItemStackDto}`, `JanelaResponse{windowId,tipo,titulo,entityId,slots,cursor}`, `ItemStackDto{itemId,count,damage}` (`count` é `string`, não numérico), `ClicarSlotRequest{slotUnificado,botaoDireito}`, `MoverItemRequest{slotOrigem,slotDestino}`, `TrocarSlotsRequest{slotUnificado,slotHotbar}`, `SlotRequest{slotUnificado}`, `SelecionarHotbarRequest{slot}`, `ConfirmarTransacaoRequest{windowId,numeroDeTransacao,aceito}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useConsultar` (inventário), `useEquipamento`, `useJanela`, `useClicarSlot`, `useShiftClicar`, `usePegarItem`, `useLargarItem`, `useMoverItem`, `useTrocarSlots`, `useLimparCursor`, `useSelecionarHotbar`, `useConfirmarTransacao`, `useFecharJanela` — todos `inventario-controller` |
| Hooks de domínio | `useInventario` (`refetchOnWindowFocus`+`staleTime` curto, sem WS push), `useInventarioActions` (uma mutation por ação de slot, cada uma invalida inventário+equipamento+janela ao terminar, sem toast por clique) |
| Componentes | `InventorySlotGrid` + `ItemSlot` (novos, `features/bots/details/components/` — sem componente de grade no Design System ainda, `DataTable` é orientado a linhas; só 1 consumidor até agora, não promovidos) |
| Dependências | Bots/list |
| **GAP confirmado (validado contra backend real)** | `GET /inventario`, `/inventario/equipamento` e `/inventario/janela` respondem `409` quando o bot não está numa sessão PLAY ativa — mesmo comportamento do Mundo (ver acima), tratado como erro normal. |

## Feature: Bots/details/mundo — **IMPLEMENTADA (EPIC-FRONT-06)**

**Tela**: Bot Details — Mundo/Jogadores/Entidades (rota `/bots/:id/mundo`)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/bots/{id}/estado` (fora de `/mundo/`, `mundo-controller`), `GET .../mundo/bloco?x=&y=&z=` (params obrigatórios), `GET .../mundo/entidades?tipo=`, `GET .../mundo/entidades/{entityId}`, `GET .../mundo/jogadores` |
| Eventos WS | Nenhum dedicado — GAP §8 |
| DTOs (gerados) | `EstadoMundoResponse{x,y,z,yaw,pitch,onGround,estaSubmerso,health,food,saturation,gamemode,dimension,chunkAtual:ChunkAtualResponse,msDesdeUltimoKeepAlive}`, `ChunkAtualResponse{chunkX,chunkZ}`, `BlocoResponse{x,y,z,blockId,metadata,solido}`, `EntidadeResponse{entityId,tipo,x,y,z,yaw,pitch}`, `JogadorResponse{nome,nomeDeExibicao}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useEstado`, `useBloco`, `useEntidades`, `useEntidade`, `useJogadores` — todos `mundo-controller` |
| Hooks de domínio | `useEstadoMundo` (polling 3s, sem push), `useMundoEntidades` (polling 5s para entidades/jogadores) + `useBlocoLookup` (consulta sob demanda, `enabled` controlado pelas coordenadas informadas pelo operador) |
| Componentes | Painel de estado, formulário de consulta de bloco, `DataTable` (reuso) para entidades/jogadores |
| Dependências | Bots/list |
| **GAP confirmado (validado contra backend real)** | `GET /estado`, `/mundo/bloco`, `/mundo/entidades`, `/mundo/jogadores` respondem `409 "Bot não está em uma sessão de jogo ativa (PLAY)"` quando o bot não está conectado — confirmado manualmente (bot criado e nunca conectado a um servidor real), tratado como erro normal (`ErrorState` + toast), sem dado simulado. |

## Feature: Bots/details/macros — **IMPLEMENTADA (EPIC-FRONT-06)**

**Tela**: Bot Details — Macros por bot (rota `/bots/:id/macros`)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/bots/{id}/macros`, `POST .../macros/{alias}` (body opcional `AtivarMacroRequest{argumentos}`), `DELETE .../macros/{alias}` — `macro-controller` |
| Eventos WS | `/ws/bots/{id}/events`, tipo `"comando"` após ativar/desativar (mesmo payload do Console) |
| DTOs (gerados) | `MacroResponse{tipo}`, `AtivarMacroRequest{argumentos}`, `ComandoResponse{resultado}` — `shared/api/generated/models`. Cruzado com `ComandoCatalogoResponse{nome,descricao,aliases,parametros}` do Catálogo (`useCatalogoMacros`, reuso — 2º consumidor real do catálogo de macros) |
| Hooks gerados (Orval, reais) | `useListar4` (`GET /macros`), `useAtivar` (`POST /macros/{alias}`), `useDesativar` (`DELETE /macros/{alias}`) — todos `macro-controller` |
| Hooks de domínio | `useBotMacros` (wrap de `useListar4` + `useAtivar`/`useDesativar`, invalida tanto `getListar4QueryKey(id)` quanto `getDetalhar2QueryKey(id)` — `BotResponse.macrosAtivas` espelha a mesma lista e é usado por `BotTable`/`BotDetailsHeader`) |
| Componentes | Tabela de macros ativas (`DataTable`, reuso), formulário de ativação (`Select` do catálogo + `Input` de argumentos) |
| Dependências | Bots/list, Catálogo de Comandos/Macros |
| **GAP confirmado (validado contra backend real, não contornado)** | (1) `MacroResponse.tipo` — única informação devolvida para uma macro ativa — frequentemente **não corresponde** a `nome`/`aliases` do Catálogo: ativar `antiafk` e reconsultar `GET /macros` devolveu `{"tipo":"TarefaAntiAFK"}`, um valor derivado internamente pelo backend (provável nome de classe), não o alias original. Resultado: a correspondência "descrição do catálogo" cai para o valor cru quando não há match, e o filtro que esconde do dropdown macros já ativas também falha nesse caso (compara `tipo` a `nome`, que não bate). (2) `DELETE /bots/{id}/macros/{alias}` usando o próprio `tipo` como `alias` (única opção disponível, já que o backend não devolve o alias original) respondeu `200 OK` duas vezes seguidas em teste manual, mas `GET /macros` continuou devolvendo a mesma macro ativa depois — a desativação não teve efeito observável com o bot desconectado. Nenhum dos dois é contornável no frontend sem o backend passar a devolver o alias original (ou um identificador estável) junto de `MacroResponse`. |

## Feature: Macros/Catálogo (global) — **IMPLEMENTADA (EPIC-FRONT-04)**

**Tela**: Catálogo (rota `/catalogo`, `features/catalogo/`, lazy-loaded, abas Comandos/Macros)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/commands`, `GET /api/v1/macros` |
| Eventos WS | Nenhum |
| DTOs (gerados) | `ComandoCatalogoResponse{nome,descricao,aliases,parametros}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useComandos` (`GET /commands`), `useMacros` (`GET /macros`) — catalogo-controller |
| Hooks de domínio | `useCatalogoComandos`/`useCatalogoMacros` (wrap dos hooks gerados, `staleTime` 5min), `useCatalogoTableState` (busca+paginação client-side, reutilizado pelas duas abas — mesmo DTO) |
| Componentes | `CatalogoTable` (especialização de `DataTable`, colunas nome/descrição/aliases/parâmetros, somente leitura — reutilizada pelas duas abas), `Tabs` (`shared/components/navigation/Tabs.tsx`, reuso) |
| Dependências | Nenhuma — paralelizável (Fase 1), confirmado |
| **Observação** | Agrupamento por categoria avaliado e não aplicado: `ComandoCatalogoResponse` não expõe nenhuma taxonomia/categoria no backend — agrupar por heurística client-side inventaria estrutura ausente na fonte de verdade. Editor de código (Monaco) não implementado — não há endpoint de criação/edição de macro customizada no backend, feature é somente leitura. |

## Feature: Proxy — **IMPLEMENTADA (EPIC-FRONT-03)**

**Tela**: Proxy (rota `/proxy`, `features/proxy/`, lazy-loaded)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `POST /api/v1/proxies`, `GET /api/v1/proxies`, `DELETE /api/v1/proxies?host=&port=&tipo=` — confirmado, sem `PUT` (ver limitação abaixo) |
| Eventos WS | Nenhum |
| DTOs (gerados) | `ProxyRequest{host,port,tipo}`, `ProxyResponse{host,port,tipo}`, `Remover1Params{host,port,tipo}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useListar1` (GET), `useAdicionar` (POST), `useRemover1` (DELETE) — proxy-controller |
| Hooks de domínio | `useProxyList` (wrap de `useListar1`, `staleTime` 5min), `useProxyTableState` (busca + paginação client-side, sem endpoint de busca/paginação no backend), `useCreateProxy`/`useDeleteProxy` (wrap dos mutations gerados + invalidação + toast de sucesso), `useUpdateProxy` (mutation composta: chama as funções geradas `remover1`+`adicionar` em sequência — não existe endpoint único de edição) |
| Componentes | `ProxyTable` (especialização de `DataTable`, sem coluna de latência — GAP), `ProxyFormModal` (criar/editar, reaproveita `Modal`), `ConfirmDialog` (reuso direto, exclusão) |
| Componente novo no Design System | `Select` atômico (`shared/components/atoms/Select.tsx`) — já especificado em `03-Design-System.md` desde a Fase 0, implementado agora por necessidade real (campo `tipo`) |
| Dependências | Nenhuma — paralelizável (Fase 1), confirmado |
| **Limitação real do backend** | Sem `PUT /api/v1/proxies` — proxy não tem identidade própria por entrada (chave natural host+port+tipo, `ProxyController.java`). "Editar" implementado como remover+adicionar client-side; se a remoção for bem-sucedida e a criação falhar, a entrada original já foi perdida (mensagem de erro específica avisa o operador a recadastrar manualmente). Duplicatas exatas são permitidas pelo backend (sem `UNIQUE`) — `remover`/`DELETE` remove só a primeira ocorrência encontrada, não necessariamente a linha exibida na tela quando há duplicatas. Sem `POST /proxies/check` (latência) — já era GAP conhecido, `ProxyTable` não tem coluna de ping. |

> Atualização (EPIC-FRONT-05): `TIPOS_DE_PROXY`/`TipoDeProxy` e a validação host+porta+tipo (`validateProxyForm`/`isProxyFormValid`/`ProxyFormValues`) foram promovidas de `features/proxy/services/proxyValidation.ts` para `shared/types/proxy.ts`/`shared/lib/proxyFormValidation.ts` — 2º consumidor real (`BotProxyModal`, troca de proxy de um bot, usa o mesmo formato de `ProxyBotRequest`). `proxyValidation.ts` mantido como reexport, nenhum contrato desta feature mudou.

## Feature: Contas/Servidores — **IMPLEMENTADA (EPIC-FRONT-04)**

**Tela**: Contas e Servidores (rota `/contas-servidores`, `features/contas-servidores/`, lazy-loaded, abas Contas/Servidores). Sem tela de navegação própria no documento original, implementada como necessária — alimenta `BotForm` da futura feature Bots/list.

| Camada | Detalhe |
|---|---|
| Endpoint REST | `POST/GET/DELETE /api/v1/contas` (`offset`/`limit` reais no backend), `POST/GET/DELETE /api/v1/servidores` (idem) — confirmado, sem `PUT` em nenhum dos dois (ver limitação abaixo) |
| Eventos WS | Nenhum |
| DTOs (gerados) | `ContaRequest{email,username,password}`, `ContaResponse{id,email,username}`, `Listar2Params{offset,limit}`, `PageResponseContaResponse{items,total,offset,limit}`; `ServidorRequest{nome,host,port}`, `ServidorResponse{id,nome,host,port}`, `ListarParams`, `PageResponseServidorResponse` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useListar2`/`useCriar1`/`useRemover2` (conta-controller); `useListar`/`useCriar`/`useRemover` (servidor-controller) |
| Hooks de domínio | `useContaList`/`useServidorList` (wrap dos hooks gerados com `limit=500`, `staleTime` 5min — busca a página real inteira e pagina client-side por decisão explícita, mesmo padrão de UX de Proxy), `useContaTableState`/`useServidorTableState` (busca+paginação client-side), `useContaMutations`/`useServidorMutations` (`useCreateConta`/`useCreateServidor`, `useDeleteConta`/`useDeleteServidor` wrap direto; `useUpdateConta`/`useUpdateServidor` mutation composta remover+criar, mesmo padrão de `useUpdateProxy`) |
| Componentes | `ContaTable`/`ServidorTable` (especialização de `DataTable`), `ContaFormModal`/`ServidorFormModal` (reaproveitam `Modal`), `ConfirmDialog` (reuso direto), `Tabs` (`shared/components/navigation/Tabs.tsx`, novo, promovido no 1º épico com 2 consumidores reais) |
| Dependências | Nenhuma — paralelizável (Fase 1), confirmado. Alimentará `BotForm` da feature Bots/list (Fase 2). |
| **Limitação real do backend** | Sem `PUT /api/v1/contas/{id}` nem `PUT /api/v1/servidores/{id}` (`conta-controller.ts`/`servidor-controller.ts` só expõem listar/criar/detalhar/remover). "Editar" implementado como remover+criar client-side — diferente da limitação de Proxy (chave natural sem `id`), aqui a entidade tem `id` próprio (UUID) e a composição client-side gera um **novo** `id` a cada edição; se a remoção for bem-sucedida e a criação falhar, a entidade original já foi perdida (mensagem de erro específica avisa o operador, mesmo padrão de `useUpdateProxy`). |

## Feature: Configurações — **IMPLEMENTADA (EPIC-FRONT-04)**

**Tela**: Configurações (rota `/configuracoes`, `features/configuracoes/`, lazy-loaded)

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/configuracao/reconnect-policy` (leitura), `PUT /api/v1/bots/{id}/auto-reconnect` (por bot, coberto pela futura feature Bots/list) |
| Eventos WS | Nenhum |
| DTOs (gerados) | `ReconnectPolicyResponse{intervaloBaseMs,jitterMaximoMs}` — `shared/api/generated/models` |
| Hooks gerados (Orval, reais) | `useReconnectPolicy` — configuracao-controller (só GET, backend não expõe mutation) |
| Hooks de domínio | `useConfiguracao` (wrap de `useReconnectPolicy`, `staleTime` 5min) |
| Componentes | `ReconnectPolicyPanel` (painel de leitura simples, `Card` + grid chave/valor — fica local à feature, único consumidor até agora) |
| Dependências | Nenhuma — paralelizável (Fase 1), confirmado. GAP §8: `/config/server`, `/config/auto-login`, `/config/miner`, `/config/global` não existem — nenhum formulário de escrita construído para eles. |

## Feature: Monitoramento

**Tela**: Monitoramento

| Camada | Detalhe |
|---|---|
| Endpoint REST | `GET /api/v1/metricas` (mesmo endpoint do Dashboard) |
| Eventos WS | `/ws/events` |
| DTOs | `MetricasResponse` (mesmo do Dashboard) |
| Hooks gerados | `useMetricas` (metricas-controller, reuso do hook gerado do Dashboard) |
| Hooks de domínio | `useDashboardMetrics` (reuso) |
| Componentes | `MetricCard`, `MetricTrendChart` (reuso do Dashboard) |
| Dependências | Extensão do Dashboard — GAP §8: métricas de rede (KB/s, pps, threads) não existem, não construir esses cards |

---

## Features bloqueadas por GAP de backend (Fase 4 — scaffolding apenas, sem consumir dado real)

| Feature | Tela | Situação |
|---|---|---|
| Spammer | Spammer | Nenhum endpoint. Sem `SpammerController`. Não construir chamada REST — aguardar backend. |
| Ferramentas | AccountChecker/BanCheck | Nenhum endpoint/controller. Aguardar backend. |
| Viewer 3D (full) | Viewer 3D | `WS /ws/viewer/{botId}` (chunks binários) inexistente. Versão REST-only usa endpoints de Bots/details/mundo (§ acima) enquanto GAP não resolve — ver decisão 20 do doc 06. |
| Minerador (config fina) | Minerador | Só existe toggle de macro (`POST/DELETE /bots/{id}/macros/minerar`, coberto por Bots/details/macros). `PUT /config/miner` inexistente — não construir tela de configuração fina de prioridade/raio. |

---

## Regra de manutenção deste documento

Esta matriz é atualizada **somente** quando: (a) o backend expõe um endpoint/evento novo que resolve um GAP listado, ou (b) uma feature nova é aprovada fora do escopo das 10 telas de `02-Arquitetura-Frontend.md`. Qualquer EPIC-FRONT que precise de dado não listado aqui deve primeiro voltar a este documento e ao 06, não decidir endpoint/hook ad-hoc dentro do épico de implementação.
