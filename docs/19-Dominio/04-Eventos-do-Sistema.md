# Eventos do Sistema

O AdvancedBot é altamente orientado a eventos assíncronos gerados pela rede e processados de forma síncrona pelo motor principal no ciclo do Tick. Este documento lista os **Eventos de Domínio** essenciais.

## 1. Eventos de Rede e Sessão

| Evento | Quem publica | Quem consome | Quando acontece | Impacto |
|---|---|---|---|---|
| `SessionConnected` | Network Manager / Proxy | UI, Macros, Plugins | Socket TCP é aberto e logado. | Libera Início do jogo. |
| `SessionDisconnected` | Protocol / Network | Session, UI, IA | Servidor corta conexão ou proxy cai. | Interrompe imediatamente IA; dispara gatilho de auto-reconnect. |
| `PacketReceived` | Network Stream | Protocol Translator | Um pacote válido foi decodificado (antes de impactar o domínio). | Ponto de interceptação usado por Bypasses e Plugins para fraudar pacotes (Spoof). |

## 2. Eventos de Mundo

| Evento | Quem publica | Quem consome | Quando acontece | Impacto |
|---|---|---|---|---|
| `BlockChanged` | Protocol (Packet 0x34/0x35) | World, Viewer, Pathfinding | Um bloco no mundo é alterado. | Pathfinding recalcula rota se o bloco mudado estiver no caminho; IA de mineração percebe que o bloco quebrou. |
| `ChunkLoaded` | Protocol (Packet 0x33/0x21) | World | Novos dados geográficos chegam. | Atualiza malha de colisão do World. |
| `ChunkUnloaded` | Protocol | World, IA | Servidor manda limpar memória. | Se a IA estava mirando algo nesse chunk, deve abortar (Reset). |
| `TimeUpdated` | Protocol (Packet 0x03) | World | Servidor dita a hora atual do dia. | Menor impacto no bot; afeta iluminação no Viewer. |

## 3. Eventos de Entidades

| Evento | Quem publica | Quem consome | Quando acontece | Impacto |
|---|---|---|---|---|
| `EntitySpawned` | Protocol | World, Macros de Mob | Novo jogador, mob, boia ou item cai no chão. | Macro de combate mira no inimigo; Auto-coleta busca o item. |
| `EntityMoved` | Protocol | World, Pathfinding | Uma entidade no raio visual altera posição. | IA recalcula intercepção se for o alvo. |
| `EntityDespawned` | Protocol | World, Macros | Entidade morre ou sai do raio de visão. | IA limpa o "Target" se for o alvo dela. |
| `PlayerHealthChanged`| Protocol | Player, IA | Bot toma dano ou regenera. | Pode ativar Macro de comer sopa ou disparar fuga. |
| `PlayerDied` | Player (via Health=0) | Session, Plugins | A vida do bot chega a 0. | Interrompe comandos, exige envio de pacote de Respawn. |

## 4. Eventos de Inventário

| Evento | Quem publica | Quem consome | Quando acontece | Impacto |
|---|---|---|---|---|
| `InventoryWindowOpened` | Protocol (Packet 0x64/0x2D)| Inventory, Macros | Bot clica num baú ou servidor força tela. | Macro de armazém começa a descarregar itens. |
| `SlotUpdated` | Protocol | Inventory | Servidor envia novo item para um slot. | Atualiza contagem local de recursos. |
| `TransactionRejected` | Protocol (Packet 0x6A/0x32)| Inventory, IA | Bot tentou clique impossível (falha de sinc). | Reseta a janela de inventário para a visão do servidor. |

## 5. Eventos de Tempo

| Evento | Quem publica | Quem consome | Quando acontece | Impacto |
|---|---|---|---|---|
| `Tick` | Session (Scheduler local) | World, Player, IA, Plugins | A cada 50ms exatos (ideal). | Motor de todas as decisões do bot. A IA não faz nada fora deste evento. |
