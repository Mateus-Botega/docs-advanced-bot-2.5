# 10 - Protocolo de Uso com IA

## Objetivo

Este documento define o protocolo oficial para utilização de Inteligências Artificiais durante todo o processo de engenharia reversa, documentação, arquitetura e reescrita do AdvancedBot em Java.

O objetivo é garantir que a IA seja utilizada como ferramenta de produtividade, mantendo consistência técnica, rastreabilidade, qualidade do código e preservação da lógica original.

A IA nunca substitui a análise técnica humana.

---

# Princípios Gerais

Toda utilização de IA deve obedecer aos seguintes princípios.

## 1. Preservação da lógica original

A IA nunca poderá modificar regras de negócio.

Ela apenas poderá:

- explicar
- documentar
- organizar
- traduzir
- sugerir melhorias
- converter sintaxe

A lógica original sempre possui prioridade.

---

## 2. Zero invenções

A IA não pode:

- inventar classes
- inventar métodos
- inventar algoritmos
- assumir comportamentos
- completar código ausente utilizando criatividade

Caso alguma informação não exista, deverá registrar:

> Informação não encontrada no código fonte.

---

## 3. Engenharia Reversa Primeiro

Toda análise deve seguir obrigatoriamente:

Código Original

↓

Entendimento

↓

Documentação

↓

Modelagem

↓

Planejamento

↓

Implementação Java

Nunca implementar antes de compreender.

---

## 4. Um componente por vez

Nunca solicitar para IA converter grandes partes do projeto.

Sempre trabalhar em pequenas unidades.

Exemplo:

✔ Classe

✔ Método

✔ Enum

✔ Interface

✔ Pacote

✔ Fluxo específico

Nunca:

❌ Projeto inteiro

❌ Módulo inteiro

❌ Milhares de linhas

---

# Fluxo Oficial de Trabalho

Cada sessão deve seguir exatamente a sequência abaixo.

## Etapa 1

Selecionar componente.

---

## Etapa 2

Localizar dependências.

---

## Etapa 3

Compreender funcionamento.

---

## Etapa 4

Documentar.

---

## Etapa 5

Validar entendimento.

---

## Etapa 6

Criar arquitetura Java.

---

## Etapa 7

Implementar.

---

## Etapa 8

Testar.

---

## Etapa 9

Registrar métricas.

---

## Etapa 10

Atualizar documentação.

---

# Tipos de Solicitações Permitidas

A IA pode ser utilizada para:

## Engenharia Reversa

- explicar código
- identificar responsabilidades
- encontrar dependências
- mapear chamadas
- localizar referências

---

## Documentação

Criar:

- Markdown
- Diagramas
- Fluxogramas
- Tabelas
- Roadmaps
- Checklists

---

## Arquitetura

Auxiliar em:

- Design Patterns
- Clean Architecture
- SOLID
- DDD (quando aplicável)
- Modularização

---

## Conversão de Linguagem

Converter:

C#

↓

Java

Preservando:

- comportamento
- regras
- nomenclaturas
- ordem lógica

---

## Revisões

Revisar:

- documentação
- código
- consistência
- duplicações
- nomenclaturas

---

## Explicações

Explicar:

- APIs
- bibliotecas
- algoritmos
- protocolos
- estruturas

---

# Solicitações Proibidas

Nunca solicitar:

## Reescrever todo projeto

Exemplo:

> Reescreva todo o AdvancedBot.

Proibido.

---

## Inventar código

Exemplo:

> Complete esse método.

Sem possuir o original.

Proibido.

---

## Alterar regras

Exemplo:

Modificar funcionamento para ficar "melhor".

Proibido.

---

## Otimizações sem validação

Toda otimização precisa ser aprovada.

Nunca alterar comportamento automaticamente.

---

## Remover código

Nenhuma remoção pode ocorrer sem análise.

---

## Alterar arquitetura durante migração

Primeiro:

Migrar.

Depois:

Refatorar.

---

# Padrão de Prompts

Todos os prompts devem seguir uma estrutura padronizada.

---

## Contexto

Descrever:

- objetivo
- módulo
- componente

---

## Escopo

Explicar exatamente o que será analisado.

---

## Restrições

Exemplo:

- Não inventar código
- Preservar lógica
- Manter nomenclatura
- Não alterar comportamento

---

## Resultado esperado

Definir claramente.

Exemplo:

- documentação
- análise
- diagrama
- conversão
- revisão

---

# Modelo de Prompt

