# Modelo Operacional e Isolamento

## Objetivo da Decisão

Este documento define formalmente o modelo operacional da aplicação e o padrão de isolamento para a nova arquitetura do AdvancedBot em Java.

O objetivo é solucionar o bloqueio das decisões arquiteturais pendentes (DEC-03, DEC-06 e DEC-08) definindo se o sistema operará com múltiplas instâncias isoladas ou através de um único núcleo gerenciador de múltiplos clientes.

## Contexto Atual do AdvancedBot C#

A arquitetura original do bot em C# possuía uma estrutura voltada para o gerenciamento acoplado e interfaces gráficas pesadas.

O sistema foi desenhado para suportar desde pequenas execuções individuais até cenários com dezenas de bots simultâneos na mesma máquina.

As principais características e limitações atuais incluem:

- **Quantidade Esperada:** Usuários frequentemente executam entre 1 e 50 bots simultâneos.
- **Limitações Atuais:** Forte acoplamento de estado global causa inconsistências quando múltiplas instâncias rodam no mesmo processo.
- **Gerenciamento de Conexão:** Cada bot possui seu próprio socket de rede e thread de controle.
- **Problemas Conhecidos:** 
  - *Reconexão:* Falhas em threads de rede não isoladas causam o encerramento abrupto do processo principal.
  - *Travamentos:* Condições de corrida na interface de usuário bloqueiam a execução de todos os bots conectados.
  - *Consumo de Memória:* Bibliotecas gráficas alocam recursos excessivos para cada instância virtual.
  - *Isolamento de Erros:* Exceções não tratadas no escopo de um bot frequentemente contaminam o estado dos demais.

## Comparação Arquitetural

Abaixo apresentamos a análise detalhada entre os modelos de execução aplicáveis ao projeto.

### Modelo A: Single-Tenant (Uma JVM por Bot)

Este modelo propõe que cada bot seja um processo autônomo do sistema operacional, executando sua própria Java Virtual Machine (JVM).

- **Memória:** Alto consumo global devido ao overhead natural da JVM para cada processo isolado.
- **CPU:** Alta concorrência no escalonador do sistema operacional, porém maior facilidade em utilizar múltiplos núcleos.
- **Estabilidade:** Máxima estabilidade. A falha crítica de um processo não afeta os outros bots.
- **Manutenção:** Maior simplicidade no código, pois não existe necessidade de isolamento lógico de contexto.
- **Deploy:** Facilidade na execução via scripts e gerenciadores de processos.
- **Atualização:** Requer reiniciar cada processo individualmente.
- **Debugging:** Extremamente simples, estado focado em apenas um jogador por sessão de debug.
- **Plugins:** Evita completamente colisão de classpath entre extensões de terceiros.
- **Spring Boot:** Totalmente compatível, operando no modelo padrão de aplicação isolada.
- **Virtual Threads:** Úteis para tarefas internas, mas o impacto no paralelismo massivo é subutilizado.
- **Structured Concurrency:** Facilita a destruição do contexto único em caso de falha de conexão.

### Modelo B: Multi-Tenant (Uma JVM para Múltiplos Bots)

Este modelo centraliza todas as conexões em uma única aplicação rodando em uma JVM compartilhada, separando os bots logicamente.

- **Memória:** Extremamente eficiente. Estruturas estáticas e metaspace da JVM são instanciados apenas uma vez.
- **CPU:** Eficiência superior no chaveamento de contexto gerenciado internamente pela própria JVM.
- **Estabilidade:** Menor estabilidade nativa. Requer tratamento de erros rigoroso para evitar crash do servidor inteiro.
- **Manutenção:** Alta complexidade. Todo gerenciamento de estado exige injeção por escopo e prevenção de vazamentos de memória.
- **Deploy:** Simplificado em um único artefato executável que carrega todos os perfis.
- **Atualização:** Interrupção total de todos os bots conectados durante uma reinicialização.
- **Debugging:** Complexo, exigindo rastreio detalhado para identificar ações de um bot específico.
- **Plugins:** Alta probabilidade de vazamento de memória ou modificações globais não autorizadas entre instâncias.
- **Spring Boot:** Requer configuração avançada de escopos customizados, elevando muito o custo de processamento.
- **Virtual Threads:** Altamente recomendadas, permitindo gerenciar milhares de conexões com custo ínfimo.
- **Structured Concurrency:** Essencial para agrupar todas as tarefas e ciclos de vida vinculados a um bot específico.

