# 16 - Padrões de Diagramas

## Objetivo

Definir o padrão oficial para criação, manutenção e atualização de diagramas durante todo o processo de migração do projeto **AdvancedBot C# → Java**.

Todos os diagramas devem representar fielmente a arquitetura existente, facilitar o entendimento do sistema e servir como documentação técnica permanente.

Os diagramas nunca substituem a documentação textual.

Eles complementam a documentação.

---

# Objetivos dos Diagramas

Cada diagrama deve possuir pelo menos um dos objetivos abaixo:

- facilitar entendimento do código
- mostrar dependências
- explicar fluxo
- representar arquitetura
- representar estrutura
- representar comunicação
- representar responsabilidades
- auxiliar revisões
- auxiliar onboarding
- facilitar manutenção

---

# Princípios

Todos os diagramas devem seguir:

- simplicidade
- clareza
- padronização
- legibilidade
- consistência
- atualização constante
- nomenclatura oficial do projeto
- granularidade adequada

Nunca criar diagramas extremamente complexos.

Sempre dividir quando necessário.

---

# Ferramentas Permitidas

Preferencialmente utilizar:

- Mermaid
- PlantUML
- Draw.io
- Excalidraw
- LucidChart
- Visio

Para documentação Markdown:

Prioridade:

1. Mermaid
2. PlantUML

---

# Organização dos Diagramas

Todos devem ficar organizados por assunto.

Exemplo:

```
docs/

    diagramas/

        arquitetura/

        fluxo/

        classes/

        banco/

        integração/

        sequência/

        estados/

        componentes/
```

---

# Convenções Gerais

Todos os diagramas devem possuir:

Título

Objetivo

Escopo

Versão

Data

Autor

Relacionamentos

Referências

---

Exemplo

```text
Título:
Fluxo da Macro de Mineração

Objetivo:
Representar o fluxo completo da execução.

Versão:
1.2

Atualizado:
2026-07-14
```

---

# Nomenclatura

Sempre utilizar nomes reais.

Exemplo correto

```
MacroMineracao

InventoryManager

PacketHandler

ConnectionManager
```

Nunca

```
Classe A

Objeto B

Processo X
```

---

# Idioma

Diagramas técnicos utilizam os nomes reais do código.

Textos auxiliares podem permanecer em português.

Exemplo

```
InventoryManager

↓

Atualiza inventário

↓

Seleciona ferramenta
```

---

# Cores

Utilizar cores apenas para diferenciação.

Exemplo

Verde

- concluído

Azul

- processamento

Amarelo

- atenção

Vermelho

- erro

Cinza

- componente externo

Jamais utilizar cores aleatórias.

---

# Tipos Oficiais de Diagramas

O projeto utiliza os seguintes diagramas.

---

## Arquitetura

Representa:

- módulos
- pacotes
- dependências
- componentes

Exemplo

```
Minecraft

↓

Network

↓

Bot

↓

Commands

↓

Macros
```

---

## Componentes

Mostra comunicação entre módulos.

Exemplo

```
MacroMineracao

↓

InventoryService

↓

PacketService

↓

MinecraftClient
```

---

## Classes

Representa:

- herança
- composição
- agregação
- interfaces

Exemplo

```
Command

↑

CommandFish

CommandMob

CommandRepair
```

---

## Sequência

Mostra ordem temporal das chamadas.

Exemplo

```
Usuário

↓

Macro

↓

Bot

↓

PacketHandler

↓

Servidor
```

---

## Fluxograma

Representa decisões.

Exemplo

```
Entrou Tick

↓

Inventário cheio?

↓

Sim

↓

Voltar Baú

↓

Não

↓

Continuar mineração
```

---

## Máquina de Estados

Muito importante para macros.

Exemplo

```
Idle

↓

Mining

↓

InventoryFull

↓

ReturningChest

↓

DepositItems

↓

Mining
```

---

## Atividades

Representa processos.

Exemplo

```
Login

↓

Selecionar servidor

↓

Entrar

↓

Inicializar bot

↓

Executar macro
```

---

## Casos de Uso

Representa interação externa.

Exemplo

```
Jogador

↓

Executar Macro

↓

Sistema

↓

Mineração Automática
```

---

## Banco de Dados

Representa:

