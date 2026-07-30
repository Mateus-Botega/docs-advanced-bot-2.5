# Plano de Construção do Frontend (CONGELADO)

> Fundação oficial da fase Frontend. Baseado exclusivamente em: backend Java (fonte da verdade), Governança (DEC-40/41/42, Estado-Atual-Migracao), Documentação de Interface (12-Interface/*). Legado C# não consultado nesta revisão — já cumpriu papel.
> Toda decisão de arquitetura abaixo é definitiva. Próximas sessões implementam EPIC-FRONT diretamente, sem reabrir estas escolhas. Mudança de decisão exige nova revisão explícita deste documento, não decisão ad-hoc dentro de um épico.
> Não implementa código.

## Status de implementação

- **EPIC-FRONT-01 (Fundação) — CONCLUÍDO.** Projeto `advancedbot-frontend/` criado na raiz do repositório com toda a infraestrutura descrita neste documento: providers, layout (AppShell/Sidebar/TopBar/Workspace), componentes atômicos/moleculares base, httpClient/wsClient, geração Orval funcional contra o OpenAPI real do backend, testes (Vitest/RTL/MSW), lint/format/typecheck/build validados. Nenhuma tela de negócio foi implementada nesta fase (por escopo). Ver registro completo na Milestone Frontend 01 em `11-Estado-Atual-Migracao.md`.
- **EPIC-FRONT-02 (Feature Dashboard) — CONCLUÍDO.** Primeira feature funcional: `features/dashboard/` (components/hooks/services/pages/tests), consumindo exclusivamente hooks gerados pelo Orval (`useMetricas`, `useListar`/`useListar1`/`useListar2`). Corrigido nesta sessão um bug de configuração do Orval (ver decisão #7 abaixo) que invertia `useQuery`/`useMutation` em todos os hooks gerados — a fundação da Fase 0 nunca havia exercitado um hook GET de verdade, então o bug só apareceu ao consumir a API pela primeira vez. Dashboard virou a rota raiz (`/`), substituindo a `FoundationPage` (removida). Ver Milestone Frontend 02 em `11-Estado-Atual-Migracao.md` para o detalhamento completo.
- **EPIC-FRONT-03 (Feature Proxy) — CONCLUÍDO.** `features/proxy/` completa (components/hooks/services/pages/tests): listagem, busca/paginação client-side, criação, edição, exclusão com confirmação — pronta para produção. Backend não tem `PUT /api/v1/proxies` (sem identidade própria por entrada, chave natural host+port+tipo) — "editar" é composto client-side (remover + adicionar, ambas funções geradas pelo Orval), documentado como limitação real do backend, não como decisão de frontend. Ver Milestone Frontend 03 em `11-Estado-Atual-Migracao.md`.
- **EPIC-FRONT-04 (Fundação Administrativa) — CONCLUÍDO.** Quatro features independentes da Fase 1 implementadas na mesma sessão: `features/contas-servidores/` (CRUD completo de Conta e Servidor, uma rota com abas), `features/catalogo/` (Comandos/Macros, somente leitura, uma rota com abas), `features/configuracoes/` (política de reconexão, somente leitura). `shared/lib/pagination.ts` e `shared/components/navigation/Tabs.tsx` promovidos a `shared` no 2º consumidor real (regra de promoção do item 10 abaixo). GAP confirmado: `conta-controller.ts`/`servidor-controller.ts` não têm `PUT` — "editar" é remover+criar client-side (mesmo padrão de Proxy), mas troca o `id` da entidade porque, ao contrário de Proxy, Conta/Servidor têm identidade própria por UUID. Ver Milestone Frontend 04 em `11-Estado-Atual-Migracao.md`.
- **EPIC-FRONT-05 (Feature Bots) — CONCLUÍDO.** Núcleo da Fase 2: `features/bots/` completa (listagem com busca+paginação client-side, criação, exclusão, ações de execução/sessão iniciar/parar/pausar/retomar/conectar/desconectar/reconectar, troca de proxy, auto-reconnect, indicador de estado, ações em lote via `Promise.allSettled`, atualização em tempo real via WS `"estado"`). Promovidos para `shared` no 2º consumidor real: `shared/types/proxy.ts` (`TIPOS_DE_PROXY`) e `shared/lib/proxyFormValidation.ts` (validação host+porta+tipo), ambos antes só em Proxy. GAP confirmado: sem "editar" bot — `BotResponse` não devolve a senha usada na criação, então a composição remover+criar (usada em Proxy/Contas/Servidores) não é segura aqui; Feature Bots oferece só criar e excluir. Ver Milestone Frontend 05 em `11-Estado-Atual-Migracao.md`.
- **EPIC-FRONT-06 (Bot Details) — CONCLUÍDO.** Núcleo completo da Fase 2 restante: `features/bots/details/` com as 5 sub-abas roteadas sob `/bots/:id` (rotas aninhadas, decisão travada #5) — Console (logs em tempo real via WS, chat, execução de comandos), Ações (movimento/interação), Inventário (inventário/equipamento/janela), Mundo (estado/bloco/entidades/jogadores), Macros (macros ativas do bot cruzadas com o Catálogo global). Canal WS por bot (`connectBotEventsSocket`) conectado pela primeira vez nesta sessão, por `BotDetailsPage` (layout), mantido vivo entre troca de abas. `BotTable` da listagem ganhou link de navegação (`username` → `/bots/{id}/console`). GAPs reais confirmados contra o backend rodando (não contornados): `DELETE /bots/{id}/macros/{alias}` responde 200 mas não remove a macro da lista quando o bot está desconectado; `MacroResponse.tipo` (única informação devolvida para macros ativas) frequentemente não bate com `nome`/`aliases` do Catálogo (ex.: ativar `antiafk` retorna `tipo:"TarefaAntiAFK"`), então a correspondência categoria-ativa↔catálogo às vezes cai para o valor cru; `GET /bots/{id}/estado`, `/mundo/*` e `/inventario/*` respondem 409 "Bot não está em uma sessão de jogo ativa (PLAY)" quando o bot não está conectado — tratado como erro normal (`ErrorState`+toast), não simulado. Ver Milestone Frontend 06 em `11-Estado-Atual-Migracao.md`.
- **EPIC-FRONT-07 em diante**: implementam as demais features do roadmap abaixo (Fase 3 — Monitoramento; Fase 4 — bloqueadas por GAP), uma de cada vez, seguindo `07-Matriz-Frontend-Backend.md`.

---

## 1. Telas existentes

10 páginas de topo (`02-Arquitetura-Frontend.md`):

1. Dashboard
2. Bots (lista + detalhe: Console, Inventário, Equipamento, Containers, Mundo, Jogadores, Comandos — sub-abas, não páginas)
3. Proxy
4. Macros
5. Minerador
6. Viewer 3D (app/rota separada, WebGL)
7. Spammer
8. Ferramentas (AccountChecker/BanCheck)
9. Monitoramento
10. Configurações

DEC-41 fala em "12 telas" — contagem por sub-recurso, não por página de navegação (Bot Details/Console/Inventário/Equipamentos/Containers/Mundo/Jogadores/Comandos são sub-abas de Bots).

## 2. Obrigatórias vs opcionais

**Core (backend pronto hoje):** Dashboard, Bots (CRUD + ações), Bot Details (Console/Ações/Inventário/Mundo/Macros-por-bot), Macros (catálogo), Proxy, Contas/Servidores, Configurações (parcial).

**Bloqueadas por falta de backend (GAP, não implementar até endpoint existir):** Viewer 3D, Spammer, Ferramentas (AccountChecker/BanCheck), Monitoramento avançado (rede/pps), Minerador (config fina).

## 3-5. Matriz Tela × Endpoint × Evento WebSocket × DTO

Ver documento dedicado [07-Matriz-Frontend-Backend.md](07-Matriz-Frontend-Backend.md) — versão definitiva e detalhada desta matriz, incluindo Hooks e Componentes por feature. Esta seção fica só como referência histórica de qual tela usa qual grupo de endpoint; a matriz de trabalho é o documento 07.

## 6. Componentes reutilizáveis entre telas

`DataTable`, `BotStatusBadge`, `SidebarBotList`, `ConsoleLogViewer`, `InventoryGrid`, `ModalConfirm`, `ToolbarActionGroup`, `SearchBox`/`FilterBar`, `NumberInput`, `MetricCard`/`StatusCard`, `Toast`. Regra de promoção: nasce dentro da feature, sobe para `shared/components` na 2ª feature que precisar (não promover prematuramente).

## 7. Estado backend vs estado só-UI

**Backend:** `estadoExecucao`, `estadoSessao`, posição/vida/inventário/janela/entidades/proxy, logs, métricas, `autoReconnect`.

**Só UI:** aba ativa, modal aberto/fechado, texto de busca não confirmado, bot focado na sidebar, tema, ordenação de tabela, scroll/filtros, spinners locais, estados finos de sessão (`Autenticando`/`Executando Macro`/`Desconectando`/`Erro`/`Banido` — inferidos no cliente via `useBotSessionState`, não existem como enum no backend).

## 8. GAPs de Backend confirmados

Registrado como GAP — **não inventar solução no React para nenhum destes**, aguardar backend:

- Endpoint único de conectar (`POST /bots/connect`) — hoje é 2 passos (criar + connect)
- Endpoints de lote (`connect-batch`, disconnect/reconnect em massa) — hoje só por `{id}` individual
- `PUT /config/server`, `/config/auto-login`, `/config/miner`, `/config/global` — nenhum existe
- `POST /macros/execute`/`/stop` globais — modelo real é por bot+alias
- `POST /spammer/start` — sem controller
- `GET /dashboard/summary` — existe `GET /metricas`, formato diferente
- `GET /export/{type}` — ausente
- `POST /proxies/check` (latência) — ausente
- `WS /ws/viewer/{botId}` (chunks binários) — ausente, sem infra de streaming binário
- `WS /ws/bots`, `/ws/logs`, `/ws/control` — ausentes; canal real é `/ws/bots/{id}/events` e `/ws/events`
- `POST /auth/login` (JWT) — ausente; auth real é API key estática (`X-API-Key`)
- Eventos tipados (`BOT_CONNECTED`, `BOT_INVENTORY_UPDATED`, `METRICS_TICK`, `CHUNK_DATA_RECEIVED`, `PROXY_LATENCY_UPDATED`) — `EventoDeBot.tipo` só emite `"log"`, `"estado"`, `"comando"` hoje
- Sem evento WS dedicado para mudança de inventário/mundo/entidade — cliente precisa refetch REST ou polling, não há push

Fora de escopo deliberado (DEC-41): Bioma, NPCs distintos, Ping real (RTT), separação estruturada chat/erro/comando no console.

## 9. Inconsistências código × documentação de Interface

Mantidas como registro histórico (não bloqueiam implementação, já resolvidas pela decisão de seguir o backend como fonte da verdade): nomenclatura `accounts` vs `contas`, conexão 1 vs 2 passos, lote nativo vs individual, 7 vs 3+3 estados, ping vs keep-alive, JWT vs API key, métricas de rede ausentes, protocolo binário Viewer3D ausente, serialização `{"value":"uuid"}` de `IdentificadorBot`, checklist desatualizado em `11-Estado-Atual-Migracao.md`.

**Regra congelada:** onde doc de Interface e backend Java divergem, o frontend implementa o backend. A doc de Interface deixou de ser normativa para os pontos listados acima.

## 10. Telas paralelizáveis

Independentes: Proxy, Contas/Servidores, Catálogo Comandos/Macros, Configurações, Dashboard/Métricas.
Acopladas (mesmo `{id}` de bot + mesmo canal WS): Bots lista + Bot Details inteiro — tratado como feature única `bots/` com sub-rotas, não páginas paralelas.
Bloqueadas: Viewer 3D, Spammer, Ferramentas, Minerador-fino.

---

## DECISÕES TRAVADAS (definitivas, não reabrir)

### 1. Gerenciamento de estado do servidor — **TanStack Query**
Cache, dedupe, invalidação e refetch automático prontos; `staleTime` configurável por endpoint (baixo para bot list/logs, alto para Contas/Servidores/Catálogo). Integra nativamente com hooks gerados por Orval (decisão #7). Alternativa SWR descartada por ecossistema de devtools/mutations mais fraco para este volume de endpoints (13 controllers).

### 2. Estado global da aplicação — **Zustand**
Cobre: bot selecionado/focado, tema (persistido), sidebar colapsada, preferências de UI. Sem Provider hell (ao contrário de Context puro), sem boilerplate de actions/reducers/middleware (ao contrário de Redux, que é overkill para este volume de estado — não há necessidade de time-travel debugging ou middleware complexo). Context API não é usado para estado mutável frequente; reservado só para composição estrutural exigida por libs de terceiros (ex. Router).

### 3. Cliente HTTP — **Axios**
Interceptors nativos para injetar `X-API-Key` em toda request e normalizar `ErrorResponse{status,error,message,path}` do `GlobalExceptionHandler` em um `AppError` tipado antes de chegar no `onError` do TanStack Query. Fetch nativo rejeitado — exigiria wrapper manual equivalente sem ganho real.

### 4. Cliente WebSocket — arquitetura definida
- Dois clients singleton: `globalEventsSocket` (`/ws/events`) e uma factory `createBotEventsSocket(botId)` (`/ws/bots/{id}/events`).
- API nativa `WebSocket` do browser — backend não usa STOMP nem Socket.IO (DEC-40), então nenhum client de protocolo extra é necessário ou compatível.
- Reconexão automática com backoff exponencial simples (max ~30s), disparada em `onclose`/`onerror`.
- Parsing centralizado do envelope `EventoDeBot{botId,tipo,timestamp,payload}` em um único parser que já resolve o formato `{"value":"uuid"}` de `IdentificadorBot` — nenhum outro ponto do código faz esse parsing manualmente.
- Dispatch via **mitt** (pub/sub de ~200 bytes) por `tipo` de evento; `tipo` desconhecido tem fallback silencioso (backend pode adicionar tipos sem quebrar o cliente).
- Eventos de baixa frequência/estruturais (`"estado"`) fazem `queryClient.setQueryData`/`invalidateQueries` no cache do TanStack Query. Eventos de alta frequência (`"log"`) vão direto por pub/sub para o `ConsoleLogViewer`, sem passar pelo cache de query (evita re-render de árvore).

### 5. Roteamento — **React Router v6 (data router / `createBrowserRouter`)**
Uma rota por feature/página de topo, `lazy()` + code splitting por rota. Viewer 3D é rota própria com import lazy isolado (Three.js nunca entra no bundle principal). Sub-abas de Bot Details são rotas aninhadas sob `/bots/:id`.

> Nota de auditoria (EPIC-FRONT-01): `npm audit` reporta uma vulnerabilidade alta em `react-router`/`react-router-dom` (GHSA-qwww-vcr4-c8h2, "RSC Mode CSRF Bypass") presente em toda a série 7.12.0-8.2.0, incluindo a última versão publicada (7.18.1, instalada). O advisory cobre exclusivamente o modo RSC/framework (server actions) — este projeto usa somente o modo biblioteca (`createBrowserRouter` em uma SPA servida por Vite, sem SSR/loaders de servidor/server actions), então a superfície vulnerável não é exercida. Mantida a versão mais recente por cobrir corretamente as demais vulnerabilidades do pacote; reavaliar quando o advisory for corrigido upstream.

### 6. Tema — estratégia definitiva
Tailwind `darkMode: 'class'`. Estado do tema (`light`/`dark`/`system`) num slice Zustand persistido em `localStorage`, aplicado como classe no `<html>`. Tokens do Design System (`03-Design-System.md`) viram CSS variables + `tailwind.config` `extend`.

### 7. Geração de DTOs — **automática via OpenAPI, obrigatória**
Confirmado no backend: `springdoc-openapi-starter-webmvc-ui:2.5.0` já presente no `pom.xml`, `application.yml` expõe `/v3/api-docs` e `/swagger-ui.html`. Portanto:
- **Ferramenta**: **Orval**, gerando a partir de `/v3/api-docs` diretamente clientes Axios + hooks TanStack Query tipados para os 13 controllers REST.
- Script `npm run generate:api` roda Orval contra o backend local (ou contra um JSON exportado em CI) e escreve em `shared/api/generated/` — **pasta sempre gerada, nunca editada à mão**.
- **DTOs manuais ficam proibidos** a partir de agora. Qualquer tipo TS que hoje seria "BotResponse escrito à mão" passa a vir do gerador. Só tipos de view-model puramente de UI (que não existem no backend, ex. filtros de tabela) continuam manuais, em `shared/types/`.
- WebSocket fica fora do OpenAPI (não é REST) — o envelope `EventoDeBot` e seus `payload` por `tipo` continuam com tipos escritos manualmente em `shared/types/ws.ts`, mantidos sincronizados manualmente com `application/registry/EventoDeBot.java` e os poucos payloads reais (`log`, `estado`, `comando`).

> **Correção de configuração (EPIC-FRONT-02):** `orval.config.ts` tinha `override.query: { useQuery: true, useMutation: true, signal: true }`. Com os dois booleanos juntos, o Orval 8.23 inverte a geração — todo `GET` vira `useMutation` e todo `POST`/`PUT`/`DELETE` vira `useQuery` (confirmado inspecionando o output real ao consumir `useMetricas` pela primeira vez). Corrigido para `{ signal: true }` apenas, deixando o Orval decidir pelo verbo HTTP (comportamento padrão, correto). `shared/api/generated` foi regenerado após a correção. Nenhuma decisão arquitetural mudou — Orval continua a ferramenta, só a config estava quebrada e nunca tinha sido exercitada contra um hook GET real até esta sessão.

### 8. Estrutura de diretórios — revisada, redundância removida
`shared/types/` deixa de guardar DTOs de REST (agora gerados) — só guarda tipos de WS e view-models de UI. Pasta `bot-details/` é removida como feature top-level separada: vira sub-pasta dentro de `features/bots/details/`, refletindo o acoplamento já identificado em §10.

```
src/
  app/                    # bootstrap, providers (QueryClientProvider, Router), tema
  shared/
    api/
      generated/          # output do Orval — NUNCA editar à mão, regenerar via npm run generate:api
      httpClient.ts        # instância Axios (interceptors X-API-Key, AppError)
      wsClient.ts           # globalEventsSocket + createBotEventsSocket, parser EventoDeBot, mitt bus
    components/            # Design System: Button, Card, Badge, Modal, Toast, DataTable, ...
    hooks/                 # genéricos não-domínio: useDebounce, usePagination
    store/                 # slices Zustand: uiStore (bot selecionado, tema, sidebar)
    types/                 # ws.ts (tipos manuais de EventoDeBot/payloads), view-models de UI
  features/
    dashboard/
    bots/
      list/                # lista, CRUD, ações start/stop/pause/resume/connect/disconnect/reconnect
      details/
        console/            # logs, chat, comandos
        acoes/              # movimento/interação
        inventario/          # inventário/equipamento/containers
        mundo/               # estado/bloco/entidades/jogadores
        macros/              # macros ativas do bot
    macros-catalogo/        # catálogo global de comandos/macros
    proxy/
    contas-servidores/
    configuracoes/
    monitoramento/
    spammer/                # scaffolding, aguarda GAP de backend
    ferramentas/            # scaffolding, aguarda GAP de backend
    viewer3d/                # rota isolada, lazy, aguarda GAP de backend (streaming binário)
  main.tsx
```

### 9. Arquitetura das Features — confirmada com o ajuste do item 8
Feature-based mantido. `bots/` passa a ser a única feature "grande", com `list/` e `details/*` como sub-módulos — reflete a realidade de acoplamento (mesmo `{id}`, mesmo canal WS), evitando duas features artificialmente separadas que sempre são desenvolvidas juntas.

### 10. Componentes reutilizáveis — estratégia confirmada
Regra de promoção mantida (nasce na feature, sobe para `shared/components` na 2ª necessidade). Lista de candidatos day-1 já conhecida (§6) pode nascer direto em `shared/components` para as 5 features da Fase 1 do roadmap, já que o reuso é certo desde o início.

> Atualização (EPIC-FRONT-03): `Select` (atômico, já especificado em `03-Design-System.md` §2 desde a Fase 0 mas não implementado até então) foi criado em `shared/components/atoms/Select.tsx` para o campo `tipo` do formulário de Proxy — completa o catálogo já congelado, não é uma decisão nova.

### 11. Estratégia de Hooks — duas camadas
- **Hooks gerados** (Orval, `shared/api/generated/`): um hook TanStack Query por endpoint REST (ex. `useGetApiV1Bots`, `usePostApiV1BotsIdStart`). Nunca editados manualmente.
- **Hooks de domínio** (escritos à mão, dentro de cada feature): combinam hooks gerados + WS pub/sub + Zustand. Ex.: `useBotList` (wrap do hook gerado + assinatura de `"estado"` via WS), `useRealtimeLogs` (assina `"log"` via mitt, não usa TanStack Query), `useBotSessionState` (infere estados finos de sessão a partir de logs/eventos, já que o backend só expõe 3+3 estados).

### 12. Estratégia de comunicação REST
Axios (`shared/api/httpClient.ts`) com `baseURL=/api/v1`, header `X-API-Key` injetado via interceptor a partir de env var. Todos os hooks gerados pelo Orval usam essa instância. Paginação offset/limit (`PageResponse<T>`) tratada nos hooks gerados sem wrapper extra. Ações em lote continuam sendo N chamadas paralelas do cliente ao endpoint individual (GAP de backend, §8) — usar `Promise.allSettled`.

### 13. Estratégia de comunicação WebSocket
Ver item 4. Reforço: nenhuma lógica de negócio no parser — ele só resolve envelope + `IdentificadorBot`. Toda lógica de "o que fazer com o evento" vive nos hooks de domínio que assinam o pub/sub.

### 14. Estratégia de tratamento de erros
Interceptor Axios converte `ErrorResponse` em `AppError{status,error,message,path}`. TanStack Query `onError` (global, via `QueryCache`/`MutationCache` do `QueryClient`) dispara Toast padronizado via Zustand action. Mapeamento por status: 400→mensagem de validação inline no formulário quando aplicável, 404→"recurso não encontrado" + eventual redirect, 409→mensagem de conflito (ex. estado inválido de bot), erro de rede/timeout→Toast genérico com botão "tentar novamente" que chama `refetch`/`mutate` de novo.

### 15. Estratégia de Loading
`isLoading` (primeira carga) e `isFetching` (refetch em background) do TanStack Query dirigem, respectivamente, Skeleton components e uma barra de progresso fina no topo. Mutations usam `isPending` para desabilitar botão + spinner inline. Sem loading global de tela cheia fora do carregamento inicial da aplicação.

### 16. Estratégia de Cache
Cache único: TanStack Query. `staleTime` por grupo de endpoint:
- Estático (Contas/Servidores/Catálogo Macros/Configuração reconnect-policy): `staleTime` alto (ex. 5 min), invalidado só por mutation.
- Dinâmico por bot (estado, inventário, mundo): `staleTime` baixo/zero, atualizado principalmente por `setQueryData` vindo de eventos WS (`"estado"`), refetch REST como fallback ao montar componente.
- Logs: fora do cache do TanStack Query — vivem só no pub/sub + buffer local do `ConsoleLogViewer` (ver item 17).

### 17. Estratégia de Atualização em Tempo Real
- Eventos estruturais (`"estado"`, `"comando"`) → `queryClient.setQueryData`/`invalidateQueries`, mantendo TanStack Query como única fonte de estado de servidor visível pelos componentes.
- Eventos de alta frequência (`"log"`) → pub/sub mitt direto para `ConsoleLogViewer`/`useRealtimeLogs`, nunca tocam o cache do TanStack Query (evitaria re-render de toda árvore a cada linha de log).
- Sem evento dedicado de inventário/mundo (GAP §8): essas telas fazem refetch REST no foco da aba/polling leve enquanto o backend não expuser evento — não inventar heurística de invalidação baseada em `"log"`.

### 18. Estratégia de Testes
- **Unit/componente**: Vitest + React Testing Library.
- **Mock de REST**: MSW (Mock Service Worker), com handlers derivados dos tipos gerados pelo Orval (mantém mock e contrato real sincronizados).
- **E2E smoke** (golden path: criar bot → conectar → ver log → desconectar): Playwright, rodando contra o backend Java real (não mock) — execução agendada para epics de fechamento, não bloqueia início dos epics de feature.
- Cobertura mínima decidida por épico individual (não fixado aqui um número global).

### 19. Estratégia de Build
Vite (dev server porta 5173, exigida pelo CORS default do backend — `ADVANCEDBOT_CORS_ORIGINS`). `vite build` para produção. Env vars via `.env`: `VITE_API_BASE_URL`, `VITE_API_KEY` (uso só em ambiente de desenvolvimento local — build de produção não deve embutir API key real em bundle público; distribuição além de dev local exige revisão de auth, fora de escopo desta decisão). Type-check via `tsc --noEmit` e lint via ESLint+Prettier rodam em CI antes de qualquer merge.

### 20. Estratégia de integração futura com Viewer 3D
Rota/módulo isolado (`features/viewer3d/`), lazy-loaded, nunca compartilha bundle com o resto do app (Three.js só carrega quando a rota é acessada). Enquanto o backend não expuser `WS /ws/viewer/{botId}` com protocolo binário de chunks (GAP §8), o Viewer 3D consome apenas os endpoints REST já existentes (`estado`, `mundo/bloco`, `mundo/entidades`, `mundo/jogadores`) para uma representação estática/low-fidelity, sem tentar simular streaming binário no cliente. Quando o backend expuser o canal dedicado, um novo `viewerSocket` isolado é plugado nesta feature sem tocar nos outros clients WS — GAP registrado, nenhuma solução de contorno implementada agora.

---

## Roadmap do Frontend

**Fase 0 — Fundação**
Setup Vite+React+Tailwind (porta 5173). Axios `httpClient` + interceptors. `wsClient` (globalEventsSocket + factory por bot). Zustand `uiStore`. React Router (data router) + rotas base. Script Orval `generate:api` funcionando contra `/v3/api-docs` do backend. Design System base (Button, Card, Badge, Modal, Toast, DataTable, NumberInput, SearchBox). Layout global (Sidebar, TopBar, Workspace, SidebarBotList). Setup Vitest+RTL+MSW.

**Fase 1 — Features independentes (paralelizáveis)**
1. Proxy (CRUD)
2. Contas/Servidores (CRUD)
3. Catálogo de Comandos/Macros (leitura)
4. Configurações (reconnect-policy, leitura)
5. Dashboard/Métricas

**Fase 2 — Feature Bots (núcleo)**
6. Bots/list — lista, CRUD, ações (depende de Contas/Servidores)
7. Bots/details/console — logs, chat, comandos (WS `"log"`/`"comando"`)
8. Bots/details/acoes — movimento/interação
9. Bots/details/inventario — inventário/equipamento/containers
10. Bots/details/mundo — estado/bloco/entidades/jogadores
11. Bots/details/macros — macros por bot (depende do catálogo, Fase 1)

**Fase 3 — Monitoramento avançado**
12. Monitoramento (extensão do Dashboard)

**Fase 4 — Bloqueadas por GAP de backend (aguardar decisão do responsável do backend antes de alocar sprint)**
13. Spammer
14. Ferramentas (AccountChecker/BanCheck)
15. Viewer 3D (versão REST-only conforme item 20, até GAP do streaming binário ser resolvido)
16. Minerador (config fina)

## Dependências entre Features

- Bots/list depende de Contas/Servidores
- Bots/details/* depende de Bots/list (precisa de `{id}` selecionado)
- Bots/details/macros depende do Catálogo de Macros
- Monitoramento depende do mesmo endpoint de Dashboard
- Fase 4 inteira depende de trabalho de backend ainda não feito — GAPs formais em §8, não sprint de frontend até resolução

## Checklist de Implementação

- [x] Setup Vite + React + Tailwind, porta 5173 (EPIC-FRONT-01)
- [x] Script `npm run generate:api` (Orval) gerando `shared/api/generated/` a partir de `/v3/api-docs` (EPIC-FRONT-01 — validado contra o backend real, 71 arquivos gerados a partir dos 13 controllers)
- [x] `httpClient` Axios com `X-API-Key` + `AppError` (EPIC-FRONT-01)
- [x] `wsClient` (globalEventsSocket + createBotEventsSocket) com parser de `EventoDeBot`/`IdentificadorBot` + mitt bus (EPIC-FRONT-01)
- [x] Zustand `uiStore` (bot selecionado, tema persistido, sidebar) + `toastStore` (EPIC-FRONT-01)
- [x] React Router data router + rota raiz (`AppShell`) — rotas de feature e lazy Viewer3D entram no respectivo EPIC-FRONT (EPIC-FRONT-01)
- [x] Design System base + layout global (Button, Input, NumberInput, SearchBox, Card, Badge, Tooltip, Modal, ConfirmDialog, DataTable, Spinner/Skeleton, EmptyState, ErrorState, PageHeader, PageContainer, Sidebar, TopBar, Workspace, AppShell) (EPIC-FRONT-01)
- [x] Setup Vitest + RTL + MSW (EPIC-FRONT-01 — 19 testes passando; handlers por feature entram nos épicos seguintes)
- [x] Feature Proxy (EPIC-FRONT-03 — CRUD completo: listar/criar/editar (composto delete+create)/excluir com confirmação, busca e paginação client-side, validado contra backend real)
- [x] Feature Contas/Servidores (EPIC-FRONT-04 — CRUD completo de Conta e Servidor, uma rota `/contas-servidores` com abas; "editar" composto client-side, GAP de backend sem `PUT`)
- [x] Feature Catálogo Comandos/Macros (EPIC-FRONT-04 — somente leitura, rota `/catalogo` com abas, busca client-side)
- [x] Feature Configurações (EPIC-FRONT-04 — somente leitura, rota `/configuracoes`, só política de reconexão exposta pelo backend)
- [x] Feature Dashboard/Métricas (EPIC-FRONT-02 — cards de métricas, distribuição por estado, tendência de CPU/TPS, GAPs registrados explicitamente na UI)
- [x] Feature Bots/list (EPIC-FRONT-05 — CRUD completo exceto edição (GAP real de backend), ações de execução/sessão, troca de proxy, auto-reconnect, ações em lote, atualização em tempo real via WS, validado contra backend real)
- [x] Feature Bots/details/console (EPIC-FRONT-06 — logs em tempo real via WS, chat, execução de comandos, limpar logs com confirmação)
- [x] Feature Bots/details/acoes (EPIC-FRONT-06 — movimento, câmera, postura, bloco alvo, entidade alvo, usar item)
- [x] Feature Bots/details/inventario (EPIC-FRONT-06 — inventário/equipamento/janela, refetch no foco, GAP: requer sessão PLAY ativa)
- [x] Feature Bots/details/mundo (EPIC-FRONT-06 — estado/posição, consulta de bloco sob demanda, entidades/jogadores, polling leve, GAP: requer sessão PLAY ativa)
- [x] Feature Bots/details/macros (EPIC-FRONT-06 — macros ativas cruzadas com Catálogo, ativar/desativar, GAP: `tipo` retornado nem sempre bate com `nome`/`aliases` do catálogo e desativação pode não refletir com bot desconectado)
- [ ] Feature Monitoramento
- [ ] Playwright smoke e2e do golden path (pode ficar para épico de fechamento)
- [ ] Confirmar com responsável do backend a ordem de resolução dos GAPs antes de iniciar Fase 4
