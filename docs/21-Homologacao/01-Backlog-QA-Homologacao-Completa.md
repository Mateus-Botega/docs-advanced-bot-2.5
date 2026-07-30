# 21.01 - Backlog Técnico de Homologação Completa (EPIC-QA-01)

> Última atualização: 2026-07-28
>
> Autor: sessão de homologação assistida (Claude Code).
>
> Escopo desta sessão: **apenas comparação funcional prática** entre o comportamento esperado do AdvancedBot e o comportamento real observado no AdvancedBot Java, através do uso da aplicação rodando com backend real (Spring Boot + PostgreSQL) e servidor Minecraft real. **O código-fonte C# não foi consultado** — todos os achados abaixo vêm exclusivamente da observação de comportamento (UI, rede, API, logs).
>
> Nenhum código foi alterado nesta sessão. Nenhuma DEC foi aberta. Nenhuma decisão arquitetural foi tomada. Este documento é só backlog.

---

## 1. Ambiente de homologação usado

- Backend: `advancedbot-java`, buildado com Maven 3.9.9 + JDK 21 (Microsoft Build of OpenJDK), rodando em `localhost:8080`.
  - Observação de processo: o repositório não tinha Maven Wrapper (`mvnw`/`mvnw.cmd`) nem Maven instalado na máquina, e o jar em `target/` estava desatualizado em relação ao código-fonte (havia `.java` mais recentes que o `.jar`). Foi necessário baixar o Maven manualmente para gerar um build atual antes de homologar. Ver item **GAP-08** abaixo.
- Banco: PostgreSQL 18.1 local, schema já migrado pelo Flyway (`V1__contas_servidores_proxies.sql`, `V2__bots.sql`).
- Frontend: `advancedbot-frontend` (Vite/React), `localhost:5173`, consumindo a API real via `X-API-Key`.
- Servidor Minecraft real usado na conexão: `olimpo.clmc.com.br:3737` (CraftLandia Olimpo), com conta cadastrada `Solk`. Login, chat do servidor, lista de jogadores online e mundo são 100% reais, não mockados.

## 2. Cenários efetivamente executados

| Cenário | Executado | Observação |
|---|---|---|
| Conectar | ✔ | Login real no servidor, console mostrou MOTD real |
| Desconectar | ✔ | Via lista de bots |
| Reconectar | ✔ | Inclusive cenário de falha (proxy inválida), ver BUG-04 |
| Auto reconnect (ligar/desligar) | ✔ | Toggle na lista de bots |
| Trocar proxy | ✔ | Ver BUG-04 e MUX-01 |
| Iniciar/parar/pausar bot | ✔ | Via lista de bots |
| Executar comando (console) | ✔ | `help` executado com sucesso no servidor real |
| Ativar/desativar macro | ✔ | Miner ativado e "desativado" — ver **BUG-01** (crítico) |
| Abrir baú / interagir com container | ✖ (não existe) | Ver **GAP-01** |
| Movimentação | ✔ (parcial) | Testado "Balançar braço"; ações de mover/olhar não exercitadas com dados reais nesta sessão por prudência com o servidor de terceiros |
| Mineração, Pesca, Mob, Herbalismo | Parcial | Miner ativado e revisado via API; demais macros não ativadas de fato no servidor real nesta sessão (risco de efeito colateral em servidor de terceiros); backend confirma que os 6 tipos de macro do catálogo respondem a POST/DELETE do mesmo endpoint genérico |
| Console | ✔ | Execução de comando e leitura de log real |
| Inventário | ✔ | Página completa revisada, ver **GAP-02** |
| Mundo | ✔ | Entidades próximas e jogadores online reais listados |
| Painel operacional (por bot) | ✔ | Aba "Painel Operacional" do bot |
| Excluir bot | ✔ (achou bug) | Ver **BUG-02** |
| Contas e Servidores (CRUD) | ✔ | Listagem confirmada, dados reais pré-existentes |
| Proxy (CRUD) | ✔ | Criação testada, ver MUX-02 |
| Catálogo (comandos/macros) | ✔ | Ver MV-01 |
| Configurações | ✔ | Ver GAP-03 |
| Dashboard/métricas | ✔ | Ver GAP-04 |

---

## 3. Achados

Cada achado tem um ID estável (`BUG-xx`, `GAP-xx`, `MUX-xx` = Melhoria UX, `MPF-xx` = Melhoria Performance, `MV-xx` = Melhoria Visual) para referência cruzada em épicos futuros.

### BUG-01 — Desativar macro não remove a macro no bot real (falso positivo de sucesso) — ✅ RESOLVIDO (Sprint QA-01, 2026-07-28)

