# Regras de Negócio

Este documento consolida as lógicas de negócio do AdvancedBot, agrupadas por contexto. Estas são as regras que ditam "como" a aplicação deve se comportar perante as situações de jogo e sistema.

## 1. Login e Autenticação

- **Autenticação Mojang**: Se a conta do bot incluir um caractere `@` e uma senha válida, o sistema DEVE contatar os servidores da Mojang para obter o `AccessToken` e `ProfileId` antes de conectar ao Minecraft.
- **Offline Mode**: Se não houver senha, o bot assume o modo "Cracked" e DEVE pular a autenticação Mojang.
- **Sessão Expirada**: Se a resposta da Mojang retornar *invalid token*, o sistema DEVE tentar re-autenticar usando a senha armazenada; se falhar, a Sessão é marcada como de erro irreversível.
- **Login in-game (AuthMe)**: Se o servidor enviar uma mensagem de chat contendo `/login` ou `/register`, a automação nativa DEVE responder automaticamente `/login <senha>` usando uma senha padrão de configuração em até 2 segundos.

## 2. Conexão e Rede

- **Reconnect Automático**: Se a conexão for perdida (ex: Kicked) e a opção Auto-Reconnect estiver ligada, o sistema DEVE aguardar o "Reconnect Delay" (em milissegundos) antes de tentar nova conexão.
- **Desconexão Proposital (Anti-AFK / Anti-Ban)**:
  - O bot PODE ser configurado para desconectar temporariamente caso outro jogador se aproxime a menos de X blocos, visando ocultar o fato de ser um bot.
  - O bot PODE se desconectar se a durabilidade de um item crítico (ex: Espada/Picareta) acabar e não houver repasse.
- **Keep Alive**: A Sessão DEVE responder pacotes KeepAlive emitidos pelo servidor com o mesmo ID exato. A falha resulta em *Timeout* em 30 segundos.
- **Protocolo de Criptografia**: Se o servidor solicitar `EncryptionRequest`, o bot DEVE gerar o Segredo AES compartilhado, responder `EncryptionResponse` e imediatamente cifrar todos os bytes futuros da Sessão.

## 3. Inventário

- **Limites de Janela**: Uma interação de clique (`WindowClick`) DEVE enviar o `ActionNumber` sequencial. Se o servidor rejeitar (`ConfirmTransaction` negativo), a operação foi inválida.
- **Hotbar**: O slot ativo (Held Item) DEVE ser um valor de 0 a 8.
- **Drops de Sobra**: Se o inventário ficar cheio durante atividades automatizadas (pesca, mineração), o bot DEVE descartar (Drop) os itens menos valiosos (ou itens não marcados na whitelist).
- **Auto-Equip**: Quando a durabilidade da ferramenta em mãos esgotar, o bot DEVE buscar automaticamente no inventário interno uma ferramenta idêntica e transferi-la para a hotbar.

## 4. IA e Pathfinding

- **Física Limítrofe**: O Pathfinding não pode traçar caminhos através de blocos sólidos (`IsSolid`). DEVE respeitar colisões precisas (AABB).
- **Jump Limits**: O bot só pode pular buracos de até 2 blocos de distância horizontais, e subir no máximo 1 bloco vertical (a não ser que haja escadas/vines).
- **Quedas**: A IA DEVE evitar cair distâncias maiores que 3 blocos para prevenir dano de queda letal contínuo.
- **Gravity e Move Packets**: O bot em voo livre (queda) DEVE enviar atualizações contínuas do eixo Y (diminuindo de acordo com a aceleração terminal do jogo) a cada Tick, sob pena do Anti-Cheat flagrá-lo como "Fly Hack".

## 5. Combate (Macros de Mob)