- tabelas
- relacionamentos
- chaves

Exemplo

```
Usuario

↓

Sessao

↓

Log
```

---

## Dependências

Representa bibliotecas.

Exemplo

```
AdvancedBot

↓

Newtonsoft

↓

ProtocolLib
```

---

## Pacotes

Representa organização.

Exemplo

```
core

network

commands

macros

utils

packets
```

---

# Diagramas Mermaid

Sempre utilizar sintaxe limpa.

Exemplo

```
graph TD

A[Bot]

B[Macro]

C[Packet]

A --> B

B --> C
```

---

# Diagramas PlantUML

Seguir padrão oficial.

Exemplo

```
@startuml

Bot --> Macro

Macro --> Packet

@enduml
```

---

# Fluxos

Todo fluxo deve conter:

Início

Processamento

Decisão

Exceções

Finalização

---

# Decisões

Sempre utilizar losangos.

Exemplo

```
Inventário cheio?

↓

Sim

↓

Baú

↓

Não

↓

Continuar
```

---

# Loops

Representar explicitamente.

Nunca ocultar repetições.

Exemplo

```
Tick

↓

Verificar estado

↓

Executar

↓

Tick
```

---

# Tratamento de Erros

Sempre representar.

Exemplo

```
Enviar pacote

↓

Erro?

↓

Sim

↓

Reconnect

↓

Não

↓

Continuar
```

---

# Threads

Quando existir concorrência.

Representar:

- Thread principal
- Worker
- Scheduler
- Timer
- Quartz

---

# Comunicação de Rede

Representar:

Cliente

Servidor

Pacotes

Eventos

Timeouts

Reconexões

---

# Dependências Externas

Sempre separar.

Exemplo

```
AdvancedBot

↓

Minecraft Server

↓

Proxy

↓

Discord

↓

Banco
```

---

# Diagramas de Migração

Cada componente importante deve possuir:

Arquitetura C#

↓

Arquitetura Java

↓

Diferenças

↓

Decisões

↓

Status

---

# Diagrama Antes x Depois

Sempre que possível.

Exemplo

```
C#

Macro

↓

Packet

↓

Socket

↓

Servidor



Java

Macro

↓

Camel

↓

NetworkService

↓

Servidor
```

---

# Atualização dos Diagramas

Obrigatório atualizar quando houver:

nova classe

novo pacote

nova arquitetura

novo fluxo

mudança estrutural

refatoração

remoção de componentes

---

# Versionamento

Cada diagrama deve registrar:

Versão

Autor

Data

Descrição da alteração

Exemplo

```
v1.0

Criação

v1.1

Adicionado fluxo de login

v1.2

Atualizado módulo Inventory
```

---

# Revisão

Durante Pull Requests verificar:

- diagramas atualizados
- nomenclatura correta
- consistência
- fluxo correto
- sem componentes inexistentes
- sem dependências incorretas

---

# Integração com Outros Documentos

Os diagramas devem possuir referências para:

- 01-Visao-Geral.md
- 02-Arquitetura.md
- 03-Roadmap.md
- 04-Checklist-Mestre.md
- 05-Procedimento-Operacional.md
- 06-Definition-of-Done.md
- 07-Controle-de-Decisoes.md
- 08-Registro-de-Sessoes.md
- 09-Metricas-do-Projeto.md
- 10-Protocolo-de-Uso-com-IA.md
- 11-Guia-de-Documentacao.md
- 12-Guia-de-Nomenclatura.md
- 13-Matriz-de-Rastreabilidade.md
- 14-Fluxo-de-Migracao.md
- 15-Boas-Praticas.md

---

# Checklist de Qualidade

Antes de considerar um diagrama concluído, verificar:

- [ ] Objetivo claramente definido
- [ ] Escopo documentado
- [ ] Nomenclatura oficial utilizada
- [ ] Componentes identificados corretamente
- [ ] Fluxo consistente
- [ ] Decisões representadas
- [ ] Tratamento de erros documentado
- [ ] Dependências representadas
- [ ] Legibilidade adequada
- [ ] Compatível com Mermaid ou PlantUML
- [ ] Versionado
- [ ] Revisado
- [ ] Relacionado aos documentos oficiais do projeto
- [ ] Atualizado conforme a implementação mais recente
```