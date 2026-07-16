# 19 - Política de Atualização

---

# Objetivo

Este documento define a política oficial de atualização de toda a documentação, artefatos técnicos, código migrado e registros produzidos durante o projeto de migração do AdvancedBot de C# para Java.

Seu objetivo é garantir que toda alteração realizada permaneça sincronizada entre código, documentação, diagramas e histórico do projeto, evitando divergências ao longo do desenvolvimento.

Esta política possui caráter obrigatório para qualquer manutenção realizada durante o projeto.

---

# Princípios Gerais

Toda atualização deve obedecer aos seguintes princípios:

- Consistência
- Rastreabilidade
- Reprodutibilidade
- Histórico preservado
- Documentação sincronizada
- Atualização incremental
- Nenhuma informação contraditória

Toda alteração deve ser considerada parte integrante da documentação do projeto.

Nunca existe alteração apenas no código.

Toda alteração gera atualização documental correspondente.

---

# Regra Fundamental

Sempre que qualquer item do projeto sofrer alteração, todos os documentos relacionados devem ser revisados.

Nunca considerar uma alteração concluída apenas modificando o código.

Uma alteração somente é considerada finalizada quando:

- código atualizado;
- documentação atualizada;
- diagramas revisados;
- matriz de rastreabilidade atualizada;
- histórico registrado;
- métricas ajustadas;
- decisões registradas quando necessário.

---

# Política de Atualização Contínua

A documentação deve evoluir junto com o projeto.

É proibido deixar documentação para atualização futura.

Sempre atualizar imediatamente após concluir uma alteração.

Jamais acumular alterações documentais.

---

# Ordem Obrigatória de Atualização

Sempre seguir a seguinte sequência:

1. Implementação
2. Testes
3. Revisão técnica
4. Atualização da documentação
5. Atualização dos diagramas
6. Atualização das métricas
7. Atualização da rastreabilidade
8. Registro da sessão
9. Registro de decisões (quando aplicável)
10. Revisão final

---

# Atualização do Código

Sempre atualizar:

- comentários
- documentação JavaDoc
- TODOs
- FIXME
- observações
- anotações de migração

Nunca deixar comentários incompatíveis com o código.

---

# Atualização dos Diagramas

Sempre revisar:

- Mermaid
- Fluxogramas
- Diagramas de sequência
- Diagramas de classes
- Diagramas de estados
- Diagramas de componentes

Sempre verificar se representam exatamente o estado atual do projeto.

---

# Atualização da Documentação Técnica

Sempre revisar:

- arquitetura
- estrutura de pacotes
- classes
- interfaces
- DTOs
- entidades
- serviços
- casos especiais
- limitações

---

# Atualização dos Guias

Sempre atualizar quando necessário:

- Guia de nomenclatura
- Guia de documentação
- Fluxo de migração
- Boas práticas
- Protocolo de IA
- Métricas
- Registro de sessões
- Controle de decisões

---

# Atualização da Matriz de Rastreabilidade

Toda alteração funcional deve refletir na matriz.

Atualizar:

- requisito
- origem
- arquivo
- classe
- método
- status
- observações

Nunca deixar requisitos sem vínculo.

---

# Atualização das Métricas

Sempre atualizar indicadores quando houver alteração.

Exemplos:

- quantidade de arquivos migrados
- quantidade de classes
- quantidade de métodos
- cobertura
- percentual concluído
- módulos restantes
- pendências

---

# Atualização do Registro de Sessões

Toda sessão deve registrar:

- data
- objetivo
- alterações
- arquivos modificados
- decisões
- problemas encontrados
- soluções
- próximos passos

---

# Atualização do Controle de Decisões

Registrar sempre que ocorrer:

- mudança arquitetural
- alteração de padrão
- nova convenção
- exceção adotada
- alteração estrutural

Cada decisão deve possuir:

- identificador
- data
- descrição
- justificativa
- impacto

---

# Atualização dos Diagramas Mermaid

Sempre validar:

- sintaxe
- renderização
- compatibilidade
- organização
- legibilidade

Nunca manter diagramas quebrados.

---

# Atualização dos Arquivos Markdown

Sempre revisar:

- títulos
- índices
- links internos
- links relativos
- referências cruzadas
- exemplos
- tabelas
- listas

---

