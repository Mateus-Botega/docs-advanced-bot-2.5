# 15 - Boas Práticas

> Documento responsável por definir todas as boas práticas obrigatórias do projeto de migração do AdvancedBot C# para Java. Este documento complementa todos os demais padrões e deve ser seguido durante toda a execução da migração.

---

# Objetivo

Garantir que toda a migração seja realizada de forma:

- consistente;
- previsível;
- rastreável;
- organizada;
- auditável;
- segura;
- incremental;
- sem perda de comportamento.

O objetivo deste documento é padronizar decisões técnicas para evitar retrabalho e divergências entre diferentes sessões de desenvolvimento.

---

# Princípios Fundamentais

Todo o projeto deve seguir os seguintes princípios.

## 1. Preservação do comportamento

A prioridade máxima da migração é preservar exatamente o comportamento existente.

Sempre que houver conflito entre:

- arquitetura bonita
- comportamento original

vence o comportamento original.

Nunca alterar regras de negócio durante a migração.

---

## 2. Migração incremental

Nunca migrar grandes blocos de código de uma única vez.

Sempre trabalhar em pequenas unidades.

Exemplos:

- uma classe
- um método
- uma funcionalidade
- um comando
- um parser
- um pacote

---

## 3. Código compilando continuamente

O projeto deve permanecer compilável o máximo possível.

Evitar deixar centenas de erros acumulados.

Sempre finalizar uma etapa antes de iniciar outra.

---

## 4. Pequenos commits

Cada alteração deve representar apenas uma responsabilidade.

Exemplos:

✔ converter PacketLogin

✔ converter ChatParser

✔ converter Inventory

Evitar commits gigantes contendo dezenas de mudanças.

---

## 5. Um problema por vez

Caso existam:

- erros de compilação
- erros de arquitetura
- erros de lógica

Resolver nesta ordem:

1. compilação
2. arquitetura
3. lógica
4. otimizações

Nunca otimizar código quebrado.

---

# Organização do Código

## Estrutura de pacotes

Sempre respeitar a arquitetura definida.

Exemplo:

```
network
commands
inventory
world
entity
pathfinding
protocol
util
gui
configuration
```

Evitar criar pacotes duplicados.

---

## Nomeação consistente

Sempre utilizar:

- PascalCase para classes
- camelCase para métodos
- camelCase para variáveis
- UPPER_CASE para constantes

Nunca misturar padrões.

---

## Um arquivo por classe

Evitar múltiplas classes públicas no mesmo arquivo.

Cada classe deve possuir seu próprio arquivo.

---

## Responsabilidade única

Cada classe deve possuir apenas uma responsabilidade.

Evitar classes gigantes.

Se necessário:

dividir em:

- Service
- Factory
- Builder
- Mapper
- Parser
- Util

---

# Conversão de Código

## Converter antes de refatorar

Primeiro:

copiar comportamento.

Depois:

melhorar arquitetura.

Nunca fazer ambos simultaneamente.

---

## Evitar reescrever lógica

Se existe uma lógica funcionando em C#:

converter.

Não reinventar.

---

## Evitar simplificações perigosas

Não remover:

- verificações
- ifs
- validações
- estados
- tratamentos

Mesmo que pareçam redundantes.

---

## Manter nomes conhecidos

Sempre que possível manter nomes semelhantes ao projeto original.

Exemplo:

```
MoveTo()

↓

moveTo()
```

Evitar renomeações desnecessárias.

---

# Tratamento de Erros

## Nunca ignorar exceções

Evitar:

```
catch(Exception e){}
```

Sempre:

- registrar
- documentar
- justificar

---

## Mensagens claras

Logs devem informar:

- classe
- método
- operação
- causa

Exemplo:

```
Erro ao converter pacote LoginPacket
```

e não apenas:

```
Erro
```

---

## Não esconder falhas

Falhas críticas devem aparecer.

Nunca mascarar problemas importantes.

---

# Comentários

## Comentar apenas quando necessário

O código deve ser autoexplicativo.

Comentários apenas para:

- decisões complexas
- regras antigas
- compatibilidade
- limitações

---

## Nunca comentar código morto

Evitar:

