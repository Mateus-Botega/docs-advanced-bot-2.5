# 09 - Setup Inicial do Projeto Java

## 1. Visão Geral

Este documento formaliza a conclusão da **Fase 1 (Inicialização Física da Plataforma Java - Milestone 1)**, detalhada no documento `07-Plano-de-Migracao-e-Estrategia-de-Implementacao.md`.

O objetivo desta fase foi estabelecer a fundação inicial, em branco (greenfield), para a reconstrução do sistema AdvancedBot, cumprindo estritamente as diretrizes de governança arquitetural sem incluir nenhuma migração de regras de negócio nesta etapa.

**Data de Inicialização:** 14 de Julho de 2026 (Data do ambiente)

---

## 2. Estrutura Criada

Foi inicializado um novo projeto Maven no diretório `C:\Users\Administrator\Desktop\ADVs\ProjetosBot\advancedbot-java`, vizinho ao projeto C# legado. A arquitetura de pacotes foi definida exatamente como estipulado no documento `08-Fundacao-Arquitetural-Java.md`.

A hierarquia final inicial é:

```text
advancedbot-java
├── pom.xml
├── src/main/java/com/advancedbot
│   ├── application/bootstrap
│   │   └── AdvancedBotApplication.java
│   ├── bot
│   │   ├── execution/
│   │   ├── lifecycle/
│   │   └── manager/
│   ├── core
│   │   ├── domain/
│   │   ├── entity/
│   │   ├── event/
│   │   └── state/
│   ├── infrastructure
│   │   ├── config/
│   │   ├── logging/
│   │   └── persistence/
│   ├── network
│   │   ├── connection/
│   │   ├── proxy/
│   │   ├── session/
│   │   └── socket/
│   ├── pathfinding/
│   └── protocol
│       ├── adapter/
│       ├── v1_5_2/
│       └── v1_8/
├── src/main/resources
│   └── application.yml
└── src/test/java/com/advancedbot
    └── application/bootstrap
        └── AdvancedBotApplicationTests.java
```

---

## 3. Dependências Adicionadas

As dependências foram restritas ao mínimo necessário para a fundação e configuradas no `pom.xml`:

1.  **Spring Boot Starter (`3.2.x`)**: Para injeção de dependências, configuração centralizada em propriedades YAML e gerenciamento do ciclo de vida da aplicação.
2.  **Spring Boot Starter Test**: Para suporte integrado a testes automatizados com JUnit 5.
3.  **Lombok**: Adicionado (como `optional`) para minimizar *boilerplate* de classes puras (`@Getter`, `@Setter`, etc.), conforme aprovação prévia.
4.  **SLF4J + Logback**: Embutidos no Spring Boot Starter, utilizados para o sistema de *logging* inicial.

---

## 4. Configurações e Preparativos

*   **Ponto de Entrada (Bootstrap):** Criada a classe `AdvancedBotApplication` com anotação `@SpringBootApplication(scanBasePackages = "com.advancedbot")`, garantindo rastreamento completo de beans e incluindo o gancho de `@PreDestroy` para futuros desligamentos graciosos (*Graceful Shutdown*).
*   **Application YAML:** Criado arquivo `application.yml` configurando os perfis da aplicação (`dev`), habilitando desligamento em fases para os *Virtual Threads* e estabelecendo os padrões visuais e níveis de log iniciais (Root `INFO`, AdvancedBot `DEBUG`).
*   **Testes Base:** Criada a estrutura de testes refletindo o main, com uma classe inicial de *context loading* que assegura que o projeto consegue inicializar sua injeção de dependências sem falhas, validando as fundações.

---

## 5. Validação com a Constituição e Decisões (ADRs)

A execução deste setup passa no crivo da governança estabelecida:

| Diretriz / Decisão | Validação |
| :--- | :--- |
| **Nenhuma regra legada foi portada (ADR-001)** | ✅ Cumprido. Nenhuma classe do legado C# foi tocada ou portada. |
| **Isolamento de Domínio (ADR-002)** | ✅ Cumprido. As hierarquias separando a `infrastructure` do `core` estão prontas. |
| **Nova infraestrutura base (DEC-05, DEC-08)** | ✅ Cumprido. Spring Boot estabelecido no projeto e YAML configurado. |

### Itens Bloqueados / Limitações

*   **Virtual Threads (DEC-03 / Java 21):** O uso de Virtual Threads exige Java 21 LTS (Project Loom), sendo incompatível com a especificação temporária anterior baseada no Java 17 (versão na qual Virtual Threads ainda não estavam disponíveis de forma estável). A correção do baseline de JDK 17 para JDK 21 remove essa restrição histórica e viabiliza a execução assíncrona concorrente dos bots.
*   **Ambiente do usuário:** Se o Maven não estiver disponível no PATH local, recomenda-se que seja instalado (ou que se adicione o Maven Wrapper nas próximas etapas) para a execução fluida dos testes de contexto e rotinas de build.

---

## 6. Próximos Passos Recomendados

Com a fundação inicial física aprovada, recomenda-se avançar para a **Fase 2 (Motor do Bot isolado)**, onde as seguintes atividades devem ser focadas:

1.  Definição e criação das interfaces e enums do domínio no pacote `core/domain` (Ex: `Location`, `Block`).
2.  Construção das Máquinas de Estado primárias no pacote `core/state`.
3.  Criação da estrutura de *EventBus* no pacote `core/event`.
