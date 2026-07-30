# 08 — Proposta de Evolução: Viewer Operacional Integrado

> Status: PROPOSTA DE ARQUITETURA FUNCIONAL — não implementado, não é DEC, não altera OpenAPI.
> Base: auditoria de 2026-07-29 sobre `advancedbot-java` (real), `advancedbot-frontend` (real), e `docs-reescrita/docs/12-Interface/01` a `07`.
> Esta proposta NÃO reabre nem substitui `04-Fluxos-do-Usuario.md` (seções 1-6, histórico) nem descarta `06-Plano-Construcao-Frontend.md`/`07-Matriz-Frontend-Backend.md` como fontes de verdade — ela parte deles.
> Não referencia o projeto C# além do que já está citado nos documentos existentes.

---

## 0. Resumo executivo

O Painel Operacional atual (`PainelOperacionalPage.tsx`) já resolveu o problema central do Viewer legado: uma tela única, sem troca de aba, compondo estado do jogador, console, inventário, equipamento, mundo/entidades e macros ativas. É uma base sólida — não um protótipo a descartar.

O que falta não é reconstrução, é **densidade de informação operacional para escala** (dezenas/centenas de bots) e **estrutura temporal** (timeline de eventos, hoje inexistente tanto no backend quanto no frontend). O plano abaixo evolui incrementalmente a partir do que existe, sem esperar por Viewer 2D/3D para entregar valor.

---

## 1. Auditoria — o que já existe (resumo, ver relatório completo da sessão para detalhe file:line)

### 1.1 Backend Java — dados reutilizáveis imediatamente

| Domínio | Endpoint/Evento | DTO | Observação |
|---|---|---|---|
| Lista de bots | `GET /bots` (paginado) | `BotResponse` (enriquecido: posição, vida, macros ativas, proxy, keepalive) | já evita N+1, ideal para grade de "Painel Multi-Bot" |
| Estado individual | `GET /bots/{id}` | `BotResponse` | idem acima, 1 bot |
| Mundo/posição | `GET /bots/{id}/estado`, `GET /mundo/*` | `EstadoMundoResponse` (x,y,z,yaw,pitch,onGround,submerso,health,food,saturation,gamemode,dimension,chunkAtual) | `409` se sessão não está em PLAY — tratar como estado, não erro |
| Entidades próximas | `GET /mundo/entidades`, `/mundo/entidades/{id}`, `/mundo/jogadores` | `EntidadeResponse`, `JogadorResponse` | polling client-side hoje (3-5s) |
| Inventário | `GET /inventario`, `/equipamento`, `/janela` | `InventarioResponse`, `EquipamentoResponse`, `JanelaResponse` | completo, já integrado ao Painel |
| Ações/comandos | 14 mutations em `AcaoController` | — | cobre movimento, olhar, quebrar/colocar bloco, interagir, usar item, chat, comando |
| Macros | `GET/POST/DELETE /bots/{id}/macros` | `MacroResponse{tipo}` | `tipo` = nome de classe Java, não alias amigável (GAP conhecido) |
| Logs | `GET/DELETE /bots/{id}/logs` | buffer plano de strings | sem estrutura, sem paginação, sem timestamp por linha |
| Métricas | `GET /metricas` | `MetricasResponse`, `MemoriaResponse`, `MotorDeTickResponse` | sem threads JVM, sem rede, sem proxy-em-uso |
| WS por bot | `/ws/bots/{id}/events` | `EventoDeBot{botId,tipo,timestamp,payload}` | tipos hoje: `"log"`, `"estado"`, `"comando"` |
| WS global | `/ws/events` | idem | mesmo envelope, todos os bots |

Ponto-chave reaproveitável: **o envelope `EventoDeBot` já carrega `timestamp`** — isso é a semente de uma timeline, mesmo sem endpoint de histórico persistido. O que falta é o frontend estruturar isso como stream cronológico ao invés de efeitos pontuais em hooks isolados.

### 1.2 Frontend React — reutilizável imediatamente

