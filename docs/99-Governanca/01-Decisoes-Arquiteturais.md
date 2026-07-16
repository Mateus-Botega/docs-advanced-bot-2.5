# 01 - Decisões Arquiteturais

## Objetivo

Este documento registra todas as decisões arquiteturais essenciais que devem ser tomadas antes do início da codificação do AdvancedBot em Java.

Ele documenta o contexto, as alternativas, a recomendação e os impactos de cada decisão para garantir alinhamento técnico.

---

## Premissas Oficiais do Projeto

Para garantir o sucesso da migração, as seguintes premissas são estabelecidas:

- A migração deve garantir paridade funcional com a versão legado C#.
- O sistema alvo utilizará Java 21 ou superior.
- Toda interface gráfica legada (Windows Forms e OpenGL) será descartada, focando em uma operação CLI ou API first.
- A arquitetura adotada deve favorecer alta escalabilidade e baixo acoplamento.
- A documentação é considerada parte integrante da entrega, não havendo código não documentado.

---

## Itens Bloqueadores da Migração

Os itens abaixo impedem o início da codificação e dependem de definição prévia.

- **Definição de Versões de Protocolo (DEC-01):** Impede a construção correta dos serializadores de rede.
- **Escolha do Framework Base (DEC-08):** Impede a configuração inicial do projeto e definição de injeção de dependências.
- **Implementação Criptográfica (DEC-04):** Impede o teste de conexões autenticadas.

---

## Decisões Obrigatórias Antes da Codificação

Esta seção lista as decisões críticas mapeadas no inventário inicial que exigem aprovação oficial.

### DEC-01 — Versões de Protocolo Suportadas

**Contexto:** O AdvancedBot suportava múltiplas versões do protocolo. A manutenção de múltiplos manipuladores é custosa.

**Problema Atual no C#:** O sistema contém diversas classes `Handler_v*` com lógica duplicada, dificultando a implementação de novos pacotes.

**Alternativas Possíveis:**
1. Manter suporte para todas as versões originais desde o início.
2. Focar exclusivamente em uma versão principal (exemplo 1.8) e expandir posteriormente.
3. Adotar uma biblioteca de abstração de protocolo de terceiros.

**Decisão Recomendada:** Alternativa 2. Iniciar com foco em protocolo único para validar a arquitetura antes de adicionar complexidade de múltiplas versões.

**Impacto na Implementação Java:** Simplifica a criação dos validadores e a abstração inicial de pacotes de rede, garantindo entrega de valor rápida.

---

### DEC-02 — Estratégia de Interface do Usuário

**Contexto:** A versão original possuía componentes de UI em Windows Forms e renderização 3D em OpenGL.

**Problema Atual no C#:** A lógica de interface gráfica está fortemente acoplada ao domínio do bot, dificultando operações sem interface visual.

**Alternativas Possíveis:**
1. Reescrever a interface usando JavaFX ou Swing.
2. Descartar interface rica e construir apenas um modelo de linha de comando.
3. Construir uma API REST e painel Web independente.

**Decisão Recomendada:** Alternativa 2. Priorizar um executável de terminal leve, isolando completamente a lógica do domínio.

**Impacto na Implementação Java:** Elimina a necessidade de migrar inúmeros arquivos de interface, reduzindo o esforço drasticamente. Foco total no núcleo da aplicação.

---

### DEC-03 — Modelo de Macros e Scripts

**Contexto:** Macros no C# utilizavam *threads* dedicadas ou simulação de execução paralela para emular o comportamento humano no jogo.

**Problema Atual no C#:** Muitas condições de corrida entre macros e o envio de pacotes na *thread* principal, além da falta de cancelamento seguro de tarefas.

**Alternativas Possíveis:**
1. Utilizar modelo de *thread* puro tradicional.
2. Utilizar executores baseados em eventos com `CompletableFuture`.
3. Implementar *Virtual Threads* ou modelo reativo.

**Decisão Recomendada:** Alternativa 3. Adotar as *Virtual Threads* do Java 21 para executar macros de forma concorrente com baixo custo computacional.

**Impacto na Implementação Java:** Mudança estrutural em todos os comandos e *plugins*, exigindo criação de uma API de interrupção confiável e isolamento de estado.

---

### DEC-04 — Provider de Criptografia AES-CFB8

**Contexto:** O protocolo de rede exige criptografia AES-CFB8 para autenticação na rede oficial da Mojang.

**Problema Atual no C#:** Implementação local propensa a falhas de compatibilidade, dificultando atualizações em ambientes complexos.

**Alternativas Possíveis:**
1. Escrever uma implementação customizada em Java puro.
2. Utilizar cifra nativa do *Java Cryptography Extension*.
3. Integrar a biblioteca BouncyCastle para garantir máxima compatibilidade.

**Decisão Recomendada:** Alternativa 3. Adotar BouncyCastle, que é largamente utilizado no ecossistema Minecraft para este fim específico.

**Impacto na Implementação Java:** Adição de uma dependência externa crítica, que garante robustez total na camada de comunicação de rede criptografada.

---

