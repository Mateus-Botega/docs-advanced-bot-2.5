# Interface e visualizador

`AdvancedBot/*Form` e `Main` são WinForms: início de sessão, proxies, contas, spam, estatísticas, login, opções, edição de macro e diálogos. `Viewer/*` renderiza `World` via OpenGL/WGL: `ViewForm`, `WorldRenderer`, `ChunkRenderer`, `AsyncChunkBuilder`, `TextureManager`, `VBO`, `Tessellator`, `Font`, `Frustum` e wrappers `GL/WGL`; `Viewer.Gui/*` desenha controles próprios.

## Java

Spring Boot não substitui desktop automaticamente. Para operação remota, usar REST + WebSocket e uma SPA; para viewer 3D, tratar como aplicação separada (web WebGL ou JavaFX/LWJGL), consumindo snapshots/eventos de mundo. Nunca colocar OpenGL ou componentes visuais no serviço de sessão.