- **Hooks de domínio** prontos para recompor em outro layout sem tocar backend: `useBotDetail`, `useRealtimeLogs`, `useBotConsole`, `useEstadoMundo`, `useMundoEntidades`, `useBlocoLookup`, `useInventario`+`useInventarioActions`, `useBotActionsPanel`, `useBotMacros`, `useBotEventsSocket`.
- **Componentes prontos**: `ConsolePanel`/`ConsoleLogViewer`, `InventorySlotGrid`/`ItemSlot`, `BotStatusBadge` (execução+sessão), `MetricCard`/`MetricTrendChart`/`StateBreakdownCard`, `DataTable`/`BotTable`.
- **Design System**: atoms completos (Button, Input, Select, Badge, Tooltip, Card, Skeleton, Spinner), layout (Sidebar/TopBar/Workspace/AppShell), feedback (Toast, EmptyState, ErrorState). Faltam: Checkbox/Switch/Radio dedicados, Chip/Tag, Accordion, CommandPalette.
- **Estado**: TanStack Query é o cache de domínio real (não Zustand) — qualquer proposta de "estado compartilhado do viewer" deve usar Query, não inventar um novo store global.
- **Padrão de canal WS já resolvido**: `wsClient.ts` mantém singleton global + factory por bot, com parser do envelope e do formato `{"value":"uuid"}`. Reaproveitar, não reimplementar.

### 1.3 O que NÃO existe (gaps confirmados, não implementar agora — só documentar)

- Timeline/histórico de eventos persistido no backend (logs são string plana, sem estrutura).
- Eventos WS tipados por domínio (`BOT_CONNECTED`, `INVENTORY_CHANGED`, `WORLD_ENTITY_UPDATE`, `METRICS_TICK` etc.) — hoje só 3 tipos genéricos.
- Indicador de saúde de conexão real (RTT/ping) — `msDesdeUltimoKeepAlive` existe mas não é round-trip, é tempo desde último keepalive recebido.
- Alias estável de macro ativa (hoje é nome de classe Java).
- XP do jogador (limitação de protocolo, não de UI).
- Streaming de chunks para Viewer 3D (nenhum scaffolding).
- Latência de proxy, métricas de rede/threads JVM.
- Endpoint de visão consolidada multi-bot além do `GET /bots` genérico (sem agregações tipo "quantos travados agora").

---

## 2. Diagnóstico operacional (o problema real a resolver)

Painel Operacional atual é ótimo para **1 bot por vez**. Em operação de farm com dezenas/centenas de bots, os problemas não resolvidos hoje são:

1. **Sem visão de rebanho**: para saber "quais bots estão travados/desconectados agora" o operador precisa abrir `/bots` (tabela) e depois abrir cada um. Não há sinalização agregada nem priorização visual de anomalia.
2. **Sem histórico correlacionável**: um bot que travou há 3 minutos não deixa rastro navegável — logs são texto corrido, eventos WS não são retidos além da sessão do hook.
3. **Sem indicador de travamento dedicado**: "travado" hoje é inferido implicitamente por `msDesdeUltimoKeepAlive` alto ou ausência de eventos, mas não há regra visual/threshold consolidada nem alerta.
4. **Polling em vez de push** para mundo/inventário/entidades — gera latência de percepção e custo de rede que escala mal com centenas de bots abertos simultaneamente (embora hoje só 1 bot fique aberto por vez, isso é um limitador para qualquer visão multi-bot futura).

---

## 3. Proposta de organização visual e layout

### 3.1 Princípio de camadas (não reinventa o AppShell existente)

```
AppShell (existente: Sidebar + TopBar)
└─ Workspace
   ├─ [NOVO] Barra de Enxame (Swarm Bar) — sempre visível quando >1 bot ativo
   │    mini-cards de status por bot, clicáveis, com badge de alerta
   └─ Painel Operacional (existente, evoluído)
        ├─ Coluna Esquerda: Identidade/Sessão (existente) + [NOVO] Indicadores de Saúde
        ├─ Centro: Console (existente) + [NOVO] Timeline de Eventos (aba ou split)
        ├─ Direita: Jogador/Inventário/Equipamento (existente)
        └─ Rodapé: Mundo/Entidades (existente) + [FUTURO] toggle Viewer 2D
```

Não propõe substituir o Painel Operacional — propõe **encaixar** Swarm Bar acima dele e Timeline dentro dele, reaproveitando 100% dos componentes de coluna já existentes.

### 3.2 Painel principal — informações prioritárias (ordem de prioridade operacional)

