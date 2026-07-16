# Macros Solk — Documentação Técnica Completa

Fontes: `CommandMob.cs`, `CommandPesca.cs`, `CommandPescaV2.cs`, `CommandMobTeleport.cs`, `MacroUtils.cs`, `ConfigMob.cs`, `FileLock.cs`.

Namespace: `AdvancedBot.Client.Commands.Solk`.

---

## Visão Geral

O pacote Solk contém macros **server-specific** (CloudCraft/Craftlandia) de alto nível, que automatizam tarefas complexas de farm utilizando máquinas de estado assíncronas (`async/await`). Diferentemente dos comandos genéricos do pacote principal, estas macros encapsulam lógica de negócio de servidor: teleporte via `/home`, interação com NPCs de venda, troca de ferramentas por baú compartilhado e gerenciamento autônomo de vida da sessão (auto-reconnect).

---

## 1. `CommandMob` — Macro de Farm de Mobs

**Alias**: `solkmob`  
**Arquivo**: `CommandMob.cs`

### 1.1 Máquina de Estado

```
EstadoMacroMob:
  RECEM_LOGOU → VERIFICAR_INVENTARIO → BUSCAR_MOB → VERIFICAR_POCAO → MATAR_MOB
                                      → TROCAR_FERRAMENTA
                                      → VENDER
                                      → MOB_NAO_ENCONTRADO
  SERVIDOR_REINICIANDO  (estado de bloqueio — só sai via RECEM_LOGOU)
  OUTRA_CONTA_IDENTIFICOU_MOB_BUGOU (estado de bloqueio — só sai via RECEM_LOGOU)
  FINALIZANDO
```

**Invariante crítica**: os estados `SERVIDOR_REINICIANDO` e `OUTRA_CONTA_IDENTIFICOU_MOB_BUGOU` só permitem transição para `RECEM_LOGOU`. Todas as chamadas a `alterarEstado()` para outros destinos são silenciosamente ignoradas quando nesses estados (bloqueio de emergência).

### 1.2 Regras de Negócio

| Regra | Detalhe |
|-------|---------|
| **Espada com Durabilidade** | `Metadata >= 1300` dispara `TROCAR_FERRAMENTA` |
| **Inventário cheio** | `< 5 slots vagos` dispara `VENDER` imediatamente |
| **Timeout de venda** | Se passarem `>= 20 minutos` sem vender (independente de slots), dispara `MOB_NAO_ENCONTRADO` para tentar desbuguar o mob |
| **Busca de mob** | Distância máxima de `3.5 blocos` (Euclidiana 3D) |
| **Tentativas** | Até 15 tentativas com `/home mob` + `15s de espera` entre cada uma antes de ir a `MOB_NAO_ENCONTRADO` |

### 1.3 Algoritmo de Hit em Mob (`hitarMob`)

O bot **não usa pathfinding** para lutar. O algoritmo é:
1. `HotbarSlot = 0` (espada)
2. `LookToBlock` na posição do mob
3. Simula um **salto falso** enviando `PacketPlayerPos` com `y + 0.42` sem alterar física local (`Player.MotionY = 0.42`, `OnGround = false`)
4. Envia `PacketUseEntity(entityID, attack=true)` em loop **850 vezes** com delay de `100ms` por hit (total ≈ 85 segundos)
5. Envia `PacketPlayerPos` com `OnGround = true` para aterrissar

**Risco anti-cheat**: 850 hits com 100ms de delay resultam em exatamente 10 hits/segundo, padrão mecanicamente perfeito e potencialmente detectável pelo GrimAC.

### 1.4 Protocolo de Troca de Ferramenta (`trocarFerramenta`)

O processo usa `FileLock` para coordenação entre múltiplas instâncias do bot que compartilham um único baú de espadas:
1. Teleporta para `/home ferramentas`
2. Detecta baú próximo (raio de 2 blocos)
3. Adquire lock de arquivo via `FileLock("trava_bau")` — timeout de 10 segundos
4. Limpa cursor (item fantasma no slot de cursor)
5. Verifica espadas extras no inventário e as move para o baú
6. Move espada quebrada (`Metadata >= 1300`) para o baú
7. Procura nova espada no baú com `Metadata <= 1000` e opcionalmente com encantamento **Flame** (`ID 20`)
8. Retenta até 3 vezes a troca com aguarda de confirmação de 200ms por check
9. Libera lock (`FileLock.Dispose`)

