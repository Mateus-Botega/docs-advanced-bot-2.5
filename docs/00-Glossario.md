# Glossário

- **Cliente/Bot**: instância `MinecraftClient`; uma conta conectada a um servidor.
- **Tick**: unidade periódica de atualização. O cliente atual executa física, fila, comandos, caminho e plugins durante o tick.
- **Packet**: mensagem binária do protocolo Minecraft. `IPacket` representa pacotes de saída; handlers decodificam entrada.
- **VarInt**: inteiro de tamanho variável usado para comprimento e IDs de pacote.
- **Handshake/Login**: fases anteriores ao jogo; podem incluir autenticação de sessão, RSA e AES.
- **World/Chunk/Section**: cache local do mundo. Chunk é 16×16; section armazena blocos por faixa vertical.
- **NBT**: formato hierárquico de tags usado por itens, configuração e dados Minecraft.
- **Inventory/OpenWindow**: inventário do jogador e janela de container aberta no servidor.
- **Macro/comando**: automação local disparada por texto iniciado por `$`; comandos podem manter estado e recebem ticks.
- **Plugin**: extensão carregada de DLL que observa ciclo de vida, chat e pacotes.
- **Proxy**: transporte SOCKS/HTTP para abrir uma conexão TCP.
- **Legacy**: protocolo 1.5.2, que usa `PacketStreamLegacy`, distinto do framing moderno.