1. Estado de conexão/saúde (é o bot está vivo e respondendo?)
2. Indicador de travamento (está vivo mas não progride?)
3. Posição + dimensão + alvo atual
4. Macro ativa (o que ele acha que está fazendo)
5. Vida/fome (risco de morte iminente)
6. Últimos comandos/ações (o que aconteceu agora há pouco)
7. Inventário (estado de recursos, menos urgente que os acima)
8. Entidades próximas (contexto de ameaça/oportunidade)

Essa ordem já é quase a ordem real do `PainelOperacionalPage.tsx` atual — a mudança proposta é tornar 1 e 2 explícitos como indicadores visuais dedicados (hoje implícitos em `BotStatusBadge` + campo cru `msDesdeUltimoKeepAlive`).

### 3.3 Indicadores propostos (usando dado que já existe, sem exigir backend novo)

| Indicador | Fonte de dado atual | Regra visual proposta |
|---|---|---|
| Saúde de conexão | `msDesdeUltimoKeepAlive` + presença de eventos WS recentes | Verde (<Xs), Amarelo (X-Ys), Vermelho (>Ys ou WS desconectado) — threshold configurável, não fixo no código |
| Atividade | timestamp do último `EventoDeBot` tipo `"estado"`/`"comando"` recebido | "Ativo há Ns" — se ultrapassa threshold sem nenhum evento novo, sinaliza possível travamento |
| Travamento | combinação: posição/estado sem mudança + macro ativa declarada + nenhum evento novo | Heurística client-side inicial (sem endpoint dedicado) — ver EPIC-VIEWER-01 |
| Macro ativa | `MacroResponse.tipo` (nome de classe) | Exibir com tooltip explicando que é nome técnico, não alias (transparência do GAP, no padrão já usado por `DashboardGapNotice`) |

### 3.4 Timeline de eventos (novo conceito, sem exigir persistência nova no início)

> Reforçado como prioridade máxima do roadmap (acima do Overlay de Debug 3.5) — motivador direto trazido em revisão: no ADV antigo também faltava um feed cronológico simples de "o que aconteceu, quando", nível suficiente pra pegar bug de macro sem precisar de detalhe fino. Exemplo de granularidade-alvo (não precisa ser mais detalhado que isso):

```
09:35:12  Conectou
09:35:15  Macro Miner iniciada
09:35:18  Moveu para X=120
09:35:20  Bloco quebrado
09:35:21  Inventário alterado
09:35:24  Recebeu dano
09:35:27  Chat: "Você encontrou minério"
09:35:31  Girou 90°
09:35:33  Começou Twerk
09:35:40  Desconectou
```

Ponto-chave: cada linha já corresponde a um evento/ação que já existe no backend hoje (conectar/desconectar via `BotController`, ativação de macro via `MacroResponse`/evento `"comando"`, movimento e quebra de bloco via `AcaoController` + evento `"estado"`, inventário via evento correlato a `InventarioResponse`, dano via `EstadoMundoResponse.health` decrescente, chat via evento `"log"`/`"comando"` de chat). Não é preciso esperar por eventos tipados por domínio (EPIC-VIEWER-03) pra ter essa timeline básica — dá pra montar isso client-side agora, com heurística simples de "mudou X, registra linha", igual ao overlay de debug da seção 3.5.

Proposta em 2 estágios:

- **Estágio 1 (frontend-only, sem mudança de backend)**: agregar client-side, por bot, um buffer ordenado de `EventoDeBot` recebidos via WS (log/estado/comando) + respostas de mutations REST relevantes (ex. resultado de ação), com timestamp local de recebimento. Renderizar como lista cronológica filtrável por tipo. Isso é uma composição nova de dado que já existe — não pede endpoint novo.
- **Estágio 2 (evolução futura de backend, documentada, não implementada)**: endpoint de histórico persistido por bot com paginação e filtro por tipo/tempo, para sobreviver a reload de página e permitir auditoria pós-incidente.

---

## 3.5 Norteador do Viewer legado — debug visual de macro (`ViewForm.cs`, doc 01 §1.11)

O Viewer 3D legado (`ViewForm`) já resolvia um problema que o Painel atual não resolve: overlay de debug mostrando FPS, coordenadas da câmera, **bloco apontado** e **informação de entidade**, com toggle F4. Esse overlay é a peça que falta hoje — sem ele, log só diz `"Minerando..."`, nunca `"Minerando onde, olhando pra quê, quebrando qual bloco"`.

