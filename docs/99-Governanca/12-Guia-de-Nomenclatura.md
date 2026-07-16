# 12 - Guia de Nomenclatura

## Objetivo

Este documento define o padrão oficial de nomenclatura utilizado durante todo o processo de engenharia reversa, documentação, análise e reescrita do AdvancedBot (C#) para Java.

A padronização de nomes garante:

- Organização do projeto.
- Facilidade de localização.
- Consistência entre documentos.
- Facilidade para futuras manutenções.
- Padronização entre código, documentação e arquitetura.

Este padrão deve ser seguido durante 100% do projeto.

---

# Princípios Gerais

Toda nomenclatura deve obedecer aos seguintes princípios:

- Clareza.
- Consistência.
- Legibilidade.
- Objetividade.
- Facilidade de pesquisa.
- Baixo nível de ambiguidade.

Nunca utilizar nomes genéricos.

Exemplos proibidos:

```
Classe1
TesteNovo
Temp
Aux
NovaClasse
Objeto2
```

Sempre utilizar nomes descritivos.

Exemplos corretos:

```
InventoryManager
FishingStateMachine
PacketDecoder
WindowManager
ReconnectController
```

---

# Idioma Oficial

Toda a documentação deve ser escrita em português.

Classes, interfaces, enums e records Java devem utilizar nomes em português (DEC-11).

Pacotes, métodos, variáveis e constantes permanecem em inglês.

## Documentação

Exemplo:

```
Estado Atual

Fluxo de Login

Sistema de Inventário
```

## Código

Exemplo:

```
GerenciadorInventario

ControladorPesca

ServicoReconexao

LeitorPacote
```

Métodos e variáveis continuam em inglês:

```
processInventory()

reconnectService.execute()
```

---

# Convenções de Escrita

## PascalCase

Utilizado para:

- Classes
- Interfaces
- Enums
- Records
- Exceptions
- DTOs

Exemplo

```
InventoryManager

PacketReader

FishingMacro

WindowController

ReconnectService
```

---

## camelCase

Utilizado para:

- Variáveis
- Métodos
- Parâmetros
- Objetos

Exemplo

```
inventory

packetReader

windowManager

currentState

playerName

processInventory()

readPacket()

updateState()
```

---

## UPPER_SNAKE_CASE

Utilizado para:

Constantes.

Exemplo

```
MAX_INVENTORY_SIZE

DEFAULT_TIMEOUT

LOGIN_PACKET

WINDOW_ID

MAX_RECONNECT_ATTEMPTS
```

---

## snake_case

Não utilizar.

Somente permitido quando exigido por:

- Banco de dados
- Configurações externas
- Arquivos legados

---

# Nomeação de Pacotes

Todos os pacotes devem permanecer em letras minúsculas.

Exemplo

```
bot

bot.inventory

bot.network

bot.protocol

bot.player

bot.gui

bot.window

bot.macro

bot.command

bot.entity

bot.item

bot.utils

bot.event
```

Nunca utilizar letras maiúsculas.

Nunca utilizar espaços.

Nunca utilizar caracteres especiais.

---

# Nomeação de Classes

As classes devem representar exatamente sua responsabilidade.

Nomes em português (DEC-11).

Formato

```
Substantivo
```

Exemplo

```
GerenciadorInventario

ControladorJogador

DecodificadorPacote

ControladorPesca

RastreadorEntidade

ServicoReconexao

GerenciadorJanela
```

Evitar verbos.

Errado

```
ManageInventory

DoFishing

OpenWindow
```

---

# Nomeação de Interfaces

Prefixo obrigatório:

```
I
```

ou

Sem prefixo, dependendo do padrão definido.

Exemplo

```
PacketHandler

InventoryRepository

WindowRenderer
```

Caso seja adotado prefixo:

```
IPacketHandler

IInventoryRepository
```

O projeto deve manter apenas um padrão.

---

# Nomeação de DTOs

Sempre finalizar com:

```
DTO
```

Exemplo

```
PlayerDTO

InventoryDTO

ItemDTO

WindowDTO

EntityDTO
```

---

# Nomeação de Records

Sempre representar objetos imutáveis.

Exemplo

```
PacketHeader

PlayerPosition

InventorySlot
```

---

# Nomeação de Enums

Sempre utilizar substantivos.

Exemplo

```
ConnectionState

FishingState

WindowType

PacketType

ItemType
```

Valores internos:

```
LOGIN

PLAY

HANDSHAKE

DISCONNECTED

CONNECTED
```

---

# Nomeação de Exceptions

Sempre terminar com:

```
Exception
```

Exemplo

```
PacketException

InventoryException

AuthenticationException

ConnectionException
```

---

# Nomeação de Serviços

Sempre finalizar com:

```
Service
```

Exemplo

```
AuthenticationService

ReconnectService

InventoryService

MovementService
```

---

# Nomeação de Controllers

Sempre finalizar com:

```
Controller
```

Exemplo

```
FishingController

PlayerController

MovementController
```

---

# Nomeação de Managers

Sempre finalizar com:

```
Manager
```

Exemplo

```
InventoryManager

WindowManager

PluginManager

ThreadManager
```

---

# Nomeação de Handlers

Sempre finalizar com:

```
Handler
```

Exemplo

```
PacketHandler

ChatHandler

EventHandler

CommandHandler
```

---

# Nomeação de Factories

Sempre finalizar com:

```
Factory
```

Exemplo

```
PacketFactory

ItemFactory

CommandFactory
```

---

# Nomeação de Builders

Sempre finalizar com:

```
Builder
```

Exemplo

```
PacketBuilder

WindowBuilder

InventoryBuilder
```

---

# Nomeação de Repositories

Sempre finalizar com:

```
Repository
```

Exemplo

```
PlayerRepository

ConfigurationRepository

InventoryRepository
```

---

# Nomeação de Utilitários

Sempre finalizar com:

```
Utils
```

ou

```
Util
```

Exemplo

```
FileUtils

MathUtils

PacketUtils

ReflectionUtils
```

---

# Nomeação de Eventos

Sempre finalizar com:

```
Event
```

Exemplo

```
PlayerJoinEvent

PlayerMoveEvent

PacketReceivedEvent

InventoryOpenEvent
```

---

# Nomeação de Listeners

Sempre finalizar com:

```
Listener
```

Exemplo

```
PacketListener

PlayerListener

InventoryListener
```

---

# Nomeação de Threads

Sempre finalizar com:

```
Thread
```

Exemplo

```
FishingThread

ReconnectThread

PacketReaderThread
```

---

# Nomeação de Estados

Sempre utilizar:

```
State
```

Exemplo

```
LoginState

FishingState

ReconnectState

CombatState
```

---

# Nomeação de Comandos

Sempre finalizar com:

```
Command
```

Exemplo

```
FishingCommand

SellCommand

RepairCommand

ReconnectCommand
```

---

# Nomeação de Macros

Sempre finalizar com:

```
Macro
```

Exemplo

```
FishingMacro

MiningMacro

RepairMacro
```

---

# Nomeação de Métodos

Métodos devem iniciar com verbo.

Exemplo

```
connect()

disconnect()

sendPacket()

receivePacket()

openInventory()

closeWindow()

calculatePath()

updateInventory()

readBuffer()

writePacket()
```

Nunca utilizar nomes vagos.

Errado

```
execute()

doStuff()

run()

teste()

aux()
```

---

# Nomeação de Variáveis

Utilizar nomes descritivos.

Correto

```
inventory

currentSlot

playerHealth

packetBuffer

windowId

selectedItem

lastReconnectTime
```

Evitar

```
obj

aux

tmp

teste

x

y

z
```

---

# Nomeação de Booleanos

Sempre iniciar com:

```
is

has

can

should
```

Exemplo

```
isConnected

isLogged

hasPermission

hasInventory

canMove

shouldReconnect
```

---

# Nomeação de Coleções

Utilizar plural.

Exemplo

```
players

entities

items

windows

commands

listeners
```

---

# Nomeação de Arquivos Markdown

Formato:

```
NN-Nome-do-Arquivo.md
```

Exemplo

```
01-Visao-Geral.md

02-Estado-Atual.md

03-Roadmap.md

04-Checklist-Mestre.md

05-Procedimento-Operacional.md

06-Definition-of-Done.md

07-Controle-de-Decisoes.md

08-Registro-de-Sessoes.md

09-Metricas-do-Projeto.md

10-Protocolo-de-Uso-com-IA.md

11-Guia-de-Documentacao.md

12-Guia-de-Nomenclatura.md
```

---

# Nomeação de Diretórios

Utilizar:

- PascalCase para módulos lógicos quando fizer parte da documentação.
- lowercase para diretórios de código.

Exemplo

Documentação

```
Arquitetura

Protocolos

Inventario

Macros

Fluxos
```

Código

```
inventory

network

protocol

entity

macro

command

window
```

---

# Nomeação de Branches

Formato

```
tipo/descricao
```

Tipos

```
feature/

bugfix/

hotfix/

refactor/

docs/

analysis/
```

Exemplos

```
feature/inventory-system

feature/fishing-macro

refactor/network-layer

docs/documentation

analysis/reverse-engineering
```

---

# Nomeação de Commits

Formato recomendado

```
tipo: descrição
```

Tipos

```
feat

fix

docs

refactor

test

style

perf

build

ci

chore
```

Exemplos

```
feat: implement inventory parser

fix: correct packet decoder

docs: update reverse engineering guide

refactor: simplify fishing state machine

test: add packet decoder tests
```

---

# Nomeação de Diagramas

Sempre utilizar nomes objetivos.

Exemplo

```
Arquitetura Geral

Fluxo de Login

Fluxo de Pesca

Fluxo de Inventário

Máquina de Estados

Arquitetura de Rede

Comunicação Cliente-Servidor
```

---

# Nomeação de Imagens

Formato

```
modulo_descricao.png
```

Exemplo

```
inventory_slots.png

login_flow.png

packet_structure.png

state_machine.png

window_layout.png
```

---

# Nomeação de Logs

Formato

```
AAAAMMDD_HHMMSS_modulo.log
```

Exemplo

```
20260714_183000_inventory.log

20260714_184500_network.log
```

---

# Glossário Padronizado

Os seguintes termos devem ser utilizados de forma consistente em toda a documentação e código:

| Português | Inglês |
|------------|---------|
| Inventário | Inventory |
| Janela | Window |
| Jogador | Player |
| Entidade | Entity |
| Mundo | World |
| Bloco | Block |
| Item | Item |
| Slot | Slot |
| Rede | Network |
| Pacote | Packet |
| Protocolo | Protocol |
| Estado | State |
| Máquina de Estados | State Machine |
| Controlador | Controller |
| Gerenciador | Manager |
| Serviço | Service |
| Manipulador | Handler |
| Repositório | Repository |
| Evento | Event |
| Ouvinte | Listener |
| Macro | Macro |
| Comando | Command |
| Reconexão | Reconnect |
| Autenticação | Authentication |
| Configuração | Configuration |
| Sessão | Session |
| Conexão | Connection |

---

# Conformidade

Toda contribuição ao projeto deverá seguir integralmente este guia de nomenclatura.

Qualquer exceção deverá ser:

- Justificada tecnicamente.
- Registrada no Controle de Decisões.
- Documentada na sessão correspondente.
- Aprovada antes da implementação.

A adoção consistente deste padrão é obrigatória para garantir uniformidade, rastreabilidade e qualidade durante todo o processo de engenharia reversa, documentação e reescrita do AdvancedBot de C# para Java.