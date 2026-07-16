# 18. Padrões de Markdown

## Objetivo

Este documento estabelece o padrão oficial de escrita em Markdown para todo o projeto de migração do AdvancedBot de C# para Java.

Seu objetivo é garantir que toda a documentação seja consistente, organizada, fácil de manter, facilmente navegável e compatível com diferentes renderizadores de Markdown, permitindo que qualquer integrante do projeto produza documentação uniforme independentemente do documento que esteja sendo editado.

Este padrão deve ser seguido em todos os arquivos Markdown do projeto sem exceções.

---

# Objetivos da Padronização

A padronização da documentação possui os seguintes objetivos:

- Garantir uniformidade visual.
- Facilitar a leitura.
- Facilitar a manutenção futura.
- Facilitar revisões.
- Evitar ambiguidades.
- Melhorar a navegação.
- Permitir reutilização de conteúdo.
- Reduzir inconsistências.
- Melhorar a experiência de novos colaboradores.
- Produzir documentação profissional.

---

# Compatibilidade Obrigatória

Toda documentação deve ser compatível com:

- GitHub Markdown
- GitLab Markdown
- Azure DevOps Wiki
- Visual Studio Code
- Obsidian
- MkDocs
- Mermaid
- Markdown Preview Enhanced

Não devem ser utilizados recursos exclusivos de um renderizador específico que prejudiquem a visualização em outro.

---

# Filosofia de Escrita

Toda documentação deve seguir os seguintes princípios:

- Clareza
- Simplicidade
- Objetividade
- Precisão
- Consistência
- Organização
- Padronização
- Reutilização
- Facilidade de manutenção

A documentação deve explicar o suficiente para qualquer desenvolvedor compreender o conteúdo sem depender do conhecimento prévio do autor.

---

# Estrutura Obrigatória dos Documentos

Sempre que aplicável, os documentos devem seguir a seguinte estrutura.

```text
Título

Objetivo

Escopo

Contexto

Pré-requisitos

Sumário

Conteúdo

Exemplos

Boas Práticas

Erros Comuns

Checklist

Referências
```

Nem todas as seções são obrigatórias para documentos pequenos, porém a organização deve ser mantida.

---

# Hierarquia dos Títulos

Cada documento deve possuir exatamente um título principal.

```markdown
# Nome do Documento
```

Os demais níveis devem seguir obrigatoriamente esta hierarquia.

```markdown
# H1

## H2

### H3

#### H4
```

Evitar utilizar H5 e H6.

Caso um conteúdo necessite níveis adicionais, o documento deverá ser dividido.

---

# Padronização dos Títulos

Todos os títulos devem utilizar Title Case.

Exemplos corretos

```text
Fluxo de Migração

Boas Práticas

Matriz de Rastreabilidade

Registro de Sessões
```

Exemplos incorretos

```text
fluxo de migração

BOAS PRÁTICAS

registro de sessões
```

---

# Espaçamento

Sempre deixar exatamente uma linha em branco entre:

- títulos
- subtítulos
- listas
- tabelas
- diagramas
- blocos de código
- citações
- parágrafos

Nunca agrupar elementos sem espaçamento.

---

# Parágrafos

Os parágrafos devem ser curtos.

Preferencialmente entre:

- duas linhas
- cinco linhas

Caso um texto fique muito extenso, ele deverá ser dividido em novos parágrafos.

---

# Comprimento das Linhas

Sempre que possível manter linhas entre:

100 e 120 caracteres.

Evitar linhas extremamente longas.

---

# Estilo da Escrita

Sempre utilizar:

- voz ativa
- linguagem técnica
- frases objetivas
- escrita impessoal
- termos consistentes

Evitar:

- linguagem informal
- opiniões pessoais
- ambiguidades
- exageros
- repetições desnecessárias

---

# Ênfase

Utilizar apenas três tipos de destaque.

Negrito

```markdown
**Importante**
```

Itálico

```markdown
*Observação*
```

Código Inline

```markdown
`PlayerInventory`
```

Evitar excesso de negrito.

---

# Código Inline

Todo elemento técnico deve utilizar código inline.

Exemplos:

Classes

```markdown
`Player`
```

Interfaces

```markdown
`IInventory`
```

Métodos

```markdown
`loadInventory()`
```

Construtores

```markdown
`Player()`
```

Variáveis

```markdown
`playerInventory`
```

Constantes

```markdown
`MAX_STACK_SIZE`
```

Enums

```markdown
`InventoryType`
```

Pacotes

```markdown
`com.advancedbot.inventory`
```

Arquivos

```markdown
`Player.java`
```

XML

```markdown
`application.xml`
```

SQL

```markdown
`TB_PLAYER`
```

---

# Listas

Todas as listas não ordenadas devem utilizar exclusivamente:

```markdown
-
```

Exemplo

```markdown
- Item
- Item
- Item
```

Nunca misturar:

- *
- +
- -

---

# Listas Numeradas

Utilizar somente quando existir ordem lógica.

```markdown
1. Primeiro
2. Segundo
3. Terceiro
```

---

# Checklists

Sempre utilizar.

```markdown
- [ ]
```

ou

```markdown
- [x]
```

Nunca utilizar símbolos alternativos.

---

# Tabelas

Sempre alinhar visualmente.

```markdown
| Campo | Tipo | Obrigatório |
|-------|------|-------------|
| Id | Long | Sim |
| Nome | String | Sim |
```

Utilizar tabelas apenas quando elas facilitarem comparações.

---

# Blocos de Código

Todo bloco deve informar a linguagem.

Java

````markdown
```java
public class Player {

}
```