### 3.5.1 Problema concreto (motivador desta seção)

Macro de mineração pode estar, sem que o log denuncie:

- olhando pra lado errado;
- quebrando bloco errado;
- presa em parede;
- girando infinitamente;
- clicando no ar;
- sem detectar entidade;
- andando em círculo.

Log mostra `"Minerando..."`. Não responde "minerando onde". Isso é sintoma direto de ausência de Viewer — não é falta de log, é falta de **correlação espacial** entre ação declarada e estado real do mundo.

### 3.5.2 O que já existe hoje para resolver isso sem esperar Viewer 3D

Todo dado necessário pro overlay de debug do legado **já está exposto pelo backend atual**, sem endpoint novo:

| Dado do overlay legado | Fonte atual (já existe) |
|---|---|
| Coordenadas da câmera/jogador | `EstadoMundoResponse.{x,y,z}` |
| Direção (yaw/pitch) | `EstadoMundoResponse.{yaw,pitch}` |
| Bloco apontado | Não existe cálculo de raycast no backend hoje — mas `GET /mundo/bloco?x=&y=&z=` permite consultar bloco em qualquer coordenada, incluindo a projetada a partir de yaw/pitch (raycast client-side simples, sem exigir mudança de backend) |
| Info de entidade | `GET /mundo/entidades/{entityId}` + `EntidadeResponse` |
| Ação em execução | Última ação REST disparada (`AcaoController`) + macro ativa (`MacroResponse.tipo`) — hoje sem correlação visual, só textual |

Ou seja: dá pra construir um **overlay de debug 2D** (não precisa ser 3D) hoje, calculando client-side o ponto mirado a partir de posição+yaw+pitch e sobrepondo isso num canvas top-down com o bloco/entidade-alvo destacado. Isso é estritamente mais barato que o Viewer 3D completo e entrega o mesmo valor de debug que motivou a pergunta.

### 3.5.3 Elemento central proposto: "o que o bot pensa que está fazendo" vs "onde ele realmente está"

O overlay de debug do legado tinha, embutido, algo que a proposta anterior (seção 5, Viewer 2D) descrevia como visualização espacial genérica — esta seção eleva isso a requisito explícito: o Viewer 2D não é só "ver posição", é **justapor 3 camadas no mesmo desenho**:

1. Posição real + direção real (linha/seta de mira a partir de yaw/pitch);
2. Bloco/entidade que a mira intercepta agora (raycast client-side sobre `GET /mundo/bloco`/`GET /mundo/entidades`);
3. Rótulo da ação/macro declarada no momento (`"Minerando"` + qual bloco-alvo a macro reportou, se disponível via evento `"comando"`).

Quando (1)+(2) divergem do que (3) diz que deveria estar acontecendo, o operador vê o bug na hora — sem precisar inferir do log.

---

## 4. Inventário integrado, chat integrado, entidades, jogador — o que muda

- **Inventário**: já integrado (`InventorySlotGrid` no Painel). Proposta: nenhuma mudança estrutural — só garantir que fique visível sem exigir scroll em telas de operação (avaliar densidade quando Swarm Bar for adicionada acima).
- **Chat**: `POST /bots/{id}/chat` já existe; hoje entra pelo Console genérico junto com logs de sistema. Proposta: no Console, permitir filtro visual "chat vs sistema vs comando" — isso é possível hoje via string matching client-side (log é texto plano), não exige mudança de backend para o filtro básico. Estrutura real do lado do servidor (campo `categoria` no evento) é melhoria futura documentada no backlog (seção 6).
- **Entidades próximas**: já exibido agrupado (`agruparEntidades`) no rodapé do Painel. Proposta: manter, mas destacar quando alvo atual (via ação `olhar-entidade`/`interagir-entidade` mais recente) corresponde a uma entidade da lista — correlação client-side entre última ação e lista de entidades, sem endpoint novo.
- **Jogador/coordenadas/direção**: já exposto via `EstadoMundoResponse`. Nenhuma mudança de dado, só de destaque visual (indicadores da seção 3.3).
- **Alvo atual / últimos comandos**: hoje só "últimos 5 comandos" na coluna esquerda. Proposta: isso se funde na Timeline de Eventos (seção 3.4) em vez de manter como lista separada e desconectada.

