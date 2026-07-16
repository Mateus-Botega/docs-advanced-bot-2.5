# Loop de tick

O tick é chamado para cada cliente pela orquestração da interface. Em jogo, ele descarrega a fila, valida timeout de handshake/keep-alive, avança `PathGuide`, chama `CommandManagerNew.Tick`, atualiza física e enfileira pacote de posição/rotação. Fora do jogo, pode disparar reconexão após ~40 ticks.

## Java

Usar `ScheduledExecutorService` dedicado por shard de sessões, com período configurável e sem bloquear I/O. O loop deve apenas serializar transições da sessão; leitura/escrita Netty e tarefas longas (pathfinding/script) ficam em executores separados. Medir duração e ticks perdidos.
