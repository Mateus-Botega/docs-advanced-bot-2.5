# 13 - Matriz de Rastreabilidade

## Objetivo

A Matriz de Rastreabilidade é o documento responsável por garantir que **todos os elementos do projeto possam ser relacionados entre si**, permitindo identificar facilmente:

- Origem de cada funcionalidade.
- Arquivos analisados.
- Classes migradas.
- Documentação correspondente.
- Decisões tomadas.
- Evidências coletadas.
- Testes realizados.
- Status da migração.

Seu principal objetivo é assegurar que nenhuma funcionalidade do projeto original seja perdida durante a migração do AdvancedBot de C# para Java.

---

# Objetivos da Matriz

A matriz deverá permitir responder rapidamente perguntas como:

- Esta classe já foi analisada?
- Esta funcionalidade já foi migrada?
- Existe documentação desta classe?
- Existe decisão arquitetural registrada?
- Existem testes realizados?
- Existe pendência?
- Quem analisou?
- Quando foi concluída?
- Onde está a implementação Java?
- Qual arquivo C# originou esta implementação?

---

# Princípios

Toda informação registrada deve ser:

- objetiva;
- verificável;
- atualizada;
- consistente;
- rastreável;
- única.

Nunca deve existir uma funcionalidade sem rastreamento.

---

# Estrutura da Matriz

Cada linha representa um item rastreável.

Exemplo:

| ID | Tipo | Origem | Destino | Documentação | Testes | Status |
|----|------|---------|----------|--------------|---------|---------|

---

# Identificador (ID)

Cada item deverá possuir um identificador único.

Exemplos:

```
CLS-001
CLS-002
PKT-015
INV-023
BOT-044
CMD-010
GUI-007
NET-012
```

Nunca reutilizar IDs.

Nunca alterar IDs após criados.

---

# Tipo

Identifica qual elemento está sendo rastreado.

Exemplos:

- Classe
- Interface
- Enum
- Packet
- Comando
- Macro
- Tela
- Sistema
- Thread
- Serviço
- Evento
- Utilitário
- Configuração
- Arquivo
- Biblioteca
- Recurso

---

# Origem

Indica o elemento existente no projeto C#.

Exemplos:

```
Inventory.cs
MinecraftClient.cs
CommandMob.cs
MacroMineracao.cs
PacketReader.cs
```

Sempre utilizar o caminho completo quando possível.

Exemplo:

```
AdvancedBot\Client\Inventory.cs
```

---

# Destino

Indica a implementação correspondente no projeto Java.

Exemplo:

```
br.com.advancedbot.inventory.Inventory
```

ou

```
src/main/java/...
```

---

# Funcionalidade

Descrição resumida do comportamento implementado.

Exemplo:

```
Gerenciamento do inventário.

Controle dos slots.

Movimentação de itens.

Atualização dos pacotes.
```

Evitar descrições genéricas.

---

# Responsável

Indicar quem realizou:

- análise;
- migração;
- revisão.

---

# Status

Utilizar exclusivamente os status padronizados do projeto.

Exemplo:

| Status | Significado |
|----------|------------|
| Não iniciado | Ainda não analisado |
| Em análise | Em estudo |
| Documentado | Documentação concluída |
| Em desenvolvimento | Migração iniciada |
| Implementado | Código desenvolvido |
| Em revisão | Revisão técnica |
| Testado | Testes concluídos |
| Validado | Equivalência comprovada |
| Concluído | Item finalizado |
| Bloqueado | Dependência externa |

---

# Documentação Relacionada

Cada item deverá apontar para os documentos correspondentes.

Exemplo:

```
01-Escopo.md

03-Roadmap.md

07-Controle-de-Decisoes.md

08-Registro-de-Sessoes.md
```

---

# Decisão Relacionada

Caso exista alguma decisão arquitetural, informar.

Exemplo:

```
DEC-012
```

ou

```
ADR-005
```

---

# Sessão Relacionada

Referenciar a sessão onde o trabalho foi realizado.

Exemplo:

```
SES-014
```

---

# Checklist Relacionado

Informar o item correspondente do Checklist Mestre.

Exemplo:

```
CHK-041
```

---

# Evidências

Informar onde estão armazenadas as evidências.

