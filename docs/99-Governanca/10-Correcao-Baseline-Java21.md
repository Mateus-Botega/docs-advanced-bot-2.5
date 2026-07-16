# 10 - Correção de Baseline Tecnológico Java 21

## 1. Motivo da Alteração
O projeto AdvancedBot Java foi concebido para utilizar os recursos de concorrência mais modernos e performáticos do ecossistema Java. No entanto, durante as etapas iniciais de bootstrap físico do repositório, o arquivo de configuração `pom.xml` foi configurado temporariamente com a versão Java 17. 
Esta alteração corrige essa inconsistência de baseline tecnológico, alinhando a implementação física com a fundação arquitetural estabelecida que prescreve o uso do Java 21 como o alinhamento oficial da stack de desenvolvimento do projeto.

---

## 2. Histórico Java 17 → Java 21
*   **Java 17 (Estágio Temporário):** Serviu como um ponto de partida inicial durante a fase de infraestrutura de bootstrap rápido do Spring Boot. Apresentava a limitação severa de não possuir suporte estável para *Virtual Threads* (Project Loom), impossibilitando o avanço correto do design orientado a concorrência assíncrona concorrente dos bots.
*   **Java 21 LTS (Estágio Corrigido e Oficial):** Versão estável mais recente de suporte de longo prazo (LTS), trazendo em sua especificação de produção a feature de *Virtual Threads* e melhorias essenciais no *Pattern Matching* e *Records*, fundamentais para o processamento limpo e eficiente dos pacotes do protocolo do Minecraft.

---

## 3. Stack Oficial Atual
A stack tecnológica oficial e homologada para o projeto **AdvancedBot Java** compreende:
*   **Linguagem:** Java 21 LTS (Microsoft OpenJDK Build / Amazon Corretto)
*   **Framework Principal:** Spring Boot 3.2.5
*   **Build System:** Maven
*   **Livraria Ajudante:** Lombok (Aprovado DEC-08)
*   **Criptografia:** BouncyCastle (DEC-04)
*   **Banco de Dados:** PostgreSQL (Planejado para etapas futuras)
*   **Frontend:** React (Planejado para controle web/dashboard)

---

## 4. Decisão Arquitetural
A decisão foi formalizada como parte da correção de governança de baseline:
*   **Deliberação:** O **Java 21 LTS** é fixado como a versão mínima e padrão obrigatória para toda a compilação, empacotamento e execução do backend.
*   **Justificativa:** Viabilidade técnica do uso nativo de *Virtual Threads* (DEC-03) sem necessidade de flags experimentais de compilação ou JREs antigas sem suporte de produção, além de maximizar a vida útil do suporte de segurança do projeto.

---

## 5. Impactos
*   **Ambiente de Desenvolvimento:** Exigência de instalação e configuração do JDK 21 nas máquinas dos desenvolvedores e nas pipelines de CI/CD.
*   **Gerenciamento de Concorrência:** O motor do bot (Core Engine) e as lógicas de Macros (DEC-03) podem utilizar livremente o modelo assíncrono não-bloqueante escalável de threads virtuais.
*   **Isolamento do Domínio:** Nenhuma alteração foi realizada no modelo de entidades ou nas regras de negócio (Core), mantendo o foco puro na governança e infraestrutura do baseline.

---

## 6. Validação Realizada
Para atestar a consistência do setup tecnológico corrigido para Java 21 LTS, a validação foi executada sob a seguinte metodologia:

*   **Comando Executado:**
    ```bash
    mvn clean test
    ```
    *(Nota: Executado sob a JVM Microsoft OpenJDK 21.0.9 e Maven 3.9)*

*   **Resultados Obtidos:**
    *   **Compilação:** O código-fonte de bootstrap e a suite de testes foram compilados com sucesso com a flag de release 21 (`Compiling with javac [debug release 21]`).
    *   **Spring Context Loading:** O contexto inicial do Spring Boot 3.2.5 inicializou sem quaisquer falhas ou advertências estruturais (`Started AdvancedBotApplicationTests in 1.008 seconds`).
    *   **Sucesso dos Testes:** Todos os testes de verificação do contexto básico de injeção de dependências foram concluídos sem erros (`Tests run: 1, Failures: 0, Errors: 0, Skipped: 0`).
    *   **Status de Build:** `BUILD SUCCESS`.

---

## 7. Estado Final
O projeto encontra-se estabilizado e homologado com o seguinte estado final de configurações:
*   `<java.version>21</java.version>` definido em `pom.xml`.
*   Suíte de testes de inicialização em conformidade total.
*   Documentação de setup inicial (`09-Setup-Inicial-Projeto-Java.md`) e fundação arquitetural (`08-Fundacao-Arquitetural-Java.md`) atualizadas removendo qualquer premissa desatualizada baseada no Java 17.
