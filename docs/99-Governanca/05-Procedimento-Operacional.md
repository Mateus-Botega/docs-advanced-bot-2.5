# 05 - Procedimento Operacional

## Objetivo

Este documento define o procedimento operacional oficial que deverá ser seguido durante todo o processo de reescrita do AdvancedBot de C# para Java.

O objetivo é garantir que todas as atividades sejam executadas de maneira padronizada, rastreável, reproduzível e documentada, evitando retrabalho, perda de funcionalidades e inconsistências entre a implementação original e a nova arquitetura.

Este documento representa o fluxo operacional diário da equipe durante a migração.

---

# Princípios Gerais

Toda implementação deverá seguir obrigatoriamente os princípios abaixo.

- Nunca desenvolver funcionalidades sem antes compreender completamente o código original.
- Nunca alterar comportamento durante a migração.
- Nunca modernizar regras de negócio durante a reescrita.
- Nunca remover funcionalidades existentes.
- Toda decisão técnica deve ser documentada.
- Todo código implementado deve possuir rastreabilidade.
- Todo componente implementado deve possuir checklist.
- Toda funcionalidade deve possuir validação.
- Todo comportamento deve ser comparado com o projeto original.

---

# Fluxo Oficial de Trabalho

Toda funcionalidade deverá seguir exatamente esta sequência.

```
Selecionar módulo

↓

Analisar documentação existente

↓

Analisar código C#

↓

Documentar comportamento

↓

Identificar dependências

↓

Definir arquitetura Java

↓

Implementar

↓

Testar

↓

Comparar comportamento

↓

Documentar evidências

↓

Atualizar checklist

↓

Concluir módulo
```

Nenhuma etapa poderá ser ignorada.

---

# Etapa 1 — Seleção da Funcionalidade

Antes de iniciar qualquer implementação deverá ser identificado:

- Nome do módulo.
- Localização no projeto original.
- Responsável.
- Prioridade.
- Dependências.
- Complexidade.
- Estado atual.

Exemplo:

```
Módulo:
Fishing

Prioridade:
Alta

Dependências:
MinecraftClient
Inventory
NetworkHandler

Status:
Não iniciado
```

---

# Etapa 2 — Leitura do Código Original

Antes de escrever qualquer linha em Java deve ser realizada uma análise completa do código C#.

Deve ser identificado:

- Classes envolvidas.
- Métodos.
- Fluxo.
- Eventos.
- Estados.
- Dependências.
- Objetos utilizados.
- Comunicação entre módulos.
- Tratamento de exceções.
- Fluxo de execução.

Tudo deve ser documentado.

---

# Etapa 3 — Documentação da Regra de Negócio

Antes da implementação deverá existir uma descrição textual contendo:

## Objetivo

O que o módulo faz.

## Quando é executado

Quando ocorre sua execução.

## Entradas

Quais informações recebe.

## Saídas

Quais informações produz.

## Dependências

Quais componentes utiliza.

## Fluxo

Descrição completa do funcionamento.

## Exceções

Situações especiais.

---

# Etapa 4 — Identificação das Dependências

Toda funcionalidade deverá possuir uma lista completa de dependências.

Exemplo

| Dependência | Tipo | Obrigatória | Situação |
|-------------|------|-------------|----------|
| MinecraftClient | Classe | Sim | OK |
| Inventory | Classe | Sim | OK |
| PacketHandler | Classe | Sim | OK |
| Timer | Classe | Sim | OK |

---

# Etapa 5 — Planejamento da Implementação

Antes da codificação deverá existir um pequeno planejamento contendo:

## Classes que serão criadas

-

-

-

## Interfaces

-

-

-

## Serviços

-

-

-

## DTOs

-

-

-

## Eventos

-

-

-

## Exceções

-

-

-

---

# Etapa 6 — Implementação

Durante a implementação deverão ser seguidas as regras definidas no Plano Mestre.

Itens obrigatórios:

- Código limpo.
- Nomes padronizados.
- Métodos pequenos.
- Responsabilidade única.
- Sem duplicação.
- Comentários apenas quando necessário.
- Organização por pacotes.
- Padrões definidos pelo projeto.

---

# Etapa 7 — Revisão Técnica

Após concluir a implementação deverá ser realizada uma revisão técnica verificando:

- Código compilando.
- Imports corretos.
- Dependências corretas.
- Arquitetura respeitada.
- Padrões seguidos.
- Nenhum código morto.
- Nenhum TODO pendente.
- Nenhum FIXME.

