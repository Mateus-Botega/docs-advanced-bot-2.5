# 07 - Controle de Decisões

# Objetivo

Este documento registra todas as decisões arquiteturais, técnicas e funcionais tomadas durante o processo de reescrita do AdvancedBot de C# para Java.

Seu objetivo é preservar o histórico completo das decisões do projeto, evitando que conhecimentos importantes fiquem apenas na memória dos desenvolvedores.

Toda decisão relevante deve ser registrada antes de sua implementação definitiva.

---

# Objetivos do Documento

- Documentar decisões importantes.
- Registrar justificativas.
- Registrar alternativas avaliadas.
- Registrar impactos.
- Evitar retrabalho.
- Facilitar auditorias futuras.
- Facilitar onboarding de novos desenvolvedores.
- Servir como histórico técnico do projeto.

---

# Quando Registrar uma Decisão

Toda decisão deve ser registrada quando envolver:

- Arquitetura.
- Frameworks.
- Bibliotecas.
- Estrutura de projeto.
- Mudanças de padrão.
- Alteração de comportamento.
- Alteração de algoritmo.
- Mudança de modelagem.
- Refatorações grandes.
- Alterações de protocolos.
- Mudanças na comunicação entre módulos.
- Mudanças de persistência.
- Mudanças de infraestrutura.
- Alterações de desempenho.
- Alterações que impactem documentação existente.

---

# O que NÃO deve ser registrado

Não registrar:

- Correções simples.
- Pequenos bugs.
- Ajustes de nomenclatura.
- Refatorações locais.
- Alterações cosméticas.
- Commits pequenos.

Esses registros pertencem ao histórico do Git.

---

# Estrutura Obrigatória de Cada Decisão

Cada decisão deve possuir exatamente os seguintes tópicos.

## Identificador

Formato:

```
DEC-0001
DEC-0002
DEC-0003
...
```

Nunca reutilizar um identificador.

---

## Título

Título curto e objetivo.

Exemplo:

```
Substituição do sistema de eventos por EventBus
```

---

## Data

Registrar a data da decisão.

Exemplo

```
2026-07-14
```

---

## Status

Utilizar apenas um dos seguintes:

- Proposta
- Em Avaliação
- Aprovada
- Implementada
- Rejeitada
- Substituída
- Cancelada

---

## Responsável

Informar quem tomou a decisão.

Exemplo

```
Mateus Botega
```

ou

```
Equipe
```

---

## Categoria

Utilizar uma categoria.

Exemplos:

- Arquitetura
- Infraestrutura
- Persistência
- Banco de Dados
- API
- Segurança
- Performance
- Organização
- Build
- Maven
- Gradle
- Interface
- Threads
- Rede
- Minecraft Protocol
- Bot Logic
- Logging
- Configuração

---

## Contexto

Descrever o problema existente.

Responder:

- O que motivou a decisão?
- Qual dificuldade existia?
- Qual limitação foi encontrada?
- O que precisava ser resolvido?

---

## Problema

Descrever exatamente qual era o problema.

Quanto mais específico, melhor.

---

## Alternativas Avaliadas

Registrar todas as alternativas consideradas.

Exemplo:

- Manter implementação original
- Reescrever completamente
- Utilizar biblioteca externa
- Criar framework próprio

Cada alternativa deve possuir vantagens e desvantagens.

---

## Decisão Tomada

Descrever exatamente o que foi decidido.

Esta seção deve ser objetiva.

---

## Justificativa

Explicar por que essa decisão foi escolhida.

Responder:

- Por que esta solução é melhor?
- Quais benefícios oferece?
- Quais riscos evita?

---

## Consequências

Registrar consequências positivas.

Exemplo:

- Código mais limpo
- Melhor manutenção
- Maior desempenho
- Menor acoplamento

Também registrar consequências negativas.

Exemplo:

- Curva de aprendizado
- Maior complexidade inicial
- Refatoração necessária

---

## Impacto

Informar quais partes do projeto foram afetadas.

Exemplo:

- Core
- Network
- Inventory
- PathFinder
- AI
- Commands
- Events
- GUI

---

## Documentação Relacionada

Relacionar documentos.

Exemplo:

- 01-Plano-Mestre-Reescrita.md
- 03-Roadmap.md
- 04-Checklist-Mestre.md
- ADR relacionada
- Issue correspondente

---

## Classes Afetadas

Listar classes.

Exemplo

```
MinecraftClient
NetworkManager
PacketReader
InventoryManager
```

---

## Arquivos Alterados

Quando possível registrar.

Exemplo

```
src/main/java/...
```

---

## Evidências

Adicionar evidências.

Exemplos:

- Diagramas
- Prints
- Benchmarks
- Logs
- Métricas
- Testes
- Capturas de tela

---

## Riscos

Registrar riscos.

Exemplo

- Compatibilidade
- Regressões
- Performance
- Complexidade
- Dependências

---

## Plano de Rollback

Caso necessário.

Descrever como desfazer a decisão.

---

## Critério de Sucesso

Como saber que a decisão foi correta?

Exemplos:

- Todos os testes passam
- Performance aumentou
- Código simplificado
- Bugs eliminados
- Cobertura mantida

---

## Observações

Espaço livre para comentários adicionais.

---

# Template Padrão

Toda decisão deve seguir exatamente este modelo.

```markdown
# DEC-000X — Título

## Data

AAAA-MM-DD

## Status

Aprovada

## Responsável

Nome

## Categoria

Arquitetura

## Contexto

...

## Problema

...

## Alternativas Avaliadas

### Alternativa 1

Vantagens

Desvantagens

### Alternativa 2

Vantagens

Desvantagens

## Decisão Tomada

...

## Justificativa

...

## Consequências

### Positivas

-

### Negativas

-

## Impacto

-

## Documentação Relacionada

-

## Classes Afetadas

-

## Arquivos Alterados

-

## Evidências

-

## Riscos

-

## Plano de Rollback

-

## Critério de Sucesso

-

## Observações

...
```

---

# Fluxo de Aprovação

Toda decisão deve seguir este fluxo:

```
Necessidade Identificada
        │
        ▼
Análise do Problema
        │
        ▼
Alternativas Avaliadas
        │
        ▼
Discussão Técnica
        │
        ▼
Escolha da Solução
        │
        ▼
Registro da Decisão
        │
        ▼
Implementação
        │
        ▼
Validação
        │
        ▼
Status = Implementada
```

---

# Boas Práticas

Sempre:

- Documentar antes da implementação.
- Escrever justificativas técnicas.
- Registrar alternativas rejeitadas.
- Atualizar o status conforme a evolução.
- Relacionar decisões dependentes.
- Referenciar documentos impactados.
- Utilizar linguagem objetiva.
- Evitar opiniões sem embasamento.
- Basear decisões em evidências sempre que possível.

---

# Rastreabilidade

Sempre relacionar uma decisão com:

- Roadmap
- Checklist
- Plano Mestre
- Procedimento Operacional
- Definition of Done
- Commits
- Pull Requests
- Issues
- Testes
- Benchmark
- Diagramas

Isso permite rastrear completamente a evolução técnica do projeto.

---

# Convenções

## Numeração

Sempre sequencial.

```
DEC-0001
DEC-0002
DEC-0003
...
```

Jamais reutilizar números.

---

## Status Permitidos

| Status | Significado |
|---------|-------------|
| Proposta | A decisão foi criada e ainda não analisada. |
| Em Avaliação | Alternativas estão sendo discutidas. |
| Aprovada | A decisão foi aceita, aguardando implementação. |
| Implementada | A solução foi aplicada ao projeto. |
| Rejeitada | A proposta foi descartada. |
| Substituída | Foi substituída por outra decisão. |
| Cancelada | A necessidade deixou de existir. |

---

# Critérios de Qualidade

Uma decisão somente é considerada completa quando:

- Possui identificador único.
- Possui contexto bem definido.
- O problema está claramente descrito.
- As alternativas foram registradas.
- A decisão está documentada.
- Existe justificativa técnica.
- Os impactos foram identificados.
- Os riscos foram avaliados.
- O plano de rollback foi definido (quando aplicável).
- Existe critério objetivo de sucesso.
- A documentação relacionada foi vinculada.
- O status está atualizado.
- O documento está consistente com os demais artefatos do projeto.