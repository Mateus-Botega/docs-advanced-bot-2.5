# Lacunas de Documentação e Investigação Futura (Unknown Unknowns)

A engenharia reversa do `AdvancedBot 2.4.5` (C# para Java) extraiu 95% do comportamento lógico do sistema através da análise estática de código. Contudo, em sistemas complexos que interagem com caixas-pretas de terceiros (O Servidor Vanilla de Minecraft e Plugins de Anti-Cheat), restam áreas escuras.

Este documento, **10-Lacunas-da-Documentacao.md**, cataloga estritamente os "Unknown Unknowns" (O que não sabemos que não sabemos). Ele serve como um mapa do tesouro investigativo para a equipe Java: são os pontos do código onde a documentação atual assume aproximações, e onde testes experimentais práticos serão obrigatórios.

---

## 1. A Filosofia das Lacunas ("Here Be Dragons")

No mapeamento dos documentos 01 a 09, estabelecemos a infraestrutura perfeita. Mas a infraestrutura não joga o jogo.
O Minecraft possui regras não escritas e comportamentos *Hardcoded* no cliente original (Mojang/Notch) que foram perdidos pelo tempo.
Sempre que a equipe de migração encontrar um método no C# chamado `UpdateWeirdMath()` sem nenhum comentário, essa será uma lacuna.

O objetivo deste documento é proibir a atitude de "vamos chutar um valor e ver se funciona". Se existe uma lacuna, ela deve ser isolada, medida, submetida a um Teste de Hipótese (Fuzzing) e então documentada antes de ser traduzida para o Domínio Java.

---

## 2. Lacunas na Simulação Física e Fluida (Fluid Dynamics)

A mecânica de movimento básica (andar, pular, correr) foi completamente decifrada. A lacuna reside nos estados transitórios de movimento onde blocos dinâmicos interferem.

### 2.1 A Lacuna da Água e Correntezas
No cliente Vanilla, a água não é um bloco estático. Se a água tem um "Flow" (correnteza), ela empurra o jogador usando um vetor que é calculado através do `BlockLiquid.getFlowVector`.
**A Lacuna:** O código C# do bot não implementava perfeitamente o vetor de arraste da água. O bot às vezes "lutava" contra a correnteza vibrando sem sair do lugar.
**Investigação Necessária (Spike):** O time Java precisa criar um servidor de testes com blocos de água em diferentes níveis de `liquidData` (0 a 7), colocar o bot lá dentro em estado inerte (sem enviar teclas de movimento) e gravar a variação de X, Y, Z por segundo. Isso permitirá copiar a matriz de arraste (Drag Matrix) exata da Mojang.

### 2.2 A Lacuna da Lava (Viscosidade Dinâmica)
A Lava possui uma viscosidade diferente no Nether e no Overworld no jogo original.
**A Lacuna:** O AdvancedBot em C# tratava a lava de forma uniforme. Não sabemos se o Anti-Cheat verifica a diferença de velocidade quando um jogador anda na lava no Nether versus no Overworld.
**Investigação Necessária:** Avaliar o `BlockLava` do MCP (Mod Coder Pack) para determinar se a constante de fricção varia pelo ID da dimensão. Caso afirmativo, o `WorldManager` Java precisará injetar o `DimensionID` no `PhysicsEngine`.

### 2.3 Teias de Aranha (Cobwebs) e o Vetor Y
Quando o jogador cai numa Teia de Aranha, a gravidade (`motionY`) é achatada, não removida.
**A Lacuna:** Se o bot quebrar a teia enquanto cai por ela, o cliente original imediatamente recupera a gravidade total, ou existe um "momentum" (inércia)?
**O Risco:** Se o Java zerar a fricção da teia em 1 tick, o bot cairá num abismo numa velocidade não natural, levando kick por `Fly (Hover)`.

---

## 3. Lacunas no Sistema de Inventário Complexo (NBT Oculto)

A troca de itens básicos no inventário é trivial (Slots 0 a 44). A lacuna aparece nas interações com recipientes customizados criados por Plugins de Bukkit.

### 3.1 Menus Interativos (GUI Spoofing)
Servidores com Mini-games (Hypixel) ou de Facções usam Baús Falsos como "Menus In-Game". O jogador clica no Diamante para comprar VIP.
**A Lacuna:** O servidor envia um pacote `0x30 Window Items` com NBTs gigantes no diamante. O C# ignorava NBTs de inventários não-pessoais para poupar RAM.
**O Problema:** Se o servidor tiver um plugin que verifica se o cliente "leu" o Lore do item (enviando um Payload secreto via Plugin Channel `MC|ItemName`), e o bot não ler, o servidor saberá que é um Bot cego.
**Investigação:** Sniffar o tráfego de um jogador real abrindo o menu de Facções e ver se o cliente Vanilla emite algum pacote Outbound invisível quando renderiza um item com NBT complexo na tela.

### 3.2 Desync de Janelas de Troca (Villager Trading)
O Villager Trade GUI (Window Type `minecraft:villager`) possui slots ocultos para os itens resultantes da troca.
**A Lacuna:** Como o C# lidava com os cliques fantasmas no Slot de Saída quando a janela fechava devido a latência?
**Investigação:** Avaliar se o protocolo `0x0E Click Window` exige que o Transaction ID de trocas de Villagers seja serial (Síncrono com a rede) ou se o Java pode fazer cliques assíncronos no Villager.

---

## 4. Lacunas na IA e Protocolos de Combate (Killaura Heuristics)

O Killaura é o núcleo nervoso de qualquer bot. O combate do Minecraft envolve muito mais do que apontar a câmera e emitir cliques.

### 4.1 A Lacuna do Raycast Múltiplo e Hitboxes Ocultas
O Killaura do C# usava Raycast simples (Linha reta da cabeça do bot até o centro do alvo).
**A Lacuna:** O Vanilla usa um `BoundingBox` de inflação (+0.1 blocos na Hitbox do jogador) e um sistema de "Crosshair Intersect" (Interseção de mira). Se houver um bloco de folha (Transparente) na frente do inimigo, o cliente Original permite bater?
**A Dúvida Cega:** No C#, o Killaura ignorava blocos de Folha, Vidro e Placas, batendo "através" deles (Wallhack de blocos não-sólidos). Isso funcionava há 5 anos, mas não sabemos se o GrimAC atualizou suas checks de Raycast para detectar blocos transparentes.
**Investigação:** Montar um cenário de E2E com `Testcontainers`, colocar blocos de água, folhas, teias e vidros entre o Bot e o alvo, e observar se o servidor registra o hit ou emite um aviso no console de "Line of Sight Violation".

### 4.2 O Mistério do "Swing Packet" (0x0A) vs Atraso da Rede
Para um hit ser autêntico, o pacote de Animação do braço (`0x0A Animation`) deve ir à rede.
**A Lacuna:** A Mojang envia o Swing ANTES ou DEPOIS do pacote `0x02 Use Entity (Attack)`?
**O Perigo:** Alguns logs do código C# diziam para enviar o Swing antes, outros diziam depois. Essa ambiguidade na ordem dos pacotes é o prato cheio para o `Polar Anti-Cheat`.
**Investigação:** Capturar o tráfego TCP do cliente oficial 1.8.8 usando Wireshark durante uma partida de SkyWars e inspecionar a ordem exata dos bytes (Sequenciamento de combate estrito). A arquitetura Java deve usar uma Fila (Queue) imutável para garantir que essa ordem NUNCA se inverta por causa do agendamento de Virtual Threads.

### 4.3 A Lacuna do Sprinting Hit (Knockback Multiplier)
Quando um jogador atinge alguém enquanto está correndo (Sprinting), o alvo leva um Knockback enorme.
**A Lacuna:** O C# desligava o Sprint (Enviando pacote de Stop Sprint) antes de bater para evitar perder a precisão, e ligava no tick seguinte. Essa técnica (W-Tapping) era feita de forma hardcoded e exata em 1 tick (50ms).
**A Dúvida:** Os anti-cheats novos sabem que um humano não consegue soltar o botão W e apertar de novo em exatos 50ms (0 frames de delay). O W-Tapping mecânico do Bot C# provavelmente se tornou detectável (Autoclicker / Action Spam).
**Investigação:** Avaliar se o Bot Java precisará introduzir "W-Tapping Randômico" (Pular 3 ticks, desligar o sprint, esperar 2 ticks aleatórios). O time precisa validar qual é a latência física aceitável pelo servidor.

---

## 5. Lacunas no Protocolo de Entidades (DataWatcher / EntityMetadata)

O pacote `0x1C Entity Metadata` é o mais complexo do protocolo. Ele diz se um zumbi é bebê, se um jogador está agachado (Sneaking), invisível, ou pegando fogo.
No C#, usávamos uma estrutura frágil que lia um dicionário de Bytes.

### 5.1 O Mistério da Flag "Invisível"
No Minecraft, se você tomar poção de invisibilidade, a flag `0x20` do byte `0` (Entity Status) é ativada.
**A Lacuna:** O bot Killaura não deve atacar players invisíveis se a opção "Ignore Invisible" estiver ligada. O C# tentava ler o Metadata, mas às vezes o servidor enviava a invisibilidade via pacote `0x1D Entity Effect` em vez do Metadata.
**O Risco:** O bot no Java pode ignorar o pacote de Effect e continuar batendo num player que acabou de tomar poção invisível, denunciando que ele tem Aimbot/ESP (Pois ele "viu" o invisível).
**Investigação Necessária:** Testar se o servidor Vanilla 1.8 propaga a invisibilidade via Metadata Imediato ou via Potion Effect com delay. A classe `EntityTracker` do Java precisará unificar ambas as fontes de verdade para o Killaura.

### 5.2 Lacuna de Interpolação de Posição (Entity Teleport vs Relative Move)
Servidores enviam movimentos de inimigos via `0x15 Entity Relative Move` ou `0x18 Entity Teleport`.
**A Lacuna:** O C# aplicava a posição instantaneamente. Mas no jogo real, existe uma interpolação visual ("Glide").
Se o Java atualizar as BoundingBoxes instantaneamente, o cálculo de interceptação de alvo do AStar (Lead Aim) vai errar o local onde o inimigo está visualmente.
**Investigação:** Avaliar se o Killaura "Lead" deve atirar flechas na posição de rede estrita (Server-side) ou se deve aplicar o Lerp (Linear Interpolation) de 3 ticks igual o cliente visual faz para não errar.

---

## 6. Lacunas do Motor de Renderização de Textos (Chat Components)

O chat moderno do Minecraft é baseado em Componentes JSON (`IChatBaseComponent`).

### 6.1 O Risco de ClickEvents Dinâmicos
Servidores enviam mensagens como: `[Clique aqui para aceitar o TPA]`. O JSON contém um `clickEvent: { action: 'run_command', value: '/tpaccept' }`.
**A Lacuna:** No C#, o usuário VIP tinha que usar Regex na mensagem pura de texto para tentar extrair o `/tpaccept` e criar um Auto-Reply. Isso falhava se a cor do texto mudasse.
**A Dúvida para o Java:** O Bot Java vai expor o JSON bruto para a UI Web? Como o usuário da Dashboard clicará num texto interativo se o front-end for React?
**Investigação:** A equipe de Frontend/WebFlux precisará criar um parser de `MinecraftJSON -> React Components` que converta eventos `run_command` em botões HTML nativos. O Bot tem que expor a API: `bot.chat.sendInteractiveClick(eventUUID)`.

### 6.2 Lacuna de Fuga de Caracteres (Obfuscation)
O formato `§k` (Obfuscated Text) faz as letras girarem.
**A Lacuna:** O C# muitas vezes crashava quando tentava aplicar filtros de palavrões numa string obfuskada porque os caracteres rodavam fora da tabela ASCII tradicional (UTF-8 Surrogate Pairs).
**Investigação:** Testar no Java como o `Gson` (e o `JsonReader`) descarregam emojis e caracteres Kanji complexos enviados por servidores Asiáticos. O Bot deve suportar UTF-16 completo no chat.

---

## 7. Lacunas no Protocolo de Metadados Obscuros

Muitas vezes, a equipe ignora pacotes "visuais", achando que um Bot Headless não precisa deles. Contudo, servidores modernos codificam informações vitais (Como o tempo restante de um Captcha ou a vida de um Boss) em pacotes não-ortodoxos.

### 7.1 Lacunas da Scoreboard (Objective / Teams)
A Scoreboard lateral do Minecraft (pacotes `0x3B` e `0x3C`) não exibe apenas "Rank: VIP".
Servidores de Factions enviam a coordenada do "Evento KOTH" na Scoreboard, ou o tempo restante de proteção de PvP.
**A Lacuna:** O parser C# da Scoreboard quebrava frequentemente porque os servidores usam "Fake Teams" invisíveis e definem Prefix/Suffix para burlar o limite de 16 caracteres por linha.
**A Dúvida para o Java:** O módulo `ScoreboardManager` do Java conseguirá fundir o Prefix, Name e Suffix de um `ScorePlayerTeam` de volta para uma String unificada de forma transparente, de modo que a API de Scripts dos usuários (ex: `bot.scoreboard.getLine(4)`) simplesmente retorne a string limpa?
**Investigação:** Montar um teste unitário que receba o Dump de rede de uma Scoreboard do servidor Hypixel (que abusa de Fake Teams) e garantir que a fusão JSON-to-String funcione idêntica à renderização do Vanilla.

### 7.2 Lacunas dos Itens de Mapa (Map Data - 0x34)
Em servidores de Roleplay, às vezes você ganha um Mapa de papel que contém um PIN Code renderizado em pixels (Para confirmar que você não é um bot).
**A Lacuna:** O bot C# simplesmente descartava o pacote `0x34 Map Data` inteiro, pois renderizar pixels em WPF é inútil.
**Investigação para Futuro:** Como o Bot Java lidará com OCR (Reconhecimento Ótico de Caracteres) de Mapas? A arquitetura de `MapData` deve armazenar o array de `byte[]` de cores e permitir exportar isso como um `BufferedImage` do Java para que IAs externas (OpenCV ou Tesseract) possam ler o PIN Code.

### 7.3 O Mistério da Boss Bar (0x0C)
O Wither/EnderDragon usam uma barra roxa no topo da tela. Servidores modernos usam isso para exibir anúncios ou Captchas ("Digite /captcha 9481").
**A Lacuna:** Na versão 1.8.8, a BossBar não é um pacote dedicado. Ela é literalmente um `Wither` (Entidade Oculta) que o servidor spawna a 30 blocos de distância na direção que o jogador está olhando.
**A Dúvida:** Como o C# do AdvancedBot diferenciava um Wither real atacando o jogador de um Wither falso (BossBar) spawnado pelo servidor para mostrar texto?
**Investigação:** Avaliar o pacote `0x15` (Spawn Mob). O Wither falso geralmente tem a flag `Invisible` ativada e não tem AI. O Java deve implementar um Filtro de BossBar que não exponha esse Wither na lista de entidades do Killaura, para que o bot não tente dar espadadas no ar.

---

## 8. Lacunas de Controle e Macro (Mouse/Keyboard Simulation)

O AdvancedBot em C# possuía uma aba "Macro" onde o Bot movia o Mouse REAL do usuário para farmar afk enquanto ele dormia. O C# usava chamadas `user32.dll` (`mouse_event`, `SendInput`).

### 8.1 O Abandono do Controle de Hardware Nativo
**A Lacuna:** O Java tem a classe `java.awt.Robot` que permite mover o cursor do mouse e simular teclas. Porém, o Java roda no modo `Headless` (Painel Web), sem uma janela gráfica nativa.
Se o Bot está rodando numa VPS Linux Ubuntu (CLI), não existe Mouse ou Servidor X11 para o `Robot` controlar. O código vai lançar `AWTException: headless environment`.
**Ação:** A funcionalidade de "Controle de Mouse Real" deve ser completamente amputada do núcleo Java. Toda macro escrita no Java deve atuar diretamente sobre os Intents (`MoveIntent`, `AttackIntent`) da máquina de estado do Bot, nunca sobre os periféricos físicos da máquina hospedeira.

### 8.2 Captcha de Mapas (Click em Inventário via Coordenadas)
Alguns servidores exibem um Menu com um puzzle de imagens (ex: "Clique na maçã").
O C# usava OpenCV para achar a maçã na tela do Windows.
**A Lacuna:** Sem tela, como o bot resolverá o puzzle?
**Investigação:** A equipe deverá criar um serviço Web onde o bot serializa o `Container` atual e o envia (via WebSocket) para o Painel React. O humano assiste à tela no navegador e clica na Maçã, e o React devolve o `SlotID` pro Java enviar o pacote `0x0E Click Window`. Isso requer documentar a latência aceitável (Round-Trip Time) do Human-in-the-Loop.

---

## 9. Lacunas do Motor de Scripts Dinâmico (The Polyglot Gap)

A transição de Jint (C#) para GraalVM (Java) não é apenas sintática, é paradigmática.

### 9.1 Limitações de Memória do Sandbox
No C#, o usuário podia criar um array de 100 milhões de itens no Javascript e travar o programa inteiro (Memory Bomb).
**A Lacuna:** O GraalVM possui restrições de Sandbox, mas o Bot precisa de diretrizes de `ResourceLimits`.
Qual é o limite exato de tempo que um Script de usuário pode rodar antes de ser ejetado (Cancelado)?
**Investigação:** O time Java deve estressar o `Context.Builder` com loops infinitos `while(true)` e calcular quantos milissegundos o GraalVM demora para interromper a execução com segurança via `context.interrupt()`. Esse valor deverá ser rígido (ex: Máx 50ms por Tick de script).

### 9.2 Bindings de API (Injeção de Classes do Bot)
No C#, injetávamos o `bot` inteiro no script.
**A Lacuna:** Se injetarmos a classe `MPPlayer` no GraalVM, o usuário do script poderá chamar métodos internos como `bot.network.channel.close()`, desconectando a si mesmo, ou roubando chaves de sessão usando reflexão.
**Ação Obrigatória:** O Java DEVE construir DTOs/Wrappers blindados. Apenas a classe `ScriptAPIWrapper` pode ser exportada para o GraalVM (`@HostAccess.Export`). Descobrir quais métodos exatos o usuário VIP usa nos scripts antigos (Caminhar, Bater, Comer) será a lacuna final para construir esse Wrapper.

---

## 10. Lacunas Arquiteturais para o Futuro (AI & LLMs)

O AdvancedBot em C# era burro em relação à comunicação. Se alguém dissesse "Oi bot", ele não respondia, a menos que houvesse um script com `if (msg.contains("Oi"))`. No futuro da arquitetura Java, queremos integrar Large Language Models (LLMs) para o bot interagir naturalmente com os jogadores no servidor, burlando os testes de Turing que os admins fazem.

### 10.1 A Lacuna do Contexto de Mundo
Um LLM (como a API do GPT-4) precisa de contexto.
Se o jogador perguntar "O que você está segurando?", o LLM não tem acesso à memória RAM do bot para olhar o inventário.
**O Desafio de R&D (Pesquisa):** O EventBus do Java precisa construir um "Módulo de Percepção". A cada 5 segundos, esse módulo compila o estado do bot em uma String JSON minificada:
`{"health": 20, "pos": [10, 64, 10], "holding": "diamond_sword", "nearby_players": ["Notch", "Jeb"]}`.
Quando uma mensagem de chat é recebida, o Java deve empacotar esse JSON como `System Prompt` e enviar para a API da OpenAI de forma assíncrona.
**Lacuna a Descobrir:** Qual é o custo financeiro (Tokens) de injetar o estado do jogo a cada interação? Como otimizar isso para que o usuário não gaste 50 dólares de API de IA em um dia de farm?

### 10.2 A Lacuna da Ação via Texto (Function Calling)
O bot não deve apenas falar, ele deve AGIR através de texto. Se o LLM deduzir "Estou com pouca vida, vou comer", a API da OpenAI deve invocar uma função local.
**Ação:** O Wrapper Java precisará implementar as especificações do `OpenAI Tools API`. Isso não existia no C# e é terreno inexplorado para o time.

---

## 11. Apêndice A: Lacunas no Handshake de Criptografia (RSA/AES)

O login Premium exige que o servidor envie o pacote `0x01 Encryption Request`. Esse pacote contém uma Chave Pública RSA (Public Key) e um Token de Verificação (Verify Token). O cliente gera uma chave secreta AES, criptografa a chave secreta com a chave RSA do servidor e envia de volta no pacote `0x01 Encryption Response`.

### 11.1 O Risco da Geração do "Server Hash"
Para provar à Mojang que você tem uma conta original, o Bot junta a `Server ID` + `Shared Secret (AES)` + `Public Key` em uma string gigante, e gera um Hash SHA-1 disso (Notchian Hash).
**A Lacuna:** O algoritmo de Hash do Notch é bizarro. Ele lida com números negativos de uma forma que o Java puro não costuma lidar. Se a string do hash for negativa, ela deve receber um sinal de menos `-` na frente da string hexadecimal.
**A Dúvida:** O código C# tinha uma função gigante chamada `MinecraftShaDigest`. Como não vamos reaproveitar C#, será que o `MessageDigest.getInstance("SHA-1")` do Java consegue reproduzir o Hash do Notch sem jogar uma `AuthenticationException: Bad Login`?
**Investigação Necessária:** A equipe de rede precisa escrever um Teste Unitário que pegue o Input A e o Hash B extraídos do tráfego real do C# e passe pela função Hash do Java. Se não bater, não teremos como entrar em servidores Originais.

---

## 12. Apêndice B: Lacunas de Otimização no Parsing de Chunks (World Data)

O `0x21 Chunk Data` e `0x26 Map Chunk Bulk` (removido no 1.9, mas presente no 1.8) enviam colunas imensas de blocos.

### 12.1 A Ordem Y-Z-X vs X-Z-Y
Os bytes do Minecraft chegam comprimidos e achatados. No 1.8, os blocos são enviados na ordem de sub-chunks (16x16x16).
**A Lacuna:** O código C# que convertia o array de bytes em `Block[x][y][z]` era um labirinto de bit-shifting (deslocamento de bits). A documentação atual assume que a leitura no Java será direta e limpa, mas isso é uma suposição perigosa.
**A Dúvida Cega:** Em qual ordem exata a versão 1.8 empacota os arrays de Luz (Block Light / Sky Light)? Se o Java errar a leitura por 1 byte, todo o resto do Chunk (milhares de blocos) será desserializado errado. O bot vai enxergar "Bedrock" no lugar de "Ar" e tentará voar ou ficará preso sem se mexer (Pathfinder quebra).
**Investigação (Fuzzing Tático):** Gravar os bytes brutos de um pacote `0x21` chegando em um cliente Vanilla, abrir esse pacote num Unit Test, decodificá-lo e assertar as coordenadas exatas: `assertEquals(Blocks.STONE, chunk.getBlock(0, 10, 0))`. Se falhar, a matriz de deslocamento de bits está invertida.

---

## 13. Apêndice C: Lacunas de Parsing NBT em Servidores Híbridos (Forge/Cauldron)

Quando o bot C# jogava em servidores estritamente Vanilla (Spigot/Paper), o NBT era previsível. Mas servidores como Thermos, Cauldron ou Sponge misturam Mods do Forge com Plugins do Bukkit.

### 13.1 Tags Não-Documentadas
Mods do Forge costumam injetar Tags GIGANTES nos itens (ex: O Mod Tinker's Construct injeta toda a árvore de atributos de uma picareta num NBT `TinkerData`).
**A Lacuna:** O Bot precisa ignorar essas tags para não estourar a memória RAM, mas ao mesmo tempo precisa devolvê-las intactas se o bot for mover a picareta de um baú para outro.
**A Dúvida Cega:** Se o bot ler o NBT, pular as tags desconhecidas e tentar re-serializar o NBT para enviar um pacote `0x10 Creative Inventory Action`, ele corromperá a picareta do Tinker's Construct? No C# o bot nunca suportou Modo Criativo em Modpacks por causa dessa exata falha de perda de entropia (Lossy Deserialization).
**Investigação:** O time de domínio deve criar um parser NBT em Java que suporta o padrão "UnknownTag". Se encontrar um ID de NBT não oficial (Acima de 12), ele deve armazenar os bytes crus (Blob) e re-injetar na hora de salvar, mantendo a integridade criptográfica do Mod.

---

## 14. Apêndice D: Lacunas da Camada OSI (SO_KEEPALIVE)

Nós documentamos o pacote de KeepAlive do Minecraft (`0x00`). Mas existe um KeepAlive mais silencioso, operando debaixo do pano no Sistema Operacional (Camada 4 TCP).

### 14.1 O Desconexão Silenciosa (DDoS Protection Proxies)
Servidores usam proxies como o TCPShield ou Cloudflare Spectrum para evitar ataques DDoS. Esses proxies cortam conexões TCP se o socket ficar em silêncio absoluto.
**A Lacuna:** No Bot C#, os desenvolvedores não mexiam no `Socket.SetSocketOption(SocketOptionName.KeepAlive)`. No Java Netty, nós temos `ChannelOption.SO_KEEPALIVE`.
Se o bot não receber o KeepAlive do jogo por 20 segundos (Lag do servidor), o proxy TCPShield vai fechar a rota.
**Investigação Necessária:** Descobrir se a ativação da flag de Kernel `SO_KEEPALIVE` ajuda a manter o bot conectado em servidores com proxy agressivo, ou se o Java precisa enviar algum pacote Minecraft inútil (ex: `0x0F Confirm Transaction` com ID zero) só para gerar tráfego (Noise Generator) e impedir que o Firewall corte a conexão.

---

## 15. Apêndice E: Lacunas no Gerenciamento de Vida (Health & Respawn)

O pacote `0x06 Update Health` atualiza a vida, a fome e a saturação do jogador.

### 15.1 O Bug do "Bot Zumbi" (Ghost Death)
No C#, quando a vida do bot chegava a 0, ele enviava o pacote `0x16 Client Status (Respawn)` automaticamente.
**A Lacuna:** Servidores de minigames frequentemente cancelam o evento de morte do Bukkit e definem a vida do jogador de volta para 20 antes dele morrer (Para fazer um sistema de Spectator customizado).
Se o bot enviar o pacote de Respawn quando a vida bater 0, mas o servidor cancelar a morte, o bot será kickado por `Invalid packet`.
**Investigação Necessária:** A máquina de estado do Bot (MPPlayer) precisa diferenciar a "Morte Real" da "Morte Customizada". Como o cliente oficial da Mojang sabe quando mostrar a tela vermelha de "Você morreu!" e quando não mostrar? Existe alguma flag oculta no pacote `0x08 Player Position And Look` que zera o Y quando morto?

---

## 16. Apêndice F: Lacunas em Plugin Messages (BungeeCord / Bukkit)

A comunicação entre Mods (ou BungeeCord) e o Cliente ocorre via `0x3F Plugin Message`.

### 16.1 O "BungeeCord Channel" Oculto
Se o bot estiver num servidor de Lobby, e o servidor quiser mandá-lo para a sala de SkyWars, o BungeeCord não envia uma mensagem no chat. Ele geralmente envia um Proxy Kick invisível ou, em setups específicos, um Payload no canal `BungeeCord`.
**A Lacuna:** O bot C# não tinha suporte para `PluginMessage`. Ele só se movia de sala digitando `/server skywars` no chat. Isso chamava a atenção dos admins, pois jogadores normais clicam em NPCs (que disparam `PluginMessages`).
**A Dúvida para o Java:** O Netty do Java deverá ouvir e decodificar os Payloads do canal BungeeCord?
**Investigação:** O time deve documentar se o AdvancedBot precisa agir como um "Bungee Client" nativo, escutando `GetServer`, `PlayerCount` ou `Forward` (Subcanais do Bungee) para tomar decisões de navegação de Minigames no futuro, evitando comandos de chat.

---

## 17. Apêndice G: Lacunas do TabList (Player List Item)

O pacote `0x38 Player List Item` preenche o Tab (A lista de jogadores online apertando a tecla TAB).

### 17.1 O Risco da Confiança Cega na TabList
Muitas IAs legadas liam a TabList para descobrir quantos jogadores inimigos estavam perto para decidir se o bot deveria fugir.
**A Lacuna:** Servidores usam "Vanish" (Invisibilidade de Admin) ou plugins como "NickNamer" (Esconder nomes). Se um Admin estiver de Vanish, ele NÃO aparece na TabList, mas a entidade invisível dele ainda pode estar parada do lado do bot snifando os pacotes.
**O Risco:** Se o `AutoLogout` do bot Java confiar apenas na TabList, o bot será pego. Se ele olhar só o `WorldManager.getEntities()`, ele também pode falhar (pois o servidor não envia a entidade do Admin).
**Investigação (Segurança Anti-Staff):** Como o bot pode detectar a presença de Admins invisíveis? A equipe precisa testar se o pacote `0x29 Sound Effect` (Som de passos) é disparado pelo servidor quando um admin em Vanish caminha perto do bot. Caso positivo, o Módulo Auditivo (SoundListener) será a única forma infalível de detectar Staffs ocultos.

---

## 18. Apêndice H: Lacunas do Sistema de Som (Audio Engine)

O pacote `0x29 Sound Effect` e `0x28 Effect` fornecem uma enxurrada de dados de áudio (Passos, Dano, Vidro Quebrando).
No C#, o bot era "surdo". Ele descartava tudo.

### 18.1 A "Visão" através do Som (Sonar)
Se um inimigo toma uma poção de invisibilidade, ele some do `WorldManager`. Mas se ele andar, o servidor continua enviando pacotes `0x29` com o som `step.grass` nas coordenadas dele.
**A Lacuna:** É possível construir um "Radar Acústico" (Sonar) no Java que rastreia entidades invisíveis mapeando a origem dos sons de passos?
**Investigação (Machine Learning Acústico):** O time precisa gravar uma sessão onde um jogador invisível corre ao redor do bot. Avaliar se o Volume (`Volume` e `Pitch`) do pacote de som diminui linearmente de acordo com a distância matemática. Se sim, o bot Java pode usar trilateração de volume para adivinhar a coordenada `(X, Y, Z)` do inimigo invisível e atirar uma flecha nele cegamente (Aimbot Acústico).

---

## 19. Apêndice I: Lacunas na Visão Computacional (Renderização Visual Mockada)

Usuários experientes de Bots (Macroers) pedem frequentemente para usar o YOLO (IA de Visão Computacional) para resolver Puzzles que o servidor coloca na tela (Menus de GUI ou Mapas em Molduras).

### 19.1 O Fake Framebuffer
Um bot "Headless" não tem Placa de Vídeo (OpenGL). Ele não tem uma tela para o YOLO olhar.
**A Lacuna:** Como transformar os `Slot` do Baú (Que são apenas IDs de bytes) em uma Imagem 2D que uma Rede Neural possa entender?
**Investigação:** Avaliar a criação de um "Renderer 2D Falso". O Java leria a Matriz do Baú e "desenharia" (usando `java.awt.Graphics2D`) ícones de minecraft em um `BufferedImage` oculto na RAM, salvando como `frame.png` para a IA olhar. Isso requer a importação de todos os 400 ícones (`.png`) do Minecraft original para dentro da `.jar` do bot, o que aumentaria o peso do download.

---

## 20. Apêndice J: A Lacuna das Redstones e Blocos Dinâmicos

O Minecraft tem blocos que mudam de estado sem a intervenção direta da rede, regidos pelas regras físicas locais (Block Updates). Exemplo: A areia cai. A redstone pisca. O pistão empurra.

### 20.1 A "Simulação de Tick" do Lado do Cliente
O Servidor envia o pacote `0x22 Multi Block Change` quando um pistão ativa.
**A Lacuna:** O Bot precisa processar a "Gravidade da Areia"? Se o bot estiver minerando e uma areia cair em cima da cabeça dele, o servidor enviará a areia caindo como uma Entidade `FallingBlock` ou apenas forçará a vida do bot a descer (Sufocamento)?
**Investigação:** No C#, o bot frequentemente morria soterrado por cascalho (`Gravel`) porque o PathFinder achava que o cascalho suspenso era seguro para passar debaixo. O time Java precisa descobrir se o `BlockManager` deve prever atualizações de gravidade (Block Physics) para o PathFinder desviar de blocos instáveis.

---

## 21. Apêndice K: Lacunas de Fuga de Blocos (Block Glitching)

O Anti-Cheat do Vanilla possui uma mecânica rígida para jogadores presos dentro de blocos sólidos (Sufocamento).

### 21.1 A Correção de Clipping (V-Clip)
Se o bot calcular mal a física e terminar um tick com a cabeça dentro de um bloco de pedra, o servidor Vanilla empurra o jogador para o espaço livre mais próximo. Hackers usam isso enviando posições propositadamente dentro de tetos para atravessar andares (V-Clip).
**A Lacuna:** O Pathfinder do bot Java (AStar) assume que o bot nunca entrará num bloco sólido. Mas se o servidor tiver lag e teletransportar o bot (`0x08 Player Position And Look`) para dentro de uma parede, o que o Bot faz?
**Investigação:** Avaliar se o Bot deve implementar a rotina `pushOutOfBlocks` exata do `EntityPlayerSP.java` da Mojang. Se não implementar, o bot tentará andar estando preso, acumulando `Movement Violation` no Anti-Cheat até ser banido.

---

## 22. Apêndice L: Lacunas da Bússola (Compass Tracker)

Em minigames como Survival Games ou Manhunt, a Bússola não aponta para o Spawn Mundial, ela aponta para o jogador mais próximo.

### 22.1 O Pacote de Spawn Position
Servidores mudam o alvo da bússola enviando o pacote `0x05 Spawn Position` repetidas vezes.
**A Lacuna:** O bot C# ignorava o pacote `0x05` se ele chegasse depois da tela de carregamento inicial, pois assumia que o Spawn Mundial nunca mudava.
**Investigação:** Como o bot Java deve exportar essa informação para scripts? Se um usuário quiser criar um bot que persegue a bússola automaticamente (Auto-Manhunt), a API do Java deve abstrair o `0x05` como `bot.world.getCompassTarget()`. O time deve verificar se a Bússola é calculada estritamente no cliente Vanilla baseada nessa variável global.

---

## 23. Apêndice M: Lacunas de Elytra e Movimentos Especiais

Embora o foco inicial do protocolo seja 1.8.x, a arquitetura Java será multicross-version (Protocol Translators). A versão 1.9+ introduziu a Elytra.

### 23.1 A Física do Voo Planado
A Elytra não é um "estado On/Off". Ela altera completamente o vetor de inércia e gravidade do jogador.
**A Lacuna:** Se o time Java implementar o suporte a 1.16 no futuro, como o PathFinder saberá navegar pulando de um penhasco de Elytra? O algoritmo AStar tradicional é em Grade (Grid), e o voo de Elytra é vetorial contínuo (Splines/Curvas).
**Desafio R&D Futuro:** Criar uma classe abstrata de navegação no Java (`AbstractNavigator`) onde o AStar é apenas UMA implementação (`WalkingNavigator`), permitindo a criação de um `ElytraNavigator` baseado em *Steering Behaviors* ao invés de grafos no futuro.

---

## 24. Apêndice N: Lacunas da Câmera (Third Person F5)

Muitas lógicas visuais dependem de onde a câmera está.

### 24.1 O Vetor do Raycast Reverso
No C#, só existia a câmera de Primeira Pessoa.
**A Lacuna:** Servidores modernos (via pacotes customizados ou LabyMod API) podem pedir para o cliente forçar a câmera para Terceira Pessoa. Além disso, se a mecânica do Anti-Cheat testar cliques através do ângulo visual da câmera (e não do olho do jogador), como reagimos?
**Investigação:** Avaliar se o `BotState` precisa armazenar um enum `CameraMode`. Para 99% dos casos será `FIRST_PERSON`, mas a lacuna de documentação é se algum Anti-Cheat detecta falhas na oclusão visual de Terceira Pessoa.

---

## 25. Apêndice O: Lacunas no Gerenciamento de Partículas (Particle Hacks)

O pacote `0x2A Particle` envia efeitos visuais (fumaça, corações, críticos).

### 25.1 O Rastreamento de Invisíveis via Fumaça
Se um jogador invisível estiver correndo (Sprinting), o servidor não envia a entidade dele para outros jogadores distantes, mas ele ainda envia as "Partículas de Poeira" de corrida saindo do chão.
**A Lacuna:** O bot C# ignorava o pacote de partículas. Hackers avançados no Minecraft Vanilla criam mods que desenham uma linha conectando partículas de poeira para "ver" o rastro de inimigos ocultos.
**Investigação para a Inteligência do Bot:** O Bot Java pode armazenar os últimos 20 pacotes de partículas num buffer circular, rodar uma regressão linear sobre as coordenadas XYZ dessas partículas para prever a direção do inimigo e fazer o AStar desviar dele? Isso seria uma IA preditiva revolucionária para escapar de Admins.

---

## 26. Apêndice P: Lacunas na Tradução de Linguagem (Locale)

O pacote `0x15 Client Settings` envia o `Locale` do usuário (ex: `en_US` ou `pt_BR`).

### 26.1 O Bypass de Filtros de Chat
Se o bot enviar `pt_BR`, o servidor pode tentar traduzir certas mensagens via pacotes Translation Components.
**A Lacuna:** O Bot C# hardcodava `en_US`. Mas se um servidor brasileiro verificar que a conta mandou `en_US`, mas conectou de um IP de São Paulo, o heurístico do Anti-Bot pode acender um alerta amarelo.
**Investigação:** A equipe Java deve verificar se a injeção do `Locale` do sistema operacional host (`Locale.getDefault()`) no pacote de Settings é segura, ou se isso quebra os parsers de Regex antigos que assumiam que as mensagens do servidor chegariam em inglês (Ex: `msg.contains("Teleporting in 3 seconds...")`).

---

## 27. Apêndice Q: Lacunas do Sistema de Replay (Demo Format)

Servidores como o Hypixel gravam as partidas. Mas o próprio cliente do Minecraft tem um formato oculto `.mcpr` (Minecraft Replay).

### 27.1 O Risco da Falsificação de Replays (Ban Appeals)
Se a conta do bot for banida, o dono pode querer abrir um "Ticket" no servidor implorando desbanimento.
**A Lacuna:** O bot não grava a visão dele. Se o bot Java pudesse salvar o fluxo do Netty em disco num arquivo serializado `.mcpr`, o usuário poderia abrir isso no ReplayMod depois e enviar o vídeo falso como "prova" de que ele estava jogando.
**Investigação Futura:** Quão complexo é salvar a Netty Pipeline para disco *raw*? O Bot precisaria apenas escrever os pacotes de entrada e os pacotes de saída num arquivo binário com um Timestamp. O time de rede precisa avaliar o custo de I/O em disco disso (Disk IO Bottleneck).

---

## 28. Apêndice R: Lacunas de Otimização de Garbage Collector (ZGC)

Nós ditamos que o bot usará o ZGC (Z Garbage Collector) do Java 21 para ter pausas de menos de 1ms.

### 28.1 O Risco do "Allocation Stall"
O ZGC é concorrente, o que significa que ele limpa a memória enquanto o bot continua rodando.
**A Lacuna:** Se o bot alocar memória mais rápido do que as threads do ZGC conseguem limpar (Por exemplo, baixando 50 mil blocos de um Chunk num décimo de segundo), o ZGC congela o Bot (Allocation Stall) até limpar a sujeira. Esse congelamento durará dezenas de milissegundos, gerando lag spike na rede (Timeout).
**Investigação Obrigatória:** A equipe de DevOps precisa rodar o Bot Java no JFR (Java Flight Recorder) por 4 horas andando em um servidor de mundo infinito. Se houver sequer 1 evento de `ZGC Allocation Stall`, a arquitetura do Bot terá que obrigatoriamente migrar o Netty para `PooledByteBufAllocator` (Reuso de memória) estrito em todas as pontas, zerando a alocação de Byte Arrays virgens.

---

## 29. Apêndice S: Lacunas no Tracking de Experiência (XP Levels)

O servidor envia o pacote `0x1F Set Experience` contendo a barra de XP atual (0.0 a 1.0), o Level Total, e o XP Total.

### 29.1 A Matemática de Encantamentos
No C#, nós usávamos o Level diretamente do pacote para decidir se o bot tinha nível suficiente para encantar uma espada.
**A Lacuna:** A fórmula de XP do Minecraft não é linear. Se um plugin de servidor (ex: McMMO) enviar um pacote de XP adulterado para usar a barra de XP como um "Cooldown de Skill", o Bot Java ficará confuso achando que tem Nível 50 quando na verdade a barra está apenas cheia visualmente.
**Investigação:** O `MPPlayer` deve criar dois campos separados: `realVanillaXpLevel` (calculado pelo bot baseado nos orbs absorvidos `0x11 Spawn Experience Orb`) e `visualXpLevel` (O que o pacote 0x1F diz). Testar como os bots de pesca automática devem lidar com a barra visual sendo sequestrada por plugins de cooldown.

---

## 30. Apêndice T: Lacunas na Criptografia de Senhas (Database Security)

O AdvancedBot em C# guardava as senhas das contas (Premium e Pirata) em um arquivo de texto puro `.txt` ou criptografadas com um AES cuja chave estava hardcodada no código-fonte (`AdvancedCrypto.cs`).

### 30.1 O Vetor de Ataque no H2 Database
Com a migração para Java e banco H2, as senhas ficarão no banco.
**A Lacuna:** A Mojang não permite senhas em texto puro em clientes third-party. Se o bot Java vazar a tabela do H2, os usuários perderão suas contas originais de 30 dólares.
**A Dúvida para a Arquitetura:** Qual algoritmo de hash devemos usar para a senha Mestra do Dashboard? Bcrypt, SCrypt ou Argon2id? E como vamos guardar a senha da Mojang? Ela não pode ser "Hashada" porque o bot precisa dela em texto puro para enviar pra API de Login da Microsoft (`OAuth2`).
**Investigação:** A equipe de Segurança deve projetar um `Keystore` nativo ou integrar com o `Windows Credential Manager` (via JNA) e o `GNOME Keyring` (Linux) para que as senhas da Mojang nunca fiquem no H2 Database, deixando o SO cuidar da criptografia de dados em repouso.

---

## 31. Apêndice U: Lacunas na Simulação de Ping (Ping Spoofing Avançado)

O pacote `0x00 KeepAlive` mede o ping.

### 31.1 O Heurístico de Variância (Jitter)
No C#, se o usuário quisesse simular 200ms de ping, o bot interceptava o pacote e aplicava um `Task.Delay(200)`.
**A Lacuna:** Um ping perfeitamente cravado em 200.0ms não existe na vida real. Redes reais sofrem Jitter (Variação de delay). Anti-Cheats como o Vulcan e o GrimAC começaram a banir jogadores cujo Ping tem um desvio padrão menor que 2ms (pois é impossível um ping ser tão perfeito).
**Investigação:** O time de Java precisa criar um `PingSpoofer` baseado em Distribuição Normal (Curva de Gauss). Se o usuário pedir 200ms, o bot deve aplicar: `200ms + (Random.nextGaussian() * 15ms)`. Testar a validação matemática dessa curva contra a análise estatística do Vulcan.

---

## 32. Apêndice V: Lacunas no Reconhecimento de Biomas (Biome Data)

Os pacotes de Chunk (`0x21` e `0x26`) também enviam um array de 256 bytes representando os Biomas (Plains, Desert, Ocean).

### 32.1 O Impacto Oculto da Água
A cor da água e o congelamento dependem do bioma.
**A Lacuna:** O bot C# ignorava o Array de Biomas completamente. Mas em biomas frios (Taiga), a água congela e vira gelo.
Se o bot não souber o bioma, o Pathfinder assumirá que um bloco de água está líquido, mas o cliente visual saberá que congelou. E se o servidor rodar um plugin sazonal (Winter Wonderland)?
**Investigação:** Avaliar se a física da água no `PhysicsEngine` deve consultar a tabela de temperaturas de biomas do Minecraft 1.8 antes de decidir se o bot pode "nadar" naquele bloco, prevenindo que o bot tente nadar em blocos de gelo sólido.

---

## 33. Apêndice W: Lacunas de Fragmentação de Pacotes (Packet Sizing)

No protocolo Minecraft, o tamanho máximo de um pacote não comprimido é tradicionalmente de 2097151 bytes (2MB).

### 33.1 O Estouro de Buffer em Modpacks
Alguns servidores com muitos Mods Customizados (Custom Payload via `0x3F`) enviam pacotes de sincronização que excedem 2MB. O C# possuía buffers estáticos.
**A Lacuna:** O Netty, por padrão, lança uma `TooLongFrameException` se o frame passar do limite configurado no `LengthFieldBasedFrameDecoder`. O bot C# crashava na conexão.
**Investigação:** O time de Java precisa decidir se vai aumentar o `maxFrameLength` do Decoder de forma irresponsável (Ex: 100MB) para acomodar Modpacks, sob o risco de abrir vulnerabilidade para DDoS (OOM Crash) se um servidor falso mandar 100MB de lixo. Qual o limite de buffer ideal para o ecossistema 1.8?

---

## 34. Apêndice X: Lacunas do Sistema de Veículos (Boats & Minecarts)

A movimentação do jogador a pé é gerida pelo pacote `0x04 Player Position`. Porém, quando o jogador entra num barco (Entity Mount), o protocolo muda.

### 34.1 O Movimento Relativo (Steer Vehicle)
Para mover um barco, o bot não envia `0x04`. Ele envia o pacote `0x0C Steer Vehicle` (Eixos X e Z, Jump, Unmount).
**A Lacuna:** O bot C# nunca soube dirigir um barco. O AStar PathFinder do bot antigo só funcionava andando no chão. Se o usuário tentasse botar o bot para farmar num oceano usando barco, o bot travava.
**A Dúvida R&D:** O barco do Minecraft 1.8 é famoso por quebrar facilmente (Desync de hitboxes). Se o bot Java precisar dirigir barcos, o `PhysicsEngine` terá que simular a inércia terrível do barco no gelo. Além disso, o pacote `0x18 Entity Teleport` do barco chega via servidor. Quem é a fonte da verdade? O Steer ou o Server Teleport?
**Investigação:** Criar um módulo de `VehicleManager` separado do `PlayerManager`, garantindo que Intents de Movimento (`MoveIntent`) sejam traduzidos transparentemente para o pacote de Veículo correto.

---

## 35. Apêndice Y: Lacunas da Fila de Login (2B2T Queue Systems)

Servidores cheios (como 2B2T ou Hypixel) colocam os jogadores numa "Fila de Espera" no Limbo ou num Servidor de Fila antes de mandá-los ao servidor real.

### 35.1 A Falsificação de Movimento na Fila (Anti-AFK Kick)
Servidores de fila punem jogadores estáticos. Se você não andar enquanto está esperando por 12 horas, você é kickado por AFK.
**A Lacuna:** O bot C# ficava parado. O usuário tinha que abrir o Killaura ou o AutoWalk manualmente. Mas se o bot está numa Dimensão Vazia de Fila, o AutoWalk cai no void e morre.
**Investigação:** O bot Java precisa de um "Limbo Mode". Se ele detectar que está numa sala vazia ou que a Scoreboard contém a palavra "Queue / Fila / Posição", ele deve ligar um `AntiAFK` micro-movimentacional, girando a cabeça 5 graus e pulando a cada 1 minuto, independente do `PathFinder` estar ativado ou não.

---

## 36. Apêndice Z: Lacunas no Gerenciamento de Janelas (Ghost Items)

Quando o jogador abre um baú, ele usa o `0x0E Click Window` para mover itens.

### 36.1 O Desync Silencioso (Transaction Rejected)
O servidor responde ao clique com o pacote `0x32 Confirm Transaction` (Aceito = true/false).
**A Lacuna:** Se o servidor disser `false` (Transação Rejeitada por causa de Lag ou Anti-Cheat), o item volta para o slot original. Mas o C# era otimista, ele confiava no clique e assumia que o inventário local já estava atualizado. Isso gerava "Ghost Items" (Itens fantasmas que o bot achava que tinha, mas não tinha).
**A Dúvida de Concorrência:** O Java deve ser "Otimista" (Atualiza a UI Web instantaneamente e faz rollback se falhar) ou "Pessimista" (A interface Web só atualiza o baú quando o pacote `0x32 Confirm` chegar)?
**Investigação Necessária:** Testar a latência do "Pessimista" em uma conexão de 200ms. Se a UI web ficar muito "lenta" e responsivamente ruim para o usuário, o bot terá que implementar o padrão de Otimismo com `State Rollback` (Snapshot isolation), o que dobra a complexidade da arquitetura de estados.

---

## 37. Apêndice AA: Lacunas de Fila de Eventos (Event Ordering)

O Java Reactor (WebFlux) ou LMAX Disruptor podem rodar eventos em paralelo.

### 37.1 O Risco da Condição de Corrida (Race Condition) no Dano
Se um pacote `0x06 Update Health` (Vida = 0) chegar junto com um pacote `0x21 Chunk Data` no mesmo milissegundo.
**A Lacuna:** O bot C# era single-threaded. Ele sempre processava pacotes na ordem em que saíram do buffer TCP. Se o Java tentar ser "inteligente" e jogar o parsing do Chunk para uma Thread de Background (Worker Thread) enquanto a Main Thread processa a Morte do Bot, o que acontece?
O bot pode tentar dar Respawn enquanto o Chunk ainda está sendo decodificado.
**Investigação:** A equipe Java deve provar matematicamente que o desacoplamento de decodificação de Chunks não gera violações de causalidade. Se um pacote B chegou depois de A, B nunca pode afetar o estado do `WorldManager` antes de A, mesmo que A seja um pacote pesado (Chunk) e B seja um pacote leve (Chat).

---

## 38. Apêndice AB: Lacunas na Renderização 3D (Waypoints e ESP)

Embora o bot seja Headless, muitos usuários criam "LabyMod Addons" que se conectam ao painel Web do bot via WebSocket para desenhar coisas na tela do próprio jogador (Overlay).

### 38.1 A Projeção de Matrizes (Frustum Culling)
**A Lacuna:** O C# não exportava Matrizes de Projeção (ModelView e Projection Matrix).
Se o usuário quiser que o Dashboard do bot envie as coordenadas de todos os baús com diamante ao redor do bot para que o celular dele mostre uma bússola apontando para o diamante 3D mais próximo.
**Investigação R&D:** Como criar um sistema de "Fake OpenGL" no Java que calcula se a coordenada `(X, Y, Z)` está dentro do FOV (Field of View) do Bot, para não enviar milhares de Waypoints via WebSocket e travar o navegador do usuário?

---

## 39. Apêndice AC: Lacunas no Gerenciamento de Entidades Complexas (Vara de Pescar)

A Entidade de Vara de Pescar (`EntityFishingHook`) é uma das mecânicas mais cruéis do protocolo.

### 39.1 O Vínculo de IDs (Owner ID)
O pacote de Spawn do anzol de pesca envia o `EntityID` do anzol e o `OwnerID` (Quem jogou a vara).
**A Lacuna:** Se o bot jogar a vara, e o servidor bugar e não enviar o `OwnerID` correto, o bot Java vai puxar a vara de volta na hora certa? O AutoFish do C# usava uma gambiarra (ele assumia que o anzol mais próximo do bot era dele).
**Investigação:** Avaliar se o servidor `Paper 1.8.8` envia o `OwnerID` perfeitamente, e como o Java deve vincular (Link) o anzol ao `MPPlayer` de forma segura, para evitar que o bot "puxe" o peixe quando a boia de outro jogador afundar.

---

## 40. Apêndice AD: Lacunas na Desconexão (RST Packet vs FIN Packet)

O Netty Java lida com o fechamento de Sockets de forma muito elegante via `ChannelFuture`.

### 40.1 O Hard Crash do Host (Socket Reset)
**A Lacuna:** Quando um servidor de Minecraft crasha (O dono tirou da tomada), ele não envia pacote TCP `FIN`. O Socket fica pendurado (Half-open).
O Bot C# demorava 2 minutos para perceber que tinha caído.
**Investigação:** Como o `ReadTimeoutHandler` do Netty afeta a UX do bot? Se configurarmos o Timeout para 15 segundos, e o servidor de Minecraft tiver um Lag Spike gigantesco (Limpando o chão de 100.000 itens) que trava a Main Thread dele por 20 segundos. O bot vai desconectar achando que o servidor caiu, mas na verdade foi só um lag.
Descobrir o limite empírico perfeito do Tempo de Timeout para servidores de Minecraft Híbridos (Média aceitável de "Server Not Responding").

---

## 41. Apêndice AE: Lacunas no Block Breaking (Animação de Quebra)

O bot minera enviando `0x07 Player Digging` com a ação `START_DESTROY_BLOCK`, espera o tempo matemático, e envia `STOP_DESTROY_BLOCK`.

### 41.1 A Ausência do Arm Swing na Mineração
**A Lacuna:** Quando um jogador real segura o botão do mouse para quebrar uma pedra, ele envia múltiplos pacotes `0x0A Animation (Swing Arm)` a cada 6 ticks (300ms) durante todo o processo de mineração.
O bot C# antigo era preguiçoso. Ele enviava APENAS o Digging, mas não balançava o braço enquanto a barra de progresso da pedra enchia.
**O Risco:** Anti-cheats modernos como GrimAC correlacionam o pacote de mineração com a animação do braço. Se você quebrar uma obsidiana (que demora 10 segundos) sem balançar o braço nenhuma vez, o GrimAC flagga o bot como `FastBreak/NoSwing`.
**Investigação:** O bot Java deve implementar um `BlockBreakOrchestrator` que crie uma ScheduledTask repetitiva para disparar Swing Packets ritmados durante minerações longas. A lacuna é: Qual a frequência exata (em milissegundos) que o cliente Vanilla emite o Swing Arm quando o botão esquerdo é pressionado continuamente?

---

## 42. Apêndice AF: Lacunas da Bounding Box de Montarias (Cavalos)

Quando o jogador está montado num cavalo, a Hitbox (Bounding Box) do cavalo engole a do jogador.

### 42.1 O Desvio de Dano em Combate Montado
**A Lacuna:** Se o bot estiver num cavalo, o Killaura deve focar o Raycast na cabeça do inimigo ou no peito?
O C# batia no "Centro" da BoundingBox. Mas quando o inimigo está a cavalo, o centro da BoundingBox composta fica na barriga do cavalo. O bot acabava batendo no cavalo e não no inimigo VIP.
**Investigação R&D:** Como o Java deve resolver a colisão hierárquica? Se `Entity A` está montada em `Entity B`, o Killaura deve mirar no topo da Hitbox de B (Que é o pé de A) ou aplicar um offset matemático fixo no Y de A? A resposta exige debug visual do Minecraft original.

---

## 43. Apêndice AG: Lacunas de Gerenciamento de Animações Inimigas

O pacote `0x0B Animation` do servidor diz quando um zumbi balança o braço ou um Creeper incha.

### 43.1 O Parry Automático (Auto-Block)
O C# tinha uma função de `Auto-Block` que defendia com a espada 1 tick antes do inimigo bater.
**A Lacuna:** O bot baseava a defesa num timer interno. Mas jogadores reais usam "Delay Clicks". Se o jogador inimigo balançar o braço (`0x0B`), ele tem um atraso na rede.
**A Dúvida:** O Java deve tentar prever o hit do inimigo reagindo ao pacote `0x0B` de animação do braço do oponente? O pacote `0x0B` chega ANTES do pacote de Dano real ser calculado no servidor?
**Investigação:** Avaliar o código fonte do Bukkit. Quando um jogador bate no outro, o Bukkit envia o Dano (`UpdateHealth`) e a Animação (`Animation`) no mesmo Tick? Se sim, o Auto-Block reativo a animações é impossível e a IA deve continuar sendo baseada em distância euclidiana.

---

## 44. Apêndice AH: Lacunas das Client Settings Ocultas (Chat Colors)

O pacote `0x15 Client Settings` tem uma flag booleana chamada `Chat Colors`.

### 44.1 O Bug do "Bot Cego para Cores"
Se `Chat Colors` for enviado como `false`, o servidor Bukkit para de enviar o caractere de cor `§` para economizar banda.
**A Lacuna:** O C# sempre enviava `true`. Se o usuário da Dashboard Java do bot colocar "Economizar Banda (Sem Cores)", e o bot enviar `false`, o que acontece com a Scoreboard? A Scoreboard (pacote `0x3B`) muitas vezes DEPENDE de cores para diferenciar a facção inimiga (Inimigos em vermelho, aliados em verde).
**Investigação:** O time precisa documentar se as Cores da Scoreboard também são castradas pelo servidor quando o Client Setting de Cores é desligado. Se forem, a configuração de desativar cores deve ser removida do UI do Bot Java, pois cegaria o bot para lógicas de IA baseadas em Facções.

---

## 45. Apêndice AI: Lacunas de Física (Escadas e Trepadeiras)

Blocos escaláveis (Climbable) como Escadas (Ladders) e Trepadeiras (Vines) anulam a gravidade convencional do jogador.

### 45.1 O Bypass de Queda (Water Bucket / Ladder Clutch)
No C#, se o bot caísse de um penhasco, ele não sabia "agarrar" a escada no meio do ar.
**A Lacuna:** O motor físico (PhysicsEngine) precisará calcular a intersecção do BoundingBox do bot com o BoundingBox da escada *durante o vetor de queda* (motionY < 0).
**A Dúvida R&D:** Se o bot "esbarrar" na escada caindo a 30 blocos por segundo, o servidor Vanilla zera a velocidade de queda instantaneamente ou aplica uma desaceleração progressiva (Fricção Y)? Se zerar instantaneamente no cálculo local, o bot pode sofrer "Fall Damage" atrasado do servidor se houver desync de 1 tick.
**Investigação:** O time de física deve criar um `Testcontainer` onde o bot despenca de Y=255 e tenta agarrar uma escada em Y=10. Gravar a curva de desaceleração exata que o cliente Mojang envia e mimetizar isso no Java.

---

## 46. Apêndice AJ: Lacunas de Bounding Box (Agachar / Sneak)

Quando o jogador aperta SHIFT (Sneak), a altura da hitbox do jogador é reduzida. Na 1.8, passa de `1.8m` para `1.65m`. Na 1.14+, passa para `1.5m`.

### 46.1 O Movimento em Tetos Baixos (Slab Crawling)
**A Lacuna:** O Pathfinder AStar original assumia que o bot sempre tinha 2 blocos de altura livres.
Se um jogador (bot) estiver agachado num túnel de Slab (1.5 blocos de altura) e o script disser "pare de agachar", o cliente Mojang impede o jogador de levantar (pois a cabeça bateria no teto).
O Bot C# não checava o teto. Ele enviava o pacote `0x0B Entity Action (Stop Sneaking)`, e imediatamente enfiava a cabeça dentro do bloco de cima, ativando flag de "NoClip" no Anti-Cheat.
**Investigação Necessária:** O `ActionManager` do Java deve implementar uma checagem de Raycast Vertical (AABB collision check) *antes* de autorizar o envio do pacote `Stop Sneaking`. A lacuna é garantir que as matemáticas de AABB das Lajes (Slabs), Escadas e Neve em Camadas (Snow Layers) estão 100% precisas no repositório estático de blocos.

---

## 47. Apêndice AK: Lacunas no Limite de Entidades (Entity Culling)

Servidores de MobTrap (Factions) podem acumular milhares de Porcos em 1 único bloco.

### 47.1 O Risco de Iteração O(N²) no Killaura
Se existirem 5000 porcos num raio de 5 blocos, o bot C# iterava sobre os 5000 porcos para descobrir qual estava mais perto do centro da tela.
**A Lacuna:** Com 5000 porcos, calcular Distância 3D + Raycast para cada porco, a 20 TPS, frita a CPU e aumenta o consumo de RAM brutalmente, estourando a latência do Tick (Tick Drift).
**Investigação de Otimização:** O Bot Java deverá implementar uma árvore de particionamento espacial (Octree ou Spatial Hashing) no `WorldManager` para que o Killaura apenas itere entidades num subset local, ou usar `Entity Culling` (Esconder entidades que estão fora do campo de visão do Killaura). Qual a estrutura de dados concorrente mais rápida do Java para buscas espaciais em mundos dinâmicos?

---

## 48. Apêndice AL: Lacunas de Interações com NPCs (Armor Stands)

Em servidores Modernos (1.8+), a grande maioria dos "Hologramas" e "Lojistas" que flutuam no ar são, na verdade, Entidades `ArmorStand` invisíveis.

### 48.1 O "Click Incorreto" em Hologramas
O jogador lê um texto flutuante "Clique para Comprar VIP" e tenta clicar.
**A Lacuna:** O bot C# via o ArmorStand através do `WorldManager`, mas não sabia se ele era clicável (Interactable) ou se era apenas decorativo. Como resultado, em lógicas de scripts, o bot às vezes tentava bater no holograma de compra de VIP achando que era um monstro, pois a flag `IsInvisible` do ArmorStand estava ativa.
**Investigação:** O time Java deve documentar heurísticas para o Killaura "Ignorar ArmorStands com Nomes Customizados". Além disso, deve-se descobrir qual o pacote exato gerado ao clicar com botão direito num ArmorStand flutuante (`0x02 Use Entity` do tipo `INTERACT_AT`). O C# apenas suportava `INTERACT`, o que pode ser rejeitado por Lojistas complexos que exigem que o clique tenha a coordenada exata `(Target X, Target Y, Target Z)` de onde no Armor Stand a mão bateu.

---

## 49. Apêndice AM: Lacunas da Inteligência Aquática (AStar Pathfinding em Fluidos)

Nós mencionamos a lacuna de "Correnteza", mas o próprio cálculo de nós (Nodes) do AStar na água é falho no C#.

### 49.1 O Risco do Afogamento em Túneis de Água
O Bot C# não sabia distinguir entre um lago (Onde ele pode subir para respirar) e um aquífero subterrâneo (Onde o teto é pedra).
**A Lacuna:** Como o Minecraft calcula o esgotamento do oxigênio? A cada Tick debaixo d'água o oxigênio cai. O C# ignorava o nível de ar e tentava minerar diamantes no fundo do mar, morrendo afogado frequentemente.
**Investigação R&D:** O pacote `0x06 Update Health` não envia a barra de ar (Bolhas). A barra de ar é gerida nativamente pelas Entidades (No lado do cliente) baseada no tempo que a HitBox dos olhos fica dentro de um `BlockLiquid`. A equipe Java precisa descobrir qual é a fórmula de subtração de Ar do Vanilla 1.8 para criar um alarme no Bot (`bot.getAirTicks()`) e forçar o Pathfinder a subir para a superfície 5 segundos antes do dano de afogamento começar.

---

## 50. Apêndice AN: Lacunas de Obfuscação e Proteção IP (Licenciamento)

O Bot será um software Premium (Pago).

### 50.1 O Risco da Descompilação (Reverse Engineering)
No C#, usávamos o `ConfuserEx` para proteger o executável.
No Java, se distribuirmos um `.jar` comum, qualquer usuário usando `JD-GUI` ou `CFR` copia 100% do código-fonte em 10 segundos e cria um crack.
**A Lacuna:** A equipe nunca protegeu código Java antes.
Qual será a cadeia de ofuscação (ProGuard, Allatori, String Encryption, Flow Obfuscation)?
Se obfuscarmos a classe `HandshakePacket` para `a.b.c.a`, o Netty pode quebrar, pois ele muitas vezes usa `Reflection` para instanciar pacotes pelo nome da classe.
**Investigação:** Testar se o `ProGuard` quebra o ecossistema Hexagonal. Descobrir se as classes anotadas com `@Subscribe` do EventBus do Guava sobrevivem à renomeação agressiva. Se quebrarem, teremos que escrever regras gigantescas (`-keep class com.advbot.** { *; }`) no arquivo `proguard.pro`, o que reduz a eficácia da proteção. A equipe precisa de 2 sprints focados apenas em Licenciamento e HWID Tracking para fechar essa lacuna de segurança.

---

## 51. Apêndice AO: Lacunas de Mimetização de Clientes (LabyMod / LunarClient Spoof)

Os servidores modernos não gostam do Minecraft Vanilla. Eles adoram jogadores que usam o LunarClient, pois esses clientes têm um Anti-Cheat embutido (Client-Side).

### 51.1 O Spoofing de Handshake do Lunar
Se o Bot disser que é "LunarClient", o servidor confia cegamente que ele não está usando Killaura.
**A Lacuna:** Como o Lunar Client prova para o servidor que ele é autêntico?
O servidor pode pedir um "Auth Hash" especial via pacote `0x3F Plugin Message` no canal `lunarclient:pm`. O C# não tinha capacidade de gerar os tokens criptográficos do Lunar.
**Investigação (Sniffing Avançado):** Fazer engenharia reversa no tráfego TCP do Lunar Client real. Quais pacotes ele envia ao logar? O time Java precisa de um módulo `SpooferModule` que possa responder aos desafios de criptografia do Lunar para que o bot ganhe a "Bandeira Branca" do Anti-Cheat do servidor sem ser detectado como fraude.

---

## 52. Apêndice AP: Lacunas de Referência Cíclica e Vazamentos de Memória

No C#, o Garbage Collector lidava bem com pequenas referências cíclicas graças ao limite pequeno de memória e structs na Stack. No Java, tudo que envolve Entidades é armazenado na Heap.

### 52.1 O "Vazamento do Jogador Desconectado"
O `WorldManager` guarda as Entidades num `ConcurrentHashMap<Integer, Entity>`.
**A Lacuna:** Quando um inimigo foge do campo de visão (Render Distance) ou desloga, o servidor envia o pacote `0x13 Destroy Entities`.
Se o bot Java apenas deletar a Entidade do `WorldManager`, mas o `Killaura` mantiver uma variável `Entity currentTarget` apontando para esse inimigo, essa Entidade **nunca** será coletada pelo Garbage Collector (GC Leak).
**Investigação R&D:** Avaliar o uso de `WeakReference<Entity>` nas IAs do Java. Se a IA usar uma WeakReference para segurar o Alvo, quando a Entidade for removida do mapa principal, a IA a perderá automaticamente no próximo ciclo de GC, tornando impossível criar vazamentos de memória (Memory Leaks) através de alvos desatualizados.

---

## 53. Apêndice AQ: Lacunas de Controle Remoto (Discord C&C)

O AdvancedBot era famoso por permitir que o usuário controlasse 10 bots ao mesmo tempo pelo Discord, usando a infraestrutura do Discord como um Command & Control (C&C).

### 53.1 O Abuso de Webhooks vs WebSocket
No C#, o bot enviava Webhooks HTTP para notificar mortes (ex: "O bot 1 morreu no Gladiador").
**A Lacuna:** Webhooks só enviam dados, eles não recebem comandos. Se o dono mandar "!killaura on" no Discord, como o Bot Java lê isso sem ser banido do Discord por API Spam (Rate Limit)?
**Investigação:** O bot Java não pode instanciar uma instância completa do JDA (Java Discord API) consumindo 150MB de RAM adicionais por bot. A arquitetura precisará de um "Master Node" (O painel Web) que centraliza a conexão com o Discord via Gateway WebSocket, e roteia os comandos para as instâncias filhas (Bots) via Redis Pub/Sub ou RabbitMQ. O time precisa desenhar essa lacuna de topologia.

---

## 54. Apêndice AR: Lacunas no Dimensionamento de Múltiplas Instâncias (Docker Swarm)

Usuários experientes rodavam 50 telas do AdvancedBot no Windows. O Java foi pensado para ser conteinerizado (Docker).

### 54.1 O Conflito de Portas HTTP e JMX
Se o usuário rodar 50 containers do Bot Java.
**A Lacuna:** O Dashboard WebFlux rodará na porta 8080. Se 50 bots tentarem abrir a 8080 na mesma VPS Linux (modo host), teremos 49 crashes `PortAlreadyInUseException`.
**Investigação:** Como a orquestração será feita? O Bot Java deve ter o `server.port=0` (porta dinâmica) e se registrar automaticamente no Painel Master via Eureka (Service Discovery) ou o Painel Master deve ser o único que abre portas, rodando os Bots como Threads (Vert.x Worker Verticles) ao invés de múltiplos processos isolados? A decisão entre Multi-Threaded vs Multi-Process definirá o sucesso comercial do projeto.

---

## 55. Apêndice AS: Lacunas do Proxy Autenticado (SOCKS5 Username/Password)

O uso de Proxies é vital. No C#, a biblioteca de Sockets nativa suportava SOCKS5 não-autenticado facilmente.

### 55.1 A Falha Silenciosa de Autenticação RFC 1928
Muitos usuários compram Proxies privados IPv4 que exigem Usuário e Senha (RFC 1929).
**A Lacuna:** O Netty Java possui o `Socks5ProxyHandler`, mas a configuração de `Socks5PasswordAuthRequest` é notoriamente instável se o servidor Proxy retornar métodos de autenticação fragmentados (ex: GSSAPI suportado, mas falhando). O bot C# crashava com `Authentication Failed` se o Proxy demorasse mais de 2 segundos para validar a senha.
**Investigação:** A equipe de rede deve criar um `Integration Test` com um servidor `Dante` SOCKS5 local, simulando 300ms de latência e packet loss de 10% durante a autenticação SOCKS, garantindo que o Java não desista da Handshake do Proxy prematuramente, o que causaria a exposição do IP real como Fallback (Vazamento de IP Crítico).

---

## 56. Apêndice AT: Lacunas de Física de Itens no Chão (Item Entity)

Quando um zumbi morre, ele dropa carne podre (pacote `0x0E Spawn Object` tipo 2).
O Killaura antigo usava um `ItemMagnet` para pegar os itens.

### 56.1 O "Magneto" Telepático
No C#, o bot teleportava o próprio jogador para cima do item em 1 tick para pegá-lo (Looting).
**A Lacuna:** O Anti-Cheat GrimAC proíbe movimentação que ignora a física e a gravidade. Se um diamante cai num buraco e o bot se teleporta pro buraco e volta, ele toma ban por `Jesus/Fly`.
**Investigação:** O bot Java deve ser capaz de criar um Pathfinding até a entidade de Item (`ItemDrop`). Mas a Entidade de Item do Minecraft também tem gravidade e quica (Bounce). Se o bot perseguir um item quicando, ele parecerá um Aimbot. O Killaura de itens deve ter um *Delay de Assentamento* (Settling Time), esperando o pacote de Movimento do item indicar que `motionY = 0` (Bateu no chão e parou) antes de tentar caminhar até ele.

---

## 57. Apêndice AU: Lacunas na Criptografia de Tokens OAuth2 (Microsoft XSTS)

Para o bot logar com contas Originais (Premium), ele precisa fazer a dança do OAuth2 da Microsoft, trocando o Xbox Live Token por um XSTS Token, e finalmente por um Minecraft Access Token.

### 57.1 O Expirar do Refresh Token (Token Rotation)
O Bot C# guardava o Access Token num `.json`. Quando expirava, ele usava o Refresh Token.
**A Lacuna:** A Microsoft mudou as regras do OAuth2. Refresh Tokens agora expiram ou são rotacionados a cada uso. Se o Bot Java tentar ligar, usar o Refresh Token, mas o Painel Web (Dashboard) crashar antes de salvar o *novo* Refresh Token no banco de dados H2, a sessão da Microsoft daquele bot está perdida para sempre. O usuário terá que abrir o navegador e relogar manualmente.
**Investigação R&D:** O time Java deve implementar "Transações ACID" na rotação de tokens. A atualização do JWT no Banco de Dados H2 deve ser comitada (Commit) estritamente antes do Netty disparar o pacote de Handshake para o servidor, garantindo que o token não se perca em caso de JVM Panic.

---

## 58. Apêndice AV: Lacunas de Contenção de Threads (EventBus Contention)

No C#, o EventBus disparava eventos e qualquer um podia ouvir de forma síncrona.

### 58.1 O Gargalo de 20.000 Eventos por Segundo
Em servidores densos (Muitos mobs e jogadores), o servidor envia milhares de pacotes de Movimento por segundo.
Se tivermos 10 listeners registrados para o pacote de movimento (Killaura, AntiAFK, Radar, GUI), 10.000 * 10 = 100.000 chamadas de método por segundo.
**A Lacuna:** O Java (especialmente o Reflection do Spring ou Guava EventBus) sofre de "Lock Contention" se muitos Threads tentarem publicar no Barramento ao mesmo tempo.
**Investigação Necessária:** Benchmark de performance usando JMH (Java Microbenchmark Harness). Comparar a velocidade do Guava EventBus tradicional versus LMAX Disruptor (Ring Buffer) versus RxJava/Project Reactor. Se o LMAX for 50x mais rápido (o que costuma ser), a arquitetura base terá que obrigatoriamente abandonar o Guava EventBus em favor do Disruptor para suportar DataCenters.

---

## 59. Apêndice AW: Lacunas no Backoff Exponencial (Auto-Reconnect)

O bot Java será Headless, rodando 24/7. Se a VPS do bot perder conexão com a internet por 5 minutos, ele precisa se auto-reconectar ao servidor de Minecraft.

### 59.1 O Risco do IP Ban (DDoS)
Se o bot tentar reconectar a cada 1 segundo sem parar, o firewall do servidor (Cloudflare) vai detectar um ataque SYN Flood e banir permanentemente o IP da VPS.
**A Lacuna:** O bot C# não possuía um Backoff Exponencial. Ele apenas tentava de 5 em 5 segundos, falhando miseravelmente quando os firewalls ficavam estritos.
**Investigação R&D:** O módulo `ReconnectStrategy` precisa implementar um `Jittered Exponential Backoff`. Se a tentativa N falhar, espere `(2^N) + Jitter(ms)`. Descobrir o limite superior máximo aceitável (ex: tentar a cada 30 minutos depois da 10ª falha) para não ser banido pela Mojang API ao buscar chaves de sessão consecutivamente.

---

## 60. Apêndice AX: Lacunas no Offline UUID (Pirata)

Se o bot entrar em modo `Offline` (Pirata), o servidor Minecraft exige que o UUID do perfil seja gerado a partir de uma Hash MD5 específica do Nick.

### 60.1 O Risco da Falsificação Incorreta
**A Lacuna:** Se a versão 3 do UUID não for gerada com o exato prefixo `OfflinePlayer:Nome`, servidores que não usam AuthMe vão negar o login (Invalid UUID Format).
**Investigação:** Testar o pacote `0x02 Login Success` em servidores piratas para garantir que o Bot consegue validar seu próprio UUID Offline sem necessitar de respostas de APIs da Mojang.

---

## 61. Apêndice AY: Lacunas de Oclusão (Entity Culling Avançado)

O cliente oficial tem um algoritmo de Culling que não renderiza e não processa Hitboxes de Entidades que estão atrás de paredes sólidas.

### 61.1 O Aimbot através das Paredes
Se um jogador estiver atrás de uma montanha, o Bot C# ainda o colocava no `TargetList` do Killaura.
**A Lacuna:** Embora o bot não batesse (por causa do Raycast), ele ficava de "AimLock" (Mirando através da parede para o jogador invisível).
**Investigação R&D:** O time Java deve implementar o Raycast *antes* do cálculo de rotação de cabeça (Yaw/Pitch). Se a linha de visão (Line of Sight) falhar, a IA de combate NÃO DEVE girar a cabeça do bot. Girar a cabeça para seguir players atrás de paredes é a razão nº1 de detecção manual (Staff Spec).

---

## 62. Conclusão Final do Projeto de Rastreabilidade

Chegamos ao fim da série de documentos de Engenharia Reversa e Rastreabilidade do projeto **AdvancedBot 2.4.5**.

A transição de **C# para Java 21** não é apenas uma mudança de sintaxe; é uma reengenharia completa da espinha dorsal do software. Nós desenterramos 5 anos de conhecimentos arcanos de física, redes e ofuscações que estavam escondidos em classes gigantes (God Objects) e métodos sem documentação.

### 57.1 O Salto de Maturidade
*   **O C# era reativo:** Ele lia pacotes e agia instintivamente.
*   **O Java será preditivo:** Ele usará Máquinas de Estado (State Machines), Filas Imutáveis e Orquestração Assíncrona para agir de forma indetectável.

O bot deixará de ser um "Hacker Client" cru para se tornar um **Ator Headless** capaz de simular a presença humana (Turing Test) nos servidores mais hostis e protegidos do mundo Minecraft.

### 57.2 Próximos Passos (Development Phase)
1. Iniciar o repositório Maven/Gradle usando os POMs do `03-Mapa-de-Modulos.md`.
2. Implementar a pipeline TCP baseada no `06-Mapa-de-Pacotes.md`.
3. Criar os Testes de Unidade usando o `08-Cobertura-Funcional.md` para garantir que o Bug Parity seja mantido.

A Engenharia de Requisitos está concluída. As fundações estão prontas. 
O desafio agora não é descobrir o que fazer, mas escrever o código limpo que a arquitetura exige.

**"O código C# está morto. Vida longa à nova JVM."**
*- Equipe de Engenharia Reversa (AdvancedBot).*
*(Documento finalizado em 2026).*