- **Mira Crítica (RayCast)**: Para atacar, o bot DEVE estar olhando (Yaw/Pitch) de forma que o raio originado de seus olhos intercepte a caixa de colisão da Entidade Alvo.
- **Alcance**: Entidades não podem ser atacadas se a distância (Euclidiana do olho até a AABB) for superior a 3.0 blocos (Minecraft <= 1.8).
- **Delay de Hit**: A automação DEVE respeitar o tempo de imunidade do inimigo (geralmente não enviar mais que 2 hits por segundo), exceto em cenários "Spammer" onde o limite de pacotes por segundo impera.

## 6. Pesca

- **Throw Event**: O bot DEVE arremessar a vara (Item Use). O servidor responde criando a entidade "Bobber" (boia).
- **Recolhimento**: O bot só DEVE puxar a vara quando um evento sonoro (`SoundEffect` ou `EntityVelocity`) específico ocorrer exatamente na localização da Boia.
- **Pitch Angular**: A vara DEVE ser lançada com um Pitch específico (ex: -10 graus) e Yaw olhando para a água para garantir aterrissagem correta.

## 7. Mineração

- **Quebra**: O bot DEVE enviar `PlayerDigging(START)`, aguardar (opcionalmente) o tempo correto baseado na ferramenta e dureza do bloco, e então enviar `PlayerDigging(FINISH)`.
- **Anti-Cheat Delay**: Para simular realidade humana (Legit), o AutoMiner DEVE adicionar um atraso em milissegundos entre o olhar para o minério e o início da quebra.

## 8. Plugins

- **Ciclo de Carregamento**: Plugins devem ser carregados DEPOIS das configurações globais, mas ANTES de qualquer conexão com servidor ser iniciada.
- **Isolamento de Falha**: Se um Plugin lançar exceção durante os callbacks de evento, isso NÃO DEVE travar (crashar) a Sessão principal.
- **Hot-Reloading**: O sistema DEVE suportar descarregar e carregar instâncias de plugin sem reiniciar o processo do bot (caso mantido o comportamento legado).

## 9. Proxy

- **Transparência**: O domínio (MinecraftClient) NÃO DEVE saber se o pacote trafega por Proxy. O roteamento (SOCKS4/5/HTTP) DEVE ser resolvido no nível da Infraestrutura/Rede antes do Protocolo iniciar.
- **Timeouts Agudos**: Proxies tendem a cair muito. A Sessão DEVE ter tolerância para encerramento abrupto da conexão sem emitir erros longos.

## 10. Viewer (Interface OpenGL)

- **Modo Passivo (Read-Only)**: O Viewer consome o modelo de Mundo, mas NUNCA DEVE disparar eventos de jogo diretamente (ex: clicar no viewer não envia pacote de quebrar bloco se a IA estiver em outro estado). A IA detém a primazia sobre a ação.

## 11. Scheduler / Tempo

- **O Tick Principal**: O bot DEVE avaliar seu ambiente a 20 TPS (Ticks por segundo, ou a cada 50ms).
- **Sincronia Servidor-Cliente**: O tick do cliente NÃO É acoplado ao pacote `TimeUpdate` do servidor. O bot gera os ticks localmente para fins de física, e tenta manter-se próximo de 20 TPS independentemente do lag de rede.

---

## 12. AuthMe — Contador de Tentativas e Limite de Envio

- **Contador Bit-Packing**: O campo `authmeCounter` armazena **dois contadores de 4 bits** em um único inteiro via `Utils.GetBits`/`SetBits`: bits 0-3 = tentativas de `/register`, bits 4-7 = tentativas de `/login`.
- **Limite por tipo**: Cada tipo (`/register`, `/login`) só é enviado no máximo **2 vezes** por sessão. Após 2 tentativas, o bot para de responder aquele prompt, mesmo que o servidor continue pedindo.
- **Email aleatório**: O placeholder `@email` no `CmdRegister` é substituído por um email sintético gerado via `NickGenerator.RandomNick(16, 1)` + `"@gmail.com"`.
- **Trigger de detecção VIP**: A mensagem exata `"✖ Apenas VIPs podem utilizar este comando!"` (sem cores) seta `PlayerVIP = false`. A flag VIP também é usada em `MacroUtils.WaitForTeleport` para determinar delay de espera (3s vs 5s).
- **Trigger de login**: A mensagem `"agora você está logado. nunca use a mesma senha do craftlandia em outros servidores."` seta `LoggedIn = true` e envia automaticamente `/vip`.