**Algoritmo de busca de espada**: busca reversa do último slot para o slot 0. Prioriza espadas com menor uso (menor Metadata). Rejeita espadas sem Flame se `config.isEspadaComFlame == true`.

### 1.5 Poção Especial (Blaze)

Condicionada à detecção de `"blaze"` no `config.homesVenda`:
- Verifica se passaram **9 minutos** desde a última poção (`ultimaPocaoBebida`)
- Procura poção `ID 373 (Items.potion), Metadata 8265` (Força I) no slot 2 (index 37)
- Tenta beber `2 vezes` com verificação por diminuição de `Count` do item
- Se não encontrar, vai a `/home Tiozaopatinhas pot` para buscar a poção no baú

### 1.6 Reações a Chat (`onReceiveChat`)

| Mensagem (case-insensitive) | Ação |
|-----------------------------|------|
| `"agora você está logado..."` (login do servidor) | Reinicia a macro (`Start()`) |
| `"identifiquei que o mob bugou"` | Transition para `OUTRA_CONTA_IDENTIFICOU_MOB_BUGOU` |
| `"o servidor ira reiniciar em breve"` | Transição para `SERVIDOR_REINICIANDO` (pausa 5 minutos via `aguardarReinicio`) |

### 1.7 Configuração (`ConfigMob`)

Arquivo JSON em `Plugins\Configs\Mob\{nick}.json`. Parâmetros:

| Campo | Padrão | Descrição |
|-------|--------|-----------|
| `homeMob` | `/home mob` | Comando de teleporte ao mob |
| `homesVenda` | `vmob` | Locais de venda separados por vírgula |
| `homeFerramentas` | `/home ferramentas` | Baú de espadas |
| `desativarMenuLoja` | `/menuloja off` | Desativa menu antes de vender |
| `cmdSpawn` | `/spawn` | Teleporte ao spawn para desbug |
| `cmdEsconder` | `/esconder` | Invisibilidade |
| `delayTeleporte` | `6000` ms | Espera pós-teleporte |
| `isEspadaComFlame` | `false` | Filtra espadas pelo encantamento Flame |

---

## 2. `CommandPesca` — Macro de Pesca V1

**Alias**: `solkpesca`  
**Arquivo**: `CommandPesca.cs`

### 2.1 Máquina de Estado

```
EstadoMacroPesca:
  RECEM_LOGOU → INICIANDO → LIMPAR_INVENTARIO → VERIFICAR_INVENTARIO
  VERIFICAR_INVENTARIO → PESCAR
                       → REPARAR
                       → BUSCAR_VARA
                       → LIMPAR_INVENTARIO
  LIMPAR_INVENTARIO → GUARDAR_ITENS | VERIFICAR_INVENTARIO
  COMPRAR_LINHA → RECEM_LOGOU
  FINALIZANDO
```

### 2.2 Regras de Negócio

| Regra | Detalhe |
|-------|---------|
| **Durabilidade da vara** | `Metadata >= 45` → estado `REPARAR` |
| **Slots mínimos** | `< 5 slots vagos` → `LIMPAR_INVENTARIO` |
| **Slots pós-limpeza** | `< 20 slots vagos` após limpeza → `GUARDAR_ITENS` |
| **Reparo** | Localiza bloco de ferro (`Blocks.iron_block`) no raio 2x4x2 e executa 2 cliques direitos |
| **Auto-reconnect** | Se `!IsBeingTicked()`, aguarda 10 segundos e chama `StartClient()` |

### 2.3 Algoritmo de Pesca

1. `HotbarSlot = 0`, `Pitch = -90` (mira para cima)
2. `LookToBlock` na posição `(x, y+5, z)` do jogador (mira para o céu)
3. Loop de **1000 iterações** com `Task.Delay(180ms)` → envia `PacketBlockPlace(ItemInHand)` (jogada da vara)
4. Verifica durabilidade a cada iteração

