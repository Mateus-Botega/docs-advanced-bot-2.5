# Revisão Arquitetural Fase 1.2

## Objetivo

Este documento apresenta a avaliação crítica das decisões arquiteturais documentadas na Fase 1.1, analisando o impacto, viabilidade e riscos para a migração do AdvancedBot.

---

## Avaliação das Decisões

### DEC-01 — Versões de Protocolo Suportadas

#### Status da Decisão

APROVADA

#### Avaliação Técnica

- **Pontos Positivos:** O foco exclusivo em um protocolo inicial reduz drasticamente o escopo de migração.
 A versão 1.8 possui um protocolo estável e largamente dominado.
- **Riscos Identificados:** Acoplamento excessivo aos formatos de pacote da versão 1.8 no domínio da aplicação.
 Isso pode dificultar a adoção futura de outras versões do jogo.
- **Impactos Arquiteturais:** Exige que a camada de comunicação seja totalmente abstraída do Core do sistema.
 O Core nunca deve conhecer os IDs de pacotes específicos.
- **Possíveis Problemas Futuros:** Refatoração massiva caso a injeção de manipuladores de pacote não seja modular.

#### Questionamentos Pendentes

Como as estruturas de dados específicas da versão serão transladadas para um modelo comum independente de protocolo?

#### Ajuste Recomendado

- **Alteração:** Incluir na documentação a obrigatoriedade da criação de um modelo de domínio de rede agnóstico.
 Recomendar o uso do padrão *Factory Method* para instanciar pacotes e isolar *handlers*.

---

### DEC-02 — Estratégia de Interface do Usuário

#### Status da Decisão

APROVADA

#### Avaliação Técnica

- **Pontos Positivos:** Separação absoluta entre interface visual e lógica de negócio do bot.
 A adoção de uma aplicação *headless* reduz exponencialmente os custos de manutenção.
- **Riscos Identificados:** Dificuldade para debugar estados internos visuais durante os testes automatizados.
 A remoção do renderizador OpenGL exigirá excelentes registros de *logs*.
- **Impactos Arquiteturais:** Toda alteração de estado do bot precisará ser comunicada via eventos.
 O sistema será totalmente arquitetado no padrão *API-First*.
- **Possíveis Problemas Futuros:** Limitação comercial caso usuários exijam um painel de controle interativo.

#### Questionamentos Pendentes

A arquitetura irá expor os eventos do bot através de uma *API* local ou *WebSockets* para ferramentas de debug?

#### Ajuste Recomendado

- **Sugerir Alternativa:** Recomendar a adoção formal da Arquitetura Hexagonal (*Ports and Adapters*).
 O domínio deve disparar eventos para portas de saída, facilitando conexões futuras com painéis Web.

---

### DEC-03 — Modelo de Macros e Scripts

#### Status da Decisão

APROVADA COM AJUSTES

#### Avaliação Técnica

- **Pontos Positivos:** O uso de *Virtual Threads* no Java 21 é o estado da arte para alta concorrência.
 Elimina os limites e gargalos das tradicionais *Thread Pools* para processos bloqueantes.
- **Riscos Identificados:** Inevitáveis condições de corrida na manipulação do estado global do *Player*.
 O acesso simultâneo de macros e pacotes de rede ao inventário pode gerar inconsistências.
- **Impactos Arquiteturais:** Exige controle rigoroso de mutação de estado.
 Entidades críticas do jogo precisarão utilizar imutabilidade ou bloqueios seguros.
- **Possíveis Problemas Futuros:** Travamento silencioso de rotinas de macro caso interrupções não sejam tratadas adequadamente.

#### Questionamentos Pendentes

Qual será o modelo de sincronização adotado para o estado global quando múltiplas macros executarem ações concorrentes?

#### Ajuste Recomendado

- **Nova Alternativa:** Adotar os princípios de *Structured Concurrency* nativos do Java moderno.
 Isso permitirá agrupar tarefas subordinadas e facilitar o cancelamento seguro durante desconexões.

---