### DEC-05 — Formato de Configuração

**Contexto:** As configurações persistentes no C# utilizavam serialização binária proprietária através de formatação em NBT.

**Problema Atual no C#:** O formato binário restringe significativamente a edição manual e amigável das configurações por usuários comuns.

**Alternativas Possíveis:**
1. Manter arquivo original para total retrocompatibilidade com perfis antigos.
2. Migrar totalmente para JSON usando bibliotecas como Jackson ou Gson.
3. Adotar YAML para maior legibilidade manual e facilidade de manipulação.

**Decisão Recomendada:** Alternativa 3. Arquivos YAML são padrão do ecossistema moderno e fáceis de editar sem corromper estruturas cruciais.

**Impacto na Implementação Java:** Requer a construção de um novo sistema de configuração estruturado e perda da retrocompatibilidade com os perfis binários antigos.

---

### DEC-06 — Formato e Sistema de Plugins

**Contexto:** O AdvancedBot carregava módulos dinâmicos externos via sistema de injeção direta de bibliotecas compiladas.

**Problema Atual no C#:** O sistema original permitia acesso irrestrito ao núcleo do bot, causando quebras frequentes e falta de versionamento explícito.

**Alternativas Possíveis:**
1. Carregamento direto de bibliotecas customizadas sem mecanismos de isolamento.
2. Adotar um sistema de módulos complexo como o *Java Platform Module System*.
3. Criar uma API exposta altamente controlada para extensões baseadas em arquivos JAR.

**Decisão Recomendada:** Alternativa 3. Implementar um carregador simples com descritores declarativos e injeção rigorosamente controlada dos serviços.

**Impacto na Implementação Java:** Necessidade de desenhar uma interface central bem definida, restrita e exaustivamente documentada para os desenvolvedores.

---

### DEC-07 — Servidor Alvo Primário

**Contexto:** O foco da nova versão afetará diretamente quais classes do domínio legado serão migradas e testadas primeiro.

**Problema Atual no C#:** O código mescla comportamentos distintos de protocolos diferentes de forma confusa, sem prioridade de suporte definida.

**Alternativas Possíveis:**
1. Foco em servidores antigos utilizando pacotes primitivos.
2. Foco no protocolo 1.8 devido a estabilidade na comunidade de PvP.
3. Foco exclusivo nas versões mais recentes e atualizadas do jogo.

**Decisão Recomendada:** Alternativa 2. A versão 1.8 estabelece o ponto médio ideal e possui ferramentas estabelecidas para validação funcional do *bypass*.

**Impacto na Implementação Java:** Esforço inicial direcionado ao manipulador correspondente e às lógicas específicas de movimentação associadas a essa época.

---

### DEC-08 — Framework Base do Projeto Java

**Contexto:** O bot original funcionava como uma aplicação local isolada sem adoção de nenhum controlador central ou inversão de controle.

**Problema Atual no C#:** O forte acoplamento e o controle de estado global dificultam severamente a criação e a manutenção de testes automatizados seguros.

**Alternativas Possíveis:**
1. Utilizar aplicação local bruta construindo um motor próprio de controle.
2. Adotar o Spring Boot para orquestração geral do ciclo de vida.
3. Utilizar o Quarkus focando em minimizar o uso de memória volátil.

**Decisão Recomendada:** Alternativa 2. Adotar o Spring Boot devido ao amadurecimento das bibliotecas do ecossistema e enorme suporte da comunidade.

**Impacto na Implementação Java:** Redefine completamente a arquitetura de controle e dita como as instâncias essenciais serão injetadas através do sistema de dependência.

---

### DEC-09 — Modelo Operacional da Aplicação

**Contexto:** Definição de como o ciclo de vida e a concorrência dos bots serão governados no ambiente Java, afetando diretamente o isolamento e o paralelismo.

**Problema Atual no C#:** O forte acoplamento de estado global causa inconsistências, travamentos e falhas em cascata quando múltiplas instâncias compartilham o mesmo processo.

**Alternativas Possíveis:**
1. Modelo Single-Tenant, com uma JVM isolada por instância ou perfil de bot.
2. Modelo Multi-Tenant, utilizando uma única JVM para executar e gerenciar múltiplos perfis simultâneos.

**Decisão Recomendada:** Alternativa 1. Adotar a execução Single-Tenant (Uma JVM por Bot), simplificando a separação de escopos e mitigando falhas generalizadas.

**Impacto na Implementação Java:** Remove a necessidade de isolamento dinâmico avançado (ex: ClassLoaders para plugins). O gerenciamento massivo de bots exigirá o uso do Spring Boot como orquestrador externo gerenciando as JVMs independentes do motor.

---

### DEC-10 — Gerenciamento e Suporte a Proxies

**Contexto:** O AdvancedBot legou nativamente o suporte a proxies. Na nova arquitetura Single-Tenant distribuída, o roteamento da comunicação precisa de suporte limpo, escalável e isolado.

**Problema Atual no C#:** O gerenciamento do proxy estava atrelado à interface e lógica de rede global, dificultando rotações ou execuções autônomas sem configuração de UI.