---

## 5. Integração futura com Viewer 2D e 3D

Não implementar agora. Documentar caminho:

- **Viewer 2D**: mais barato — pode ser derivado inteiramente de dados já existentes (`EstadoMundoResponse.chunkAtual`, `EntidadeResponse`, `JogadorResponse`) plotados em canvas/SVG top-down. Não depende de nenhum endpoint novo para uma primeira versão (mapa de blocos ao redor pode vir de `GET /mundo/bloco` chamado em grade, com custo de N chamadas — aceitável para um raio pequeno, não escalável para épocas futuras sem endpoint de "bloco em lote").
- **Viewer 3D**: depende do gap já identificado — `WS /ws/viewer/{botId}` de chunks binários não existe. Bloqueado até essa evolução de backend (fora de escopo desta sessão, só documentado).
- Ambos devem entrar no Painel Operacional como um **toggle de painel adicional** (reaproveitando o Workspace/Drawer do AppShell), não como página separada — mantém o princípio de tela única já estabelecido pelo EPIC-PROD-03.

---

## 6. Evoluções de backend desejáveis (documentar apenas, não implementar)

| Evolução | Motivação | Prioridade sugerida |
|---|---|---|
| Eventos WS tipados por domínio (`BOT_CONNECTED`, `INVENTORY_CHANGED`, `WORLD_ENTITY_UPDATE`, `MACRO_CHANGED`, `METRICS_TICK`) | Elimina polling client-side de mundo/inventário/entidades, essencial para escalar Swarm Bar a centenas de bots | ALTA |
| Alias estável em `MacroResponse` (nome de classe → nome amigável do catálogo) | Elimina inconsistência UI já documentada, necessário para indicador de macro confiável | ALTA |
| Endpoint de histórico de eventos persistido, paginado, por bot | Sustenta Timeline de Eventos além da sessão de página | MÉDIA |
| Campo de saúde de conexão real (RTT) em vez de só tempo-desde-keepalive | Indicador de saúde de conexão mais preciso | MÉDIA |
| `GET /bots?resumo=true` ou endpoint de agregação (contagem por estado, contagem de travados) | Swarm Bar em escala de centenas não deve exigir client agregar `GET /bots` inteiro a cada poll | MÉDIA |
| `WS /ws/viewer/{botId}` de chunks binários | Pré-requisito único do Viewer 3D | BAIXA (não bloqueia nada do roadmap incremental abaixo) |
| Categoria estruturada de evento de log (chat/sistema/comando/erro) | Filtro de Console deixa de depender de string matching | BAIXA |
| Métricas de rede/threads JVM/proxy-em-uso | Fecha gaps já sinalizados no próprio `DashboardGapNotice` | BAIXA |

---

## 7. Classificação de melhorias

### ALTA

**7.1 — Swarm Bar (visão de rebanho)**
- Problema operacional: nenhuma visão agregada de múltiplos bots sem abrir um a um.
- Impacto: alto — é o gargalo nº1 de operação em escala.
- Solução proposta: barra de mini-cards reaproveitando `BotStatusBadge` + `MetricCard`, alimentada por `GET /bots` já existente (paginado/polled), com badge de alerta usando heurística client-side de travamento (seção 3.3).
- Componentes envolvidos: novo componente composto a partir de `BotStatusBadge`, `Card`, `Tooltip`, hook `useBotList` já existente.
- Dependências: nenhuma de backend para v1 (usa GAP-aware: `GET /bots` genérico, sem agregação — aceitar polling reduzido).
- Esforço estimado: médio (2-3 dias, é composição de existente).

**7.2 — Indicadores de saúde/atividade/travamento explícitos**
- Problema: sinal de "algo errado" só existe implícito em campo cru.
- Impacto: alto — reduz tempo de detecção de incidente.
- Solução: 3 indicadores visuais (seção 3.3) na coluna esquerda do Painel e nos mini-cards da Swarm Bar.
- Componentes: `BotStatusBadge` extendido, novo `HealthIndicator` (a nomear no design system quando implementado).
- Dependências: nenhuma nova de backend (thresholds configuráveis no frontend).
- Esforço estimado: pequeno (1-2 dias).

