# 09 - Métricas do Projeto

## Objetivo

Este documento define todas as métricas utilizadas para acompanhar a evolução da reescrita completa do AdvancedBot de C# para Java.

Seu objetivo é permitir o acompanhamento quantitativo e qualitativo do projeto durante todas as fases da migração, garantindo que o progresso possa ser medido de forma objetiva.

As métricas devem ser atualizadas continuamente ao longo do projeto.

---

# Objetivos das Métricas

As métricas existem para responder perguntas como:

- Quanto do projeto já foi analisado?
- Quanto já foi documentado?
- Quanto já foi reescrito?
- Quanto já foi validado?
- Quanto ainda falta?
- Qual é o nível de risco atual?
- Existem gargalos?
- O projeto está evoluindo conforme planejado?

---

# Frequência de Atualização

As métricas devem ser atualizadas:

- ao finalizar uma sessão de trabalho;
- ao concluir uma etapa importante;
- antes de iniciar uma nova fase;
- sempre que um módulo mudar de status.

Nunca deixar métricas desatualizadas.

---

# Organização

O documento deverá possuir as seguintes seções.

```
1. Visão Geral
2. Indicadores Gerais
3. Métricas de Documentação
4. Métricas de Análise
5. Métricas de Reescrita
6. Métricas de Testes
7. Métricas de Cobertura
8. Métricas de Qualidade
9. Métricas de Risco
10. Métricas de Pendências
11. Evolução Histórica
12. Dashboard Geral
```

---

# 1. Visão Geral

Apresenta um resumo executivo do projeto.

Exemplo:

- Data da atualização
- Responsável
- Etapa atual
- Fase atual
- Percentual geral
- Situação geral

Exemplo:

| Item | Valor |
|------|-------|
| Última atualização | |
| Responsável | |
| Etapa atual | |
| Situação | |
| Percentual Geral | |

---

# 2. Indicadores Gerais

Resumo de alto nível.

Indicadores obrigatórios:

- Total de módulos
- Módulos analisados
- Módulos documentados
- Módulos reescritos
- Módulos testados
- Módulos homologados

Exemplo

| Indicador | Valor |
|-----------|------:|
| Total de módulos | |
| Analisados | |
| Documentados | |
| Reescritos | |
| Testados | |
| Homologados | |

---

# 3. Métricas de Documentação

Controla a documentação produzida.

Indicadores:

- Documentos previstos
- Documentos concluídos
- Documentos em andamento
- Documentos pendentes

Exemplo

| Documento | Status |
|------------|--------|
| README | |
| Plano Mestre | |
| Estado Atual | |
| Roadmap | |
| Checklist | |
| Procedimento Operacional | |
| Definition of Done | |
| Controle de Decisões | |
| Registro de Sessões | |
| Métricas | |

---

# Percentual da documentação

Sempre calcular.

```
Documentação =

(Documentos concluídos /
Documentos previstos)

x100
```

---

# 4. Métricas de Análise

Mede a evolução da engenharia reversa.

Indicadores:

- Classes analisadas
- Interfaces analisadas
- Enums analisados
- Estruturas analisadas
- Eventos analisados
- Protocolos analisados
- Pacotes analisados

Tabela

| Item | Total | Concluído | % |
|------|-------|-----------|---|
| Classes | | | |
| Interfaces | | | |
| Enums | | | |
| Eventos | | | |
| Protocolos | | | |

---

# 5. Métricas de Reescrita

Controla a implementação em Java.

Indicadores:

- Classes implementadas
- Interfaces implementadas
- DTOs implementados
- Serviços implementados
- Utilitários implementados
- Eventos implementados

Tabela

| Item | Total | Feito | % |
|------|-------|-------|---|
| Classes | | | |
| DTOs | | | |
| Serviços | | | |
| Eventos | | | |

---

# Percentual da Reescrita

```
Implementação

=

Itens implementados

/

Itens previstos

x100
```

---

# 6. Métricas de Testes

Avalia a validação funcional.

Indicadores

- Casos de teste previstos
- Casos executados
- Casos aprovados
- Casos reprovados
- Casos bloqueados

Tabela

| Indicador | Valor |
|-----------|------:|
| Testes previstos | |
| Executados | |
| Aprovados | |
| Reprovados | |
| Bloqueados | |