# Atualização das Referências Cruzadas

Sempre verificar se documentos relacionados permanecem consistentes.

Exemplo:

Guia de documentação

↓

Fluxo de migração

↓

Boas práticas

↓

Matriz de rastreabilidade

↓

Diagramas

↓

Registro de sessões

Toda referência deve apontar para conteúdo existente.

---

# Atualização dos Exemplos

Sempre que alterar uma regra:

atualizar também os exemplos.

Nunca deixar exemplos incompatíveis com o padrão.

---

# Atualização das Convenções

Sempre que surgir nova convenção:

- registrar
- justificar
- documentar
- divulgar
- aplicar em todo o projeto

---

# Atualização das Regras de IA

Sempre atualizar quando houver:

- novo prompt
- novo fluxo
- nova instrução
- novo padrão
- melhoria operacional

---

# Atualização da Estrutura de Pastas

Quando houver reorganização:

atualizar:

- diagramas
- documentação
- índices
- referências
- árvore de diretórios

---

# Atualização da Arquitetura

Sempre documentar:

- novos módulos
- novos componentes
- novas integrações
- remoções
- substituições
- refatorações

---

# Atualização de Casos Especiais

Toda exceção deve possuir documentação.

Informar:

- motivo
- impacto
- solução
- alternativas
- limitações

---

# Atualização de Dependências

Sempre registrar:

- biblioteca adicionada
- biblioteca removida
- versão
- motivo
- impacto

---

# Atualização de Compatibilidade

Sempre informar:

- versão Java
- versão Spring
- bibliotecas
- compatibilidade
- requisitos mínimos

---

# Atualização de Limitações

Sempre registrar:

- funcionalidades pendentes
- limitações conhecidas
- diferenças em relação ao C#
- riscos
- melhorias futuras

---

# Atualização de Histórico

Nunca apagar histórico.

Sempre acrescentar novos registros.

Histórico deve permanecer cronológico.

---

# Política de Versionamento

Toda documentação deve possuir:

- versão
- data
- responsável (quando aplicável)
- resumo da alteração

Exemplo:

| Versão | Data | Alteração |
|---------|------|-----------|
| 1.0 | 2026-07-14 | Documento inicial |
| 1.1 | AAAA-MM-DD | Inclusão de novas regras |
| 1.2 | AAAA-MM-DD | Atualização de convenções |

---

# Política de Revisão

Antes de considerar uma atualização concluída verificar:

- documentação consistente
- diagramas atualizados
- exemplos corretos
- links funcionando
- índices atualizados
- rastreabilidade completa
- histórico registrado

---

# Checklist Obrigatório de Atualização

## Código

- [ ] Código atualizado
- [ ] Comentários revisados
- [ ] JavaDoc atualizado

## Documentação

- [ ] Markdown atualizado
- [ ] Índices revisados
- [ ] Links revisados

## Diagramas

- [ ] Mermaid atualizado
- [ ] Fluxogramas atualizados
- [ ] Diagramas renderizando

## Arquitetura

- [ ] Componentes revisados
- [ ] Estrutura atualizada

## Métricas

- [ ] Indicadores atualizados
- [ ] Percentuais recalculados

## Rastreabilidade

- [ ] Matriz atualizada
- [ ] Requisitos vinculados

## Histórico

- [ ] Sessão registrada
- [ ] Decisões registradas

## Revisão Final

- [ ] Nenhuma inconsistência encontrada
- [ ] Documentação sincronizada
- [ ] Projeto consistente

---

# Política de Auditoria

Periodicamente realizar auditoria completa verificando:

- consistência entre documentos
- documentação desatualizada
- diagramas incompatíveis
- exemplos inválidos
- arquivos órfãos
- referências quebradas
- nomenclatura padronizada
- rastreabilidade completa

Toda inconsistência encontrada deve ser corrigida imediatamente.

---

# Política de Conclusão

Uma atividade somente poderá ser considerada concluída quando:

- implementação finalizada;
- testes concluídos;
- documentação atualizada;
- diagramas revisados;
- métricas atualizadas;
- rastreabilidade completa;
- histórico registrado;
- decisões documentadas (quando aplicável);
- revisão final aprovada.

Nenhuma entrega é considerada finalizada enquanto existir qualquer divergência entre código, documentação, diagramas ou registros do projeto.