**7.3b — Overlay de Debug Visual (mira + bloco/entidade alvo + ação declarada)**
- Problema operacional: log textual ("Minerando...") não responde onde/o quê/pra onde — bug de macro (olhar errado, quebrar bloco errado, preso em parede, girar infinito, clicar no ar, não detectar entidade, andar em círculo) fica invisível até virar prejuízo.
- Impacto: alto — é a lacuna de debug citada como motivador desta revisão, reduz drasticamente tempo de diagnóstico de macro.
- Solução proposta: canvas 2D top-down client-side (seção 3.5) combinando posição+direção real (`EstadoMundoResponse`), raycast client-side sobre bloco/entidade mirado (`GET /mundo/bloco`, `GET /mundo/entidades`) e rótulo da ação/macro declarada (evento `"comando"` + `MacroResponse.tipo`), destacando divergência entre o que a macro diz fazer e o que está de fato acontecendo no mundo.
- Componentes envolvidos: novo `DebugOverlayCanvas`, reaproveita `useEstadoMundo`, `useMundoEntidades`, `useBlocoLookup`, `useBotConsole`.
- Dependências: nenhuma de backend para v1 (raycast é geometria client-side sobre dado já exposto).
- Esforço estimado: médio (3-5 dias) — é o item de maior valor/esforço da lista ALTA.

**7.3 — Timeline de Eventos (estágio 1, client-side)** — PRIORIDADE MÁXIMA da lista ALTA
- Problema: histórico de ações/comandos disperso, sem correlação cronológica. Considerado pelo próprio operador como melhoria mais importante do roadmap, acima até do overlay visual (7.3b) — feed de linha do tempo simples ("Conectou", "Moveu para X=120", "Bloco quebrado", "Recebeu dano", "Chat: ...", "Girou 90°", "Desconectou") já é suficiente pra pegar bug de macro sem exigir visualização espacial.
- Impacto: alto — essencial para diagnosticar "o que aconteceu antes de travar".
- Solução: buffer ordenado client-side agregando WS+REST já recebidos, renderizado como lista filtrável, substituindo "últimos 5 comandos" isolado.
- Componentes: novo `EventTimeline`, reaproveitando `useBotEventsSocket`, `useBotConsole`.
- Dependências: nenhuma de backend no estágio 1.
- Esforço estimado: médio (3-4 dias).

### MÉDIA

**7.4 — Eventos WS tipados por domínio (backend)**
- Problema: polling de mundo/inventário/entidades não escala para visão multi-bot.
- Impacto: médio-alto no médio prazo, mas não bloqueia nada hoje (só 1 bot aberto por vez atualmente).
- Solução: proposta documentada na seção 6, não implementar agora.
- Componentes: `EventoDeBot`, `NotificadorDeEventos`, todos os hooks de polling do frontend passam a assinar em vez de poll.
- Dependências: mudança de backend + adaptação de hooks existentes.
- Esforço estimado: grande (backend: média; frontend: pequena, pois hooks já existem, só trocam polling por assinatura).

**7.5 — Alias estável de macro ativa**
- Problema: `tipo` = nome de classe Java, inconsistente com catálogo.
- Impacto: médio — confunde operador, mas tem workaround de tooltip hoje.
- Solução: backend passa a resolver alias a partir do catálogo ao montar `MacroResponse`.
- Componentes: `MacroResponse`, `CatalogoController`, `useBotMacros`.
- Dependências: backend.
- Esforço estimado: pequeno-médio.

**7.6 — Endpoint de agregação de bots (`resumo`/contagens)**
- Problema: Swarm Bar em centenas de bots não deve recalcular tudo client-side a cada poll.
- Impacto: médio, cresce com escala.
- Solução: endpoint novo de agregação (seção 6), documentar agora, implementar quando volume justificar.
- Componentes: `BotController`, Swarm Bar.
- Dependências: backend.
- Esforço estimado: pequeno (backend) uma vez priorizado.

**7.7 — Toggle de Viewer 2D no Painel**
- Problema: sem visualização espacial, operador infere posição só por números.
- Impacto: médio — ajuda mas não bloqueia operação atual.
- Solução: canvas/SVG top-down usando dados já existentes (seção 5).
- Componentes: novo `Viewer2DPanel`, `useEstadoMundo`, `useMundoEntidades`, `useBlocoLookup`.
- Dependências: nenhuma de backend para v1 limitada em raio pequeno.
- Esforço estimado: médio-grande.

