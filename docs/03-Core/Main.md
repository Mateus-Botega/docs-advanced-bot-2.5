# Main (interface e orquestração)

`AdvancedBot/Main.cs` é o formulário principal e proprietário da lista de clientes, painel de usuários, chat, estatísticas, proxies e ações de menu. É o coordenador de apresentação, não uma regra de domínio.

Funções observadas: abrir `Start`, selecionar cliente/todos, enviar chat/comandos, abrir viewer/editor de macro, importar/exportar listas, exibir logs e encerrar clientes. `Program.FrmMain` torna o formulário uma dependência global de rede, plugins e clientes.

## Java

Separar em `SessionController`/WebSocket ou CLI, `BotSessionService`, `ProxyService` e `TelemetryService`. A interface consulta projeções e publica comandos; não mantém nem manipula diretamente `MinecraftClient`.