```java
// antigo código
// if (...)
// ...
```

Utilizar controle de versão.

---

# Uso da IA

## A IA não toma decisões arquiteturais

Toda decisão importante deve ser validada.

A IA apenas auxilia.

---

## Sempre fornecer contexto

Ao solicitar conversões:

informar:

- classe
- dependências
- objetivo
- arquivos relacionados

---

## Não converter arquivos isoladamente

Sempre considerar:

- chamadas
- dependências
- interfaces
- herança

---

# Controle de Qualidade

Antes de finalizar qualquer etapa verificar:

- compila
- mantém comportamento
- segue arquitetura
- segue nomenclatura
- segue documentação

---

# Performance

Performance NÃO é prioridade durante a primeira migração.

Prioridade:

1 comportamento

2 compatibilidade

3 estabilidade

4 organização

5 performance

---

# Refatorações

Refatorações devem ocorrer apenas quando:

- comportamento estiver preservado
- testes estiverem passando
- classe estiver totalmente migrada

Nunca durante a conversão inicial.

---

# Dependências

Antes de adicionar uma nova biblioteca perguntar:

Existe solução usando Java padrão?

Se sim:

preferir Java padrão.

Evitar dependências desnecessárias.

---

# Reutilização

Sempre reutilizar componentes existentes.

Evitar duplicação.

Antes de criar uma nova classe verificar se já existe equivalente.

---

# Métodos

Preferir métodos pequenos.

Objetivo recomendado:

20~40 linhas.

Caso ultrapasse muito esse tamanho:

avaliar extração.

---

# Constantes

Nunca utilizar números mágicos.

Substituir por constantes nomeadas.

Exemplo:

```
public static final int MAX_PACKET_SIZE = 8192;
```

---

# Configurações

Toda configuração deve ser centralizada.

Evitar valores espalhados.

---

# Testabilidade

Sempre escrever código que possa ser testado.

Evitar acoplamentos desnecessários.

---

# Compatibilidade

Sempre preservar compatibilidade com:

- protocolo Minecraft
- pacotes
- formatos
- arquivos
- configurações

---

# Leitura de Código

Priorizar código legível.

Código claro vale mais do que código extremamente compacto.

---

# Revisão

Antes de concluir qualquer funcionalidade revisar:

- imports
- warnings
- TODOs
- FIXME
- comentários
- duplicações

---

# Checklist de Boas Práticas

Antes de finalizar uma tarefa confirmar:

- [ ] comportamento preservado
- [ ] nomenclatura correta
- [ ] arquitetura respeitada
- [ ] documentação atualizada
- [ ] sem código morto
- [ ] sem TODO esquecidos
- [ ] sem warnings importantes
- [ ] sem duplicações desnecessárias
- [ ] tratamento de erro adequado
- [ ] logs claros
- [ ] dependências justificadas
- [ ] classe compilando
- [ ] testes executados (quando existirem)
- [ ] rastreabilidade atualizada
- [ ] decisão registrada quando necessário

---

# Antipadrões Proibidos

Nunca realizar:

- alterar regra de negócio durante a migração
- otimizar prematuramente
- reescrever funcionalidades completas sem necessidade
- remover validações existentes
- ignorar exceções
- copiar código duplicado
- criar classes gigantes
- criar dependências circulares
- quebrar compatibilidade
- deixar código sem documentação quando exigida
- alterar arquitetura sem registrar decisão
- finalizar sessão com arquivos inconsistentes

---

# Fluxo Recomendado para Cada Classe

Para cada classe seguir obrigatoriamente a sequência:

1. analisar a classe original
2. identificar dependências
3. registrar rastreabilidade
4. converter estrutura
5. converter atributos
6. converter construtores
7. converter métodos
8. corrigir compilação
9. validar comportamento
10. documentar diferenças
11. atualizar checklist
12. registrar conclusão da sessão

---

# Critérios de Excelência

Uma migração é considerada excelente quando:

- preserva integralmente o comportamento original;
- possui rastreabilidade completa;
- segue todos os padrões documentados;
- mantém arquitetura consistente;
- possui documentação atualizada;
- apresenta código limpo e legível;
- facilita futuras evoluções sem comprometer a compatibilidade.