> **Status:** Corrigido no backend (`MacroController`/`Comando`/`GerenciadorDeComandos`, ver [11-Estado-Atual-Migracao.md](../99-Governanca/11-Estado-Atual-Migracao.md#sprint-qa-01-2026-07-28--correção-dos-bugs-críticos-do-backlog-bug-01-bug-02-bug-04)). Validado end-to-end contra o bot real (Solk/olimpo.clmc.com.br): ativar Miner pela UI, clicar "Desativar", `GET /macros` confirma lista vazia.

**Classificação:** BUG (crítico)

**Descrição:** Ao ativar uma macro (ex.: Miner) via `POST /api/v1/bots/{id}/macros/{chaveDoCatalogo}` (ex. `miner`), o backend passa a rastreá-la internamente pelo nome de classe da tarefa (ex. `TarefaMineracao`). O botão "Desativar" da aba Macros do frontend chama `DELETE /api/v1/bots/{id}/macros/{nomeDaTarefa}` (ex. `TarefaMineracao`) — mas o backend espera a chave de catálogo (`miner`) nesse mesmo endpoint. O DELETE com o nome errado retorna `200 OK` e a UI mostra a notificação "Macro desativada", mas a macro **continua ativa e em execução no bot real**.

**Como reproduzir:**
1. Conectar um bot em um servidor real.
2. Na aba "Macros", ativar qualquer macro do catálogo (ex. Miner).
3. Clicar "Desativar" na macro ativa.
4. Observar a notificação de sucesso "Macro desativada".
5. Consultar `GET /api/v1/bots/{id}/macros` — a macro ainda aparece na lista.
6. Confirmação do causador: `DELETE /api/v1/bots/{id}/macros/miner` (chave de catálogo) remove de fato; `DELETE /api/v1/bots/{id}/macros/TarefaMineracao` (nome da tarefa, o que a UI realmente envia) responde 200 mas não remove nada.

**Impacto:** Alto. O operador acredita ter parado uma macro (ex. mineração, farm de mob, pesca) que continua rodando sem controle no bot conectado a um servidor real de terceiros — risco de dano ao personagem (queda em lava, gasto de recursos, banimento por comportamento incontrolável) sem qualquer aviso na interface.

**Prioridade:** Crítica — deve ser tratado antes de qualquer uso operacional real do sistema com macros.

**Proposta de solução:** Padronizar o identificador usado nas duas pontas do ciclo de vida da macro. Duas opções:
 (a) Front-end passar a enviar sempre a chave de catálogo no DELETE (precisa mapear tarefa ativa → chave de catálogo, o que a resposta de `GET /macros` hoje não expõe — só devolve `tipo`, ver GAP relacionado);
 (b) Back-end aceitar tanto a chave de catálogo quanto o nome de classe da tarefa no mesmo endpoint DELETE, normalizando internamente.
Qualquer que seja a opção, adicionar teste de integração que ative e desative uma macro e confirme, via segunda consulta, que ela realmente saiu da lista de macros ativas.

**Dependências:** Nenhuma dependência externa. Fix isolado no controller/serviço de macros ativas (backend) e/ou no client HTTP do frontend.

---

### BUG-02 — Botão "Excluir" bot não executa nenhuma ação — ✅ RESOLVIDO / NÃO REPRODUZIDO (Sprint QA-01, 2026-07-28)

> **Status:** Não reproduzido no código-fonte atual do frontend — `BotTable.tsx`, `BotsPage.tsx` e `ConfirmDialog.tsx` já têm o handler, o diálogo de confirmação e o `DELETE /bots/{id}` corretamente ligados. Validado ao vivo: bot descartável criado, "Excluir" clicado, diálogo confirmado, bot removido da lista. Hipótese mais provável é que o comportamento observado na homologação original refletia um build de frontend desatualizado (mesma classe de problema do GAP-06/GAP-08). Nenhuma alteração de código foi necessária — ver [11-Estado-Atual-Migracao.md](../99-Governanca/11-Estado-Atual-Migracao.md#sprint-qa-01-2026-07-28--correção-dos-bugs-críticos-do-backlog-bug-01-bug-02-bug-04).

**Classificação:** BUG

**Descrição:** Na lista de Bots, o botão "Excluir" de uma linha não dispara nenhuma requisição de rede, não abre diálogo de confirmação e não produz notificação. O bot permanece na lista indefinidamente. Confirmado por inspeção de rede (zero requisições após o clique, em duas tentativas) e por chamada direta `DELETE /api/v1/bots/{id}`, que funciona corretamente (204) quando feita fora da UI.

**Como reproduzir:**
1. Ir para a lista de Bots com pelo menos um bot cadastrado.
2. Clicar em "Excluir" na linha do bot.
3. Nada acontece — sem diálogo, sem notificação, sem requisição de rede.

**Impacto:** Alto. Não há forma de remover um bot pela interface; único caminho é acesso direto à API ou ao banco.

**Prioridade:** Alta.

**Proposta de solução:** Revisar o binding do evento `onClick` do botão "Excluir" na tabela de bots — outros botões da mesma linha (Pausar, Parar, Desconectar, Reconectar, Proxy, Auto ON/OFF) funcionam corretamente, então o problema é local a este botão específico. Provavelmente falta o handler ou o diálogo de confirmação não foi conectado ao componente de tabela.

**Dependências:** Nenhuma.

---

### BUG-03 — `GET /api/v1/bots/{id}/commands` retorna 500 em vez de erro semântico — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** `GlobalExceptionHandler` capturava `HttpRequestMethodNotSupportedException` no handler genérico (`Exception.class`) e devolvia 500 — adicionado handler dedicado que devolve 405. Validado: `GET /commands` agora responde `405 Method Not Allowed` com mensagem específica.

**Classificação:** BUG

**Descrição:** Uma requisição `GET` no endpoint usado pelo POST de execução de comando (`/api/v1/bots/{id}/commands`) retorna `500 Internal Server Error` com mensagem genérica ("Erro interno ao processar a requisição"), em vez de `404`/`405` (método não suportado nesse verbo) ou de fato devolver o histórico de comandos, se essa for a intenção do endpoint.

**Como reproduzir:**
```
curl -H "X-API-Key: dev-local-key" http://localhost:8080/api/v1/bots/{id}/commands
```
Resposta: `500` com corpo `{"message":"Erro interno ao processar a requisição", ...}`.

**Impacto:** Médio. Não afeta o fluxo feliz (POST de execução funciona), mas indica que o tratamento de erros do controller de comandos é genérico demais — outros erros reais de execução de comando podem estar sendo mascarados da mesma forma (500 genérico em vez de mensagem específica).

**Prioridade:** Média.

**Proposta de solução:** Revisar o `ComandoController` (ou equivalente) para responder corretamente a verbos não suportados (405) e, se houver necessidade de expor histórico de comandos executados via GET, implementar esse endpoint de fato em vez de deixá-lo cair no handler de erro genérico.

**Dependências:** Nenhuma.

---

### BUG-04 — Troca de proxy não é aplicada à conexão ativa; auto-reconnect alterna proxies sozinho sem aviso na UI — ✅ RESOLVIDO (Sprint QA-01, 2026-07-28)

> **Status:** Corrigido no backend, duas causas raízes distintas — ver [11-Estado-Atual-Migracao.md](../99-Governanca/11-Estado-Atual-Migracao.md#sprint-qa-01-2026-07-28--correção-dos-bugs-críticos-do-backlog-bug-01-bug-02-bug-04). (1) `CasoDeUsoTrocarProxy` agora força reconexão imediata quando o bot já está conectado. (2) `GerenciadorDeReconexao` não rotaciona mais a proxy em qualquer falha genérica — só no motivo de kick documentado ("muitas contas"), fiel ao legado. Validado ao vivo contra o bot real Solk: troca de proxy em bot conectado força reconexão real (posição do personagem mudou); proxy inválida cadastrada retorna erro real de conexão em vez de sucesso silencioso; 4 checagens consecutivas com auto-reconnect ligado confirmaram que a proxy não alterna mais sozinha. Bot restaurado ao estado normal (conectado, sem proxy) ao final do teste.

**Classificação:** BUG / comportamento não documentado (requer confirmação de regra de negócio)

**Descrição:** Duas observações relacionadas:
1. Trocar a proxy de um bot **já conectado** (via botão "Proxy" na lista de bots) atualiza o registro (`PUT /api/v1/bots/{id}/proxy` → 204, notificação "Proxy do bot atualizada") mas **não desconecta nem reconecta o bot** — ele continua "Conectado" usando a conexão TCP direta original, sem passar pela nova proxy. A nova proxy só é usada na próxima tentativa de conexão/reconexão.
2. Com Auto-reconnect ativado e a proxy configurada apontando para um SOCKS5/HTTP inexistente, o motor de reconexão automática, a cada nova tentativa, **alternou sozinho entre as duas proxies cadastradas no sistema** (`127.0.0.1:1080 SOCKS5` ⇄ `10.0.0.5:3128 HTTP`) sem nenhuma ação do operador e sem qualquer indicação visual de que isso estava acontecendo — só foi percebido consultando `GET /api/v1/bots/{id}` repetidamente.

**Como reproduzir:**
1. Conectar um bot sem proxy.
2. Cadastrar 2 proxies em "Proxy" (ex. uma SOCKS5 e uma HTTP, ambas sem servidor real por trás).
3. No bot conectado, clicar "Proxy" e trocar para a proxy SOCKS5 inválida. Observar que o bot continua "Conectado" (sem proxy real).
4. Ativar "Auto reconnect" no bot e forçar uma falha de conexão (ex. clicar Reconectar). Observar `GET /api/v1/bots/{id}` repetidas vezes — o campo `proxy` muda de valor entre as tentativas sem intervenção manual.

**Impacto:** Alto. Comportamento não documentado em lugar nenhum da interface; o operador pode achar que uma proxy específica está em uso quando o sistema já trocou para outra automaticamente. Se alguma das proxies rotacionadas expuser IP real (ex. do próprio operador) em vez de anonimizar, isso pode ser um risco operacional sério dependendo do caso de uso do bot.

**Prioridade:** Alta — precisa de esclarecimento antes de decidir se é bug a corrigir ou funcionalidade a documentar/expor.

**Proposta de solução:**
- Confirmar com o responsável de produto/negócio se "rotação automática de proxy em falha de reconexão" é um comportamento intencional (herdado do sistema legado) ou uma falha de implementação.
- Se intencional: expor claramente no painel qual proxy está sendo tentada em cada ciclo de reconexão, e documentar a regra.
- Se não intencional: corrigir para que o motor de reconexão sempre reutilize a proxy explicitamente configurada no bot, sem trocar sozinho.
- Independentemente da decisão acima, corrigir o item 1: trocar a proxy de um bot conectado deveria deixar claro que a mudança só vale a partir da próxima reconexão (mensagem explícita), ou forçar reconexão automaticamente com confirmação do usuário.

**Dependências:** Esclarecimento de regra de negócio (não é possível decidir isso só olhando o comportamento observado; e por instrução desta sessão o código C# não foi consultado).

---

### GAP-01 — Não existe ação de UI para abrir/interagir com baú (container) — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** Adicionado formulário "Clicar bloco (abrir baú / interagir)" na aba Ações (X/Y/Z + botão esquerdo/direito), reaproveitando o comando `ClicarBloco` já catalogado via o endpoint genérico `POST /commands` (sem endpoint REST novo). Ao concluir, invalida a query de `GET /inventario/janela` para a seção "Janela" do Inventário atualizar sozinha. Validado contra o bot real: requisição disparada corretamente (confirmado via rede — coordenada fora de alcance retornou `{"resultado":"ERRO"}`, sem efeito colateral no mundo real).

**Classificação:** GAP

**Descrição:** Não há nenhum botão na aba "Ações" do bot para clicar/interagir com um bloco (o que seria necessário para abrir um baú, fornalha, etc.). O catálogo de comandos lista um comando genérico `ClicarBloco` (`clicarbloco <x> <y> <z> <botão 0-1>`) que plausivelmente cobre esse caso de uso, mas ele só pode ser disparado manualmente digitando o comando cru na aba Console — não há campo/formulário dedicado. A aba "Inventário" já tem uma seção "Janela" preparada para exibir o conteúdo de um container aberto (`GET /api/v1/bots/{id}/inventario/janela`), mas como não há como abrir uma janela pela UI, essa seção nunca é exercitada (retorna 404 "Bot não tem nenhuma janela/container aberto" sempre).

**Como reproduzir:**
1. Abrir um bot conectado, ir para a aba "Ações". Não há nenhum botão relacionado a interagir/clicar em bloco.
2. Ir para a aba "Inventário" — seção "Janela" sempre mostra "Nenhuma janela aberta".
3. Único caminho funcional hoje: digitar `clicarbloco X Y Z 1` manualmente no Console.

**Impacto:** Médio-alto para qualquer fluxo que dependa de containers (baú, fornalha, funil): usuário não tem forma amigável de operar essas interações.

**Prioridade:** Alta (o roteiro de homologação cita "abrir baú" como cenário esperado).

**Proposta de solução:** Adicionar na aba "Ações" um formulário "Clicar bloco" (X, Y, Z, botão) reaproveitando o comando `ClicarBloco` já catalogado, e garantir que a seção "Janela" da aba Inventário atualize e exiba o conteúdo assim que uma janela for aberta.

**Dependências:** Nenhuma — backend já expõe `ClicarBloco` no catálogo e `GET /inventario/janela`; o trabalho é só de frontend.

---

### GAP-02 — Botões de slot de inventário/equipamento sem nome acessível — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** `ItemSlot` agora usa o mesmo texto do tooltip como `aria-label` ("Slot N: Vazio" ou "Slot N: #itemId ×count"); slots de inventário, hotbar, janela e cursor passam `label` explícito. Validado via árvore de acessibilidade real: todos os botões de slot têm nome (ex. `"Slot 36: #3 ×24"`).

**Classificação:** GAP (acessibilidade)

**Descrição:** Na aba "Inventário" (painel e página dedicada), todos os slots de equipamento e inventário são renderizados como `<button>` sem `aria-label` nem texto identificável — aparecem na árvore de acessibilidade apenas como `button` genérico, exceto quando têm uma quantidade textual visível (ex. "24"). Um leitor de tela não consegue anunciar qual item está em cada slot.

**Como reproduzir:** Inspecionar a árvore de acessibilidade da aba/página de Inventário com um bot conectado com itens.

**Impacto:** Médio. Afeta acessibilidade e também dificulta testes automatizados de UI (não há como selecionar um slot específico por nome).

**Prioridade:** Média.

**Proposta de solução:** Adicionar `aria-label` dinâmico em cada botão de slot com o nome do item (ex. "Slot 3: Picareta de Diamante x1") e "Slot vazio" quando aplicável.

**Dependências:** Nenhuma.

---

### GAP-03 — Tela "Configurações" é somente leitura — ⏸ ADIADO (Sprint QA-02, 2026-07-28)

> **Status:** Fora do escopo da Sprint QA-02 por decisão explícita do responsável — depende de decisão de produto sobre quais parâmetros ficam editáveis via UI, não é bug técnico.

**Classificação:** GAP

**Descrição:** A tela "Configurações" exibe apenas 2 valores fixos (intervalo base e jitter máximo da política de reconexão), sem nenhuma forma de edição. Não há campos para editar API key, CORS, timeouts, ou qualquer outro parâmetro operacional — qualquer mudança exige editar `application.yml`/variáveis de ambiente e reiniciar o backend.

**Como reproduzir:** Acessar `/configuracoes` — não há nenhum input, apenas texto.

**Impacto:** Médio. Impede operação/ajuste fino do sistema sem acesso ao servidor/deploy.

**Prioridade:** Média.

**Proposta de solução:** Definir com o responsável de produto quais parâmetros devem de fato ser editáveis via UI (ex.: política de reconexão) versus quais devem continuar exclusivos de infraestrutura (ex.: credenciais de banco, API key) e implementar formulário para os primeiros.

**Dependências:** Decisão de produto sobre escopo de configuração exposta via UI.

---

### GAP-04 — Lacunas de métricas já autodetectadas pelo próprio dashboard — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** As 3 lacunas fechadas no backend: contagem de threads via `ManagementFactory.getThreadMXBean()` (`MetricasResponse.threadCount`), proxy associada a bot via join em `ProxyController.listar()` (`ProxyResponse.botsEmUso`), heap vs. non-heap via `ManagementFactory.getMemoryMXBean()` (`MemoriaResponse.naoHeapUsadaMb`). Frontend: Dashboard ganhou cards "Memória (Non-Heap)" e "Threads da JVM" (removido o `DashboardGapNotice`, que ficou obsoleto); tabela de Proxy ganhou coluna "Em uso por". Validado ao vivo: Dashboard mostrando 48 threads e memória non-heap reais; tabela de Proxy com a coluna nova.

**Classificação:** GAP

**Descrição:** O próprio Dashboard já expõe uma seção "Métricas não disponíveis no backend (GAP)" listando 3 lacunas conhecidas: threads da JVM não exposta, proxy em uso por bot não associável no `ProxyResponse`, e heap vs. memória não-heap não diferenciados (`MemoriaResponse` só expõe heap). Achado confirmado nesta sessão via observação direta da tela — registrado aqui para garantir que entre no backlog formal e não fique "perdido" apenas como nota de UI.

**Impacto:** Baixo/médio, depende de necessidade de observabilidade operacional em produção.

**Prioridade:** Baixa/Média.

**Proposta de solução:** Se for necessário observabilidade mais completa: expor contagem de threads via `ManagementFactory.getThreadMXBean()`, associar proxy a bot no `ProxyResponse` (join simples), e adicionar heap/non-heap separados via `MemoryMXBean`.

**Dependências:** Nenhuma.

---

### GAP-05 — Semântica ambígua entre "Iniciar" e "Conectar" — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** Adicionado tooltip explicativo nos botões "Iniciar" e "Conectar" da lista de bots, documentando que "Conectar" também inicia automaticamente. Comportamento em si não foi alterado (nenhuma regra de negócio nova) — só documentação inline. Validado via DOM real (tooltip renderiza o texto esperado ao focar/passar o mouse).

**Classificação:** GAP (dúvida de especificação)

**Descrição:** Um bot recém-criado aparece com estado `Parado`/`Desconectado` e dois botões distintos, "Iniciar" e "Conectar". Ao clicar apenas em "Conectar" (sem antes clicar "Iniciar"), o bot passou diretamente para `Executando`/`Conectado` — ou seja, "Conectar" parece implicitamente também "Iniciar" o bot. Não ficou claro, apenas pelo uso da aplicação, se isso é intencional (Conectar = Iniciar + Conectar) ou se há um caso em que os dois precisam ser acionados separadamente.

**Como reproduzir:** Criar um bot novo, sem clicar em "Iniciar", clicar direto em "Conectar" — observar que o estado de execução também muda para "Executando".

**Impacto:** Baixo — não impede o uso, mas gera confusão sobre o que cada botão faz de fato, especialmente para quem só conhece o comportamento pela UI.

**Prioridade:** Baixa.

**Proposta de solução:** Documentar explicitamente (tooltip ou texto de ajuda) a relação entre estado de execução e estado de sessão, e o que cada botão realmente aciona.

**Dependências:** Nenhuma.

---

### MUX-01 — Diálogo "Trocar proxy" não reaproveita proxies cadastradas nem permite remover proxy — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** `BotProxyModal` ganhou combobox "Proxy cadastrada" (lista as proxies de `/proxy`, com "Digitar manualmente" como fallback) e botão dedicado "Remover proxy" (envia host vazio, que `ProxySupport.resolver` já trata como "sem proxy"). Validado ao vivo: modal do bot Solk mostra as 2 proxies cadastradas no combobox e o botão "Remover proxy".

**Classificação:** Melhoria UX

**Descrição:** O diálogo de troca de proxy de um bot só oferece campos manuais (Host/Porta/Tipo), mesmo quando já existem proxies cadastradas na tela "Proxy". Também não há opção de "remover proxy" (voltar o bot para conexão direta) — o único jeito de "sair" de uma proxy é digitar dados de outra proxy manualmente.

**Como reproduzir:** Cadastrar 2+ proxies em "Proxy". Abrir o diálogo "Proxy" de um bot — nenhuma delas aparece pré-preenchida ou selecionável; não há botão "remover proxy".

**Impacto:** Médio — atrito operacional, risco de erro de digitação, e ausência de rollback rápido para conexão direta.

**Prioridade:** Média.

**Proposta de solução:** Adicionar combobox de seleção entre proxies cadastradas (com opção de digitar manualmente como fallback) e um botão explícito "Remover proxy" no mesmo diálogo.

**Dependências:** Nenhuma.

---

### MUX-02 — Formulário "Nova proxy" não reseta estado do campo "Tipo" entre aberturas — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** Causa raiz: a `key` do `ProxyFormModal` no modo "create" era sempre a mesma string literal, então React nunca remontava o componente entre aberturas (o `useState` interno mantinha o valor anterior). `ProxyPage` agora incrementa um contador a cada clique em "Nova proxy" e inclui esse contador na `key`, forçando remount (e reset do formulário) toda vez.

**Classificação:** Melhoria UX

**Descrição:** Depois de cadastrar uma proxy escolhendo um "Tipo" (ex. SOCKS5), ao reabrir o modal "Nova proxy" para cadastrar a próxima, o campo "Tipo" já aparece pré-selecionado com o valor usado da última vez, em vez de voltar ao placeholder "Selecione o tipo".

**Como reproduzir:** Cadastrar uma proxy com Tipo = SOCKS5. Fechar e reabrir "Nova proxy" — o combobox de Tipo já vem com SOCKS5 selecionado.

**Impacto:** Baixo — risco de cadastrar proxy com tipo errado por descuido, mas fácil de perceber e corrigir.

**Prioridade:** Baixa.

**Proposta de solução:** Resetar o estado do formulário (todos os campos) toda vez que o modal é aberto.

**Dependências:** Nenhuma.

---

### MUX-03 — Mensagem de erro genérica ao falhar reconexão ("Conflito de estado") — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** Backend (`FabricaDeConexaoMinecraftV1_8`) passou a anexar a causa raiz da falha de conexão na mensagem (timeout, host desconhecido, conexão recusada, sem rota até o host), em vez de só "Falha ao conectar em host:porta". Frontend (`errorMessages.ts`) usa esse prefixo estável para escolher um título específico ("Timeout de conexão", "Host desconhecido", "Conexão recusada", "Host inacessível") em vez do genérico "Conflito de estado" — sem precisar de novo status/campo no contrato REST. Conflitos de estado reais (não relacionados a conexão) continuam com o título genérico.

**Classificação:** Melhoria UX

**Descrição:** Ao falhar uma tentativa de reconexão (ex. proxy inválida), a notificação exibida tem o título "Conflito de estado" — que não descreve a causa real (falha ao conectar via proxy/host). O texto secundário ("Falha ao conectar em host:porta") é mais informativo, mas o título de destaque induz o operador a pensar em um problema de concorrência/estado interno, não em uma falha de rede/proxy comum.

**Como reproduzir:** Configurar uma proxy inválida em um bot e clicar "Reconectar" — a notificação de erro usa o título "Conflito de estado".

**Impacto:** Baixo/médio — dificulta diagnóstico rápido pelo operador, especialmente em uso não técnico.

**Prioridade:** Média.

**Proposta de solução:** Diferenciar mensagens por causa raiz (timeout, proxy inválida, autenticação recusada, host desconhecido etc.), usando título compatível com o problema real em vez de um rótulo genérico de conflito HTTP (409).

**Dependências:** Nenhuma.

---

### MPF-01 — Polling individual e frequente por bot aberto, sem consolidação — ⏸ ADIADO (Sprint QA-02, 2026-07-28)

> **Status:** Fora do escopo da Sprint QA-02 por decisão explícita do responsável — exige decisão arquitetural maior (WebSocket/SSE vs. consolidar em snapshot), o próprio backlog já pedia para não abrir DEC sem priorização dedicada.

**Classificação:** Melhoria Performance

**Descrição:** Ao manter a página de um bot aberta (Painel Operacional), o frontend dispara repetidamente, em intervalo curto, requisições `GET` separadas para `/estado`, `/mundo/entidades`, `/mundo/jogadores` (e outras, dependendo da aba) — observado dezenas de requisições em poucos segundos de uso durante esta sessão. Não há evidência de uso de WebSocket/SSE para atualização em tempo real; tudo parece ser polling via REST.

**Como reproduzir:** Abrir o painel de um bot conectado e observar a lista de requisições de rede por um curto período — repetição de `GET .../estado`, `.../mundo/entidades`, `.../mundo/jogadores` em sequência muito próxima.

**Impacto:** Baixo com poucos bots/abas abertas; pode não escalar bem com múltiplos bots e múltiplos operadores monitorando ao mesmo tempo (carga desnecessária no backend e no banco/estado em memória).

**Prioridade:** Baixa (hoje), Média se o número de bots simultâneos crescer.

**Proposta de solução:** Avaliar consolidar essas chamadas em um único endpoint de "snapshot" do painel, ou migrar para push via WebSocket/SSE para estado de bot conectado, reduzindo o número de requisições HTTP independentes.

**Dependências:** Decisão arquitetural (fora do escopo desta sessão — não abrir DEC agora, só registrar a necessidade).

---

### MV-01 — Macro "Twerk" sem descrição no Catálogo — ✅ RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** `ComandoTwerk.descricao()` preenchida ("Fica agachando e levantando repetidamente."). Validado: aba Macros do Catálogo mostra a descrição.

**Classificação:** Melhoria Visual

**Descrição:** Na aba "Macros" da tela "Catálogo", todas as macros têm uma descrição preenchida na coluna "Descrição", exceto "Twerk", que aparece com a célula vazia.

**Como reproduzir:** Acessar `/catalogo`, aba "Macros" — linha "Twerk" tem a coluna Descrição em branco.

**Impacto:** Muito baixo — cosmético/inconsistência de dados de catálogo.

**Prioridade:** Baixa.

**Proposta de solução:** Preencher a descrição da macro Twerk no catálogo (fonte de dados do catálogo, não é alteração de código de produção).

**Dependências:** Nenhuma.

---

### GAP-06 — Build não reprodutível em máquina limpa (sem Maven Wrapper, jar desatualizado) — ✅ PARCIALMENTE RESOLVIDO (Sprint QA-02, 2026-07-28)

> **Status:** Maven Wrapper adicionado (`mvnw`/`mvnw.cmd`/`.mvn/wrapper/`, gerado via `mvn wrapper:wrapper -Dmaven=3.9.9`). `.gitignore` criado (o repositório não tinha nenhum) excluindo `target/`; os ~1500 arquivos de `target/` que já estavam versionados foram destrackeados (`git rm -r --cached target`, arquivos continuam em disco). Não totalmente fechado: a garantia de "jar sempre refeito antes de homologação/deploy" depende de processo/CI, fora do escopo desta sessão.

**Classificação:** GAP (DX/CI, não funcional)

**Descrição:** O repositório `advancedbot-java` não possui Maven Wrapper (`mvnw`/`mvnw.cmd`), nem Maven estava instalado na máquina usada para esta homologação. Além disso, o artefato `target/advancedbot-java-1.0.0-SNAPSHOT.jar` estava desatualizado em relação ao código-fonte (havia dezenas de arquivos `.java` mais recentes que o `.jar`), o que teria produzido uma homologação sobre código antigo caso não fosse detectado e corrigido antes de começar.

**Como reproduzir:** Em uma máquina limpa, sem Maven instalado, tentar buildar o projeto sem baixar/instalar Maven manualmente — não há caminho (`mvnw`) disponível no próprio repositório.

**Impacto:** Alto para onboarding/CI/homologação reprodutível — qualquer nova máquina (ou pipeline de CI) precisa de Maven pré-instalado manualmente, e não há garantia de que o jar versionado (se algum dia for versionado) reflita o código-fonte atual.

**Prioridade:** Alta (bloqueia reprodutibilidade de build/homologação).

**Proposta de solução:** Adicionar Maven Wrapper ao repositório (`mvn wrapper:wrapper`) e garantir que `target/` esteja no `.gitignore` (não versionar artefatos buildados) ou, se necessário versionar, garantir que o build seja sempre refeito antes de qualquer homologação/deploy.

**Dependências:** Nenhuma.

---

## 4. Agrupamento em épicos futuros

| Épico proposto | Achados incluídos | Motivação do agrupamento |
|---|---|---|
| **EPIC-QA-01.1 — Confiabilidade de Estado de Macros e Bots** | BUG-01, BUG-02 | Ambos são casos onde a UI reporta sucesso mas o estado real do backend/bot não muda — risco operacional direto |
| **EPIC-QA-01.2 — Confiabilidade de Reconexão e Proxy** | BUG-04, MUX-01, MUX-03 | Tudo relacionado ao ciclo de vida de proxy/reconexão e à comunicação desse ciclo ao operador |
| **EPIC-QA-01.3 — Tratamento de Erros de API** | BUG-03 | Padronização de respostas de erro do backend |
| **EPIC-QA-01.4 — Interação com Containers/Mundo** | GAP-01 | Fecha a lacuna de "abrir baú" e interações de bloco |
| **EPIC-QA-01.5 — Acessibilidade e Consistência de Catálogo** | GAP-02, MUX-02, MV-01 | Itens de qualidade de interface de baixo risco, agrupáveis numa mesma leva de polimento |
| **EPIC-QA-01.6 — Configuração Operacional** | GAP-03 | Depende de decisão de produto sobre superfície de configuração |
| **EPIC-QA-01.7 — Observabilidade** | GAP-04, GAP-05 | Métricas e clareza de estado/semântica de botões |
| **EPIC-QA-01.8 — Performance de Atualização de Painel** | MPF-01 | Possível trabalho arquitetural maior (WebSocket/SSE) — precisa de DEC própria quando for priorizado |
| **EPIC-QA-01.9 — Reprodutibilidade de Build** | GAP-06 | DX/CI, não é funcional do produto mas bloqueia homologações futuras |

---

## 5. Cenários do roteiro original ainda não exercitados a fundo nesta sessão

Por prudência (o servidor de destino é real, de terceiros, e macros como Pesca/Mob/Herbalismo alteram estado do personagem no mundo), os seguintes cenários foram só parcialmente cobertos e merecem uma nova rodada dedicada, idealmente em servidor de teste próprio:

- Macros Pesca, Mob, Herbalismo, DropAll, Twerk, AntiAFK, Follow — só Miner foi de fato ativado/observado no servidor real.
- Movimentação de fato (mover/olhar com coordenadas reais) e quebra/colocação de bloco — testado só "Balançar braço" como ação inofensiva.
- Cenário de reconexão por queda de rede real (só foi simulada falha via proxy inválida, não uma queda de conexão de fato).

---

## 6. EPIC-PROD-04 — Auditoria de Produtividade Operacional (2026-07-28)

> Escopo: **exclusivamente experiência de uso/produtividade operacional** do AdvancedBot Java —
> Dashboard, Painel Operacional, Lista de Bots, Sidebar, fluxo de criação de Bot, navegação,
> monitoramento e escalabilidade visual para dezenas/centenas de bots. **Não é uma busca por bugs
> nem por problemas arquiteturais** (isso já foi coberto nas seções 1-5 acima, EPIC-QA-01). Código
> C# não foi consultado — auditoria feita sobre o estado atual do frontend/backend Java, docs de
> `docs-reescrita/docs/12-Interface/` e o próprio backlog de homologação. Nenhuma DEC foi aberta.

### Achados classificados

Cada achado: problema, impacto operacional, solução proposta, componentes envolvidos,
dependências, esforço estimado, prioridade.

#### OPX-01 — Dashboard sem nenhum atalho para a lista de Bots — ALTA — ✅ IMPLEMENTADO

- **Problema**: Nenhum `MetricCard` (Bots conectados/executando/pausados/desconectados) nem linha do
  `StateBreakdownCard` era clicável. O operador via "23 bots pausados" no Dashboard e precisava ir
  manualmente para `/bots` e procurar esses 23 bots por conta própria, sem filtro herdado.
- **Impacto operacional**: Alto com dezenas/centenas de bots — o Dashboard, que deveria ser o ponto
  de partida da operação, virava um beco sem saída informativo.
- **Solução implementada**: `MetricCard`s de estado e linhas do `StateBreakdownCard` agora navegam
  para `/bots?estado=<ESTADO>`, reaproveitando o novo filtro por estado (OPX-02).
- **Componentes**: `MetricCard`, `StateBreakdownCard` (prop `onItemClick` nova, opcional,
  retrocompatível), `DashboardPage`. Nenhum componente novo.
- **Dependências**: OPX-02 (filtro por estado na lista de Bots).
- **Esforço**: Pequeno (poucas horas).

#### OPX-02 — Lista de Bots só filtra por texto livre, sem filtro por estado — ALTA — ✅ IMPLEMENTADO

- **Problema**: Único campo de busca (`SearchBox`) filtrava por username/host/porta/estado como
  texto livre — sem um filtro dedicado e explícito por estado de execução/sessão.
- **Impacto operacional**: Alto para escala — achar "todos os pausados" entre 100+ bots exigia
  digitar o nome exato do estado e ainda misturava com host/username que contivessem o mesmo texto.
- **Solução implementada**: `Select` "Filtrar por estado" (Todos/Executando/Pausado/Parado/
  Conectado/Conectando/Desconectado) ao lado do `SearchBox`, usando nova função
  `filterBotsByEstado` (`features/bots/services/filterBots.ts`), combinável com a busca textual.
- **Componentes**: `Select` (já existente), `BotsPage`. Nenhum componente novo.
- **Dependências**: Nenhuma.
- **Esforço**: Pequeno.

#### OPX-03 — "Selecionar todos" da lista de Bots só cobre a página atual (10 itens) — ALTA — ✅ IMPLEMENTADO

- **Problema**: Com paginação fixa de 10 itens, "selecionar todos" nunca selecionava mais que os 10
  bots da página exibida — para agir em lote sobre 100+ bots o operador precisava repetir a seleção
  página por página.
- **Impacto operacional**: Alto — é o ponto que mais diretamente contradiz o objetivo de
  "escalabilidade visual para dezenas/centenas de bots": ações em lote reais em escala eram
  inviáveis pela UI.
- **Solução implementada**: Quando a página inteira está selecionada e existem mais bots além dela
  no recorte filtrado atual, aparece o botão "Selecionar todos os N bots filtrados" (e,
  inversamente, "Limpar seleção" quando todos já estão selecionados). Baseado no novo
  `filteredIds` retornado por `useBotTableState`.
- **Componentes**: `Button` (variant ghost, já existente), `BotsPage`. Nenhum componente novo.
- **Dependências**: OPX-02 (o recorte filtrado precisa existir para "todos os filtrados" fazer
  sentido).
- **Esforço**: Pequeno.

#### OPX-04 — Busca/filtro/página da lista de Bots perdidos ao navegar para um bot e voltar — ALTA — ✅ IMPLEMENTADO

- **Problema**: Página/busca eram `useState` local de `BotsPage`; ao clicar num bot (`/bots/:id`) e
  voltar, o componente era desmontado/remontado e o contexto (busca, página) resetava para o
  estado inicial.
- **Impacto operacional**: Alto — em uma lista de centenas de bots, é comum abrir o Painel
  Operacional de um bot específico a partir de um recorte filtrado/paginado e depois querer voltar
  exatamente para aquele recorte, não para a página 1 sem filtro.
- **Solução implementada**: `useBotTableState` reescrito para persistir `q`/`estado`/`page` na URL
  (`useSearchParams` do `react-router-dom`, já em uso no projeto) em vez de `useState` local. Isso
  também é o que viabiliza o deep-link do Dashboard (OPX-01) — `?estado=PAUSADO` na URL. `SearchBox`
  ganhou prop opcional `defaultValue` (retrocompatível) para refletir o termo restaurado da URL.
- **Componentes**: `SearchBox` (prop nova opcional), `useBotTableState` (reescrito, hook específico
  de Bots — não altera o `useTableState` genérico usado por Proxy/Contas/Servidores/Catálogo).
- **Dependências**: Nenhuma.
- **Esforço**: Médio (poucas horas, envolveu ajustar 2 testes existentes que colidiam com as novas
  opções do `Select` de estado — `Executando`/`Conectado` apareciam tanto como opção do filtro
  quanto como badge da linha).

#### OPX-05 — Ações em lote disparam N requisições paralelas sem limite nem progresso incremental — ALTA — ⏸ NÃO IMPLEMENTADO NESTA SESSÃO

- **Problema**: `useBatchBotAction` dispara `Promise.allSettled` com uma chamada HTTP por bot
  selecionado, todas em paralelo, sem limite de concorrência; o botão fica em loading até todas
  resolverem e só então mostra um toast agregado — sem barra de progresso incremental.
- **Impacto operacional**: Alto em escala real (centenas de bots) — uma ação em lote sobre 200 bots
  dispara 200 requisições simultâneas, sem visibilidade de progresso durante a espera.
- **Solução proposta**: Limitar concorrência (ex.: lote de N em N) no client, e usar `ProgressBar`
  (componente já especificado no Design System, seção 2, mas ainda não implementado) para progresso
  incremental "Processando X de N".
- **Componentes envolvidos**: `useBatchBotAction`, novo componente atômico `ProgressBar`.
- **Dependências**: Nenhuma DEC — mas exige construir um componente do Design System ainda não
  implementado (`ProgressBar`), o que ultrapassa o critério "reutilizar exclusivamente componentes
  existentes" definido para esta sessão. Recomendado para uma sprint dedicada.
- **Esforço**: Médio.

#### OPX-06 — Sidebar sem indicador de contagem de bots por estado — MÉDIA — não implementado

- **Problema**: `NavItem` já tem uma prop `badge?: ReactNode` não utilizada por nenhum item de
  navegação — nenhum badge (ex.: contagem de bots pausados/desconectados) aparece no menu lateral.
- **Impacto operacional**: Médio — o operador só percebe bots parados/desconectados ao entrar no
  Dashboard ou na lista de Bots; a sidebar (sempre visível) não dá esse sinal ambiente.
- **Solução proposta**: Alimentar a prop `badge` do item "Bots" com a contagem de bots que exigem
  atenção (pausados + desconectados + erro).
- **Componentes envolvidos**: `Sidebar`/`AppShell`, `Badge` (já existente).
- **Dependências**: Precisa de uma fonte de dados montada globalmente (hoje as métricas só são
  buscadas na página Dashboard) — adicionar isso ao layout fixo implica um novo polling sempre
  ativo, o que foi considerado fora do escopo "sem mudança arquitetural" desta sessão.
- **Esforço**: Pequeno a médio.

#### OPX-07 — Criação de Bot é sempre unitária, sem criação/importação em lote — MÉDIA — não implementado

- **Problema**: `BotFormModal` cria exatamente 1 bot por submissão; não há campo de quantidade,
  upload de lista nem "duplicar bot". Para 50-200 bots o operador repete o fluxo completo N vezes.
- **Impacto operacional**: Médio-alto para provisionamento inicial de uma frota grande, mas não
  afeta a operação do dia a dia (bots já criados).
- **Solução proposta**: Endpoint de lote no backend (`POST /bots/batch` ou similar) + formulário de
  importação (CSV/lista colada) no frontend.
- **Componentes envolvidos**: Novo endpoint backend, `BotFormModal` ou tela nova de importação.
- **Dependências**: Requer nova API no backend — mudança arquitetural, precisa de DEC própria.
  Fora do escopo desta sessão (regra explícita: não abrir novas DEC).
- **Esforço**: Grande.

#### OPX-08 — Virtualização de linhas do `DataTable` nunca ativa na tela de Bots — BAIXA — não implementado

- **Problema**: `DataTable` tem lógica de virtualização com threshold de 200 linhas, mas `BotTable`
  sempre recebe no máximo 10 linhas (paginação client-side), então a virtualização nunca entra em
  ação — não é um problema hoje, só uma capacidade não utilizada.
- **Impacto operacional**: Baixo — vira relevante só se o tamanho de página for aumentado no
  futuro (ex.: para reduzir cliques de paginação em 200 bots).
- **Solução proposta**: Nenhuma ação necessária agora; reavaliar se o `PAGE_SIZE` da lista de Bots
  for aumentado.
- **Componentes envolvidos**: `DataTable`, `useBotTableState`.
- **Dependências**: Nenhuma.
- **Esforço**: N/A (nenhuma ação nesta sessão).

#### OPX-09 — Custo agregado de WebSocket + polling por bot com Painel Operacional aberto — BAIXA (nesta sessão) — não implementado

- **Problema**: Cada bot com Painel Operacional/Console aberto mantém um WebSocket dedicado mais 2
  pollings HTTP de 3s enquanto a página estiver montada. Já registrado como **MPF-01** na seção 3
  deste documento (adiado na Sprint QA-02 por decisão explícita do responsável — exige decisão
  arquitetural maior).
- **Impacto operacional**: Seria alto com muitas abas de Painel Operacional abertas
  simultaneamente, mas o padrão de uso (um operador acompanhando um bot por vez) reduz o impacto
  prático hoje.
- **Solução proposta**: Ver MPF-01 — consolidar em snapshot único ou WebSocket dedicado para
  vida/coordenadas/entidades.
- **Dependências**: Decisão arquitetural — não abrir DEC agora, só manter registrado (já coberto por
  MPF-01, sem achado novo aqui).
- **Esforço**: N/A (já registrado, sem ação nova).

### Resumo do que foi implementado nesta sessão (EPIC-PROD-04)

Implementados **somente** os achados ALTA que não alteram arquitetura, não exigem DEC nova,
reaproveitam componentes existentes e mantêm o Design System atual: **OPX-01, OPX-02, OPX-03,
OPX-04**. OPX-05 é ALTA mas foi propositalmente deixado para uma sprint dedicada por exigir um
componente do Design System ainda não construído (`ProgressBar`). OPX-06 a OPX-09 (MÉDIA/BAIXA)
não foram implementados, conforme escopo definido para esta sessão.

**Validação**: build (`tsc -b`), lint (`eslint .`) e suíte de testes do frontend (120 testes, 26
arquivos, 2 ajustados para não colidir com as novas opções do filtro de estado) passando. Validado
ao vivo contra o backend real (Spring Boot + PostgreSQL) e o bot real `Solk` conectado a
`olimpo.clmc.com.br:3737`: clique em "Bots conectados" no Dashboard navegou para
`/bots?estado=CONNECTED` com o filtro já aplicado e selecionado no `Select`; navegação para o
Painel Operacional do bot Solk e volta (botão "voltar" do navegador) preservou o filtro na URL;
sem erros de console.

---

## 7. EPIC-VIEWER-01 — Evolução do Painel Operacional (2026-07-29)

> Escopo: exclusivamente evoluir o Painel Operacional já existente (`PainelOperacionalPage.tsx`,
> Milestone 45/EPIC-PROD-03) com indicadores de saúde/atividade/travamento e direção do jogador —
> **nenhuma página nova, nenhum endpoint/evento novo, nenhuma DEC**. Baseado na proposta
> `docs-reescrita/docs/12-Interface/08-Proposta-Viewer-Operacional.md`.

### Auditoria prévia (obrigatória antes de qualquer código, confirma o que já existia)

Contra o código atual (não contra a proposta original, que já previa "Swarm Bar" multi-bot fora
deste épico — reduzido de propósito nesta sessão para um único bot, por instrução direta):

- **Painel Operacional já reúne** status/conta/servidor/proxy/auto-reconnect/macros ativas/últimos
  comandos (coluna esquerda), console completo (`ConsolePanel`, centro), vida/fome/coordenadas/
  dimensão/equipamento/inventário/janela (coluna direita) e entidades próximas/jogadores online/
  consulta de bloco (rodapé) — confirmado lendo `PainelOperacionalPage.tsx` linha a linha. **Não
  precisou ser recriado**, só estendido.
- **"Entidades próximas" já mostra nome real de mob/nick de jogador** (`agruparEntidades`,
  `RegistroDeMobs`) e **inventário já mostra ícone+nome real do item** (`ItemSlot`, sprite sheet
  extraído do legado) — corrigido numa sessão anterior (registro de 2026-07-28 na Milestone 45,
  sem número de épico formal). A proposta do doc 08 (seção 1.2) já sinalizava isso como "já
  reaproveitável"; confirmado que também já está com dado de exibição correto, não só cru.
- **`BotStatusBadge` já existe e já é usado** na coluna de status (execução + sessão, dois badges).
- **`msDesdeUltimoKeepAlive` existe em `BotResponse`/`EstadoMundoResponse` mas nunca era lido em
  nenhum componente** (`grep` confirmado só em `shared/api/generated/models`) — GAP do doc 08
  confirmado ainda válido, é o dado-base usado agora para o indicador de Saúde de conexão.
  Chatgrupo em `estado?.msDesdeUltimoKeepAlive ?? bot?.msDesdeUltimoKeepAlive` que a proposta
  apontava mede o tempo desde o keepalive de rede, é uma leitura de estado no momento da resposta
  HTTP, não um RTT real (mesma limitação já registrada no doc 08, DEC-41 aplicável).
- **Nenhum indicador de "travamento aparente" existia** — confirmado ausente em toda a busca por
  "travad"/"lock" no frontend. Implementado nesta sessão como heurística puramente client-side.
- **Direção (yaw/pitch) já vinha do backend** (`EstadoMundoResponse`/`PosicaoResponse`) mas nunca
  era exibida no Painel (só coordenadas x/y/z) — GAP de exibição simples, corrigido.
- **`MPF-01`/`OPX-09` (polling+WebSocket agregado por bot aberto) permanecem adiados** — decisão
  arquitetural fora de escopo desta sessão, não tocada. O `timestamp` do envelope WS (`EventoDeBot`)
  já existia e já trafegava por `wsBus`, mas nenhum hook o capturava — usado agora (novo hook
  `useUltimaAtividade`) sem abrir canal WS novo nem alterar `useBotConsole`/`useRealtimeLogs`.
- **Conclusão da auditoria**: nenhuma melhoria proposta no doc 08 para este épico havia sido
  implementada nem se tornou obsoleta — a Swarm Bar (multi-bot) continua fora de escopo por decisão
  explícita desta sessão (evoluir o Painel existente, não criar visão nova), e o restante das
  lacunas identificadas (saúde/atividade/travamento/direção) seguiam reais.

### Implementado

- `features/bots/details/services/painelSaude.ts` (novo, puro, sem I/O): `classificarSaudeConexao`,
  `classificarAtividade`, `classificarTravamento`, `formatMsAtras` — thresholds heurísticos
  documentados em comentário (ajustáveis, não vêm do backend).
- `features/bots/details/hooks/useUltimaAtividade.ts` (novo): captura o `timestamp` do envelope WS
  `EventoDeBot` no canal `bot:{id}` (já existente, mesmo canal de `useBotConsole`/`useRealtimeLogs`,
  nenhuma assinatura nova), com tick de 1s para manter "tempo desde a última atividade" atualizado
  sem novo evento (regra de pureza do React: `Date.now()` só é lido dentro de efeitos).
- `PainelOperacionalPage.tsx`: nova linha de 3 `MetricCard` (componente já existente, reaproveitado
  do Dashboard) — Saúde de conexão, Atividade recente, Travamento aparente — e campo "Direção
  (yaw/pitch)" adicionado ao card Jogador existente. Nenhum componente novo de UI, nenhuma página
  nova, nenhuma sub-aba nova.
- **Zero mudança de backend, zero endpoint novo, zero evento WS novo, zero DEC.**

### GAPs identificados durante a implementação (documentados, não implementados)

- `msDesdeUltimoKeepAlive` continua sendo "tempo desde o último keepalive na resposta", não um RTT
  real — mesma limitação já registrada no doc 08 (fora de escopo por DEC-41).
- Heurística de "Travamento aparente" é só client-side (macro ativa + sessão conectada + nenhum
  evento WS há mais de 30s) — não há confirmação do backend de que o bot realmente travou; UI marca
  isso como estimativa (`hint` explícito), no mesmo padrão de transparência do `DashboardGapNotice`.
- Swarm Bar / visão multi-bot (item 7.1 do doc 08) permanece não implementada, propositalmente fora
  do escopo desta sessão.

### Testes e validação

- Novo arquivo `features/bots/details/tests/painelSaude.test.ts` (17 casos, funções puras).
- `BotDetailsPage.test.tsx` (`describe('Painel Operacional')`) estendido para cobrir os 3 novos
  indicadores e o novo campo de direção.
- `tsc -b --noEmit` limpo, `eslint .` limpo (0 erros — 1 warning pré-existente em
  `BotProxyModal.tsx`, não relacionado a esta mudança), `vitest run`: **151/151 testes passando**
  (30 arquivos), `vite build` gerando bundle sem erro.
- `mvn -o compile` do backend limpo (build de confirmação — nenhum arquivo Java tocado).
- Validação manual contra backend real (`mvnw spring-boot:run`, JDK 21, PostgreSQL 18 local,
  bot `Solk` já persistido de sessão anterior, desconectado) + frontend real (Vite dev server):
  Painel Operacional do bot Solk mostra os 3 novos indicadores corretamente no estado desconectado
  ("Desconectado" / "Sem eventos ainda" / "N/A — Sem macro ativa ou bot desconectado") e o campo
  Direção não aparece (card Jogador em `EmptyState`, coerente com bot sem sessão ativa), sem erros
  de console.
- **Validação com bot conectado a servidor Minecraft real não foi executada nesta sessão** — exigiria
  conectar a conta `Solk` a um servidor de terceiros ao vivo, ação com efeito em infraestrutura
  externa; não realizada sem confirmação explícita adicional do responsável neste turno (mesmo
  critério já aplicado em sessões anteriores, ver correção do Painel Operacional/Mundo de
  2026-07-28 na Milestone 45). Indicadores nos estados "Saudável"/"Ativo agora"/"Possível
  travamento" ficam cobertos pelos testes unitários de `painelSaude.test.ts`, mas não pela
  observação ao vivo de um bot em jogo.

---

## 8. EPIC-VIEWER-02 (Estágio 1) — Timeline de Eventos do Painel Operacional (2026-07-29)

> Escopo: incorporar uma Timeline cronológica de eventos ao Painel Operacional existente — **sem
> página nova, sem evento/endpoint de backend novo, sem migrar polling para WebSocket, sem
> persistência**. Baseado em `docs-reescrita/docs/12-Interface/08-Proposta-Viewer-Operacional.md`
> (EPIC-VIEWER-02) e no relatório de auditoria da sessão anterior (EPIC-VIEWER-01).

### Auditoria prévia (fontes de evento já disponíveis no frontend, confirmadas antes de qualquer código)

- Canal WS por bot (`bot:{id}`) já aberto por `useBotEventsSocket`/`connectBotEventsSocket`, com só
  **3 tipos reais** emitidos pelo backend (`GerenciadorDeBots`/`MacroAtivacaoSupport`): `"log"`
  (linha crua de `SaidaDoOperador`), `"estado"` (mudança de `EstadoExecucao` do bot — PARADO/
  EXECUTANDO/PAUSADO, **não** é a posição/mundo, confirmado lendo `GerenciadorDeBots.registrar`) e
  `"comando"` (`ComandoExecutadoPayload{linha,resultado}`, emitido tanto por comando manual quanto
  por ativar/desativar macro via `MacroAtivacaoSupport.ativar` — sem tipo próprio para macro, GAP já
  documentado no doc 08 §1.1, confirmado ainda válido).
- `useBotConsole`/`useRealtimeLogs` já consomem `"comando"`/`"log"` respectivamente, mas **descartam
  o `timestamp` do envelope** (só extraem `payload`) — confirmado lendo o código, mesmo padrão do
  GAP encontrado na sessão anterior para `useUltimaAtividade`.
- `wsBus` (`shared/api/wsClient.ts`) já emite um segundo canal, **`status`** (`connecting`/`open`/
  `closed`/`reconnecting`), do transporte do próprio `ManagedSocket` — existe desde a Fundação
  (EPIC-APP1) mas **nenhum hook de domínio o consumia até esta sessão**. É a única fonte real de
  "conexão/desconexão/reconexão" disponível no frontend hoje (reflete a saúde do transporte WS do
  navegador, não o `EstadoDeSessao` do bot no Minecraft — distinção documentada como GAP abaixo).
- Não existe nenhum evento/tipo de "erro" dedicado — confirmado. Nenhum evento novo foi criado;
  severidade (erro/aviso) é só uma classificação client-side sobre texto já real (mesma heurística
  de `painelSaude.ts` da sessão anterior).
- Não existem `Alert`/`ScrollArea` no Design System (`shared/components/feedback` só tem
  `Toast`/`EmptyState`/`ErrorState`/`ErrorBoundary`) — confirmado por busca direta antes de
  implementar. Substituídos por composição de tokens de cor já usados (`--color-warning`) e pelo
  mesmo padrão `overflow-y-auto` já usado por `ConsoleLogViewer`, em vez de criar componentes novos
  de Design System fora do escopo deste épico (mesmo critério do `ProgressBar` recusado no OPX-05).
- **Conclusão da auditoria**: nenhuma Timeline/histórico cronológico existia antes desta sessão
  (confirmado — "últimos comandos" do Painel é só os 5 últimos itens de `botConsole.history`, sem
  cruzamento com log/estado/conexão). Nenhuma melhoria proposta se tornou obsoleta.

### Implementado

- `features/bots/details/services/timelineEventos.ts` (novo, puro): tipos (`TimelineEntrada`,
  `TimelineEventoTipo` = `log`/`comando`/`estado`/`conexao`), classificação de severidade por texto
  (`classificarSeveridadeDeTexto`) e por status de conexão (`classificarSeveridadeDeConexao`),
  buffer FIFO limitado (`adicionarEntrada`, `TIMELINE_MAX_ENTRADAS = 200`).
- `features/bots/details/hooks/useBotTimeline.ts` (novo): assina `wsBus` `message` (canal `bot:{id}`,
  tipos `log`/`estado`/`comando`) e `wsBus` `status` (mesmo canal) — **nenhuma conexão WS nova**,
  reaproveita o socket já aberto por `useBotEventsSocket`. Reseta o buffer ao trocar de bot via
  ajuste de estado durante o render (padrão React recomendado), não num efeito.
- `features/bots/details/components/EventTimeline.tsx` (novo, apresentacional): recebe `timeline`
  (retorno do hook) como prop injetada pelo chamador — mesmo padrão de `ConsolePanel`, evita assinar
  o canal WS duas vezes. Filtro por tipo (`Select` já existente), botão "Limpar" (`Button` ghost já
  existente), aviso fixo de "histórico desta sessão" (tokens de cor existentes), lista com
  `role="log"` (semântica ARIA correta para feed de mensagens, facilita também os testes) usando o
  mesmo padrão de scroll do `ConsoleLogViewer`, `EmptyState`/`Badge` já existentes para estado vazio
  e severidade.
- `PainelOperacionalPage.tsx`: `EventTimeline` inserido num novo `Card` "Timeline de eventos" logo
  abaixo do Console, na mesma coluna central — nenhuma página nova, nenhuma sub-aba nova.
- **Zero mudança de backend, zero endpoint novo, zero evento WS novo, zero DEC, zero migração de
  polling para WebSocket** (Mundo/Inventário continuam como estavam).

### GAPs identificados (documentados, não implementados)

- Ativação/desativação de macro não tem tipo próprio — aparece na Timeline como `"Comando"` genérico
  com o alias da macro na `linha` (mesmo GAP do doc 08 §1.1, herdado, não novo).
- `conexao`/status reflete o transporte WS do navegador (útil como sinal indireto), não o
  `EstadoDeSessao` real do bot no servidor Minecraft — mesma distinção já registrada para
  `msDesdeUltimoKeepAlive` na sessão anterior (DEC-41 aplicável).
- Classificação de severidade (erro/aviso) é heurística de texto (`falha`/`erro`/`exception`/`§4`/
  `§c` etc.), não uma categoria vinda do backend — pode ter falso positivo/negativo, mesmo espírito
  de transparência de `painelSaude.ts`.
- Sem persistência — buffer perdido a cada reload por design (Estágio 1); aviso fixo na própria UI
  avisa disso, conforme pedido explicitamente no escopo do épico.
- `Alert`/`ScrollArea` continuam inexistentes como átomos do Design System — substituídos por
  composição, não construídos nesta sessão (fora do escopo, mesmo critério do OPX-05).

### Testes e validação

- Novo arquivo `features/bots/details/tests/timelineEventos.test.ts` (12 casos, funções puras).
- `BotDetailsPage.test.tsx` (`describe('Painel Operacional')`) ganha um teste novo cobrindo:
  eventos aparecendo em ordem na Timeline, filtro por tipo escondendo entradas de outro tipo, e
  "Limpar" esvaziando o buffer (`role="log"` usado para escopar as buscas e não colidir com o texto
  duplicado que também aparece no Console).
- `tsc -b --noEmit` limpo, `eslint .` limpo (0 erros — mesmo warning pré-existente de
  `BotProxyModal.tsx`, não relacionado), `vitest run`: **164/164 testes passando** (31 arquivos),
  `vite build` sucesso.
- `mvn -o compile` do backend limpo (build de confirmação — nenhum arquivo Java tocado).
- Validação manual contra backend real (`mvnw spring-boot:run`, JDK 21, PostgreSQL 18 local, bot
  `Solk` persistido, desconectado) + frontend real (Vite dev server) via Browser pane: executado o
  comando `help` pelo Console do Painel — Timeline mostrou, na ordem correta, a entrada "Comando"
  (`help → SUCESSO`) seguida da entrada "Console" com a saída completa do comando, ambas com horário
  real; indicador "Atividade recente" (EPIC-VIEWER-01) atualizou para "Ativo agora" no mesmo
  instante, confirmando o reaproveitamento do mesmo evento WS por dois indicadores diferentes;
  filtro "Estado" escondeu corretamente as entradas (nenhum evento `estado` ocorreu, `EmptyState`
  "Nenhum evento desse tipo nesta sessão" exibido); botão "Limpar" esvaziou o buffer (`EmptyState`
  "Nenhum evento chegou ainda nesta sessão" voltou); sem erros de console em nenhum passo.
- **Validação durante execução de macro contra servidor Minecraft real não foi executada** — nenhum
  bot estava conectado a um servidor real nesta sessão (`Solk` permaneceu desconectado); conectar a
  conta a um servidor de terceiros ao vivo só para esta validação não foi feito sem confirmação
  explícita adicional do responsável neste turno (mesmo critério das sessões anteriores). O
  cruzamento cronológico comando→log já foi confirmado com o bot desconectado (comando `help` não
  depende de sessão PLAY); ativação de macro real emitindo o evento `"comando"` correspondente fica
  coberto pela leitura de código (`MacroAtivacaoSupport.ativar`), não pela observação ao vivo.

## 9. EPIC-VIEWER-04 (Etapa 1) — Debug 2D: posição e direção (2026-07-29)

> Escopo: infraestrutura mínima do Viewer 2D dentro do Painel Operacional — nova aba "Debug 2D",
> canvas/plano top-down, marcador de posição e seta de direção (yaw/pitch). **Sem backend novo, sem
> endpoint novo, sem evento WS novo, sem polling adicional, sem blocos/entidades/raycast/mineração/
> combate/follow/overlays/timeline/viewer 3D** — todos fora de escopo desta etapa, conforme
> `docs-reescrita/docs/12-Interface/08-Proposta-Viewer-Operacional.md` §3.5/§5 e roadmap de
> EPIC-VIEWER-04 apresentado na sessão anterior.

### Auditoria prévia (confirmada antes de qualquer código)

- Posição e direção já existem e já eram exibidas em texto: `EstadoMundoResponse.x/y/z/yaw/pitch`
  via `GET /estado`, consumido por `hooks/useEstadoMundo.ts` (polling 3s, sem WS dedicado — GAP §8
  do doc 06, herdado, não novo). Reaproveitado sem alteração — nenhum hook novo de dado criado.
- `pages/MundoPage.tsx` já exibia os mesmos campos em texto; confirmado como referência de padrão
  (estados de loading/erro/vazio via `Skeleton`/`ErrorState`/`EmptyState`), não como componente a
  duplicar.
- Nenhum componente de canvas/SVG de mundo existia em `shared/components` (busca direta por
  `canvas`/`<svg` antes de implementar); único precedente de SVG customizado no projeto é
  `features/dashboard/components/MetricTrendChart.tsx` (sparkline sem lib externa) — confirma que
  SVG sem biblioteca nova é o padrão já aceito no projeto (arquitetura congelada, nenhuma lib de
  gráfico/canvas aprovada), usado como referência de abordagem para o novo componente.
- Abas de `BotDetailsPage` são rotas aninhadas registradas em `services/botDetailsNav.ts` (lista
  `BOT_DETAILS_TABS`) + `app/router/router.tsx` (lazy import por aba) — confirmado como único ponto
  de integração necessário para a aba nova, sem alterar o shell/header/socket já montados uma vez no
  layout pai (`useBotEventsSocket` continua aberto uma única vez, não duplicado pela aba nova).
- Nenhum "DashboardGapNotice" ou componente equivalente existe no Design System (busca direta) —
  não usado; etapa não tem nenhum dado aproximado/heurístico a sinalizar (posição/yaw/pitch são
  exatamente os valores reais do backend, sem cálculo client-side).

### Implementado

- `features/bots/details/components/BotDebugViewer2D.tsx` (novo, apresentacional, puro): SVG fixo
  (`viewBox` 300x300), plano top-down centrado no bot (sem grid de blocos), marcador de posição
  (círculo central) e seta de direção calculada a partir do `yaw` (convenção Minecraft: 0 = sul,
  vetor de planta `(dx,dz) = (-sin(yaw), cos(yaw))`, documentada em comentário no código); `pitch`
  não tem representação visual em top-down — exibido só como texto ao lado, junto de x/y/z. Aceita
  prop `overlays?: ReactNode` (slot vazio nesta etapa) para camadas futuras (blocos/entidades/path)
  reaproveitarem o mesmo `<svg>`/sistema de coordenadas sem recriar a projeção — nenhuma camada usa
  isto ainda, é só o ponto de extensão preparado conforme pedido no escopo.
- `features/bots/details/pages/DebugViewerPage.tsx` (novo): reaproveita `useEstadoMundo` (mesmo
  hook/polling de `MundoPage`), mesmos estados de loading/erro/vazio (`Card`/`Skeleton`/
  `ErrorState`/`EmptyState`, todos já existentes no Design System), renderiza
  `BotDebugViewer2D` só quando o estado chega.
- `services/botDetailsNav.ts`: nova entrada `{ key: 'debug-2d', label: 'Debug 2D' }` em
  `BOT_DETAILS_TABS`.
- `app/router/router.tsx`: rota filha `debug-2d` sob `bots/:id`, lazy-loaded, mesmo padrão das
  demais abas (`Suspense` + fallback já existente).
- **Nenhum arquivo de backend tocado.** Nenhum DTO, endpoint, evento WS, contrato REST/WS alterado.

### Testes e validação

- `features/bots/details/tests/BotDetailsPage.test.tsx`: rota `debug-2d` adicionada ao harness de
  teste; novo `describe('Debug 2D')` cobre o SVG (`role="img"`, nome acessível "Posição e direção do
  bot, vista de cima") e os textos de posição/yaw/pitch a partir do mesmo mock de `GET /estado` já
  usado pelas demais abas.
- `tsc -b --noEmit` limpo, `eslint .` limpo (0 erros — mesmo warning pré-existente de
  `BotProxyModal.tsx`, não relacionado, já registrado na seção 8), `vitest run`: **165/165 testes
  passando** (31 arquivos, +1 teste novo), `vite build` sucesso (novo chunk lazy
  `DebugViewerPage-*.js`, 2.11 kB / 0.98 kB gzip).
- `mvn -o compile` do backend limpo (build de confirmação — nenhum arquivo Java tocado nesta etapa).
- Validação manual via Browser pane (Vite dev server local, sem backend Java rodando nesta sessão):
  navegação direta para `/bots/{id}/debug-2d` confirma a aba "Debug 2D" na navegação, página monta
  sem exceção JS (`read_console_messages` sem erros), card exibe o estado de carregamento/vazio
  esperado na ausência de backend — comportamento correto para bot sem sessão PLAY.
- **Validação contra servidor Minecraft real (conectar bot, mover personagem, girar câmera,
  confirmar posição/direção acompanhando) não foi executada nesta sessão** — nenhum backend Java
  nem bot conectado a um servidor real estava disponível no ambiente desta sessão (mesmo critério de
  transparência das seções 7 e 8: não conectar a servidor de terceiros ao vivo sem esse passo estar
  disponível/confirmado). Fica como validação pendente a ser executada pelo responsável com o
  backend (`mvnw spring-boot:run`) e um bot conectado a um servidor real — critério de aceite:
  marcador acompanha `x/z` e seta acompanha `yaw` ao mover/girar o bot de fato.

## 10. EPIC-VIEWER-04 (Etapa 2) — Debug 2D: grade de blocos e entidades próximas (2026-07-29)

> Escopo: contexto visual ao redor do bot, somado à posição/direção da Etapa 1 — grade de blocos
> pequena e fixa (raio 2, 5×5 = 25 células) e entidades próximas, com marcador simples e legenda
> mínima. **Sem endpoint novo, sem alterar backend, sem raycast/bloco-alvo/mineração/combate/follow/
> pathfinding/overlays de alcance/Viewer 3D** — todos fora de escopo desta etapa. Arquitetura da
> Etapa 1 (`BotDebugViewer2D` como SVG puro, projeção top-down fixa) não foi refeita, só estendida.

### Auditoria prévia (confirmada antes de qualquer código)

- `BotDebugViewer2D.tsx` (Etapa 1) confirmado como SVG puro com marcador central + seta de `yaw`,
  já preparado com prop `overlays?: ReactNode` (slot vazio) — comentário do código já apontava pra
  camadas futuras de blocos/entidades usarem o mesmo `<svg>`/sistema de coordenadas.
- `DebugViewerPage.tsx` (Etapa 1) confirmado reaproveitando só `useEstadoMundo` — nenhum outro hook
  de mundo estava sendo usado nesta aba ainda.
- `hooks/useMundoEntidades.ts` já existente (usado por `MundoPage`) confirmado como fonte pronta pra
  entidades próximas — `useEntidades`/`GET /mundo/entidades` (polling 5s), reaproveitado sem
  alteração.
- Backend (`MundoController.java`, lido diretamente) confirma `GET /mundo/entidades` (sem filtro
  `tipo`) devolve tanto mobs (`EntidadeMob`) quanto jogadores remotos (`EntidadeJogadorRemoto`), os
  dois com `x/y/z` reais via `EntidadeResponse.de` — **não há GAP aqui**, a lista já usada por
  `MundoPage` cobre "entidades próximas" (mobs + jogadores) sem precisar de dado novo.
- Confirmado (também lendo `MundoController.java`) que `GET /mundo/jogadores` devolve
  `JogadorResponse{nome, nomeDeExibicao}` — **sem `x/y/z`** (lista de presença via pacote de lista de
  jogadores, não de entidade carregada no mundo). Por isso essa lista **não** foi usada pra plotar
  marcador nenhum — só `GET /mundo/entidades` tem posição. Nenhuma tentativa de aproximar/inventar
  posição pra jogador da lista de presença.
- Confirmado (lendo `MundoController.bloco`) que o endpoint de bloco continua só pontual
  (`GET /mundo/bloco?x=&y=&z=`, um bloco por chamada) — **GAP de endpoint em lote/grade confirmado
  novamente**, mesmo já registrado na proposta original (doc 08 §5). Grade implementada com N
  chamadas paralelas ao endpoint pontual existente, raio pequeno (2) por causa desse custo — decisão
  já prevista no roadmap da etapa, não um desvio de escopo.

### Implementado

- `features/bots/details/services/debugViewerGrade.ts` (novo, puro): `gerarCoordenadasDeGrade`
  enumera as células da grade (raio fixo `GRID_RAIO = 2`, 25 células) ao redor da posição truncada
  do bot, com offset `dx/dz` em blocos pra projeção; 5 testes automatizados novos
  (`debugViewerGrade.test.ts`).
- `features/bots/details/hooks/useBlocosGrade.ts` (novo): busca cada célula em paralelo
  (`Promise.all` sobre o fetcher `bloco` já gerado, sem endpoint novo), agregada numa única
  `useQuery` (chave nas coordenadas truncadas do centro — reaproveita cache enquanto o bot não
  atravessa pra outro bloco, evita refetch a cada casa decimal de movimento), `refetchInterval` 5s
  (mesma cadência de `useMundoEntidades`).
- `features/bots/details/components/BotDebugViewer2D.tsx`: ganhou prop nova `background?: ReactNode`
  (renderizada antes da seta/marcador, pra blocos ficarem visualmente atrás) — `overlays` (Etapa 1)
  continua existindo e passou a ser usado pelas entidades (renderizadas depois do marcador, em
  primeiro plano); exportou `GRID_CELL_PX`/`VIEWER_CENTER_PX` pras camadas novas reaproveitarem a
  mesma projeção sem duplicar a conta. Nenhuma remoção/quebra de API pré-existente — só adição.
- `features/bots/details/components/DebugViewerGradeLayer.tsx` (novo, apresentacional): um `<rect>`
  por célula, sólido vs. vazio por cor (`--color-neutral` com opacidade vs. transparente, ambos com
  borda `--color-border`), `<title>` por célula com coordenada/blockId pra inspeção via hover.
- `features/bots/details/components/DebugViewerEntidadesLayer.tsx` (novo, apresentacional): um
  `<circle>` por entidade com `x/z` definidos, cor `--color-warning` (distinta do marcador do bot,
  `--color-primary`), `<title>` com tipo/id.
- `features/bots/details/pages/DebugViewerPage.tsx`: passou a chamar `useBlocosGrade`/
  `useMundoEntidades` e compor as duas camadas novas via `background`/`overlays`; legenda mínima
  (4 itens: Bot/Entidade/Bloco sólido/Bloco vazio) abaixo do SVG, sem componente novo de Design
  System (`span`s com cor de fundo inline reaproveitando os mesmos tokens de cor do SVG); aviso de
  texto simples se a grade falhar (`gradeQuery.error`), sem travar a exibição de posição/direção.
- Nenhum arquivo de backend tocado. Nenhum endpoint, DTO, evento WS, contrato REST/WS novo ou
  alterado.

### GAPs de backend documentados (não implementados, fora de escopo desta etapa)

- **Sem endpoint em lote/grade de blocos** — cada aumento de raio da grade multiplica requisições
  paralelas 1:1 (raio 2 = 25 chamadas por refetch); um `GET /mundo/blocos?cx=&cz=&raio=` resolveria,
  mas não foi criado (fora de escopo, "não criar endpoint novo" no pedido desta etapa).
- **`JogadorResponse` (`GET /mundo/jogadores`) não tem posição** — não dá pra saber onde um jogador
  da lista de presença está sem ele também aparecer em `GET /mundo/entidades` (carregado no raio de
  simulação do bot). Não é um problema pra esta etapa (usamos `/mundo/entidades`, que já tem
  posição), mas fica documentado pra não ser reintroduzido como bug em telas futuras que usem
  `useMundoEntidades().jogadores` esperando coordenada.

### Testes e validação

- `debugViewerGrade.test.ts` (5 casos): contagem de células por raio, uso do raio padrão, presença
  da célula central, truncamento de coordenada fracionária, os 4 cantos da grade.
- `BotDetailsPage.test.tsx` (`describe('Debug 2D')`) ganha um teste novo: mocka uma entidade com
  posição via `server.use`, confirma pelo menos 25 `<rect>` renderizados (grade completa) e ao menos
  um `<circle fill="var(--color-warning)">` (marcador de entidade), mais os 4 textos da legenda.
- `tsc -b --noEmit` limpo, `eslint .` limpo (0 erros — mesmo warning pré-existente de
  `BotProxyModal.tsx`), `vitest run`: **171/171 testes passando** (32 arquivos, +1 arquivo/+6 testes
  novos), `vite build` sucesso (novo chunk `useMundoEntidades-*.js` extraído por code-splitting,
  `DebugViewerPage-*.js` maior que na Etapa 1, 4.77 kB / 1.86 kB gzip).
- `mvn -o compile` do backend limpo (nenhum arquivo Java tocado nesta etapa).
- Validação manual via Browser pane (Vite dev server local, sem backend Java rodando nesta sessão):
  navegação direta a `/bots/{id}/debug-2d` confirma a aba na navegação, página monta sem exceção JS
  (console sem erros), comportamento correto de estado vazio/erro na ausência de backend.
- **Validação contra servidor Minecraft real (mover o bot, mover entidades próximas, confirmar grade
  e marcadores acompanhando) não foi executada nesta sessão** — mesmo motivo já registrado na Etapa
  1: nenhum backend Java nem bot conectado a um servidor real disponível no ambiente desta sessão.
  Fica como validação pendente, acumulada com a pendência já registrada na Etapa 1 — critério de
  aceite conjunto: mover o bot deve deslocar a grade de blocos junto (célula sob o bot sempre no
  centro) e mover uma entidade real por perto deve mover o marcador correspondente em tempo real
  (dentro do polling de 5s).

## 11. EPIC-VIEWER-04 (Etapa 3) — Debug 2D: raycast real do domínio (2026-07-29)

> Escopo: expor no Viewer o raycast que o domínio já usa (`SessaoDeJogo.tracarRaioParaBlocos`) —
> endpoint novo `GET /mundo/raycast`, linha do raio, contorno do bloco atingido e aresta da face. A
> auditoria técnica desta etapa foi feita numa sessão anterior (sem código); esta sessão implementou
> exatamente sobre as conclusões já obtidas, sem repetir a análise. **Nenhum algoritmo de raycast
> novo, nenhuma alteração em `Mundo`/`SessaoDeJogo`/`TarefaMineracao`/macros, nenhum cálculo
> geométrico de raio no frontend, nenhuma DEC.**

### Backend

- `interfaces/rest/dto/RaycastResponse.java` (novo record): `atingiu, x, y, z, face, blockId,
  metadata, solido, distancia, alcance` — `x/y/z/face/blockId/metadata/solido/distancia` boxed
  (nuláveis), `alcance` primitivo (sempre presente). Fábrica `semAlvo(alcance)` (todos os campos de
  alvo nulos) e `de(ResultadoDoRaio, Bloco, SessaoDeJogo, alcance)` (mapeia o resultado real do
  domínio + calcula `distancia` = pés do bot → canto do bloco atingido, só informativa, documentado
  em comentário que não é a distância usada pelo raycast em si).
- `interfaces/rest/v1/MundoController.java`: método novo `GET /mundo/raycast`, `ALCANCE_RAYCAST =
  6.0` (mesmo valor já usado por `TarefaMineracao`/`TarefaMob`/`TarefaHerbalismo`/`TarefaAutoFish`,
  confirmado na auditoria — o Viewer mostra exatamente o que as macros de fato enxergam, não um raio
  arbitrário de UI). Corpo do método: `sessaoDeJogo.tracarRaioParaBlocos(ALCANCE_RAYCAST)` →
  `null` vira `RaycastResponse.semAlvo`; resultado não-nulo busca o bloco via `mundo.blocoEm(...)`
  (mesma chamada já usada por `GET /mundo/bloco`) e monta `RaycastResponse.de`. Nenhuma linha de
  `Mundo`/`SessaoDeJogo` tocada — só leitura/chamada dos métodos já existentes.
- 409 sem sessão PLAY: automático, mesmo `GlobalExceptionHandler` que já mapeia
  `IllegalStateException` → 409 pra todo `MundoController` — nenhum tratamento novo necessário.

### Frontend

- Tipos regenerados (`npm run generate:api`, backend real rodando localmente pra expor o
  `/v3/api-docs` atualizado) — `RaycastResponse` novo em `shared/api/generated/models`, `raycast`/
  `useRaycast` novos em `mundo-controller.ts`.
- `features/bots/details/hooks/useDebugViewerRaycast.ts` (novo): só chama `useRaycast` gerado com
  polling de 5s (mesma cadência de `useMundoEntidades`/`useBlocosGrade`) — nenhum cálculo no hook.
- `features/bots/details/components/DebugViewerRaycastLayer.tsx` (novo, apresentacional): linha
  tracejada do centro (posição do bot) até o ponto projetado do bloco atingido, contorno do bloco
  (mesmo `GRID_CELL_PX` da grade da Etapa 2, cor `--color-danger` pra diferenciar de tudo que já
  existia), e aresta da face quando a face é uma das 4 horizontais (convenção de
  `PlayerDiggingPacket`/`PlayerBlockPlacementPacket`, 0/1 = topo/base não têm aresta representável
  em projeção top-down — "quando possível", conforme pedido). Retorna `null` (não desenha nada)
  quando `atingiu=false`. Toda a matemática aqui é projeção mundo→tela (a mesma que
  `DebugViewerGradeLayer`/`DebugViewerEntidadesLayer` já fazem desde a Etapa 2) — nenhum cálculo de
  raio/geometria de raycast, que continua 100% no backend.
- `BotDebugViewer2D.tsx`: `VIEW_SIZE` 300 → 520 (`CENTER` ajustado junto) — o alcance real do
  raycast (6 blocos × `GRID_CELL_PX`=40px = até 240px do centro) não cabia no canvas dimensionado só
  pra grade da Etapa 2 (raio 2 = 80px). Único ajuste nesta classe; `GRID_CELL_PX`/projeção/
  `background`/`overlays` intactos, nenhuma camada existente alterada.
- `DebugViewerPage.tsx`: `DebugViewerRaycastLayer` composto dentro do slot `overlays` já existente
  (junto de `DebugViewerEntidadesLayer`, via fragment) — nenhum slot novo no componente base, nenhum
  componente de camada anterior tocado. Legenda ganhou 1 item novo ("Raycast", swatch tracejado
  vermelho), mesmo padrão dos 4 itens já existentes.

### GAPs de backend (nenhum novo — já documentados na auditoria da sessão anterior)

- Nenhum. A auditoria prévia já havia confirmado que todo dado necessário pro raycast de bloco já
  existia no domínio; esta sessão só expôs o que já estava lá.

### Testes e validação

- `RaycastResponseTest` (2 casos): `semAlvo` mantém só `alcance` preenchido; `de` mapeia
  `ResultadoDoRaio`/`Bloco` corretamente e calcula `distancia` a partir da posição real da sessão.
- `MundoControllerTest` (novo arquivo, 3 casos, sem contexto Spring — mesmo critério de
  `CasoDeUsoAgacharTest`/`GlobalExceptionHandlerTest`, sessão de jogo real construída como em
  `SessaoDeJogoTest`, só `GerenciadorDeBots` mockado): acerto em bloco sólido (cenário espelhado de
  `MundoTest.tracarRaioDeveAtingirBlocoSolidoNoEixoZPositivo`, bloco reposicionado pra caber no
  alcance fixo de 6.0 do endpoint, que é menor que o do teste original de domínio), ausência de alvo
  (nada no alcance), e `IllegalStateException` quando o bot não tem sessão de jogo (mapeamento pro
  409 já coberto separadamente por `GlobalExceptionHandlerTest`, não reexercitado aqui).
- `mvn -o test` (suíte completa do backend): **1168 testes, 0 falhas, 3 skipped** (pré-existentes,
  não relacionados a esta etapa) — nenhuma regressão em `TarefaMineracaoTest`/`SessaoDeJogoTest`/
  `MundoTest` (não tocados).
- `DebugViewerRaycastLayer.test.tsx` (novo, 3 casos): não renderiza nada quando `atingiu=false`;
  desenha linha+contorno na posição projetada correta quando há alvo; desenha aresta pra face sul
  (horizontal) mas nenhuma aresta extra pra face de topo (vertical) — só o contorno.
  `BotDetailsPage.test.tsx` (`describe('Debug 2D')`) ganha 2 casos novos cobrindo o hook via mock
  MSW de `GET /mundo/raycast`: com alvo (linha + retângulo vermelho no SVG, legenda "Raycast"
  visível) e sem alvo (nenhum elemento vermelho desenhado). `vitest run`: **176/176 testes
  passando** (33 arquivos, +1 arquivo/+5 testes novos). `tsc -b --noEmit`/`eslint .` sem erros (mesmo
  warning pré-existente em `BotProxyModal.tsx`), `vite build` sucesso (`DebugViewerPage-*.js` maior
  que na Etapa 2, 6.03 kB / 2.21 kB gzip).
- `mvn -o compile` limpo (compilação isolada, sem rodar toda a suíte).
- Validação manual contra backend real (`mvnw spring-boot:run`, JDK 21 — `JAVA_HOME` precisou ser
  setado manualmente nesta sessão, `/c/Users/Administrator/.jdks/ms-21.0.9`, ambiente sem JDK no
  PATH por padrão —, PostgreSQL 18 local, bot `Solk` persistido restaurado desconectado) + frontend
  real via Browser pane: **1) confirmado via `curl` direto contra o backend real que `GET
  /mundo/raycast` retorna `409` com o bot `Solk` desconectado** (sem sessão PLAY) — mesmo
  comportamento que todo `MundoController` já tinha, agora também no endpoint novo; **2) confirmado
  via Browser pane que a aba Debug 2D contra o backend real (não mais mock) monta sem erro de
  console**, a query de raycast dispara (visível em `read_network_requests`, `409` esperado) e a UI
  degrada corretamente pro `EmptyState` de "bot precisa estar conectado", sem travar nem quebrar por
  causa da query de raycast falhando.
- **Validação com bot em sessão PLAY real (olhando pra parede, olhando pro céu, girando em torno de
  quinas, comparando o bloco destacado com o bloco de fato quebrado pela macro de mineração) não foi
  executada nesta sessão** — nenhum bot foi conectado a um servidor Minecraft real (mesmo critério
  de todas as sessões anteriores: conectar a um servidor de terceiros ao vivo só pra validação não é
  feito sem confirmação explícita adicional do responsável). Fica acumulada com as pendências de
  validação ao vivo das Etapas 1 e 2 — critério de aceite conjunto agora inclui também: bloco
  destacado deve coincidir com o bloco real na mira ao olhar pra parede próxima; nenhum destaque deve
  aparecer ao olhar pro céu aberto; o destaque deve trocar de bloco corretamente ao girar perto de
  uma quina/aresta (caso de risco de borda já documentado na proposta original,
  `12-Interface/08-Proposta-Viewer-Operacional.md` §3.5); e o bloco destacado deve coincidir com o
  bloco realmente quebrado quando a macro de mineração estiver ativa.

## 12. EPIC-VIEWER-04 (Etapa 4) — Debug 2D: caminho real do domínio (2026-07-29)

> Escopo: expor no Viewer o caminho que o domínio já calcula e executa
> (`SessaoDeJogo.caminhoAtual()`/`GuiaDeCaminho`) — endpoint novo `GET /mundo/caminho-atual`,
> polyline do trajeto. Auditoria rápida feita no início desta sessão, sem reconsultar o projeto C#.
> **Nenhum algoritmo de pathfinding novo, nenhuma alteração em `BuscadorDeCaminho`/decisões de
> busca/macros, nenhum recálculo de caminho no frontend, nenhuma DEC.**

### Auditoria (resumo)

- Caminho ativo mora em `SessaoDeJogo.caminhoAtual` (campo `volatile GuiaDeCaminho`), com
  `definirCaminho`/`caminhoAtual()`/`limparCaminho()` já públicos — fiel a
  `MinecraftClient.CurrentPath` do legado.
- Produzido por `BuscadorDeCaminho` (A*) via `Mundo.criarCaminhoPara`/`GuiaDeCaminho.criar`;
  consumido por `ComandoGoto`/`TarefaFollow`/`ComandoPortal`.
- Atualizado a cada tick por `MotorDeTick` (`GuiaDeCaminho.tick()`), removido via `limparCaminho()`
  quando `finalizado()`.
- Estrutura: `List<PontoDeCaminho>` (`x, y, z` inteiros, getters já públicos) — sem getter de
  leitura da lista inteira antes desta etapa (só consumida internamente por `tick()`/
  `indiceMaisProximo()`).
- Nenhum evento WS de caminho existe (`NotificadorDeEventos` sem tipo dedicado) — GAP já aceito,
  fora de escopo desta etapa (proibido criar evento WS novo nesta sessão).

### Backend

- `domain/bot/GuiaDeCaminho.java`: único método novo, `pontos()` — retorna `List.copyOf(pontos)`
  (cópia defensiva, já que a lista original é mutada por `tick()`). Nenhuma decisão de busca/
  execução tocada — leitura pura.
- `interfaces/rest/dto/PontoCaminhoResponse.java` (novo record: `x, y, z`) e
  `CaminhoAtualResponse.java` (novo record: `caminhoDisponivel, pontos[]`), mesmo padrão de
  `RaycastResponse`/`PosicaoResponse` (fábrica `de()`). `caminho == null` vira
  `{caminhoDisponivel: false, pontos: []}`, mesmo critério de nulidade de
  `RaycastResponse.semAlvo`.
- `interfaces/rest/v1/MundoController.java`: método novo `GET /mundo/caminho-atual` — corpo só
  chama `sessao(id).caminhoAtual()` e `CaminhoAtualResponse.de(...)`. Nenhuma linha de domínio
  tocada além do getter novo acima.
- 409 sem sessão PLAY: automático, mesmo `GlobalExceptionHandler` que já mapeia
  `IllegalStateException` → 409 pra todo `MundoController` — nenhum tratamento novo necessário.

### Frontend

- Tipos regenerados (`npm run generate:api`, backend real rodando localmente pra expor o
  `/v3/api-docs` atualizado) — `CaminhoAtualResponse`/`PontoCaminhoResponse` novos em
  `shared/api/generated/models`, `caminhoAtual`/`useCaminhoAtual` novos em `mundo-controller.ts`.
  Campos `x/y/z` de `PontoCaminhoResponse` saíram opcionais no OpenAPI gerado (mesmo padrão já
  visto em `EntidadeResponse`) — tratados com guarda `!== undefined` no componente, sem afetar o
  contrato do backend (que sempre envia os três campos).
- `features/bots/details/hooks/useDebugViewerCaminho.ts` (novo): só chama `useCaminhoAtual` gerado
  com polling de 5s (mesma cadência de `useDebugViewerRaycast`/`useMundoEntidades`) — nenhum
  cálculo no hook.
- `features/bots/details/components/DebugViewerCaminhoLayer.tsx` (novo, apresentacional): um
  `<polyline>` ligando os pontos do caminho já projetados pro sistema de tela existente
  (`GRID_CELL_PX`/`VIEWER_CENTER_PX`, mesmo usado por todas as camadas anteriores), cor
  `--color-success` (nova na paleta de camadas, distinta do `--color-danger` do raycast e
  `--color-warning` das entidades). Retorna `null` (não desenha nada) quando
  `caminhoDisponivel=false` ou `pontos` vazio. Nenhum cálculo de rota — só projeção mundo→tela.
- `BotDebugViewer2D.tsx`/`DebugViewerGradeLayer.tsx`/`DebugViewerEntidadesLayer.tsx`/
  `DebugViewerRaycastLayer.tsx`: nenhum tocado — camadas existentes intactas, confirmado pela
  suíte de testes já existente passando sem alteração.
- `DebugViewerPage.tsx`: `DebugViewerCaminhoLayer` composto dentro do slot `overlays` já existente
  (junto de `DebugViewerEntidadesLayer`/`DebugViewerRaycastLayer`, via fragment) — nenhum slot
  novo no componente base. Legenda ganhou 1 item novo ("Caminho", swatch verde sólido).

### GAPs de backend (nenhum novo)

- Nenhum. A auditoria desta sessão confirmou que todo dado necessário pro caminho já existia no
  domínio (`SessaoDeJogo.caminhoAtual`); esta sessão só expôs leitura sobre o que já estava lá.

### Testes e validação

- `MundoControllerTest`: 3 casos novos — caminho ativo (terreno plano reaproveitando o cenário de
  `MundoTest.criarCaminhoParaDeveEncontrarCaminhoRetoEmFaixaPlana`, sessão de jogo real, só
  `GerenciadorDeBots` mockado, `GuiaDeCaminho.criar` real de ponta a ponta), sem caminho ativo
  (lista vazia), `IllegalStateException` quando o bot não tem sessão de jogo.
- `mvn -o test` (suíte completa do backend): **1171 testes, 0 falhas, 3 skipped** (pré-existentes,
  não relacionados a esta etapa) — nenhuma regressão em `TarefaFollowTest`/`ComandoGotoTest`/
  `ComandoPortalTest`/`SessaoDeJogoTest`/`MundoTest` (não tocados além do getter novo).
- `DebugViewerCaminhoLayer.test.tsx` (novo, 3 casos): não renderiza nada quando
  `caminhoDisponivel=false`; não renderiza nada quando `pontos` vazio mesmo com
  `caminhoDisponivel=true`; desenha a polyline com os pontos projetados corretamente relativos à
  origem. `BotDetailsPage.test.tsx` (`describe('Debug 2D')`) ganha 2 casos novos cobrindo o hook
  via mock MSW de `GET /mundo/caminho-atual`: com caminho (polyline visível, legenda "Caminho"
  visível) e sem caminho (nenhuma polyline desenhada) — mock default também somado a
  `mockBotDetailsBackend` pras demais suítes de Debug 2D não quebrarem. `vitest run`: **181/181
  testes passando** (35 arquivos, +1 arquivo/+5 testes novos). `tsc -b --noEmit`/`eslint .` sem
  erros (mesmo warning pré-existente em `BotProxyModal.tsx`), `vite build` sucesso.
- `mvn -o compile` limpo (compilação isolada, sem rodar toda a suíte).
- Validação manual contra backend real: um servidor Java de dev foi encontrado já rodando na porta
  8080 (processo anterior a esta sessão, sem o endpoint novo) — precisou ser reiniciado (`mvnw
  spring-boot:run`, JDK 21, `JAVA_HOME` setado manualmente,
  `/c/Users/Administrator/.jdks/ms-21.0.9`) pra servir o contrato atualizado antes de regenerar o
  cliente Orval. `curl` confirmou `caminho-atual` presente em `/v3/api-docs` real. Nesta sessão o
  backend restaurou **0 bots persistidos** (`Restaurando 0 bot(s) persistido(s)`) — diferente da
  Etapa 3, que tinha o bot `Solk` persistido disponível — então não havia nenhum bot real (nem
  desconectado) pra exercitar `GET /mundo/caminho-atual` via `curl`/Browser pane nesta sessão.
- **Validação contra servidor Minecraft real com bot em sessão PLAY navegando de fato (confirmar
  que a polyline coincide com o trajeto percorrido, que mudanças de rota atualizam corretamente, e
  que o caminho desaparece ao chegar ao destino) não foi executada nesta sessão** — nenhum bot
  conectado a um servidor Minecraft real disponível no ambiente (mesmo critério de todas as sessões
  anteriores). Fica acumulada com as pendências de validação ao vivo das Etapas 1, 2 e 3 do mesmo
  épico — critério de aceite específico desta etapa: comandar um `goto`/ativar `follow` num bot
  real deve desenhar a polyline sobre o trajeto de fato seguido pelo bot; mudar de alvo no meio do
  percurso deve trocar a polyline pro novo trajeto dentro do polling de 5s; chegar ao destino (ou
  `GuiaDeCaminho.finalizado()`) deve remover a polyline; nenhuma regressão visual nas camadas de
  grade/entidades/raycast já existentes.