**Nota**: O servidor Craftlandia detecta o impacto do anzol e devolve o item de pesca automaticamente. A lógica de puxar a vara **não está implementada** nesta versão — o servidor gerencia isso.

### 2.4 Filtro de Itens (`deveGuardarItem`)

Usa um dicionário `requisitosEncantamentos` que mapeia `ItemID` para uma lista de **alternativas** de requisitos. Um item é guardado se atender **todos** os requisitos de qualquer uma das alternativas:

| Item | Alternativa 1 | Alternativa 2 |
|------|--------------|--------------|
| Espada (276) | Afiada V + Inquebrável III + Pilhagem III | Afiada V + Inquebrável III + Pilhagem II |
| Picareta (278) | Eficiência V + Inquebrável III + Fortuna III | — |
| Capacete (310) | Proteção IV + Inquebrável III | — |
| Peito (311) | Proteção IV + Inquebrável III | — |
| Calça (312) | Proteção IV + Inquebrável III | — |
| Bota (313) | Proteção IV + Inquebrável III | — |

### 2.5 Detecção de Baús (`detectarBaus`)

- Varre de `posY-4` a `posY+4` e de `posZ-3` a `posZ` na posição X atual
- Detecta `Blocks.chest` (ID 54) e `Blocks.trapped_chest` (ID 146)
- Retorna baús **ordenados do mais alto para o mais baixo** (prioriza baús superiores)

### 2.6 Gerenciamento de Home Temporária (`isHomeGuardarTempSetada`)

Quando um baú com slots vagos é encontrado, o bot executa `/sethome guardartemp` **3 vezes consecutivas** (para garantir o registro) e ativa a flag. Nas próximas visitas, usa `/home guardartemp` em vez de `/home guardar`, evitando varredura completa de baús.

---

## 3. `CommandPescaV2` — Macro de Pesca V2

**Alias**: `solkpesca2`  
**Arquivo**: `CommandPescaV2.cs`

### 3.1 Diferenças em relação à V1

| Aspecto | V1 | V2 |
|---------|----|----|
| **Mapeamento** | Hardcoded / detecção dinâmica limitada | Fase `INICIANDO` mapeia todos os blocos e baús |
| **Baús separados** | 1 baú genérico de guardar | `bausDeVara`, `mkb`, `mkbDisco`, `mkbPeixe` — mapeados por área relativa |
| **Compra de linha** | Via `/home` fixo | Tenta primeiro placa local mapeada; fallback para warp loja |
| **Armazenamento** | Baú único detectado dinamicamente | 3 destinos distintos: peixe, disco, ferramentas/armadura |

### 3.2 Mapeamento de Baús (`mapearBausProximos`)

Após teleporte ao pesqueiro, busca baús em áreas relativas ao jogador:

| Destino | Área (relativa ao jogador) |
|---------|--------------------------|
| `bausDeVara` | `-1,-1,-1` a `+1,+1,+1` |
| `mkb` (ferramentas) | `-3,-3,-5` a `+6,+2,+3` |
| `mkbDisco` (discos) | `-3,-3,-4` a `-1,+2,-3` |
| `mkbPeixe` (peixes) | `+1,-3,-4` a `+2,+2,-3` |

**Bug documentado**: O loop em `buscarBausPorArea` usa condições `x >= fimX` com incremento `x++` — quando `inicioX < fimX`, o loop **nunca executa** (condição inicial falsa). As áreas de mapeamento estão **invertidas** no código.

### 3.3 Itens Armazenáveis e Encantamentos

Itens essenciais (nunca descartados): `fishing_rod (346)`, `string (287)`.

Encantamentos para armazenar ferramentas:
- **Pá (277)**: Eficiência V + Inquebrável III  
- **Picareta (278)**: Eficiência V + Inquebrável III + Fortuna III
- **Machado (279)**: Eficiência V + Inquebrável III + Fortuna III
- **Espada (276)**: Afiada V + Inquebrável III
- **Armaduras (310-313)**: Proteção IV + Inquebrável III