---

## 13. Roteamento de Mensagens de Chat (`SendMessage`)

- **Discriminador de prefixo**: Se a mensagem começa com `'$'`, ela é roteada para `CmdManager.RunCommand()` (executa comando local). Caso contrário, vai para `PacketChatMessage`.
- **Limite de chat**: Mensagens enviadas ao servidor são truncadas em **99 caracteres** (substring 0-98) se excederem esse limite.
- **Hook de plugins**: Todos os plugins registrados recebem `onSendChat(msg, client)` **antes** da mensagem ser processada localmente ou enviada à rede.

---

## 14. Desconexaão Especial por IP Sobrelotado

- Se o servidor enviar um pacote de kick com a mensagem exata `"Já existem muitas contas conectadas com esse IP."` (sem cores), o bot automaticamente avança para o **próximo proxy** da lista (`Program.FrmMain.Proxies.NextProxy()`) antes de tentar a reconexão.

---

## 15. Quebra Rápida de Bloco (`BreakBlock` em `MinecraftClient`)

- O método `BreakBlock(HitResult hit)` envia **em sequência imediata** (sem delay): `PacketSwingArm` + `PacketPlayerDigging(START)` + `PacketPlayerDigging(FINISH)`. Essa é a mineração de instakill (sem respeitar o tempo de quebra). Só funciona em servidores sem anti-cheat de digging ou em blocos quebráveis em 0 ticks.

---

## 16. Regras da Pesca (Macros Solk)

- **Limítacao de ciclo**: Um ciclo de pesca consiste em exatamente **1000 lançamentos de vara** com `180ms` entre cada um (180 segundos por ciclo), independente do inventário estar cheio.
- **Reparo via Bloco de Ferro**: O mecanismo de reparo de vara não usa bigorna. Executa 2 cliques diretos consecutivos no bloco de ferro (mecânica de server custom).
- **Lock inter-processo**: A macro de mob usa `FileLock` (arquivo em `%TEMP%`) para evitar que dois bots na mesma máquina abram o baú de espadas simultaneamente. Timeout de 10 segundos.
- **Venda por loja de placa**: A venda de drops é feita por cliques em placa de loja (`ShopSign`) com `sneak on/off` para cada hit, com `500ms` de delay entre cada fase.

---

## 17. Inventário — Máquina de Estado do Cursor (`Inventory.Click`)

- **Estado global de cursor**: `Inventory.ClickedItem` é um campo estático compartilhado por todas as instâncias de `Inventory`. Isso implica que múltiplos clientes em um mesmo processo compartilham o cursor — potencial race condition.
- **Regras de click esquerdo/direito**:
  - Se cursor vazio + click esquerdo: pega item inteiro do slot
  - Se cursor vazio + click direito: pega metade do item
  - Se cursor tem item + slot vazio + click esquerdo: deposita item inteiro
  - Se cursor tem item + slot vazio + click direito: deposita 1 unidade
  - Se cursor e slot têm o mesmo item + click esquerdo: empilha até 64; excedente volta ao cursor
  - Se cursor e slot têm itens diferentes: troca os itens (swap)
- **Offset de baú**: Se `isChestOpen == true`, o slot é decrementado em 9 antes de gerar o `PacketClickWindow` (pois baús abertos possuem slots relativos ao inventário do jogador deslocados).
- **ActionNumber**: `TransactionId` é incrímentado por referência em cada click (`++TransactionId`); é por instância de `Inventory`, não por sessão.