### BAIXA

**7.8 — Histórico de eventos persistido (estágio 2 da Timeline)**
- Problema: Timeline se perde ao recarregar página.
- Impacto: baixo no curto prazo (operação contínua raramente recarrega), alto para auditoria pós-incidente.
- Solução: endpoint de histórico paginado (seção 6).
- Dependências: backend + storage.
- Esforço estimado: grande.

**7.9 — Categoria estruturada de log (chat/sistema/comando/erro)**
- Problema: filtro de console depende de string matching.
- Impacto: baixo, workaround client-side já viável.
- Solução: campo `categoria` no evento de log.
- Dependências: backend.
- Esforço estimado: pequeno.

**7.10 — Viewer 3D**
- Problema: overlay 2D (7.3b) mostra mira/alvo/ação de forma esquemática, mas não mostra como o mundo real parece ao redor do bot nem a interação de fato acontecendo dentro do servidor. Confirmado como requisito real do operador (não só "nice to have") — objetivo explícito: ver mira e ação da conta em contexto visual completo do mundo.
- Impacto: alto valor de diagnóstico/confiança operacional, mas não bloqueia operação diária (Timeline + Overlay 2D cobrem a maior parte do debug de macro sem ele) — por isso sequenciado por último, não por baixa importância, e sim por maior esforço e por depender de peça de backend inexistente.
- Solução: streaming de chunks binários (seção 5), render Three.js/react-three-fiber no frontend, view sob perspectiva do bot ou freecam (equivalente ao `ViewForm` legado, doc 01 §1.11).
- Dependências: `WS /ws/viewer/{botId}` (não existe — pré-requisito de backend inteiro), engine de render 3D no frontend.
- Esforço estimado: muito grande — maior item de todo o roadmap.

**7.11 — Métricas de rede/threads JVM/proxy-em-uso**
- Problema: Dashboard tem gaps já sinalizados (`DashboardGapNotice`).
- Impacto: baixo — não bloqueia operação de bots, só observabilidade de infraestrutura.
- Solução: instrumentação adicional no backend.
- Dependências: backend.
- Esforço estimado: médio.

---

## 8. Plano de implementação — épicos incrementais

> Ordem de execução sugerida (não é numeração de prioridade): **EPIC-VIEWER-02 primeiro** — Timeline de Eventos é o item de maior valor/menor esforço apontado nesta revisão, sem dependência de backend. EPIC-VIEWER-01 e 04 podem seguir em paralelo ou logo depois.

### EPIC-VIEWER-01 — Painel Operacional Integrado (Swarm + Saúde)
- Objetivo: dar visão de rebanho e detecção rápida de anomalia sem exigir mudança de backend.
- Escopo: Swarm Bar (7.1) + Indicadores de saúde/atividade/travamento (7.2).
- Dependências: nenhuma de backend. Depende de `GET /bots`, `useBotList`, `BotStatusBadge`, `MetricCard` existentes.
- Riscos: heurística de travamento client-side pode gerar falso positivo/negativo sem endpoint dedicado — mitigar com threshold configurável e rótulo "estimado" na UI.
- Critérios de aceite: operador consegue, sem abrir bot individual, identificar quais bots estão saudáveis/degradados/travados a partir da tela de lista ou de dentro de um bot aberto.

### EPIC-VIEWER-02 — Timeline de Eventos (estágio 1)
- Objetivo: substituir "últimos comandos" isolado por stream cronológico correlacionável.
- Escopo: 7.3, client-side apenas.
- Dependências: `useBotEventsSocket`, `useBotConsole` existentes. Nenhuma de backend.
- Riscos: buffer client-side se perde em reload — comunicar limitação na UI (mesmo padrão de transparência do `DashboardGapNotice`).
- Critérios de aceite: dentro do Painel, operador vê lista cronológica combinando logs, comandos e mudanças de estado relevantes, filtrável por tipo, sem sair da tela.

