# 11 - Estado Atual da Migração

> Última atualização: 2026-07-15
>
> Documento responsável por registrar o estado oficial da migração do AdvancedBot C# para Java.
>
> Este documento deve ser atualizado ao final de cada milestone concluída.

---

# 1. Objetivo

Este documento representa a fotografia oficial do projeto.

Toda IA, desenvolvedor ou colaborador deve consultá-lo antes de iniciar novas implementações.

Seu objetivo é evitar perda de contexto entre sessões e impedir regressões arquiteturais.

---

# 2. Status Geral

Projeto: AdvancedBot Java

Status:

☑ Planejamento

☑ Governança

☑ Fundação Arquitetural

☑ Baseline Tecnológica

☐ Migração do Domínio

☐ Infraestrutura Completa

☐ Interface React

☐ Testes Integrados

☐ Release

---

# 3. Stack Oficial

## Backend

- Java 21 LTS
- Spring Boot 3.2.5
- Maven

## Frontend

- React

## Banco

- PostgreSQL

## Arquitetura

- Clean Architecture
- Hexagonal Architecture
- DDD (quando aplicável)
- SOLID

---

# 4. Estado da Governança

| Documento | Status |
|-----------|--------|
| 00 | ✔ |
| 01 | ✔ |
| 02 | ✔ |
| 03 | ✔ |
| 04 | ✔ |
| 05 | ✔ |
| 06 | ✔ |
| 07 | ✔ |
| 08 | ✔ |
| 09 | ✔ |
| 10 | ✔ |

---

# 5. Milestones

## Milestone 1

Status

Concluído

Objetivos

- Estrutura Maven
- Spring Boot
- Organização inicial
- Fundação arquitetural

Resultado

Aprovado

---

## Milestone 2

Status

Concluído

Objetivos

- Consolidação da baseline Java 21
- Ajustes de documentação

Resultado

Aprovado

---

## Milestone 3

Status

Em andamento

Objetivo

Início da migração incremental do domínio C#

Progresso

- Pacotes `domain.bot` e `application.usecase` criados (DEC-12).
- Primeira entidade: `Bot` (agregado raiz, identidade + configuração de conexão).
- Primeiros Value Objects: `IdentificadorBot`, `EnderecoServidor`, `CredenciaisBot`.
- Primeiro Use Case: `CasoDeUsoCriarBot`.
- Sem regras de negócio completas do legado (rede, protocolo, macros) — fora do escopo desta etapa.

---

# 6. Escopo Implementado

Atualmente existem apenas componentes estruturais.

Implementado:

- Estrutura Maven
- Spring Boot
- Organização dos módulos (camadas `domain`/`application`/`infrastructure`/`interfaces`, DEC-12)
- Documentação
- Arquitetura
- Primeira entidade de domínio (`Bot`) e Value Objects associados
- Primeiro Caso de Uso (`CasoDeUsoCriarBot`)

Não implementado:

- Serviços
- Adaptadores
- Repositórios
- APIs REST
- Scheduler
- Rede/Protocolo Minecraft
- Macros
- Cliente Minecraft completo

---

# 7. Restrições do Projeto

É proibido:

- Converter automaticamente código C#
- Criar regras de negócio sem validação
- Alterar comportamento do legado
- Implementar funcionalidades fora da milestone atual

Toda evolução deve ocorrer de forma incremental.

---

# 8. Decisões Arquiteturais Congeladas

## Plataforma

Java 21 LTS

## Framework

Spring Boot

## Build

Maven

## Frontend

React

## Banco

PostgreSQL

## Arquitetura

Clean + Hexagonal

Estas decisões somente podem ser alteradas mediante nova ADR.

---

# 9. Pendências Conhecidas

Lista das atividades ainda não iniciadas.

Exemplo:

- Modelagem do domínio
- Definição dos Bounded Contexts
- Estratégia de persistência
- Migração dos bots
- Migração do protocolo Minecraft
- Scheduler distribuído

---

# 10. Próxima Milestone (continuação da Milestone 3)

Nome

Migração do Núcleo do Domínio

Objetivos

- Definir pacotes do domínio ✔ (`domain.bot`, `application.usecase` — DEC-12)
- Criar primeiras entidades ✔ (`Bot`)
- Criar primeiros Value Objects ✔ (`IdentificadorBot`, `EnderecoServidor`, `CredenciaisBot`)
- Criar primeiros Use Cases ✔ (`CasoDeUsoCriarBot`)

Sem implementar regras completas do legado.

Pendente para concluir a Milestone 3: demais entidades do núcleo do domínio (ex.: sessão, estado de conexão) ainda não modeladas — aguardando priorização.

---

# 11. Próximo Prompt Esperado

Resumo da próxima atividade que deverá ser executada pela IA.

Continuar a Milestone 3 (Migração do Núcleo do Domínio): modelar as próximas entidades/Value Objects do domínio do bot ainda não cobertas (ex.: estado de sessão), mantendo o escopo restrito ao domínio puro, sem rede/protocolo/persistência.

---

# 12. Histórico de Execuções

| Data | Milestone | Resultado |
|-------|-----------|-----------|
| 2026-07-14 | Fundação da Plataforma | ✔ |
| 2026-07-15 | Correção Java 21 | ✔ |
| 2026-07-15 | Milestone 3 (incremento 1): Bot, Value Objects e CasoDeUsoCriarBot | ✔ |

---

# 13. Critérios para Continuidade

Antes de iniciar qualquer nova tarefa é obrigatório verificar:

- Estado deste documento
- Plano de Migração
- Fundação Arquitetural
- Decisões Arquiteturais
- Baseline Tecnológica

Caso exista conflito, este documento deve ser atualizado antes da implementação.