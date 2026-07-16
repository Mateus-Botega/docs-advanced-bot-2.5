# Guia de Documentação

## Objetivo

Este documento define o padrão oficial de documentação utilizado durante todo o processo de reescrita do AdvancedBot originalmente desenvolvido em C# para Java.

Toda documentação produzida durante o projeto deverá seguir este guia, garantindo:

- Uniformidade.
- Facilidade de manutenção.
- Rastreabilidade.
- Clareza técnica.
- Histórico completo das decisões.
- Padronização entre todas as etapas da migração.

Este guia deve ser considerado a referência principal para qualquer novo documento criado no projeto.

---

# Princípios da Documentação

Toda documentação deverá seguir os seguintes princípios.

## Clareza

A documentação deve ser compreensível mesmo para alguém que nunca participou do projeto.

Evitar:

- textos ambíguos;
- explicações vagas;
- abreviações sem definição.

Sempre explicar:

- o objetivo;
- o motivo;
- o funcionamento;
- as consequências.

---

## Objetividade

Cada seção deve abordar apenas um assunto.

Evitar:

- textos extremamente longos;
- mistura de assuntos;
- informações irrelevantes.

---

## Precisão

Nunca documentar algo baseado em suposição.

Toda informação deve possuir origem conhecida.

Exemplos:

- código-fonte original;
- testes executados;
- comportamento observado;
- documentação oficial;
- evidências registradas.

---

## Atualização Contínua

A documentação deve evoluir junto com o projeto.

Sempre que ocorrer:

- alteração de arquitetura;
- mudança de comportamento;
- correção importante;
- nova descoberta;
- refatoração significativa;

a documentação correspondente deverá ser atualizada.

---

## Rastreabilidade

Toda informação importante deve permitir identificar sua origem.

Sempre que possível registrar:

- arquivo analisado;
- classe;
- método;
- commit;
- decisão relacionada;
- sessão em que foi descoberta.

---

# Estrutura dos Documentos

Todos os documentos devem seguir uma estrutura semelhante.

Exemplo:

```text
Título

Objetivo

Escopo

Descrição

Detalhamento

Observações

Referências

Histórico
```

Nem todos os documentos precisarão possuir todas as seções, porém a organização deverá permanecer consistente.

---

# Estrutura dos Títulos

Utilizar sempre títulos hierárquicos.

```markdown
# Título principal

## Seção

### Subseção

#### Detalhe
```

Nunca utilizar níveis sem necessidade.

Evitar:

```markdown
##### Texto

###### Texto
```

---

# Organização do Conteúdo

Os documentos devem ser divididos em pequenas seções.

Evitar blocos enormes de texto.

Preferir:

- listas;
- tabelas;
- exemplos;
- diagramas;
- fluxos.

---

# Padrão para Listas

Listas simples:

```markdown
- Item
- Item
- Item
```

Listas numeradas:

```markdown
1. Passo
2. Passo
3. Passo
```

Nunca misturar numeração com marcadores na mesma sequência.

---

# Uso de Tabelas

Sempre que houver comparação de informações utilizar tabelas.

Exemplo:

| Item | Descrição |
|------|-----------|
| Classe | Classe responsável |
| Método | Método analisado |
| Status | Situação atual |

---

# Blocos de Código

Todo trecho de código deve utilizar markdown.

Exemplo:

````markdown
```java
public void conectar() {

}
```