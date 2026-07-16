# 00 — README — Documentação do Legado C#

---

| Campo              | Valor                                    |
|--------------------|------------------------------------------|
| **Código**         | LEG-00                                   |
| **Versão**         | 1.0.0                                    |
| **Status**         | Ativo                                    |
| **Responsável**    | Equipe de Migração AdvancedBot           |
| **Última Atualização** | 2026-07-14                           |
| **Tipo**           | Índice / Visão Geral                     |
| **Escopo**         | Documentação técnica do sistema legado   |
| **Documento Pai**  | docs/99-Governanca/00-README.md          |
| **Documentos Relacionados** | Todos os documentos desta pasta |

---

## Objetivo

Esta pasta contém o inventário arquitetural completo do sistema legado AdvancedBot, desenvolvido originalmente em C#.

A documentação aqui reunida representa a **fase de levantamento técnico** do projeto de reescrita para Java.

Nenhuma migração deve ser iniciada antes que todos os documentos desta pasta sejam revisados e aprovados.

---

## Finalidade

A documentação do legado tem as seguintes finalidades:

- Registrar fidedignamente o comportamento atual do sistema.
- Identificar todos os componentes, classes e dependências.
- Mapear os fluxos de negócio principais.
- Identificar pontos críticos e riscos para a migração.
- Servir como referência técnica durante toda a reescrita.
- Garantir rastreabilidade completa entre C# e Java.

---

## Conteúdo desta Pasta

| Documento | Código | Descrição |
|-----------|--------|-----------|
| `01-Visao-Geral-do-Sistema.md` | LEG-01 | Finalidade, contexto e arquitetura atual |
| `02-Estrutura-da-Solucao.md` | LEG-02 | Projetos, namespaces e responsabilidades |
| `03-Catalogo-de-Classes.md` | LEG-03 | Catálogo completo de classes e interfaces |
| `04-Fluxos-de-Negocio.md` | LEG-04 | Fluxos principais do sistema |
| `05-Dependencias-e-Bibliotecas.md` | LEG-05 | Bibliotecas externas e equivalentes Java |
| `06-Pontos-Criticos-da-Migracao.md` | LEG-06 | Riscos, complexidades e redesigns |
| `07-Diagrama-Arquitetural.md` | LEG-07 | Diagramas Mermaid da arquitetura legada |
| `08-Matriz-de-Migracao-Inicial.md` | LEG-08 | Rastreabilidade C# → Java |

---

## Metodologia Utilizada

O levantamento foi realizado por análise direta do código-fonte C#, incluindo:

- Leitura dos arquivos `.cs` de todos os módulos.
- Análise do arquivo de solução `AdvancedBot_Crack.sln`.
- Análise do arquivo de projeto `AdvancedBot_Crack.csproj`.
- Levantamento de todas as dependências externas.
- Mapeamento de fluxos de execução por rastreamento de chamadas.
- Identificação de padrões arquiteturais e anti-padrões.

---

## Escopo do Levantamento

O levantamento cobre integralmente:

- O projeto único `AdvancedBot_Crack` (solução Visual Studio 2022).
- Todos os módulos organizados como pastas dentro do projeto único.
- Todas as dependências externas declaradas no `.csproj` e presentes como DLLs.
- A camada de interface gráfica (Windows Forms).
- A camada de protocolo de rede (Minecraft Protocol).
- O sistema de scripts e macros.
- O sistema de plugins.
- O motor de renderização 3D.

---

## Limitações do Levantamento

Os seguintes itens **não** são cobertos por este levantamento:

- Código de macros de terceiros não incluídos no repositório.
- Arquivos de configuração gerados em tempo de execução (`conf.dat`).
- Logs de erro gerados em produção (`errlogs/`).
- Plugins externos (`.dll` / `.abp`) não presentes no repositório.

---

## Aprovação Necessária

Antes de qualquer conversão de código C# → Java, é obrigatório:

1. Revisar e aprovar todos os documentos desta pasta.
2. Registrar a aprovação em `docs/99-Governanca/08-Registro-de-Sessoes.md`.
3. Registrar decisões arquiteturais pendentes em `docs/99-Governanca/07-Controle-de-Decisoes.md`.

---

## Referências

- `docs/99-Governanca/11-Guia-de-Documentacao.md`
- `docs/99-Governanca/12-Guia-de-Nomenclatura.md`
- `docs/99-Governanca/14-Fluxo-de-Migracao.md`
- `docs/99-Governanca/20-Constituicao-do-Projeto.md`
