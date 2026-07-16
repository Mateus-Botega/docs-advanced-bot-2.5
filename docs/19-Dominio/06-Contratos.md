# Contratos e Fronteiras (Contracts)

Para garantir a migração saudável do sistema para Java ou outras plataformas, o acoplamento deve ser substituído por contratos claros (Interfaces). Este documento formaliza quem chama quem e quais são as expectativas (Design by Contract).

## 1. Contrato: Rede -> Protocolo (`ProtocolTranslator`)

A rede não sabe nada de Minecraft. Ela recebe bytes criptografados ou não, comprime ou descomprime, e entrega pacotes crus.

- **Quem chama quem**: A thread de leitura de soquete (`NetworkStream`) notifica o `ProtocolTranslator`.
- **Pré-condições**:
  - O buffer entregue contém ao menos 1 pacote completo (varint length devidamente lido na versão 1.7+).
  - A criptografia AES e compressão Zlib já devem ter sido resolvidas.
- **Pós-condições**:
  - O `ProtocolTranslator` DEVE consumir todos os bytes do pacote.
  - Se houver erro de parsing (falha ao mapear ID), ele deve abortar silenciosamente ou logar, sem crashar a Sessão.

## 2. Contrato: Protocolo -> Modelo Local (`World` e `Session`)

O tradutor de protocolo interpreta os pacotes e muda as entidades.

- **Quem chama quem**: `ProtocolTranslator` invoca métodos nos agregados `World`, `Player` ou dispara Eventos.
- **Pré-condições**:
  - As modificações devem ocorrer de maneira isolada (thread-safe se a implementação requerer, ou preferencialmente inseridas em uma Fila de Ações do Tick para serem processadas de forma síncrona).
- **Pós-condições**:
  - O estado do Mundo agora é consistente com o servidor.
  - Eventos como `BlockChanged` SÃO emitidos caso a alteração afete terceiros.

## 3. Contrato: Tick Engine -> Agentes / IA (`Agent`)

O motor de tempo cede CPU para as automações (Pathfinding, AutoMiner).

- **Quem chama quem**: `Session` -> loop no catálogo de `Agents` ativos -> chama `Agent.onTick()`.
- **Pré-condições**:
  - A Sessão DEVE estar no estado `Playing`.
  - A fila de pacotes recebidos no último Tick já foi aplicada ao Modelo. (A IA olha para o mundo "atualizado").
- **Pós-condições**:
  - O Agente PODE enfileirar pacotes de saída (Movimento, Interação) na fila de envio da rede.
  - O Agente NÃO DEVE bloquear a thread por mais do que poucos milissegundos (sem threads infinitas no Tick).

## 4. Contrato: Agentes -> Ação no Servidor (Command Output)

Os Agentes não possuem referências diretas ao soquete TCP. Eles pedem ações de alto nível para a Sessão.

- **Quem chama quem**: `Agent` -> `Session.ActionDispatcher`.
- **Pré-condições**:
  - Um pedido como `SendBlockBreak(Vector3i)` exige que o bloco esteja adjacente (o Agente verifica física antes de pedir).
- **Pós-condições**:
  - O Dispatcher traduzirá a ação abstrata no respectivo pacote usando o `ProtocolTranslator` da versão ativa e colocará no `NetworkStream`.

## 5. Contrato: Core -> Plugins (`IPlugin`)

O bot permite código de terceiros.

- **Quem chama quem**: `PluginManager` chama os *Hook Points* (`onTick`, `onPacketOut`, `onChat`) definidos pela Interface do Plugin.
- **Pré-condições**:
  - O Plugin não tem permissão para substituir instâncias de agregados base (World, Player). Ele atua observando ou modificando filas.
- **Pós-condições**:
  - O retorno de um Hook booleano (ex: `onPacketOut` retornando `false`) PODE e DEVE cancelar o envio ou processamento do pacote.
