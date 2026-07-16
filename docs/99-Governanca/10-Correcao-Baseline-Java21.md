# Correção de Baseline Tecnológico Java 21

## 1. Contexto

O projeto AdvancedBot Java foi inicialmente configurado com o Java 17 durante a etapa de bootstrap e inicialização física da plataforma. Essa configuração representou apenas um ponto de partida inicial temporário e não a decisão final.

A arquitetura oficial previamente estabelecida e definida para a nova plataforma estipula o uso do Java 21. Com isso, corrigimos essa inconsistência temporária para que a fundação do projeto reflita de maneira exata as diretrizes arquiteturais acordadas, garantindo que o Java 21 seja a versão oficial e padrão do backend a partir deste momento.

---

## 2. Stack Oficial

A pilha de tecnologias oficial definida para o projeto AdvancedBot é composta por:

*   **Backend:** Spring Boot + Java 21
*   **Frontend:** React
*   **Database:** PostgreSQL
*   **Build:** Maven
*   **Architecture:** Clean Architecture + Hexagonal Architecture

---

## 3. Decisão Arquitetural

### Decisão
O **Java 21** é a versão oficial da plataforma AdvancedBot Java.

### Motivações
*   **Versão LTS (Long-Term Support):** O Java 21 é a versão de suporte de longo prazo mais recente e estável, o que garante correções de segurança e suporte da comunidade por muitos anos.
*   **Maior Longevidade:** Evita a necessidade de migrações precoces de versão da linguagem em um futuro próximo.
*   **Melhor Suporte Futuro e Recursos:** Acesso nativo a novos recursos estáveis da JVM, incluindo aprimoramentos significativos em Virtual Threads (Project Loom) e Pattern Matching, fundamentais para a performance concorrente do motor do bot.
*   **Alinhamento Arquitetural:** Conformidade exata com as especificações gerais de design definidas para a migração do AdvancedBot C# para Java.

---

## 4. Impactos

*   **Compatibilidade Garantida:** A compatibilidade com o Spring Boot 3.2.5 foi testada e validada com sucesso sob a JVM do Java 21.
*   **Sem Alteração de Domínio:** O Core da aplicação continua isolado de dependências e livre de frameworks externos, sem qualquer alteração lógica.
*   **Nenhuma Regra de Negócio Legada Implementada:** A correção é restrita apenas ao baseline tecnológico do projeto de fundação, mantendo o escopo estritamente focado no alinhamento da governança.
*   **Preparação para Próxima Fase:** O projeto agora está tecnicamente pronto para o início do desenvolvimento da Fase 2 (Core Engine Domain e State Machine).

---

## 5. Estado final

O estado final das configurações tecnológicas após esta correção é:

*   **Java:** 21
*   **Framework:** Spring Boot
*   **Build:** Maven
*   **Frontend:** React
*   **Database:** PostgreSQL
*   **Architecture:** Clean + Hexagonal