---

# Etapa 8 — Testes

Todos os comportamentos deverão ser testados.

Checklist mínimo:

- Fluxo principal
- Fluxos alternativos
- Tratamento de erros
- Casos extremos
- Performance
- Consumo de memória
- Concorrência
- Integração

Resultado:

```
[ ] Aprovado

[ ] Reprovado

Observações:

__________________________________
```

---

# Etapa 9 — Comparação com o Projeto Original

Todo módulo implementado deverá ser comparado com o código C#.

Itens para comparação:

- Fluxo.
- Estados.
- Eventos.
- Resultado final.
- Mensagens.
- Logs.
- Exceções.
- Performance.
- Comportamento.

Caso exista diferença deverá ser documentada.

---

# Etapa 10 — Registro das Evidências

Ao concluir a implementação deverão ser registradas evidências.

Exemplos:

- Prints.
- Logs.
- Testes.
- Comparações.
- Commits.
- Diagramas.
- Análises.

Todas as evidências deverão ficar registradas.

---

# Etapa 11 — Atualização da Documentação

Após concluir o módulo deverão ser atualizados obrigatoriamente:

- Estado Atual
- Roadmap
- Checklist Mestre
- Histórico de Alterações
- Arquitetura
- Documento específico do módulo

Nenhuma implementação poderá permanecer sem documentação.

---

# Etapa 12 — Encerramento

O módulo somente poderá ser considerado concluído quando:

- Código implementado.
- Código revisado.
- Testes executados.
- Comparação realizada.
- Checklist atualizado.
- Evidências registradas.
- Documentação atualizada.

Caso contrário o módulo permanecerá em andamento.

---

# Critérios de Qualidade

Todo módulo deverá atender aos seguintes critérios.

## Funcional

- Mesmo comportamento do C#
- Sem regressões
- Funcionalidade completa

## Arquitetural

- Organização correta
- Pacotes corretos
- Responsabilidade única

## Código

- Legível
- Padronizado
- Sem duplicação

## Documentação

- Atualizada
- Completa
- Rastreável

---

# Critérios para Aprovação

Uma implementação será considerada aprovada somente quando atender simultaneamente:

- Código compilando
- Todos os testes aprovados
- Mesmo comportamento do original
- Sem diferenças funcionais
- Documentação concluída
- Checklist atualizado
- Evidências registradas

---

# Tratamento de Pendências

Caso uma implementação não possa ser concluída deverá ser registrada uma pendência contendo:

## Identificador

Exemplo:

```
PEND-001
```

## Descrição

Descrição completa do problema.

## Impacto

Baixo

Médio

Alto

Crítico

## Responsável

Nome do responsável.

## Dependências

Lista de bloqueios.

## Próximos passos

Plano para resolução.

---

# Registro Diário de Trabalho

Cada sessão de desenvolvimento deverá possuir um registro.

Modelo:

| Data | Responsável | Módulo | Atividade | Resultado |
|------|-------------|---------|-----------|-----------|
| YYYY-MM-DD | | | | |

---

# Boas Práticas

Durante toda a migração recomenda-se:

- Trabalhar módulo por módulo.
- Evitar alterações paralelas.
- Manter commits pequenos.
- Documentar todas as decisões.
- Validar frequentemente.
- Atualizar o Roadmap.
- Atualizar o Checklist Mestre.
- Revisar antes de concluir.
- Priorizar fidelidade ao comportamento original.

---

# Não é Permitido

Durante a reescrita não será permitido:

- Alterar regras de negócio.
- Adicionar funcionalidades não planejadas.
- Remover funcionalidades existentes.
- Ignorar documentação.
- Ignorar testes.
- Ignorar checklist.
- Ignorar revisão técnica.
- Mesclar módulos diferentes em uma mesma implementação.
- Concluir atividades sem evidências.

---

# Referências

Este procedimento operacional deve ser utilizado em conjunto com os seguintes documentos:

- 00-README.md
- 01-Plano-Mestre-Reescrita.md
- 02-Estado-Atual.md
- 03-Roadmap.md
- 04-Checklist-Mestre.md
- 06-Arquitetura.md
- 07-Padrões-de-Codificação.md
- Documentação específica de cada módulo

---

# Histórico de Alterações

| Versão | Data | Autor | Alterações |
|---------|------|--------|------------|
| 1.0 | YYYY-MM-DD | | Criação do Procedimento Operacional |