# 05 - Auditoria do Legado C#

## Objetivo

Este documento apresenta a auditoria arquitetural detalhada do sistema legado AdvancedBot em C#.

Seu objetivo é mapear a estrutura atual, catalogar componentes, identificar acoplamentos e definir a estratégia de migração para cada módulo rumo à nova arquitetura Java 21, alinhada com as decisões arquiteturais (DEC-01 a DEC-10).

## Contexto

A migração de C# para Java exige um inventário preciso para evitar a transferência de dívida técnica.

O legado apresenta forte acoplamento entre interface gráfica, lógica de rede e regras de negócio, exigindo uma reestruturação profunda baseada no descarte de camadas visuais (DEC-02) e adoção de um modelo orquestrado por Spring Boot (DEC-08).

## Estrutura de Diretórios C#

Abaixo está o mapeamento dos principais diretórios e módulos do projeto legado.

- `AdvancedBot.Client`: Núcleo do domínio, entidades, inventário, e gerenciamento de estado.
- `AdvancedBot.Client.Commands`: Implementação dos comandos de bot e macros.
- `AdvancedBot.Client.Handler`: Manipuladores de protocolo de rede (`1.5.2`, `1.7`, `1.8`, `1.9`).
- `AdvancedBot.Client.PathFinding`: Lógica de movimentação automatizada (A*).
- `AdvancedBot.Client.Packets`: Serialização e desserialização de pacotes Minecraft.
- `AdvancedBot.Crypto`: Funções criptográficas originais.
- `AdvancedBot.Plugins`: Mecanismo original de carregamento de extensões C#.
- `AdvancedBot.Script`: Parser e engine de scripts antigos.
- `AdvancedBot.Viewer`: Interface de renderização 3D baseada em OpenGL.
- Interfaces Gráficas: Classes como `AdvancedBot.Main`, `AdvancedBot.Spammer` e `AdvancedBot.AccountChecker` (Windows Forms).

## Catalogação e Estratégia de Migração

Cada módulo foi avaliado e classificado segundo seu futuro na arquitetura Java.

### Componentes para Descartar

Módulos que não farão parte da nova arquitetura.

- `AdvancedBot.Viewer`: Descartado devido à DEC-02 (Remoção da UI e renderização 3D).
- Interfaces Windows Forms: Descartadas a favor do uso de linha de comando (CLI) ou APIs web.
- `AdvancedBot.Script`: Descartado devido à complexidade desnecessária de um interpretador proprietário.

### Componentes para Substituir

Módulos que serão trocados por soluções prontas do ecossistema Java.

- `AdvancedBot.Crypto`: Substituído pela biblioteca BouncyCastle para suporte AES-CFB8 (DEC-04).
- Sistema de Configuração: Substituído de arquivos binários NBT para YAML (DEC-05).
- `AdvancedBot.Plugins`: Substituído por um carregador simples e isolado (DEC-06).

### Componentes para Reescrever

Módulos cujo comportamento deve ser mantido, mas com nova estrutura técnica.

- `AdvancedBot.Client.Handler`: Reescrito com foco exclusivo no protocolo 1.8 inicialmente (DEC-01 e DEC-07).
- `AdvancedBot.Client.Commands`: Reescrito utilizando Virtual Threads para concorrência segura (DEC-03).
- Conexões de Rede: Reescritas para injeção de dependência de Proxies (DEC-10).

### Componentes para Migrar Diretamente

Lógicas essenciais e algoritmos que serão transcritos com poucas alterações.

- Lógica de `PathFinding` (Algoritmo A*).
- Entidades básicas (`Player`, `Item`, `Inventory`).
- Constantes de blocos e itens (`Blocks.cs`, `Items.cs`).

## Dependências e Acoplamentos Arquiteturais

A auditoria identificou os seguintes acoplamentos severos no C#.

### Acoplamento Global de Estado

O legado utiliza estados globais e variáveis estáticas, o que gerava falhas quando múltiplos bots rodavam no mesmo processo.

A solução adotada será o modelo Operacional Single-Tenant, rodando uma JVM por instância de bot (DEC-09).

### Acoplamento de Rede e Interface

A lógica de recepção de pacotes possuía acoplamento com atualizações de interface visual.

A solução é o descarte completo da interface visual, garantindo processamento em *background*.

### Condições de Corrida nas Macros

Os comandos (`CommandBreakBlock`, `CommandAntiAFK`) interagiam livremente com fluxos assíncronos, causando *race conditions*.

A solução é o isolamento de escopo por Virtual Threads e a orquestração segura gerenciada pelo Spring Boot (DEC-08).

## Riscos de Migração

Foram mapeados os seguintes riscos durante a análise técnica.

- **Comportamentos Ocultos de Anti-Cheat**: Lógicas não documentadas no `PathFinding` e movimentação que evitam detecções.
- **Falha de Paridade em Criptografia**: Risco alto na fase de *Handshake*, exigindo validação rigorosa com BouncyCastle.
- **Sobrecarga de Threads**: Criação inadequada de macros que pode esgotar os recursos na reescrita para Java, mitigado pelo uso de *Virtual Threads*.

## Mapeamento de Módulos para o Futuro Java

A estrutura de pacotes no ecossistema Spring Boot seguirá este mapeamento inicial.

| Origem C# | Destino Java | Função |
|-----------|--------------|--------|
| `AdvancedBot.Client` | `com.advancedbot.core.domain` | Modelos de domínio. |
| `AdvancedBot.Client.Handler` | `com.advancedbot.core.network` | Manipuladores e abstração 1.8. |
| `AdvancedBot.Client.Commands` | `com.advancedbot.core.macro` | Comandos em Virtual Threads. |
| `AdvancedBot.Crypto` | `com.advancedbot.core.security` | Wrappers do BouncyCastle. |
| Configurações NBT | `com.advancedbot.config` | Beans do Spring e arquivos YAML. |
| Arquivos de Proxy | `com.advancedbot.core.proxy` | Injeção de Proxy via Socket. |