---

# Taxa de aprovação

```
Aprovação

=

Testes aprovados

/

Testes executados

x100
```

---

# 7. Métricas de Cobertura

Avalia a cobertura da migração.

Itens medidos

- Funcionalidades
- Classes
- Pacotes
- Eventos
- Protocolos
- Comandos
- Interfaces gráficas

Tabela

| Item | Cobertura |
|------|-----------|
| Funcionalidades | |
| Classes | |
| Eventos | |
| Protocolos | |
| UI | |

---

# 8. Métricas de Qualidade

Avalia a qualidade da implementação.

Indicadores

- Bugs encontrados
- Bugs corrigidos
- Bugs pendentes
- Refatorações realizadas
- Refatorações pendentes

Tabela

| Indicador | Valor |
|-----------|------:|
| Bugs encontrados | |
| Corrigidos | |
| Pendentes | |
| Refatorações | |

---

# Índice de Correção

```
Correção

=

Bugs corrigidos

/

Bugs encontrados

x100
```

---

# 9. Métricas de Risco

Avalia o risco atual do projeto.

Categorias

- Técnico
- Arquitetural
- Dependências
- Performance
- Comunicação
- Escopo

Tabela

| Categoria | Nível | Tendência |
|-----------|--------|-----------|
| Técnico | | |
| Arquitetura | | |
| Dependências | | |
| Performance | | |
| Escopo | | |

---

# Escala de risco

Sempre utilizar:

```
Muito Baixo

Baixo

Médio

Alto

Crítico
```

---

# Tendência

Sempre informar.

Valores possíveis

```
Melhorando

Estável

Piorando
```

---

# 10. Métricas de Pendências

Controla o backlog restante.

Indicadores

- Pendências abertas
- Em andamento
- Resolvidas
- Canceladas

Tabela

| Situação | Quantidade |
|-----------|-----------:|
| Abertas | |
| Em andamento | |
| Resolvidas | |
| Canceladas | |

---

# 11. Evolução Histórica

Registrar a evolução do projeto.

Nunca apagar registros antigos.

Exemplo

| Data | Documentação | Análise | Reescrita | Testes | Total |
|------|--------------|----------|-----------|---------|-------|
| | | | | | |

---

# 12. Dashboard Geral

Resumo executivo.

Modelo

```
██████████░░░░░░░░░░

Documentação

████████████░░░░░░

Análise

███████░░░░░░░░░░░

Reescrita

████░░░░░░░░░░░░░░

Testes

██░░░░░░░░░░░░░░░░

Homologação
```

Também apresentar em porcentagem.

Exemplo

```
Documentação.....45%

Análise..........32%

Reescrita........18%

Testes...........5%

Homologação......0%
```

---

# Indicadores Obrigatórios

O documento deverá acompanhar, no mínimo:

- Percentual de documentação
- Percentual de análise
- Percentual de implementação
- Percentual de testes
- Percentual de homologação
- Quantidade de bugs
- Quantidade de riscos
- Quantidade de pendências
- Cobertura funcional
- Cobertura técnica
- Evolução semanal
- Evolução mensal

---

# Regras Gerais

As métricas devem:

- ser objetivas;
- possuir origem verificável;
- ser atualizadas continuamente;
- refletir a situação real do projeto;
- nunca conter estimativas sem identificação;
- utilizar a mesma unidade de medida durante todo o projeto;
- permitir comparação histórica.

---

# Critérios de Atualização

Sempre atualizar as métricas quando ocorrer:

- conclusão de documentação;
- conclusão de análise;
- implementação de funcionalidades;
- alteração de arquitetura;
- descoberta de novos riscos;
- execução de testes;
- correção de bugs;
- mudança de escopo;
- encerramento de uma sessão de trabalho.

---

# Boas Práticas

- Nunca apagar históricos.
- Nunca alterar métricas antigas sem justificativa.
- Registrar a data de cada atualização.
- Utilizar tabelas padronizadas.
- Manter percentuais consistentes.
- Atualizar o dashboard após qualquer alteração significativa.
- Validar os números antes de registrar.
- Utilizar as métricas como base para tomada de decisão durante toda a reescrita do projeto.