### EPIC-VIEWER-03 — Eventos de Domínio no Backend
- Objetivo: eliminar polling de mundo/inventário/entidades e habilitar Timeline persistente.
- Escopo: 7.4 (eventos tipados) + 7.5 (alias de macro) + 7.9 (categoria de log).
- Dependências: EPIC-VIEWER-01/02 em produção (para validar quais eventos realmente importam antes de fixar contrato de backend).
- Riscos: mudança de contrato WS pode quebrar consumidores existentes do envelope genérico — exige versionamento ou período de coexistência dos tipos antigos (`"log"`,`"estado"`,`"comando"`) com os novos.
- Critérios de aceite: hooks de mundo/inventário/entidades passam de polling para assinatura reativa; macro ativa exibe alias correto; sem regressão nas telas existentes.

### EPIC-VIEWER-04 — Viewer 2D + Overlay de Debug Visual
- Objetivo: visualização espacial leve reaproveitando dados já expostos, com foco primário em debug de macro (motivador direto: "não consigo ver onde está o bug da macro").
- Escopo: 7.7 (Viewer 2D) + 7.3b (Overlay de Debug: mira, bloco/entidade alvo, ação declarada), tratados como uma entrega única — o overlay de debug É o Viewer 2D em sua v1, não uma camada posterior.
- Dependências: idealmente após EPIC-VIEWER-03 (eventos push), mas viável antes com polling atual em escopo reduzido.
- Riscos: custo de N chamadas para bloco em grade sem endpoint de lote — limitar raio na v1, documentar necessidade futura de endpoint de bloco em lote. Raycast client-side é aproximação geométrica sobre snapshot discreto de blocos, pode divergir de decisão real do servidor em bordas de bloco — comunicar como "estimado", igual padrão `DashboardGapNotice`.
- Critérios de aceite: painel opcional mostra posição real, direção (seta de mira), bloco/entidade que a mira intercepta agora e rótulo da ação/macro declarada, lado a lado, em vista top-down, sem travar a UI. Operador consegue, olhando só pra esse painel, responder "o bot está fazendo o que ele diz que está fazendo, no lugar certo?" sem ler log.

### EPIC-VIEWER-05 — Histórico Persistido e Auditoria
- Objetivo: sustentar Timeline além da sessão de página, habilitar diagnóstico pós-incidente.
- Escopo: 7.8.
- Dependências: EPIC-VIEWER-03 (formato de evento já estabilizado antes de persistir).
- Riscos: volume de dados em operação de centenas de bots — exige política de retenção/paginação desde o design do endpoint.
- Critérios de aceite: histórico de eventos de um bot sobrevive a reload e é consultável por período/tipo.

### EPIC-VIEWER-06 — Viewer 3D
- Objetivo: confirmado pelo operador como objetivo real do roadmap — visualização imersiva de onde a conta está mirando, o que está fazendo e como está interagindo dentro do servidor, além do que overlay 2D/Timeline já entregam em texto/esquema.
- Escopo: 7.10 — streaming de chunks binários, render 3D (Three.js/react-three-fiber), perspectiva do bot ou freecam (equivalente funcional ao `ViewForm` legado).
- Dependências: `WS /ws/viewer/{botId}` (não existe — pré-requisito de backend inteiro, precisa ser especificado/implementado antes deste épico começar), engine de render 3D no frontend.
- Riscos: maior esforço de todo o roadmap; deve ser sequenciado por último não por baixa prioridade, mas porque Timeline (EPIC-VIEWER-02) e Overlay 2D (EPIC-VIEWER-04) cobrem a maior parte do debug de macro com esforço muito menor primeiro.
- Critérios de aceite: operador abre visão 3D de um bot e vê, em tempo real, o mundo ao redor, pra onde ele mira e a ação/interação em curso (quebrar bloco, atacar entidade, etc.) — nível de detalhe equivalente ao `ViewForm` legado (overlay de coordenadas, bloco apontado, info de entidade) sobre a arquitetura Java/React atual.

---

## 9. O que esta proposta deliberadamente não decide

- Não fixa nomes finais de componentes/eventos (isso é DEC, fora de escopo desta sessão).
- Não altera `06-Plano-Construcao-Frontend.md` nem `07-Matriz-Frontend-Backend.md` — quando EPIC-VIEWER-01 em diante for de fato implementado, a Matriz 07 deve ser atualizada seguindo sua própria regra de manutenção (só quando endpoint/evento novo existir de fato).
- Não propõe descartar nenhum componente ou hook existente — todo o plano é aditivo sobre a base real auditada.
