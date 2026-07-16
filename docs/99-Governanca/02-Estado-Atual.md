# 02 - Estado Atual do Sistema

## 1. Objetivo deste Documento

Descrever detalhadamente o estado atual do sistema antes da reescrita.

Este documento representa uma engenharia reversa da aplicação existente, contendo:

- Arquitetura atual.
- Tecnologias utilizadas.
- Estrutura de projetos.
- Componentes existentes.
- Fluxos de execução.
- Regras de negócio implementadas.
- Integrações existentes.
- Dependências externas.
- Pontos frágeis conhecidos.
- Débitos técnicos.
- Comportamentos que devem ser preservados na nova implementação.

Este documento NÃO descreve a solução futura.

A finalidade é responder:

> "Como o sistema funciona atualmente?"

---

# 2. Identificação do Sistema

## 2.1 Informações Gerais

| Campo | Valor |
|---|---|
| Nome do Sistema | |
| Versão Atual | |
| Linguagem Atual | |
| Framework Principal | |
| Banco de Dados | |
| Ambiente de Execução | |
| Data da Análise | |
| Responsável pela Análise | |

---

# 3. Visão Geral da Aplicação

## 3.1 Descrição Geral

Descrever:

- Qual problema o sistema resolve.
- Qual o objetivo principal.
- Quem utiliza.
- Como é utilizado atualmente.
- Qual o fluxo geral da aplicação.

Exemplo:
O sistema é responsável por automatizar processos X,
realizando integração entre Y e Z.

---

## 3.2 Características Principais

Listar funcionalidades existentes:

- [ ] Funcionalidade 1
- [ ] Funcionalidade 2
- [ ] Funcionalidade 3

---

# 4. Arquitetura Atual

## 4.1 Visão Arquitetural

Descrever o modelo atual:

Exemplo:
Aplicação Desktop
|
|
Camada de Serviços
|
|
Banco de Dados


---

## 4.2 Componentes Principais

| Componente | Responsabilidade | Tecnologia |
|---|---|---|
| | | |

---

## 4.3 Fluxo de Inicialização

Descrever passo a passo:

1. Aplicação inicia.
2. Configura dependências.
3. Carrega configurações.
4. Inicializa serviços.
5. Executa processos automáticos.

---

# 5. Estrutura Atual do Projeto

## 5.1 Estrutura de Diretórios

Representar a estrutura atual:
Projeto
├── Pasta
│ ├── Arquivo
│ └── Arquivo
└── Pasta

---

## 5.2 Organização dos Pacotes

Para cada pacote:

## Nome do Pacote

Responsabilidade:

Classes principais:

- Classe A
- Classe B

Dependências:

- Dependência X

---

# 6. Tecnologias Utilizadas

## 6.1 Linguagens

| Linguagem | Versão | Uso |
|-|-|-|
| | | |

---

## 6.2 Frameworks e Bibliotecas

| Biblioteca | Versão | Finalidade |
|-|-|-|
| | | |

---

## 6.3 Ferramentas Externas

| Ferramenta | Finalidade |
|-|-|
| | |

---

# 7. Banco de Dados Atual

## 7.1 Tecnologia

Informações:

- Banco utilizado.
- Versão.
- Tipo de conexão.
- Driver utilizado.

---

## 7.2 Estrutura de Dados

Listar tabelas:

| Tabela | Objetivo |
|-|-|
| | |

---

## 7.3 Relacionamentos

Descrever:
Tabela A
|
|
Tabela B


---

## 7.4 Regras Persistidas

Documentar:

- Triggers.
- Procedures.
- Views.
- Índices.
- Constraints.

---

# 8. Módulos Existentes

Cada módulo deve possuir sua própria documentação.

---

# 8.X Nome do Módulo

## Objetivo

Descrever responsabilidade.

---

## Responsabilidades

Lista:

- Responsabilidade 1
- Responsabilidade 2

---

## Classes Principais

| Classe | Responsabilidade |
|-|-|
| | |

---

## Fluxo de Execução

Descrever:
Entrada
|
Processamento
|
Saída


---

## Dependências

Internas:

- Classe A
- Serviço B


Externas:

- API
- Banco
- Biblioteca

---

## Regras de Negócio

Documentar:

### Regra 1

Descrição:

Entrada:

Processamento:

Resultado esperado:

---

## Pontos de Atenção

Registrar:

- Bugs conhecidos.
- Código legado.
- Comportamentos inesperados.

---

# 9. Serviços e Processamentos Automáticos

## 9.1 Jobs / Schedulers

| Nome | Frequência | Responsabilidade |
|-|-|-|
| | | |

---

## 9.2 Threads / Processamentos Paralelos

Documentar:

- Criação.
- Controle.
- Sincronização.
- Locks.
- Concorrência.

---

# 10. Integrações Externas

## 10.1 Sistema Integrado

Nome:

Objetivo:

---

## Comunicação

Tipo:

- REST
- SOAP
- Socket
- Arquivo
- Banco

---

## Fluxo
Sistema A
|
|
Sistema Atual
|
|
Sistema B


---

# 11. Configurações Atuais

## Arquivos de Configuração

| Arquivo | Finalidade |
|-|-|
| | |

---

## Variáveis Importantes

| Nome | Descrição |
|-|-|
| | |

---

# 12. Segurança Atual

Documentar:

- Autenticação.
- Autorização.
- Criptografia.
- Controle de acesso.
- Dados sensíveis.

---

# 13. Logs e Monitoramento

## Sistema de Logs

Informar:

- Biblioteca utilizada.
- Local dos arquivos.
- Formato.

---

## Eventos Importantes

| Evento | Descrição |
|-|-|
| | |

---

# 14. Testes Existentes

## Testes Automatizados

Informar:

- Framework.
- Cobertura.
- Localização.

---

## Testes Manuais

Descrever:

- Cenários existentes.
- Passos executados.
- Resultado esperado.

---

# 15. Problemas Conhecidos

## Problema 1

Descrição:

Impacto:

Local:

Possível causa:

Solução atual:

---

# 16. Débitos Técnicos

Registrar:

| Item | Impacto | Prioridade |
|-|-|-|
| | | |

---

# 17. Limitações Atuais

Documentar:

- Limitações arquiteturais.
- Limitações tecnológicas.
- Limitações de performance.
- Limitações de manutenção.

---

# 18. Dependências Críticas

Listar componentes que não podem ser removidos sem impacto:

| Dependência | Motivo |
|-|-|
| | |

---

# 19. Regras de Negócio Descobertas

Centralizar todas as regras importantes encontradas.

## Regra

Descrição:

Origem:

Classe responsável:

Impacto:

---

# 20. Pontos que Devem ser Mantidos na Reescrita

Lista:

- Funcionalidade X.
- Comportamento Y.
- Regra Z.

---

# 21. Pontos que Devem ser Melhorados

Lista:

- Arquitetura.
- Performance.
- Segurança.
- Testabilidade.
- Manutenção.

---

# 22. Conclusão da Análise Atual

Resumo:

- Estado atual encontrado.
- Principais componentes.
- Principais riscos.
- Próximas etapas da reescrita.