```text
Contexto:

Estamos realizando engenharia reversa do AdvancedBot em C# para Java.

Objetivo:

Analisar exclusivamente a classe X.

Restrições:

- Não inventar código.
- Não alterar lógica.
- Apenas explicar.
- Documentar tudo.

Resultado esperado:

Gerar documentação técnica detalhada em Markdown.
```

---

# Regras para Conversão C# → Java

Sempre preservar:

- lógica
- sequência
- regras
- nomes importantes
- responsabilidades

Jamais converter utilizando criatividade.

---

## Conversão permitida

Sintaxe.

---

## Conversão proibida

Mudança de comportamento.

---

# Uso para Documentação

A IA deverá produzir documentação contendo:

Objetivo

Descrição

Fluxo

Dependências

Entradas

Saídas

Exceções

Observações

Pendências

Referências

---

# Uso para Engenharia Reversa

Sempre responder:

O que faz?

Como faz?

Quem chama?

Quem depende?

Quais eventos dispara?

Quais estados altera?

Quais arquivos utiliza?

Quais objetos modifica?

---

# Uso para Arquitetura

Antes de sugerir arquitetura:

Compreender totalmente o código.

Nunca aplicar Design Patterns desnecessariamente.

Sempre justificar.

---

# Uso para Refatoração

Refatorações somente serão permitidas após:

- migração concluída
- testes aprovados
- documentação completa

---

# Uso para Testes

A IA poderá auxiliar na criação de:

- testes unitários
- testes de integração
- cenários
- casos de teste
- mocks
- validações

Sempre baseados na lógica original.

---

# Registro Obrigatório das Sessões

Toda sessão utilizando IA deverá registrar:

Data

Objetivo

Componente

Prompt utilizado

Resultado

Pendências

Decisões

Arquivos alterados

Tempo gasto

---

# Checklist Antes de Perguntar à IA

Antes de iniciar uma nova interação, verificar:

- O objetivo está claro?
- O componente foi definido?
- As restrições foram informadas?
- Existe documentação suficiente?
- O escopo está limitado?
- O resultado esperado foi definido?

Somente após todas as respostas positivas a solicitação deverá ser enviada.

---

# Checklist Após Receber a Resposta

Verificar:

- A resposta respeitou o escopo?
- Houve invenção de informações?
- A lógica foi preservada?
- Existem inconsistências?
- Há necessidade de validação manual?
- A documentação foi atualizada?
- O checklist mestre foi atualizado?
- As métricas foram registradas?
- As decisões foram documentadas?

---

# Boas Práticas

- Trabalhar em sessões curtas.
- Documentar imediatamente após cada análise.
- Validar todas as respostas.
- Utilizar prompts objetivos.
- Manter rastreabilidade completa.
- Registrar todas as decisões.
- Revisar antes de implementar.
- Priorizar clareza em vez de velocidade.
- Utilizar linguagem técnica consistente.
- Evitar múltiplos objetivos na mesma solicitação.

---

# Métricas de Qualidade das Interações

Cada interação poderá ser avaliada pelos seguintes critérios:

| Critério | Objetivo |
|----------|----------|
| Clareza do Prompt | Solicitação objetiva e completa |
| Precisão da Resposta | Responde exatamente ao solicitado |
| Fidelidade ao Código Original | Nenhuma alteração indevida da lógica |
| Rastreabilidade | Permite identificar origem das informações |
| Reutilização | Resultado pode ser reutilizado em outras etapas |
| Consistência | Alinhamento com os padrões do projeto |
| Completude | Cobertura integral do escopo definido |

---

# Controle de Versões dos Prompts

Sempre que um prompt for reutilizado ou evoluído, registrar:

- Identificador do prompt.
- Objetivo.
- Data de criação.
- Data da última atualização.
- Alterações realizadas.
- Sessões em que foi utilizado.

Isso garante repetibilidade e melhoria contínua das interações com IA.

---

# Critérios de Conclusão

O protocolo será considerado corretamente seguido quando:

- Todas as interações utilizarem prompts padronizados.
- Nenhuma resposta da IA introduzir lógica inexistente.
- Toda documentação gerada estiver vinculada ao componente analisado.
- Todas as decisões forem registradas.
- As métricas forem atualizadas ao final de cada sessão.
- Houver rastreabilidade completa entre prompts, respostas e artefatos produzidos.
- O processo de migração permanecer fiel ao comportamento do sistema original.

---

# Histórico de Alterações

| Versão | Data | Autor | Alterações |
|---------|------|--------|------------|
| 1.0 | AAAA-MM-DD | Equipe do Projeto | Criação do Protocolo de Uso com IA |