---

## 4. `CommandMobTeleport` — Macro de Mob via Corner Glitch

**Alias**: `solk`  
**Arquivo**: `CommandMobTeleport.cs`

### 4.1 Propósito

Macro experimental que implementa o algoritmo **Corner Glitch** para atravessar paredes e atacar mobs através de blocos sólidos (para farms onde o mob está atrás de uma parede).

### 4.2 Algoritmo `TentarAtravessarComCornerGlitch`

1. Calcula o canto mais próximo do mob (`Math.Floor/Ceiling` da posição X e Z do mob)
2. Move-se até o ponto de aproximação: `canto - (sinal * 0.31)` (encostado na borda da parede)
3. Tenta 150 vezes empurrar a posição `0.25 blocos` na direção da parede com delay de `125ms`
4. Para cada tentativa: chama `Player.SetPosition(x + push, y, z + push)` diretamente (sem física)
5. Considera sucesso se a distância ao mob for `< 2.0 blocos` após o push

**Limitação**: O estado `FINALIZANDO` não implementa nenhuma ação. O ataque em si (`PacketUseEntity`) está comentado no código (`//int qntDeHits = 850; //for...`).

**Movimento gradual** (`MoveToLocationInSteps`): move em passos de `0.3 blocos` com `250ms` entre cada passo, enviando `SetPosition` diretamente sem emitir pacotes de rede.

---

## 5. `MacroUtils` — Utilitários Compartilhados

**Arquivo**: `MacroUtils.cs`

### 5.1 Abertura de Baús (`abrirBau`)

Tenta abrir um baú em até **10 iterações**:
1. `Player.LookToBlock(x, y, z)` + `Task.Delay(1000ms)`
2. `RightclickBlock(coordBau)` + `Task.Delay(1000ms)`
3. Verifica `Client.OpenWindow != null`; se aberto, retorna `true`

**Custo**: até 20 segundos por tentativa de abertura de baú.

### 5.2 Interação de Compra/Venda em Placa (`vender`, `comprar`)

```
vender(client, qntDeHitNaPlaca, homeParaVoltar):
  1. RayCastBlocks(6.0) — obtém HitResult da placa sob mira
  2. Para cada hit (1..qntDeHitNaPlaca):
     a. Envia PacketEntityAction(playerID, 0) — sneak on
     b. Envia "$clickblock x y z 0" (clique esquerdo)
     c. Task.Delay(500ms)
     d. Envia PacketEntityAction(playerID, 1) — sneak off
     e. Task.Delay(500ms)
  3. Teleporta de volta se homeParaVoltar != null
```

**Regra implícita**: A venda é feita via clique de placa de loja (ShopSign). O bot precisa estar mirando a placa ao invocar `vender()`.

### 5.3 `FileLock` — Mutex Inter-Processo

Implementa trava baseada em arquivo (`%TEMP%\{lockName}.lock`) para coordenar múltiplas instâncias de bot que compartilham recursos (baú de espadas):

- Tentativa a cada `200ms` por até `timeoutMs` (padrão 10 segundos)
- Usa `FileShare.None` (exclusão completa)
- `IDisposable`: libera o stream e deleta o arquivo ao sair do `using`

**Limitação**: Funciona apenas em instâncias na mesma máquina (compartilham %TEMP%). Não funciona em multi-máquina.

### 5.4 `selecionarMelhorItem`

Percorre a hotbar (slots 36–44) e seleciona a ferramenta com maior `DiggingHelper.ToolStrengthVsBlock()` para o bloco alvo. Se a ferramenta não pode coletar o bloco (`CanHarvestBlock == false`), multiplica a velocidade por `0.2` para penalização.

### 5.5 `WaitForTeleport`

Retardo adaptativo baseado na flag `PlayerVIP`:
- `PlayerVIP == true`: aguarda **3000ms**
- `PlayerVIP == false`: aguarda **5000ms**

---

## 6. `SkySurvival` — Bypass de Captcha de Inventário

**Arquivo**: `AdvancedBot.Client.Bypassing/SkySurvival.cs`  
**Namespace**: `AdvancedBot.Client.Bypassing`

