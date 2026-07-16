# Scheduler

Não há `Scheduler` como classe no código legado. Há `Thread`, `ThreadPool`, callbacks APM (`BeginRead/BeginWrite`), `Task.Factory.StartNew`, `Task.Delay` e `Thread.Sleep`, usados sem política única.

## Java

Definir `SessionScheduler` com três grupos: event-loop de rede, executor CPU para pathfinding/decodificação pesada e scheduler para ticks/delays. Todos os trabalhos recebem `CancellationToken` equivalente (`CompletableFuture`/cancelamento) associado à sessão; desligamento deve aguardar e liberar transporte.
