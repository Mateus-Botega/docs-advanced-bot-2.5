# Fluxos Funcionais — AdvancedBot 2.4.5

Esta pasta documenta o **comportamento funcional** do sistema, não a estrutura de classes. Cada documento descreve um fluxo completo de ponta a ponta: por que cada etapa existe, quais invariantes devem ser preservadas, quais falhas são possíveis e o que deve ser reproduzido numa migração Java.

## Índice de fluxos

| # | Fluxo | Arquivo | Status |
|---|---|---|---|
| 01 | Login e autenticação | [01-Login.md](01-Login.md) | ✅ |
| 02 | Reconexão automática | [02-Reconexao.md](02-Reconexao.md) | ✅ |
| 03 | Spawn e join game | [03-Spawn.md](03-Spawn.md) | ✅ |
| 04 | Recebimento e envio de chat | [04-Chat.md](04-Chat.md) | ✅ |
| 05 | Movimento e física | [05-Movimento.md](05-Movimento.md) | ✅ |
| 06 | Inventário e janelas | [06-Inventario.md](06-Inventario.md) | ✅ |
| 07 | Pathfinding e execução de rota | [07-Pathfinding.md](07-Pathfinding.md) | ✅ |
| 08 | Mineração automática | [08-Mineracao.md](08-Mineracao.md) | ✅ |
| 09 | Pesca automática | [09-Pesca.md](09-Pesca.md) | ✅ |
| 10 | Combate (KillAura / Mob) | [10-Combate.md](10-Combate.md) | ✅ |
| 11 | Plugins e extensibilidade | [11-Plugins.md](11-Plugins.md) | ✅ |
| 12 | Macros e scripts | [12-Macros.md](12-Macros.md) | ✅ |
| 13 | Troca de mundo / respawn | [13-Respawn.md](13-Respawn.md) | ✅ |
| 14 | Desconexão e encerramento | [14-Desconexao.md](14-Desconexao.md) | ✅ |
| 15 | Anti-AFK e automações passivas | [15-AntiAFK.md](15-AntiAFK.md) | ✅ |

## Como usar estes documentos

Cada fluxo segue o mesmo template de 17 seções:

1. **Objetivo** — por que este fluxo existe, qual problema resolve.
2. **Evento iniciador** — o que dispara o fluxo (pacote, comando, timer, callback).
3. **Componentes envolvidos** — classes participantes com seu papel.
4. **Ordem de chamadas** — sequência exata de métodos.
5. **Estados percorridos** — máquinas de estado afetadas.
6. **Threads envolvidas** — qual thread executa cada passo.
7. **Eventos publicados** — o que este fluxo dispara para outros subsistemas.
8. **Eventos consumidos** — o que este fluxo precisa receber para funcionar.
9. **Objetos modificados** — estado que muda ao longo do fluxo.
10. **Estruturas compartilhadas** — dados que outras sessões/threads também leem.
11. **Possíveis falhas** — o que pode dar errado em cada etapa.
12. **Recuperação de erro** — como o sistema responde a falhas.
13. **Fluxograma Mermaid** — visão de alto nível.
14. **Diagrama de sequência Mermaid** — interações entre componentes.
15. **Regras de negócio** — decisões e restrições funcionais.
16. **Dependências entre módulos** — o que cada componente precisa dos outros.
17. **Impacto para migração Java** — o que deve ser preservado e o que pode ser melhorado.

## Relação com documentação de classes

Os documentos de fluxo **complementam**, não substituem, a documentação por módulos em `docs/03..17`. Use os fluxos para entender o comportamento; use os módulos para entender os contratos de cada classe.