### DEC-04 — Provider de Criptografia AES-CFB8

#### Status da Decisão

APROVADA

#### Avaliação Técnica

- **Pontos Positivos:** A biblioteca *BouncyCastle* provê uma implementação robusta, altamente testada e homologada.
 Evita anomalias frequentes encontradas em protocolos customizados de handshake.
- **Riscos Identificados:** Introdução de conflitos de *classpath* caso o projeto adote bibliotecas de terceiros incompatíveis.
- **Impactos Arquiteturais:** Introduz dependência externa pesada diretamente no pacote de inicialização de rede.
- **Possíveis Problemas Futuros:** Dificuldade em atualizações futuras se a biblioteca for abandonada ou alterar assinaturas críticas.

#### Questionamentos Pendentes

A biblioteca será isolada utilizando técnicas de *Shading* no processo de *build* para evitar quebras em tempo de execução?

#### Ajuste Recomendado

- **Documentação Complementar:** Definir o uso imediato de interfaces para encapsular a lógica criptográfica.
 Isso permitirá a troca transparente de provedor sem afetar o comportamento de handshake.

---

### DEC-05 — Formato de Configuração

#### Status da Decisão

APROVADA COM AJUSTES

#### Avaliação Técnica

- **Pontos Positivos:** Arquivos YAML são fáceis de editar, amplamente aceitos e integráveis nativamente no ecossistema Spring.
- **Riscos Identificados:** Quebra da aplicação devido à tipagem incorreta pelo usuário final em edições manuais livres.
- **Impactos Arquiteturais:** Requer implementação de uma camada de mapeamento estrito na inicialização do contexto.
- **Possíveis Problemas Futuros:** Perda irreparável dos dados configurados pelos usuários nas versões C# legadas.

#### Questionamentos Pendentes

Existirá algum conversor oficial, ainda que mínimo, para transformar configurações NBT em YAML durante a transição?

#### Ajuste Recomendado

- **Alteração:** Adicionar a etapa obrigatória de validação de esquemas YAML na inicialização da aplicação.
 Considerar a construção de um pequeno utilitário isolado para converter os perfis antigos.

---

### DEC-06 — Formato e Sistema de Plugins

#### Status da Decisão

NECESSITA INVESTIGAÇÃO

#### Avaliação Técnica

- **Pontos Positivos:** Criar uma API rigorosamente controlada impede acessos diretos nocivos ao núcleo do sistema.
 Aumenta a confiabilidade do produto frente ao sistema legado inseguro.
- **Riscos Identificados:** Sistemas customizados de injeção de classes em Java são propensos a vazamentos severos de memória.
- **Impactos Arquiteturais:** Exige o particionamento da estrutura em módulos estritos de API e Implementação.
- **Possíveis Problemas Futuros:** Colisão de dependências (*Classpath Collision*) entre *plugins* de origens distintas.

#### Questionamentos Pendentes

O carregamento de *plugins* ocorrerá apenas na inicialização (*Startup*) ou exigirá suporte complexo de *Hot-Reload*?

#### Ajuste Recomendado

- **Sugerir Alternativa:** Adiar a implementação de *Custom ClassLoaders* para futuras versões de maturidade.
 Adotar provisoriamente o mecanismo padrão `ServiceLoader` (SPI) do Java para orquestrar *plugins* iniciais.

---

### DEC-07 — Servidor Alvo Primário

#### Status da Decisão

APROVADA

#### Avaliação Técnica

- **Pontos Positivos:** O foco exclusivo na versão 1.8 foca o esforço da equipe nas mecânicas principais.
 A maturidade dos servidores dessa versão garante métricas confiáveis de teste.
- **Riscos Identificados:** Focar demasiadamente na adaptação ao *Anti-Cheat* Sunshine pode inviabilizar o bot em conexões *Vanilla*.
- **Impactos Arquiteturais:** Inicialmente os *handlers* focarão exclusivamente nas leis físicas da versão 1.8.
- **Possíveis Problemas Futuros:** Acoplamento com anomalias propositais do anti-cheat Raizlandia.

