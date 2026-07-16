# Documento de Governança

| Campo | Valor |
|--------|-------|
| **Código** | GOV-000 |
| **Título** | README da Governança do Projeto |
| **Versão** | 1.0.0 |
| **Status** | Oficial |
| **Data de Criação** | 2026-07-14 |
| **Última Atualização** | 2026-07-14 |
| **Responsável** | Equipe de Arquitetura |
| **Escopo** | Projeto Completo |
| **Tipo** | Governança |
| **Leitura Obrigatória** | Sim |

---

# README — Governança da Documentação

## Objetivo

Esta pasta estabelece as normas oficiais que regem toda a documentação do projeto de reescrita do **AdvancedBot**.

Ela define:

- como a documentação deve ser escrita;
- como deve ser mantida;
- quais padrões devem ser utilizados;
- como qualquer Inteligência Artificial deverá trabalhar;
- como decisões técnicas devem ser registradas;
- como garantir consistência durante toda a vida útil do projeto.

Nenhum documento desta pasta descreve funcionalidades do AdvancedBot.

Todos os documentos desta pasta descrevem **como a documentação do projeto deve ser produzida e mantida**.

---

# Missão

Transformar toda a documentação do projeto em uma **especificação técnica oficial**, permitindo que qualquer desenvolvedor ou Inteligência Artificial consiga compreender, evoluir e implementar o sistema sem depender do código legado em C#.

---

# Objetivos

A Governança possui os seguintes objetivos principais:

- padronizar toda a documentação;
- eliminar ambiguidades;
- evitar documentos duplicados;
- impedir perda de conhecimento;
- garantir rastreabilidade;
- facilitar auditorias;
- permitir colaboração entre diferentes modelos de IA;
- reduzir retrabalho;
- reduzir consumo de tokens;
- garantir consistência entre documentação e implementação.

---

# Escopo

Esta pasta é responsável exclusivamente por definir regras de documentação.

Ela não substitui:

- documentação funcional;
- documentação técnica;
- documentação arquitetural;
- documentação do domínio.

Ela apenas estabelece **como essas documentações deverão ser produzidas**.

---

# Estrutura da Governança

```
99-Governanca/
│
├── 00-README.md
├── 01-Decisoes-Arquiteturais.md
├── 01-Plano-Mestre-Reescrita.md
├── 02-Estado-Atual.md
├── 02-Revisao-Arquitetural-Fase-1.2.md
├── 03-Modelo-Operacional-e-Isolamento.md
├── 03-Roadmap.md
├── 04-Arquitetura-Tecnica-da-Plataforma.md
├── 04-Checklist-Mestre.md
├── 05-Procedimento-Operacional.md
├── 06-Definition-of-Done.md
├── 07-Controle-de-Decisoes.md
├── 08-Registro-de-Sessoes.md
├── 09-Metricas-do-Projeto.md
├── 10-Protocolo-de-Uso-com-IA.md
├── 11-Guia-de-Documentacao.md
├── 12-Guia-de-Nomenclatura.md
├── 13-Matriz-de-Rastreabilidade.md
├── 14-Fluxo-de-Migracao.md
├── 15-Boas-Praticas.md
├── 16-Padroes-de-Diagramas.md
├── 17-Padroes-de-Mermaid.md
├── 18-Padroes-de-Markdown.md
├── 19-Politica-de-Atualizacao.md
└── 20-Constituicao-do-Projeto.md
```

---

# Ordem Obrigatória de Leitura

Toda IA, desenvolvedor ou colaborador deverá seguir obrigatoriamente esta ordem antes de executar qualquer atividade.

```text
README

↓

Plano Mestre

↓

Estado Atual

↓

Roadmap

↓

Checklist Mestre

↓

Procedimento Operacional

↓

Definition of Done

↓

Protocolo de Uso com IA

↓

Guia de Documentação

↓

Documentação Técnica
```

Nenhuma atividade deverá ser iniciada antes da leitura destes documentos.

---

# Princípios Fundamentais

Toda documentação deste projeto deverá seguir os princípios abaixo.

## Fonte Única da Verdade

Cada assunto deverá possuir um único documento oficial.

É proibida a duplicação de conteúdo.

---

## Rastreabilidade

Toda informação deverá possuir origem claramente identificável.

Sempre que possível deverá existir referência para:

- documento relacionado;
- módulo correspondente;
- fluxo funcional;
- regra de negócio;
- código legado.

---

## Consistência

Todos os documentos deverão utilizar:

- mesma terminologia;
- mesma estrutura;
- mesmo padrão visual;
- mesmo padrão de diagramas;
- mesmo padrão de nomenclatura.

---

## Evolução Contínua

A documentação nunca deverá ser descartada.

Ela deverá ser continuamente refinada.

Correções deverão preservar o histórico.

---

## Neutralidade Tecnológica

A documentação funcional deverá descrever comportamentos.

Ela não deverá depender da linguagem C# nem da futura implementação Java.

---

# Regras Gerais

É obrigatório:

- atualizar documentação antes da implementação;
- revisar documentos existentes antes de criar novos;
- preservar histórico de alterações;
- manter rastreabilidade;
- seguir os templates oficiais.

É proibido:

- criar documentação duplicada;
- alterar arquitetura sem registro de decisão;
- remover informações sem justificativa;
- modificar nomenclatura oficial;
- resumir documentação técnica;
- criar documentação baseada em suposições.

---

# Convenções Gerais

Toda documentação deverá utilizar:

- Markdown (.md)
- UTF-8
- Mermaid para diagramas
- Tabelas Markdown
- Cabeçalhos hierárquicos
- Linguagem técnica objetiva
- Português brasileiro

---

# Fluxo Oficial do Projeto

```text
Engenharia Reversa

↓

Documentação

↓

Auditoria

↓

Correções

↓

Congelamento da Especificação

↓

Modelagem Java

↓

Implementação

↓

Testes

↓

Validação

↓

Release
```

Nenhuma fase deverá iniciar sem que a fase anterior esteja aprovada.

---

# Público-Alvo

Esta documentação destina-se a:

- Arquitetos de Software
- Desenvolvedores Java
- Desenvolvedores C#
- Especialistas em Minecraft Protocol
- Engenheiros de Software
- Revisores Técnicos
- Modelos de Inteligência Artificial

---

# Atualização deste Documento

Este documento deverá ser atualizado sempre que houver mudanças na estrutura da governança.

Mudanças de arquitetura deverão ser registradas no documento **Controle de Decisões**.

---

# Histórico de Alterações

| Versão | Data | Responsável | Alterações |
|---------|------|-------------|------------|
| 1.0.0 | 2026-07-14 | Equipe de Arquitetura | Criação inicial do documento. |

---

# Observação Final

Este documento é considerado a porta de entrada oficial da documentação.

Todo colaborador humano ou Inteligência Artificial deverá iniciar sua leitura por este arquivo antes de consultar qualquer outro documento do projeto.

O não cumprimento desta regra compromete a consistência da documentação e aumenta significativamente o risco de divergências arquiteturais durante a reescrita do AdvancedBot para Java 21.