**Alternativas Possíveis:**
1. Configurar proxies via argumentos e propriedades globais da JVM.
2. Implementar suporte isolado instanciando e injetando objetos `java.net.Proxy` diretamente na criação dos *sockets* de conexão.
3. Delegar o roteamento de rede para um proxy reverso externo de infraestrutura.

**Decisão Recomendada:** Alternativa 2. A injeção direta de configurações proxy no *socket* de cada bot reforça o isolamento estrito proposto na DEC-09.

**Impacto na Implementação Java:** A parametrização de rede do bot necessitará de campos explícitos para configurações de Proxy. A arquitetura deverá permitir flexibilidade para uma futura adição de *pools* automáticos e rotação de IPs.

---

### DEC-11 — Idioma de Nomenclatura de Classes

**Contexto:** O [12-Guia-de-Nomenclatura.md](12-Guia-de-Nomenclatura.md) exigia nomes de classes Java exclusivamente em inglês. Ao iniciar a Milestone 3 (Migração do Núcleo do Domínio), o responsável pelo projeto solicitou nomes de classes em português.

**Problema Atual:** Divergência entre a governança aprovada (nomes em inglês) e a preferência explícita do responsável pelo projeto para o código de domínio.

**Alternativas Possíveis:**
1. Manter nomes de classes em inglês, conforme guia original.
2. Adotar nomes de classes em português em todo o código Java, atualizando o guia de nomenclatura.

**Decisão Tomada:** Alternativa 2. Nomes de **classes, interfaces, enums e records** passam a ser escritos em português. Nomes de **pacotes, métodos, variáveis e constantes** permanecem em inglês, conforme o guia original (nenhuma solicitação de alteração para esses itens).

**Justificativa:** Preferência explícita do responsável pelo projeto (Mateus Botega), aprovada em sessão de 2026-07-15, para iniciar a Milestone 3.

**Impacto na Implementação Java:** Atualiza a seção "Idioma Oficial" do [12-Guia-de-Nomenclatura.md](12-Guia-de-Nomenclatura.md). Aplica-se a todo código Java escrito a partir desta decisão; classes já existentes (ex: `AdvancedBotApplication`) não são renomeadas retroativamente sem necessidade funcional.

**Data:** 2026-07-15

**Responsável:** Mateus Botega

---

### DEC-12 — Estrutura de Pacotes: Camadas Clean Architecture

**Contexto:** O [08-Fundacao-Arquitetural-Java.md](08-Fundacao-Arquitetural-Java.md) definia uma estrutura de pacotes orientada a features (`core`, `network`, `protocol`, `bot`, `pathfinding`, `inventory`, `automation`, `proxy`, `scheduler`, `persistence`, `api`), sem pacotes explícitos de camada (`domain`, `application`, `infrastructure`, `interfaces`). Isso diverge do princípio já fixado no CLAUDE.md e na Decisão Arquitetural Congelada "Clean + Hexagonal", que exige preservar a separação entre domínio, aplicação, infraestrutura e interfaces.

**Problema Atual:** Ao iniciar a Milestone 3 (primeiras entidades, Value Objects e Use Cases do domínio), não havia pacote `domain` nem `application` onde alocar esse código sem violar a estrutura já aprovada.

**Alternativas Possíveis:**
1. Manter a estrutura orientada a features do 08-Fundacao, alocando entidades em `core` e casos de uso dentro do próprio pacote `bot`.
2. Reestruturar os pacotes em camadas Clean Architecture (`domain`, `application`, `infrastructure`, `interfaces`), mantendo os nomes de features já documentados como subpacotes dentro de cada camada.

**Decisão Tomada:** Alternativa 2. A estrutura esperada passa a ser:

```
com.advancedbot
 ├── domain            # Entidades, Value Objects, regras de negócio puras (antigo "core")
 │    ├── bot
 │    ├── network
 │    ├── protocol
 │    ├── pathfinding
 │    ├── inventory
 │    └── automation
 ├── application       # Casos de uso / orquestração de regras de domínio
 │    └── usecase
 ├── infrastructure    # Persistência, proxy, scheduler, configuração, logs
 │    ├── persistence
 │    ├── proxy
 │    └── scheduler
 └── interfaces        # Controllers / API exposta para front-end e dashboard
      └── api
```

**Justificativa:** Alinha a estrutura física de pacotes ao princípio Clean + Hexagonal já congelado, sem descartar nenhum dos módulos de feature já documentados no 08-Fundacao (apenas os reorganiza como subpacotes de camada). Aprovada explicitamente pelo responsável pelo projeto para desbloquear a Milestone 3.

**Impacto na Implementação Java:** Atualiza a seção "2. Estrutura Inicial do Projeto Maven" do [08-Fundacao-Arquitetural-Java.md](08-Fundacao-Arquitetural-Java.md). Módulos ainda não criados (network, protocol, pathfinding, etc.) adotam a nova estrutura ao serem implementados em milestones futuras; nenhum código existente precisa ser movido nesta sessão.

**Data:** 2026-07-15

**Responsável:** Mateus Botega