## Avaliação de Impacto das Decisões Bloqueadas

A escolha do modelo operacional resolve os impasses das seguintes decisões:

### DEC-03 — Virtual Threads

No modelo Single-Tenant, as Virtual Threads são subutilizadas. No modelo Multi-Tenant, elas se tornam a espinha dorsal da arquitetura de concorrência, permitindo escalar o número de bots sem estourar o limite de threads nativas do sistema.

### DEC-06 — Plugins

O modelo Single-Tenant permite um sistema de plugins simples e tradicional. O modelo Multi-Tenant exigiria isolamento via carregadores de classe customizados e injeção controlada, adicionando enorme complexidade técnica.

### DEC-08 — Spring Boot

No modelo Single-Tenant, o Spring Boot possui um enorme overhead de memória para instanciar centenas de processos. No modelo Multi-Tenant, gerenciar escopos customizados no Spring é altamente complexo e penaliza o roteamento.

## Recomendação de Arquitetura Final

**Decisão Recomendada:** Modelo A - Single-Tenant (Uma JVM por Bot).

### Justificativa Técnica

Para atingir a paridade funcional sem introduzir alta complexidade de isolamento lógico de memória, a execução de uma JVM por bot elimina dezenas de categorias de defeitos.

O foco é a robustez, facilidade de uso e isolamento total, delegando o agrupamento das aplicações para gerenciadores externos via linha de comando. Para mitigar o alto consumo de RAM, o projeto evitará frameworks pesados, preferindo injeção de dependência nativa ou frameworks otimizados para rápida inicialização.

### Riscos Aceitos

- **Overhead de Memória Base:** Cada bot possuirá um consumo basal da JVM (cerca de 30MB extras por processo).
- **Gerenciamento Descentralizado:** O controle sobre as instâncias precisa ser feito através de comandos de terminal ou processos do sistema.

### Riscos Mitigados

- **Travamentos e Falhas:** Loops infinitos em macros afetarão somente a instância isolada.
- **Condição de Corrida:** Não existe estado global compartilhado entre bots, garantindo a integridade dos pacotes de rede.
- **Colisão de Plugins:** Módulos estendem apenas a instância local, sem necessidade de isolamento de bibliotecas.

## Registro da Decisão Arquitetural Proposta

### DEC-09 — Modelo Operacional da Aplicação

Status:
PROPOSTA

Motivação:
Desbloquear o andamento do projeto solucionando as decisões arquiteturais dependentes da definição de como o ciclo de vida e concorrência dos bots serão governados no ambiente Java.

Alternativas Consideradas:
1. Modelo Single-Tenant, possuindo uma JVM isolada por perfil ou bot.
2. Modelo Multi-Tenant, possuindo uma única JVM rodando múltiplos perfis simultâneos.

Decisão:
Adotar a execução Single-Tenant (Uma JVM por Bot), removendo a necessidade de gerir escopos lógicos e prevenindo falhas em cascata, mantendo a arquitetura isolada.

Consequências:
Elimina a necessidade de carregamento dinâmico avançado de plugins. Evita vazamentos de memória compartilhados. Exige revisão da adoção do Spring Boot (revertendo DEC-08) em favor de uma aplicação Java nativa sem reflexão massiva na inicialização. Permite aprovar as decisões DEC-03 e DEC-06 em formas simplificadas.

Impactos:
Requer documentação e scripts auxiliares em Bash ou PowerShell para gerenciar a abertura em lote de diversos bots para usuários que demandam alta quantidade simultânea.
