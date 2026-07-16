# 09 - Arquitetura de Execução Distribuída

> **Status:** Oficial
>
> **Versão:** 1.0
>
> **Data:** Julho/2026
>
> **Projeto:** AdvancedBot Java
>
> **Documento de Arquitetura**

---

# 1. Objetivo

Este documento estabelece a arquitetura oficial de execução do AdvancedBot Java.

Diferentemente da implementação legada em C#, onde múltiplos bots compartilhavam um único processo da aplicação desktop, a nova arquitetura foi concebida para suportar escalabilidade horizontal, isolamento de falhas, alta disponibilidade e facilidade de manutenção.

Esta decisão arquitetural é considerada definitiva e servirá como base para todas as fases posteriores do projeto.

---

# 2. Motivação

O AdvancedBot C# foi originalmente desenvolvido como uma aplicação desktop monolítica.

Sua arquitetura seguia aproximadamente o seguinte modelo:

```
AdvancedBot.exe

├── Interface gráfica
├── Bot 01
├── Bot 02
├── Bot 03
├── ...
└── Bot N
```

Embora esta abordagem funcione para dezenas de bots, ela apresenta limitações importantes quando utilizada em ambientes de larga escala.

Principais problemas identificados:

- Todo o sistema compartilha o mesmo processo.
- Um erro crítico pode interromper todos os bots.
- Deadlocks afetam toda a aplicação.
- Vazamentos de memória acumulam-se em um único processo.
- O Garbage Collector possui pausas maiores.
- A interface gráfica permanece carregada mesmo quando não é necessária.
- Escalabilidade horizontal praticamente inexistente.
- Reinicialização obrigatória de todos os bots em caso de falha.

Considerando que o objetivo desta reescrita é suportar centenas de contas simultâneas executando macros de forma contínua, optou-se por abandonar definitivamente este modelo.

---

# 3. Objetivos da Nova Arquitetura

A arquitetura deverá atender aos seguintes requisitos:

- Escalar para centenas de bots simultâneos.
- Isolar completamente falhas entre bots.
- Permitir distribuição entre múltiplos servidores.
- Eliminar dependência da interface gráfica para execução.
- Possibilitar reinicialização individual de bots.
- Simplificar manutenção.
- Facilitar monitoramento.
- Reduzir consumo de recursos.
- Permitir futura orquestração automatizada.

---

# 4. Arquitetura Geral

O sistema será dividido em dois grandes domínios:

## Control Plane

Responsável pela administração do ambiente.

Componentes:

- Frontend React
- API Spring Boot
- Bot Manager
- Banco de Dados PostgreSQL
- Serviços administrativos
- Autenticação
- Monitoramento
- Configuração

---

## Data Plane

Responsável exclusivamente pela execução dos bots.

Cada Worker executará:

- conexão Minecraft;
- autenticação;
- processamento de pacotes;
- execução das macros;
- movimentação;
- inventário;
- combate;
- mineração;
- pesca;
- reparo;
- venda;
- demais automações.

Workers não possuem interface gráfica.

Workers não possuem regras administrativas.

Workers não realizam gerenciamento do sistema.

Sua única responsabilidade é executar um bot.

---

# 5. Arquitetura Física

```
                 React

                   │

            Spring Boot API

                   │

          Bot Manager Service

                   │

     ┌─────────────┼─────────────┐

     ▼             ▼             ▼

 Worker 01     Worker 02     Worker 300

     │             │             │

 Minecraft     Minecraft     Minecraft
```

---

# 6. Modelo de Execução

Cada conta será executada em um processo Java independente.

Exemplo:

```
java -jar worker.jar --botId=1

java -jar worker.jar --botId=2

java -jar worker.jar --botId=3
```

Cada processo possuirá:

- Heap independente.
- Garbage Collector independente.
- Threads independentes.
- Socket independente.
- Estado interno independente.

Nenhum Worker compartilhará memória com outro Worker.

---

# 7. Bot Manager

O Bot Manager é responsável pelo ciclo de vida dos Workers.

Suas responsabilidades incluem:

- iniciar Workers;
- interromper Workers;
- reiniciar Workers;
- monitorar processos;
- detectar falhas;
- registrar logs;
- controlar estado dos bots;
- limitar quantidade de Workers por máquina.

O usuário nunca executará diretamente um Worker.

Toda criação de processos ocorrerá exclusivamente através do Bot Manager.

---

# 8. Fluxo de Inicialização

Fluxo esperado:

```
Usuário

↓

Painel React

↓

API

↓

Bot Manager

↓

ProcessBuilder

↓

Worker

↓

Minecraft Server
```

O usuário apenas solicita a inicialização do bot.

Toda a orquestração será transparente.

---

# 9. Interface do Usuário

A interface gráfica deixa de ser responsável pela execução dos bots.

Ela passa a exercer apenas funções administrativas.

Exemplos:

- cadastrar contas;
- cadastrar proxies;
- configurar servidores;
- selecionar macros;
- iniciar bots;
- parar bots;
- visualizar inventário;
- acompanhar estatísticas;
- visualizar logs;
- administrar usuários.

Toda execução ocorre nos Workers.

---

# 10. Worker

Cada Worker representa exatamente um bot.

Responsabilidades:

- conectar ao servidor Minecraft;
- executar login;
- interpretar pacotes;
- controlar movimentação;
- executar IA;
- executar macros;
- enviar eventos;
- receber comandos.

O Worker não conhece a existência de outros bots.

Não existe compartilhamento de estado entre Workers.

---

# 11. Comunicação

Os Workers não devem acessar diretamente o banco de dados.

Toda comunicação administrativa ocorrerá através da API.

Exemplos:

Worker → API

- status
- inventário
- localização
- estatísticas
- logs
- eventos
- alertas

API → Worker

- iniciar macro
- parar macro
- reconectar
- alterar configuração
- desligar

A implementação poderá utilizar WebSocket, gRPC ou outro protocolo apropriado, desde que mantenha comunicação bidirecional eficiente e de baixa latência.

---

# 12. Banco de Dados

Somente a API possui responsabilidade sobre persistência.

Os Workers devem permanecer desacoplados da camada de banco de dados.

Benefícios:

- maior segurança;
- menor acoplamento;
- centralização das regras;
- auditoria simplificada;
- facilidade de evolução.

---

# 13. Escalabilidade Horizontal

A arquitetura foi projetada para suportar múltiplos servidores físicos ou virtuais.

Exemplo:

```
Servidor A

100 Workers

Servidor B

100 Workers

Servidor C

100 Workers
```

Todos conectados ao mesmo ambiente administrativo.

A expansão do sistema deverá ocorrer através da adição de novos servidores, sem necessidade de alterações estruturais na aplicação.

---

# 14. Isolamento de Falhas

Uma falha em um Worker não poderá comprometer os demais.

Exemplo:

```
Worker 157

↓

Erro inesperado

↓

Processo encerrado

↓

Bot Manager detecta

↓

Novo Worker iniciado

↓

Demais Workers continuam operando normalmente
```

Esta característica representa um dos principais objetivos da nova arquitetura.

---

# 15. Consumo de Recursos

O Worker deverá ser extremamente leve.

Por esta razão, não utilizará Spring Boot.

O Worker será desenvolvido utilizando Java 21 puro e bibliotecas específicas apenas para suas necessidades.

Exemplos:

- Netty (rede);
- Jackson (serialização);
- Logback (logs);
- Bibliotecas do protocolo Minecraft.

Não serão utilizados:

- Spring Boot;
- Tomcat;
- Hibernate;
- JPA;
- Bean Container;
- Component Scan;
- Reflection desnecessária.

Esta decisão reduz significativamente o consumo de memória e melhora a capacidade de escalabilidade.

---

# 16. Benefícios Esperados

Com esta arquitetura espera-se obter:

- isolamento completo entre bots;
- reinicialização individual;
- maior estabilidade;
- redução de falhas sistêmicas;
- melhor aproveitamento do Garbage Collector;
- menor consumo de memória por Worker;
- facilidade de monitoramento;
- escalabilidade horizontal;
- distribuição entre múltiplos servidores;
- maior facilidade de manutenção.

---

# 17. Compatibilidade com as Macros

Toda a lógica de negócio existente no AdvancedBot C# será migrada para os Workers.

Exemplos:

- mineração;
- pesca;
- combate;
- reparo;
- venda;
- coleta;
- movimentação;
- navegação;
- gerenciamento de inventário;
- demais automações.

A mudança arquitetural não altera o comportamento funcional das macros.

Ela altera exclusivamente a forma como os bots são executados e gerenciados.

---

# 18. Diretrizes Obrigatórias

São consideradas regras obrigatórias do projeto:

- Cada Worker representa exatamente um bot.
- Workers nunca compartilharão memória.
- Workers não possuirão interface gráfica.
- Workers não acessarão diretamente o banco de dados.
- Toda persistência ocorrerá através da API.
- Todo gerenciamento ocorrerá pelo Bot Manager.
- O sistema deverá suportar escalabilidade horizontal.
- O sistema deverá permitir distribuição entre múltiplos servidores.
- Toda implementação futura deverá respeitar esta arquitetura.

---

# 19. Conclusão

A arquitetura distribuída definida neste documento substitui definitivamente o modelo monolítico utilizado pela versão legada em C#.

A separação entre **Control Plane** e **Data Plane**, associada à execução de um processo independente por bot, estabelece uma base sólida para a evolução do AdvancedBot Java, priorizando escalabilidade, estabilidade, isolamento de falhas e facilidade de manutenção.

Esta decisão passa a ser considerada parte da arquitetura oficial do projeto e deverá orientar todas as implementações futuras.