#### Questionamentos Pendentes

A engenharia do *bypass* AC será desenvolvida acoplada no núcleo ou isolada como módulo interceptador dinâmico?

#### Ajuste Recomendado

- **Sugerir Alternativa:** Documentar oficialmente que comportamentos de anulação de *Anti-Cheat* serão desenvolvidos externamente.
 Manter o código central fiel às documentações e comportamento nativo do jogo.

---

### DEC-08 — Framework Base do Projeto Java

#### Status da Decisão

NECESSITA INVESTIGAÇÃO

#### Avaliação Técnica

- **Pontos Positivos:** O *Spring Boot* proporciona maturidade inquestionável, ciclo de vida estruturado e ótima testabilidade.
- **Riscos Identificados:** Excesso de consumo de memória RAM se comparado com aplicações desenvolvidas em Java puro.
 O *overhead* pode comprometer usuários com infraestruturas de baixa capacidade.
- **Impactos Arquiteturais:** Acoplamento global com os contêineres e fluxo de vida estipulado pela biblioteca *Spring*.
- **Possíveis Problemas Futuros:** Impossibilidade de rodar múltiplas instâncias isoladas (*Single-Tenant*) na mesma máquina por limites de memória.

#### Questionamentos Pendentes

O projeto implementará execução concorrente através de *Multi-Tenancy* (Muitos Bots num Spring Context) ou rodará processos distintos?

#### Ajuste Recomendado

- **Alteração:** Não consolidar o *Spring Boot* até ser avaliado o impacto arquitetural da criação de centenas de sessões em uma única JVM.
 Recomendar teste prático rápido comparando Injeção Pura e *Spring* no cenário focado do bot.

---

## Parecer Arquitetural Geral

### Maturidade Atual da Arquitetura

As decisões tomadas apresentam alto grau de alinhamento com práticas de modernização Java e arquiteturas leves.
A exclusão das interfaces legadas e adoção das novidades do Java 21 como *Virtual Threads* trazem benefícios gigantescos.
Contudo, existem incertezas críticas no aspecto do paralelismo e orquestração.

### Riscos Antes da Codificação

1. Falta de definição sobre o modelo de injeção (*Single* ou *Multi-Tenant*) que dita diretamente a viabilidade de memória do *Spring Boot*.
2. Sistemas de injeção modular para os *plugins*, podendo provocar quebras estruturais caso desenhados sem isolamento restrito de classes.
3. Tratamento incorreto do estado compartilhado global acessado em tempo real por inúmeras rotinas de rede e automação de macros.

### Decisões que Podem ser Congeladas

As seguintes diretrizes encontram-se robustas para execução inicial:

- **DEC-01:** Suporte ao protocolo 1.8.
- **DEC-02:** Operação *CLI/Headless*.
- **DEC-04:** BouncyCastle como provedor criptográfico.
- **DEC-05:** Configurações formatadas em YAML.
- **DEC-07:** Foco primário na adaptação do comportamento 1.8.

### Decisões que Precisam Aguardar Análise do Código Legado

- **DEC-03:** É necessário rastrear todas as zonas onde o legado possuía bloqueios síncronos e condições de corrida antes de usar *Virtual Threads*.
- **DEC-06:** Entender qual era o nível de acesso que as extensões antigas (*DLLs*) usufruíam no C# antes de formular as permissões da interface em Java.
- **DEC-08:** A viabilidade de uso irrestrito do *Spring Boot* precisará ser repensada se as instâncias antigas compartilhavam forte escopo estático de forma desestruturada.

### Próximos Passos Recomendados da Governança

1. Levantar quais foram os pontos críticos de colisão de pacotes e macros no mapeamento de erros antigos.
2. Formular o escopo e interfaces restritas que isolarão a biblioteca de comunicação de rede do domínio central.
3. Decidir formalmente o padrão de injeção de contexto (*Multi* ou *Single Tenant*).
4. Atualizar o arquivo de Decisões Arquiteturais oficial incorporando os ajustes indicados.