### 6.1 Propósito

Resolve captchas do servidor **SkySurvival** que consistem em uma janela de inventário com título no formato `"Clique n<item>"` e o bot deve clicar nos slots que contêm o item especificado.

### 6.2 Algoritmo

1. Verifica se `OpenWindow.Title` começa com `"Clique n"`
2. Extrai a última palavra do título como chave (`key`)
3. Busca `key` no dicionário `ItemNames`
4. Itera os slots do inventário, aplicando `Match(flags, item, items)`

**Formato de item**: `items[0]` = flags de modo, `items[i]` para `i >= 1` = `(itemID << 4) | metadata`

**Modos de correspondência**:
- `MATCH_SINGLE (1)`: para no primeiro match
- `MATCH_ANY (2)`: retorna todos os slots
- `MATCH_ALL (4)`: (definido mas não usado)
- `IGNORE_METADATA (256)`: ignora metadata na comparação

### 6.3 Itens Reconhecidos

| Palavra-chave | Tipo de Match | Itens |
|--------------|--------------|-------|
| `livro` | SINGLE | ID 257, meta 5440 |
| `batata` | SINGLE | ID 257, meta 6272 |
| `biscoito` | SINGLE | ID 257, meta 5712 |
| `cenoura` | SINGLE | ID 257, meta 6256 |
| `maçã` | SINGLE | ID 257, meta 4160 |
| `cabeça` | SINGLE | ID 257, meta 6352 |
| `cabeças` | ANY | ID 260, meta 6352 |
| `peixe` | SINGLE | ID 257, meta 5584 |
| `picareta` | ANY + IGNORE_META | todos os `*_pickaxe` |
| `machado` | ANY + IGNORE_META | todos os `*_axe` |
| `pá` | ANY + IGNORE_META | todos os `*_shovel` |
| `enchada` | ANY + IGNORE_META | todos os `*_hoe` |
| `comidas` | ANY + IGNORE_META | todos os `cooked*` + 4160, 5712 |
| `ferramentas` | ANY + IGNORE_META | todos os `*_axe, *_shovel, *_hoe, *_pickaxe` |
| `peixes` | ANY | ID 260, meta 5584 |

**Geração dinâmica via Reflection**: O método `AnyItem()` usa `typeof(Items).GetFields()` para varrer todos os campos da classe `Items` e filtrar por sufixo/prefixo de nome, gerando a lista de IDs dinamicamente em tempo de inicialização.

---

## 7. `WorldCraftBP` — Bypass SkySurvival

**Arquivo**: `AdvancedBot.Client.Bypassing/WorldCraftBP.cs`

*Este arquivo é pequeno e faz parte do subsistema de bypass. Documentação pendente de leitura completa.*

---

## 8. Dependências e Limitações

### Dependências do Módulo Solk

```
CommandMob / CommandPesca / CommandPescaV2
    → MinecraftClient (estado, inventário, rede)
    → MacroUtils (utilitários compartilhados)
    → ConfigMob (configuração por nick, JSON)
    → FileLock (coordenação inter-processo — apenas local)
    → World (detecção de baús e blocos)
    → Entity (posição, movimento)
    → Items / Blocks (constantes)
    → Viewer.ViewForm (highlight de blocos — acoplamento com UI)
```

### Limitações Conhecidas

1. **Acoplamento com UI**: `CommandPescaV2` usa `AdvancedBot.Viewer.ViewForm.BlocksToHighlight` diretamente — impede uso headless.
2. **FileLock não distribuído**: Coordenação de baú compartilhado só funciona na mesma máquina.
3. **async void Tick()**: Fire-and-forget — exceções são silenciadas, e `isProcessing` pode travar em `true` se o Tick lançar exceção no `try` antes do `finally`.
4. **Loop de pesca hardcoded**: 1000 iterações × 180ms = 180 segundos de pesca por ciclo (independente de inventário cheio).
5. **Bug em `buscarBausPorArea` (V2)**: Loop com condição invertida nunca executa.
6. **Sem cancelamento de Task**: Tasks iniciadas por `async void Tick()` continuam em background após desconexão.
