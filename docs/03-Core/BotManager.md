# Gerenciador de bots

Não existe uma classe `BotManager` no legado: a coleção de sessões está dispersa entre `Main.Clients` e `Start`. O papel é: criar clientes, impedir conta duplicada, limitar sessões ativas, aplicar atraso e atualizar lista visual.

## Java

`BotSessionManager` deve ser o único dono de `ConcurrentHashMap<SessionId, BotSession>`. Expor `create`, `stop`, `find`, `list` e `broadcast`; aplicar `ConnectionPolicy` (limite, taxa, proxy) antes da criação. Eventos de sessão alimentam UI e métricas.
