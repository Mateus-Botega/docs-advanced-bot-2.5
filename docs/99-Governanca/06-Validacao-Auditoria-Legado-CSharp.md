# 06 - Validação da Auditoria do Legado C#

## 1. Objetivo da Validação

Este documento oficializa a revisão arquitetural independente da auditoria técnica do legado C# (conforme descrito no documento `05-Auditoria-do-Legado-CSharp.md`). Esta etapa valida se o inventário levantado e a estratégia de migração definida estão totalmente alinhados com a arquitetura Java planejada (Java 21, Spring Boot, modelo Single-Tenant e isolamento de processos) antes do início da fase de implementação, garantindo a ausência de riscos técnicos ocultos e o respeito à Constituição do Projeto.

## 2. Revisão da Classificação de Migração

Abaixo encontra-se a avaliação e validação da classificação de componentes propostas pela auditoria.

### 2.1. Migrado Diretamente

A estratégia de migração direta para componentes de domínio foi validada. Componentes como:
- **Pathfinding (Algoritmo A*)**: Pode ser reaproveitado, dado que lógicas matemáticas são facilmente transcritas de C# para Java mantendo a paridade funcional.
- **Entidades Base (Player, Item, Inventory)**: Podem ser migradas diretamente como Modelos de Domínio (`com.advancedbot.core.domain`), mantendo sua estrutura e invariantes de estado.
- **Estruturas Matemáticas e Constantes (Blocks.cs, Items.cs)**: Perfeitamente adaptáveis como `enums` ou classes de constantes em Java, reaproveitáveis de forma segura.

### 2.2. Reescrito

A estratégia de reescrever módulos complexos e integrados ao comportamento assíncrono ou *threads* foi validada. Devem ser reescritos:
- **Comunicação de protocolo Minecraft (Handler)**: Validado. Necessita ser reescrito focando no protocolo 1.8 (DEC-01 e DEC-07) em `com.advancedbot.core.network`, adotando padrões assíncronos modernos.
- **Sistema de Commands e Máquina de Estados**: Validado. O gerenciamento de concorrência e *race conditions* será mitigado utilizando a reescrita com Virtual Threads (DEC-03).
- **Processamento Concorrente e Macros**: Validado. A adoção de escopo isolado por Virtual Threads garantirá paralelismo sem vazamento de estado global.

### 2.3. Substituído

A substituição de lógicas proprietárias ou defasadas por soluções maduras do ecossistema Java foi validada:
- **Configurações NBT → YAML**: Validado (DEC-05). Adotar YAML facilita a manutenção externa.
- **Criptografia → BouncyCastle**: Validado (DEC-04). A troca por BouncyCastle é obrigatória para a estabilidade do AES-CFB8 no *handshake* da rede.
- **Sistema de Plugins**: Validado (DEC-06). Deve ser adotado um carregador de API de extensões isolado, rejeitando a abordagem legada que causava quebras ao núcleo.

### 2.4. Descartado

A estratégia de descarte foi confirmada, promovendo uma arquitetura limpa:
- **Viewer e Interface gráfica**: Validado (DEC-02). Toda a interface de usuário baseada em Windows Forms, WPF e renderização OpenGL será removida, focando numa API REST com *dashboard* React e operação em CLI no motor do bot.
- **Interpretador de Script proprietário**: Validado. Descarte justificado por complexidade e acoplamento desnecessários.

## 3. Análise de Dependências

A auditoria revelou problemas cruciais no legado que foram validados em relação à nova arquitetura:

- **Acoplamentos Encontrados**: O forte acoplamento entre lógica de rede e interface (Viewer/Forms) foi adequadamente isolado pela decisão de descartar o visual do bot.
- **Estados Globais Existentes**: O uso severo de estados globais que causava concorrência indesejada foi validado. O modelo **Single-Tenant (DEC-09)**, garantindo uma JVM por bot (`Java Core` isolado), eliminará o compartilhamento perigoso de memória.
- **Dependências Circulares e Herdadas**: Serão mitigadas através da inversão de controle gerenciada pelo Spring Boot.
- **Compatibilidade Arquitetural**: A estratégia valida totalmente a compatibilidade com a Arquitetura Técnica da Plataforma, adotando o isolamento do "Motor do Bot" (Java puro/Virtual Threads) em processos independentes e orquestrados pelo backend.

## 4. Avaliação de Riscos da Migração

Abaixo os riscos técnicos analisados com suas respectivas estratégias de mitigação:

| Risco | Impacto | Probabilidade | Mitigação |
|---|---|---|---|
| Diferenças de Tipagem e Sintaxe C# → Java | Baixo | Alta | Uso intensivo de testes e mapeamento direto dos domínios. Cuidado especial com *unsigned bytes* nas lógicas de pacotes. |
| Alteração do modelo de *threads* (Virtual Threads) | Alto | Média | Desenvolver e testar o núcleo de `Commands` isoladamente, validando paridade de comportamento com as macros C#. |
| Mudança de Persistência (NBT para YAML) | Médio | Alta | Perda de compatibilidade antiga. Será mitigado pela criação de configurações unificadas em YAML, priorizando robustez da nova plataforma. |
| Implementação de Criptografia falhar no *Handshake* | Alto | Alta | Integrar BouncyCastle no início do desenvolvimento e validar com testes de conexão usando proxy local de Minecraft. |
| Comportamento oculto (Anti-Cheat/Pathfinding) não portado | Alto | Média | Inspeção cirúrgica das classes de bypass e de movimentação legadas durante a transcrição do domínio. |
| Compatibilidade de versões Minecraft (1.5.2 e 1.8) | Médio | Alta | Abstração de pacotes (*Protocol Layer*). Foco inicial restrito à versão 1.8 para validação plena da arquitetura (DEC-01 e DEC-07). |

## 5. Pontos Bloqueadores Antes do Desenvolvimento

Após a validação da auditoria, verificou-se o status dos bloqueadores para o início do desenvolvimento:

- **Modelagem dos módulos Java**: **Desbloqueado**. A matriz e o inventário do legado fornecem a estrutura clara de diretórios (`com.advancedbot.core.domain`, `com.advancedbot.core.network`, etc.).
- **Criação dos pacotes e Definição das entidades**: **Desbloqueado**. Os modelos já foram identificados como passíveis de migração direta.
- **Implementação do Core**: **Desbloqueado**. Com as decisões DEC-01 a DEC-10 formalmente registradas, a arquitetura da plataforma definida e a auditoria validada, não existem mais bloqueios arquiteturais.

## 6. Decisões confirmadas após auditoria

As seguintes decisões arquiteturais (DEC) permanecem válidas e inalteradas após a validação da auditoria do legado:

- **DEC-01**: Foco em protocolo único inicial (versão 1.8).
- **DEC-02**: Descarte completo da interface gráfica e adoção de CLI/API.
- **DEC-03**: Adoção de Virtual Threads (Java 21) para Macros e Scripts.
- **DEC-04**: Utilização do BouncyCastle como provider criptográfico AES-CFB8.
- **DEC-05**: Substituição de persistência NBT para configuração YAML.
- **DEC-06**: Implementação de um sistema simples e isolado de Plugins.
- **DEC-07**: Servidor alvo inicial sendo protocolo Minecraft 1.8.
- **DEC-08**: Adoção de Spring Boot como Orquestrador backend.
- **DEC-09**: Execução do motor em modelo Operacional Single-Tenant (Uma JVM por Bot).
- **DEC-10**: Injeção de configurações de Proxy diretamente na instância do *socket* TCP.

## 7. Aprovação Arquitetural

Status:

[x] Auditoria aprovada para implementação