Exemplo:

- screenshots
- logs
- arquivos
- testes
- comparações
- capturas
- relatórios

---

# Testes

Registrar:

- testes unitários;
- testes integrados;
- testes funcionais;
- validações manuais.

Exemplo:

```
Teste de login realizado.

Teste de movimentação aprovado.

Teste de inventário aprovado.
```

---

# Critérios de Equivalência

Informar como foi validado que a implementação Java reproduz corretamente o comportamento do código C#.

Exemplo:

- mesma sequência de execução;
- mesmo pacote enviado;
- mesmo resultado;
- mesmo fluxo;
- mesmo comportamento visual;
- mesma lógica.

---

# Dependências

Registrar dependências do item.

Exemplo:

```
Inventory depende de:

PacketReader

PacketWriter

MinecraftClient
```

---

# Impactos

Registrar quais módulos utilizam esta funcionalidade.

Exemplo:

```
Macro de mineração

Macro de pesca

Sistema de login

Comandos
```

---

# Pendências

Registrar pendências encontradas.

Exemplo:

```
Necessário validar comportamento após reconnect.

Necessário confirmar algoritmo original.

Necessário revisar tratamento de exceções.
```

---

# Riscos

Registrar riscos conhecidos.

Exemplo:

- comportamento diferente do original;
- biblioteca incompatível;
- dependência externa;
- código não documentado;
- classe parcialmente decompilada.

---

# Observações

Campo livre para registrar informações importantes.

Exemplo:

```
Classe parcialmente ofuscada.

Necessário revisar após implementação do PacketReader.

Código original possui comportamento não documentado.
```

---

# Atualização da Matriz

A matriz deverá ser atualizada sempre que ocorrer:

- nova análise;
- nova documentação;
- migração concluída;
- alteração arquitetural;
- conclusão de testes;
- validação;
- revisão;
- descoberta de dependências;
- mudança de status.

Nunca deixar atualizações para sessões futuras.

---

# Boas Práticas

- Atualizar imediatamente após concluir uma atividade.
- Nunca deixar campos obrigatórios vazios.
- Utilizar descrições objetivas.
- Manter nomenclatura padronizada.
- Garantir consistência com os demais documentos.
- Validar todas as referências antes de registrar.
- Manter histórico rastreável.
- Evitar duplicidade de registros.

---

# Campos Obrigatórios

Todo item deverá conter obrigatoriamente:

- ID
- Tipo
- Origem
- Destino
- Funcionalidade
- Responsável
- Status
- Documentação
- Evidências
- Testes
- Critérios de Equivalência
- Dependências
- Data de atualização

---

# Modelo de Registro

| Campo | Valor |
|--------|-------|
| ID | CLS-021 |
| Tipo | Classe |
| Origem | AdvancedBot\Client\Inventory.cs |
| Destino | br.com.advancedbot.client.Inventory |
| Funcionalidade | Gerenciamento do inventário do jogador |
| Responsável | Nome do responsável |
| Status | Implementado |
| Documentação | 03-Roadmap.md |
| Decisão | DEC-008 |
| Sessão | SES-011 |
| Checklist | CHK-021 |
| Evidências | Comparação entre C# e Java |
| Testes | Testes unitários e funcionais aprovados |
| Critério de Equivalência | Fluxo idêntico ao código original |
| Dependências | PacketReader, PacketWriter |
| Impactos | Sistema de mineração e pesca |
| Pendências | Validar reconnect |
| Riscos | Diferença de sincronização de pacotes |
| Observações | Classe revisada integralmente |
| Última Atualização | AAAA-MM-DD |

---

# Critério de Conclusão

Um item somente poderá ser considerado **Concluído** quando:

- tiver sido completamente analisado;
- estiver documentado;
- possuir implementação Java correspondente;
- possuir rastreabilidade completa;
- possuir testes realizados;
- possuir evidências registradas;
- possuir equivalência validada;
- possuir documentação relacionada;
- possuir dependências identificadas;
- não possuir pendências críticas em aberto.

A Matriz de Rastreabilidade deve permanecer sincronizada com todos os demais documentos do projeto, funcionando como o principal mecanismo de auditoria, acompanhamento e validação da migração do AdvancedBot de C# para Java.