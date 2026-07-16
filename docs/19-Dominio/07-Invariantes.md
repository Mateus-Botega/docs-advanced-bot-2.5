# Invariantes do Sistema

As Invariantes são regras matemáticas ou lógicas absolutas dentro do domínio. Se alguma dessas regras for violada em tempo de execução, o sistema está em um estado corrompido, e a Sessão deve ser idealmente abortada, pois a continuação resultaria em banimento do bot ou *crashes* silenciosos.

## 1. Sessão e Rede

1. **Estado Exclusivo de IO**: Nunca é permitido enviar pacotes lógicos de jogo (`SendPlayerMove`, `SendChatMessage`) se o estado da Sessão não for estritamente `Playing`.
2. **Ordem de Criptografia**: Se a Sessão ativar Criptografia, 100% dos bytes enviados/recebidos a partir daquele momento e até a desconexão devem passar pelo decodificador AES. Não há fluxo híbrido.

## 2. Inventário

1. **Limites de Hotbar**: O `SelectedSlot` (Hotbar) ativo do jogador deve ser sempre um inteiro no intervalo fechado `[0, 8]`.
2. **Consistência de Ação**: O `ActionNumber` de cliques no inventário (`WindowClick`) deve sempre ser incrementado (ou cíclico dependendo do tipo) para cada transação e deve bater com as rejeições/confirmações do servidor.
3. **Pilha Válida (Stack Size)**: A quantidade de itens (Count) em um único `ItemStack` num Slot não pode exceder o limite máximo daquele tipo de item (ex: 64 para blocos, 1 para espadas, 16 para ender pearls).

## 3. Física e Geometria do Mundo

1. **Coordenadas de Chunk**: A relação entre `BlockPosition(x, y, z)` e `ChunkPosition(cx, cz)` deve sempre satisfazer matematicamente `cx = floor(x / 16)` e `cz = floor(z / 16)`.
2. **Altura Y**: As coordenadas Y (`Vector3i.Y`) no Mundo Normal e Nether têm limite absoluto `[0, 255]` no formato original do legado 1.8 (antes da 1.18).
3. **Nulidade do Vazio (Void)**: Tentar extrair propriedades de blocos (`GetBlock`) fora da fronteira geométrica ou além da `RenderDistance` atual recebida do servidor deve resultar incondicionalmente no tipo neutro `Air`.

## 4. Pathfinding e IA

1. **Intercepção Proibida**: Um nó (`Node`) de caminho calculado pelo A* nunca pode coincidir espacialmente com a `AABB` (Axis-Aligned Bounding Box) de um bloco sólido (`IsSolid == true`).
2. **Tempo Mínimo de Tick**: Um Agente não pode enviar múltiplos pacotes sequenciais da mesma ação (ex: bater, quebrar) no intervalo de um único Tick (50ms). Se fizer isso, será classificado como ataque DDoS/Spam pelo servidor. Ação real consome tempo real.

## 5. Eventos de Domínio

1. **Sequenciamento de Entidade**: O evento `EntityDespawned(ID)` ou `EntityMoved(ID)` não pode ser invocado se o respectivo `EntitySpawned(ID)` não tiver sido recebido (exceto se a entidade de origem for o próprio jogador, cujo ID vem no login).
2. **Bloqueio Cíclico**: Um callback disparado por um Evento nunca deve aguardar de forma síncrona/bloqueante um segundo evento cuja conclusão dependa da liberação da thread do primeiro (Deadlock arquitetural). O fluxo de eventos é unidirecional.

---

## 6. Conexão Assíncrona e Limites de Recursão

1. **Profundidade de Recursão**: O método `ConnectAndHandshakeAsync(callDepth, flags)` em `MinecraftClient` tem um limite estrito de **`callDepth <= 3`**. Se excedido, lança `Exception("Recursion exception: callDepth > 3")` e aborta a conexão. Esse limite existe porque o fluxo de conexão pode chamar a si mesmo recursivamente para Autenticar, Pingar e Conectar.
2. **Flags de Fase de Conexão**: O parâmetro `flags` usa bitmask para registrar quais fases já foram executadas nessa chamada recursiva: `flag 0x01` = mundo limpo, `flag 0x02` = autenticado Mojang, `flag 0x04` = ping enviado. Isso evita que as fases sejam executadas mais de uma vez por conexão.
3. **Timeout de Handshake**: Se o estado de conexão ficar em `connStatus == 0` (aguardando JoinGame) por mais de **20 segundos** após o envio do handshake, a sessão é desconectada por `Disconnect("Por algum motivo, a conexão não foi completada.")`. O tick é quem verifica esse timeout (`Utils.GetTickCount64() - handshakeStart > 20000`).
4. **Timeout de Keep Alive**: Se `keepAliveTicks` exceder **750 ticks** (≈ 37,5 segundos a 20 TPS) sem ser resetado por um pacote KeepAlive do servidor, a sessão é desconectada. Na versão legada (v1.5.2), esse mesmo timeout é **200 ciclos de poll** com `Thread.Sleep(10ms)` = ≈ 2 segundos.

---

## 7. Física do Jogador — Constantes e Regras Imutáveis

1. **Hitbox do Jogador**: Largura = `0.6m` total (raio `0.3` em X e Z), Altura = `1.8m`. O ponto de origem é `PosY - 1.62` (olho = 1.62m acima da base da AABB). Esses valores são hard-coded em `Entity.SetPosition`.
2. **Velocidade de pulo**: `MotionY = 0.42` ao saltar (exato, invariante de protocolo).
3. **Gravidade**: `MotionY -= 0.08` por tick (aceleração terminal).
4. **Resistência vertical do ar**: `MotionY *= 0.98` por tick.
5. **Resistência horizontal do ar**: `MotionX *= num7; MotionZ *= num7` onde `num7` depende do bloco sob os pés.
6. **Atrito do bloco**: Blocos normais usam `0.6 * 0.91 = 0.546`; gelo e gelo embalado (`blockId 79` e `174`) usam `0.98 * 0.91 = 0.8918`.
7. **Velocidade de escada**: `MotionX` e `MotionZ` são clampeados para `[-0.14, 0.14]`; `MotionY` é clampeado para `>= -0.14` (ascenção forçada ao colidir horizontalmente na escada = `MotionY = 0.2`).
8. **Teia de Aranha**: Multiplica o movimento por `(0.25, 0.05, 0.25)` e zera as velocidades (`MotionX/Y/Z = 0`).
9. **Velocidade base no chão**: `GetMoveSpeed()` retorna base `0.1`. Sprint multiplica por `1.3`. Potão de velocidade (`potion ID 1`) multiplica por `1.0 + 0.2 * (nível + 1)`. Potão de lentidão (`potion ID 2`) multiplica por `1.0 - 0.15 * (nível + 1)`. Velocidade negativa é clampeada para `0.0`.
10. **Zona de Portal**: Se o bot detectar que está numa zona de portal (`BlockID 90`) enquanto está no chão, a `MoveQueue` é limpa imediatamente (previne que o pathfinder continue enviando movimentos durante a teleportação de dimensão).
11. **Física fora de chunk**: Se o chunk onde o jogador está (`floor(PosX) >> 4`, `floor(PosZ) >> 4`) não existir no `World` local e `AABB.MinY > 0.0`, o sistema força `MotionY = -0.1` para simular gravidade mínima no vazio (previne detecção de fly).
