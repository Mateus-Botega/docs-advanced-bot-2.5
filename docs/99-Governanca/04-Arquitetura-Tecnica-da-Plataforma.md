# Arquitetura Técnica da Plataforma

## Objetivo

Este documento define a arquitetura técnica de alto nível da nova plataforma AdvancedBot.

Ele estabelece o relacionamento e a integração entre a interface de usuário web (React), o backend orquestrador (Spring Boot) e o núcleo do bot em execução isolada (Java Core), além de tratar a abstração de versões do Minecraft e a gestão de comunicação de rede.

---

## Escopo

A arquitetura abrange a estruturação do sistema para suportar:

- O isolamento total do motor do bot (modelo Single-Tenant).
- O gerenciamento simplificado de múltiplas instâncias através de uma interface web unificada.
- A comunicação assíncrona baseada em WebSockets e API REST.
- A abstração das diferentes versões do protocolo de rede do Minecraft (focada nas versões 1.5.2 e 1.8).
- A configuração avançada e individual de proxies, permitindo rotação de rede e mitigação de banimentos.

---

## Contexto

Conforme determinado na DEC-09 (Modelo Operacional da Aplicação), o novo AdvancedBot abandonará interfaces gráficas acopladas nativas e adotará um modelo voltado à estabilidade e facilidade de manutenção.

A plataforma utilizará Spring Boot como backend orquestrador e React como interface web, fornecendo um *dashboard* para gerenciar bots isolados. O motor do bot operará como uma aplicação Java autônoma (Single-Tenant), desacoplada de dependências pesadas, minimizando o impacto de vazamentos de memória e falhas globais.

---

## Visão Geral da Arquitetura

O sistema será constituído de três grandes componentes independentes.

### 1. Frontend Web (React)

Responsável pela interface de usuário (UI).
Atuará como o painel central para o usuário visualizar, configurar e disparar ações nos bots.

- Comunicação de comandos (iniciar, parar, alterar configuração) através de API REST.
- Recepção em tempo real de métricas, status, eventos do jogo e logs do console por meio de conexões WebSocket.

### 2. Backend Orquestrador (Spring Boot)

Atuará como gerenciador de instâncias e ponte entre o Frontend e o Motor do Bot.

- Gerencia o ciclo de vida dos processos do motor do bot (iniciando instâncias de JVM separadas).
- Proverá *endpoints* de API seguros para o React.
- Traduzirá e distribuirá mensagens via *message brokers* leves ou *sockets* locais para as instâncias do bot em execução.

### 3. Motor do Bot (Java Core)

O núcleo do aplicativo responsável exclusivamente pelas regras de negócio, tomada de decisões (IA e Macros) e conexão de rede direta com o servidor Minecraft.

- Funciona de forma isolada, em uma JVM autônoma (UMA JVM = UM BOT).
- Desenvolvido em Java nativo de alta eficiência, totalmente desacoplado do Spring Boot.
- Toda falha, exceção não tratada, ou bloqueio na comunicação de um bot não afetará o orquestrador e nem outras instâncias ativas.

---

## Isolamento do Motor do Bot

Seguindo estritamente a DEC-09, o isolamento será garantido por processos independentes no sistema operacional.

A comunicação IPC (Inter-Process Communication) será implementada de maneira leve (ex: sockets TCP locais, *named pipes* ou Redis, caso aplicável). O motor será configurado para não conter nenhuma biblioteca gráfica e nenhuma sobrecarga do framework Spring, maximizando a economia de memória por instância (uso pretendido abaixo de 50-80MB por bot ativo).

---

## Abstração de Versões do Minecraft

O sistema precisará suportar a conexão em múltiplos servidores com diferentes lógicas de rede, principalmente o foco em Minecraft 1.5.2 e 1.8.

Para garantir baixa complexidade e facilidade de expansão:

- **Protocol Layer:** Uma camada de abstração que separará a lógica de tradução de pacotes (serialização e desserialização) da lógica de domínio de movimentação e inventário.
- Injeção dinâmica de *handlers* de acordo com a versão definida na configuração inicial do bot.
- Pacotes específicos de versões serão traduzidos para eventos genéricos e invariantes mapeados no domínio de eventos interno do sistema.

---

## Decisão Arquitetural: Gerenciamento de Proxy

A funcionalidade herdada do legado de C# de camuflar instâncias através de proxies é crucial. Este requisito originou a necessidade de formalizar uma nova regra de integração e isolamento na rede.

### DEC-10 — Gerenciamento e Suporte a Proxies

**Contexto:**
A nova arquitetura demanda um modelo estável de conexão por proxies que opere isoladamente dentro de cada processo autônomo (Single-Tenant).

**Problema:**
No C#, a configuração de proxy frequentemente sofria com acoplamento na interface gráfica, dificultando implementações de *Proxy Pools* dinâmicos e controle refinado do tráfego TCP por instâncias massivas.

**Alternativas Avaliadas:**
1. Configuração global na JVM através de System Properties (`-Dhttp.proxyHost`).
2. Configuração local e injeção do objeto genérico `java.net.Proxy` no construtor direto do `Socket` de rede de cada bot.
3. Delegação do roteamento total da infraestrutura para uma ferramenta externa de redirecionamento (Proxy Reverso ou Forwarder local).

**Decisão Oficial:**
**Alternativa 2**. A injeção parametrizada da configuração no nível de *socket* nativo garante máximo isolamento e segurança, mantendo coesão com a decisão Single-Tenant e eliminando riscos de poluição do escopo global da JVM.

**Impacto Arquitetural:**
Cada instância carregará em sua inicialização os metadados referentes a (IP, Porta, Protocolo e Credenciais). A estrutura de conexão principal deverá disponibilizar interfaces limpas para no futuro receber um módulo de rotação (Proxy Pool), que em caso de bloqueio ou desconexão, consiga solicitar um novo nó de rede de forma autônoma antes de se reconectar, sem necessidade de intervenção do orquestrador ou da interface de usuário React.
